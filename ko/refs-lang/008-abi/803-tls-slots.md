# 8.3. mutator TLS 슬롯 레이아웃

각 OS 스레드(= 스케줄러 워커 = mutator)는 스레드 로컬 저장소(TLS)의 정해진 자리에 런타임 정보를 둡니다. 컴파일러가 내보내는 코드와 런타임이 같은 TLS 슬롯을 읽고 씁니다. 그래서 이 문서가 각 슬롯의 오프셋과 타입을 정의합니다.

## 8.3.1. TLS 슬롯 구조

TLS 슬롯은 `thread_local` 블록 `rok_tls` 하나에 모여 있습니다. 런타임이 이 블록을 정의하고 내보냅니다. 각 슬롯은 블록 안의 고정 오프셋에 있습니다. 컴파일된 코드는 블록 주소에 슬롯의 오프셋을 더해 접근합니다. Rok 함수는 블록 주소를 호출자에게서 받습니다(8.3.8).

`p`는 바이트 단위의 포인터 폭입니다.

| Offset   | 슬롯                        | 타입 | 의미                                    |
|----------|-----------------------------|------|-----------------------------------------|
| 0        | `rok_tls_shadow_stack_top`  | ptr  | shadow stack 체인의 머리                |
| p        | `rok_tls_tlab_cursor`       | ptr  | TLAB의 다음 할당 위치                   |
| 2p       | `rok_tls_tlab_limit`        | ptr  | TLAB의 끝                               |
| 3p       | `rok_tls_wb_log_cursor`     | ptr  | write barrier log의 위치                |
| 4p       | `rok_tls_wb_log_limit`      | ptr  | write barrier log의 끝                  |
| 5p       | `rok_tls_satb_cursor`       | ptr  | SATB 버퍼의 위치 (동시 mark 중)         |
| 6p       | `rok_tls_satb_limit`        | ptr  | SATB 버퍼의 끝                          |
| 7p       | `rok_tls_task_worker`       | ptr  | 이 스레드의 executor worker (GC 객체, 없으면 0) |
| 8p       | `rok_tls_stack_guard`       | [usize; 2] | `[lo, hi)` 범위의 fault는 이 스레드의 스택 오버플로 (`[0, 0)`이면 모름, [8.5.7](805-extern-symbols.md#스택-오버플로))   |
| 10p      | `rok_tls_worker_id`         | u64  | worker id                               |
| 10p+8    | `rok_tls_yield_flag`        | u8   | safepoint에서 `yield`해야 하는지        |
| 10p+9    | `rok_tls_mutator_state`     | u8   | mutator 상태                            |

**ABI 계약**은 블록 심볼 `rok_tls`와 각 슬롯의 오프셋, 타입, 의미입니다. 슬롯 이름은 가리키려고 붙인 이름일 뿐 심볼이 아닙니다. 블록은 8바이트로 정렬되고 크기는 최소 10p+10바이트입니다. 마지막 ABI 슬롯 뒤에 구현이 정하는 슬롯이 와도 됩니다.

**구현 정의 슬롯**. 런타임은 스케줄링, 통계, 프로파일링, 디버깅 등에 쓰는 스레드 로컬 상태를 ABI 슬롯 뒤나 다른 곳에 더 둘 수 있습니다. 컴파일러와 외부 패키지는 이 상태를 참조하면 안 됩니다.

> [!NOTE]
> Rok의 런타임은 다음 슬롯을 더 둡니다.
>
> - 10p+16: `task_lane`
> - 11p+16: `static_inits`
> - 12p+16: `held_locks`. 이 스레드가 쥔 lock입니다. 세션 호스트에서는 스레드가 사라지면 그 스레드가 block 중에 쥐고 있던 lock을 해제합니다.

## 8.3.2. 접근 메커니즘

OS와 아키텍처마다 TLS 접근 메커니즘이 다릅니다.

| 플랫폼         | 메커니즘                              |
|----------------|---------------------------------------|
| Linux x86_64   | `%fs:offset` 또는 `%gs:offset`        |
| Linux aarch64  | `tpidr_el0`                           |
| macOS x86_64   | `%gs:offset`                          |
| Windows x86_64 | TEB (Thread Environment Block) offset |
| `wasm32-wasi-threads` | `__tls_base` global + offset   |
| 그 밖의 WASM (스레드 없음) | 블록은 일반 global         |

컴파일러는 타깃에 맞는 명령어를 내보냅니다. 세부 처리는 LLVM의 `__thread`와 TLS 모델이 알아서 합니다.

## 8.3.3. 슬롯별 의미

### `rok_tls_yield_flag`(u8)

[GC](../000-meta/007-glossary.md#gc-garbage-collector)가 safepoint를 요청할 때 설정하는 플래그입니다. mutator는 폴링할 때 이 플래그를 검사합니다. bit 0은 런타임의 정지 요청이며 런타임이 올리고 내립니다. 나머지 bit는 임베딩 호스트(예: REPL의 인터럽트)가 쓰며 런타임은 건드리지 않습니다. 폴링 코드는 플래그가 0이 아니면 언제나 `rok_runtime_yield`를 호출합니다.

자세한 메커니즘은 [배리어와 루트](../007-runtime/704-barriers-and-roots.md)의 Safepoint Polling 항목에서 정의합니다.

### `rok_tls_tlab_cursor`(ptr)

[할당](../007-runtime/702-allocation.md)에서 다루는 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer)의 다음 할당 위치를 가리킵니다.

```
allocate(size):
    tmp = tlab_cursor
    new_cursor = tmp + size
    if new_cursor > tlab_limit:
        slow path
    tlab_cursor = new_cursor
    return tmp
```

이 네 명령어가 TLAB fast path입니다. 컴파일러는 이 명령어를 인라인으로 내보냅니다.

### `rok_tls_tlab_limit`(ptr)

TLAB의 끝을 가리킵니다. 커서가 이 값을 넘으면 slow path로 들어갑니다.

### `rok_tls_shadow_stack_top`(ptr)

shadow stack 체인의 머리입니다. 자세한 내용은 8.3.8에서 정의합니다.

### `rok_tls_satb_cursor` / `rok_tls_satb_limit`(ptr)

동시 mark([7.5](../007-runtime/705-concurrent-collection.md)) 중 write barrier slow path가 기록하는 SATB 버퍼입니다. 가득 차면 수집기에 넘깁니다. mark 밖에서는 쓰지 않습니다.

### `rok_tls_wb_log_cursor` / `rok_tls_wb_log_limit`(ptr)

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md)에서 정한 write barrier log 버퍼의 커서와 끝입니다.

```
write_barrier(p, q):
    if !needs_log: return
    if wb_log_cursor >= wb_log_limit: flush
    *wb_log_cursor++ = p
```

대부분의 동작은 atomic이 아니고 스레드 안에서만 일어납니다.

두 슬롯은 스레드마다 있는 4 KiB log chunk를 가리킵니다. 가득 찬 chunk는 lock-free 스택에 올리고, 수집기가 이 스택을 비웁니다([7.4.2](../007-runtime/704-barriers-and-roots.md#742-write-barrier-incremental-update)).

### `rok_tls_mutator_state`(u8)

mutator의 현재 상태입니다.

| 값    | 의미                          |
|-------|-------------------------------|
| `0`   | RUNNING. 일반 실행            |
| `1`   | GC_COOPERATING. GC와 협력 중  |
| `2`   | PARKED. sleep / wait 중       |
| `3` ~ | 예약                          |

이 값은 GC 사이클 동기화와 safepoint 협력에 씁니다.

### `rok_tls_worker_id`(u64)

이 스레드의 워커 id입니다. 스케줄러가 내부에서 [work-stealing](../000-meta/007-glossary.md#work-stealing)이나 통계 등에 씁니다.

## 8.3.4. 슬롯 폭과 정렬

각 슬롯의 폭은 자기 타입의 폭입니다. 슬롯은 그 폭에 맞게 정렬된 오프셋에 놓입니다.

## 8.3.5. 32-bit와 64-bit의 차이

포인터 슬롯은 p바이트(64비트에서 8, 32비트에서 4)입니다. 그래서 오프셋은 표에 정한 대로 32비트와 64비트에서 다릅니다.

## 8.3.6. TLS 슬롯의 초기화

워커가 시작될 때 슬롯을 다음과 같이 초기화합니다.

```
rok_thread_init():
  rok_tls_yield_flag = 0
  rok_tls_tlab_cursor = NULL          // slow path 가 첫 할당 시 채움
  rok_tls_tlab_limit = NULL
  rok_tls_shadow_stack_top = NULL     // 빈 shadow stack 체인 (8.3.8)
  rok_tls_wb_log_cursor = wb_log_base
  rok_tls_wb_log_limit = wb_log_base + LOG_SIZE
  rok_tls_mutator_state = RUNNING
  rok_tls_worker_id = (allocated)
  rok_tls_task_worker = 0
  rok_tls_stack_guard = [guard_lo, guard_hi)   // 스택 아래의 guard 범위
```

스케줄러는 워커를 시작할 때 `rok_thread_init`을 호출합니다. 이 함수는 [extern 심볼](805-extern-symbols.md)에 속합니다.

## 8.3.7. TLS 슬롯의 종료

워커가 끝날 때는 다음 단계를 거칩니다.

```
rok_thread_shutdown():
  // log flush
  // shadow stack walk (마지막 mark 도 가능하면)
  // worker_id 반환
```

## 8.3.8. 컴파일러 출력 형식

블록은 런타임이 정의하고 내보내는 `thread_local` 전역입니다. 컴파일러는 블록을 외부 선언으로 참조하고, 블록 주소에 슬롯의 오프셋을 더한 곳에서 값을 읽습니다. Rok 함수는 이 주소를 호출자에게서 숨은 마지막 인자로 받습니다(8.6.6). 그래서 Rok 함수끼리 호출할 때는 TLS를 읽지 않습니다. Rok 바깥에서 들어오는 코드는 주소를 한 번 직접 얻어(`llvm.threadlocal.address`) 넘겨 줍니다. C `main`, `extern "C"` 정의의 C 진입점, `#[export_name]` Rok 함수, `any` dispatcher, lazy `static`의 초기화 함수, 그리고 함수 값이 거치는 thunk(8.6.11)가 여기에 해당합니다.

```mlir
; LLVM IR - 정의는 런타임, 컴파일러 산출물은 external 선언
@rok_tls = external thread_local global [N x i8]

%t = call ptr @llvm.threadlocal.address(ptr @rok_tls)   ; rok 로 들어오는 곳에서 한 번
                                                         ; (rok 함수: 마지막 인자)

; safepoint poll (yield_flag 는 10p+8)
%flag_p = getelementptr i8, ptr %t, i64 88               ; p = 8 일 때 10p+8
%flag = load i8, ptr %flag_p
%check = icmp ne i8 %flag, 0
br i1 %check, label %slow, label %cont
```

그다음 LLVM 백엔드가 이 코드를 타깃별 TLS 명령어로 낮춥니다.

ELF에서 `rok_tls`는 initial-exec TLS 모델을 씁니다. 컴파일러가 접근 코드를 처음부터 이 모델로 내보내므로 링커의 relaxation에 기대지 않습니다. 런타임은 항상 실행 파일에 정적으로 링크되므로 이 모델로 충분합니다. Mach-O는 플랫폼의 TLV 디스크립터 방식을 씁니다. toolchain이 TLS 모델을 고를 수 없는 구현은 대신 general-dynamic을 써도 됩니다.

### shadow stack 머리

`rok_tls`의 offset 0 슬롯(`rok_tls_shadow_stack_top`)은 [배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-내보내는-shadow-stack)에서 정의한 [shadow stack](../000-meta/007-glossary.md#shadow-stack) 체인의 머리입니다. 곧 가장 안쪽 `ShadowFrame`의 주소이며, 없으면 null입니다. 체인 머리는 이 슬롯 하나뿐입니다. 컴파일러가 내보내는 코드는 함수를 지나는 각 경로의 첫 safepoint 앞에서 프레임을 체인에 잇습니다. 그리고 프레임을 이은 경로의 모든 `return` 직전에 프레임을 체인에서 뗍니다(8.4.7).

LLVM의 `llvm.gcroot`와 `llvm_gc_root_chain`은 쓰지 않습니다. `llvm_gc_root_chain`은 프로세스 전역 하나뿐이라 스레드별 체인을 표현할 수 없습니다.

## 8.3.9. 안정성

8.3.1 표의 오프셋, 타입, 의미는 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface)의 일부입니다. 바꿀 때는 [개요](README.md)의 안정성 정책을 따릅니다. 오프셋을 바꾸면 major `break`입니다. 새 슬롯은 기존 슬롯 뒤의 오프셋을 받으므로 minor 호환 변경입니다. 기존 슬롯의 타입이나 의미를 바꾸는 것도 major `break`입니다.

구현 정의 슬롯은 ABI에 속하지 않습니다. 그래서 같은 major 버전 안에서도 자유롭게 바뀔 수 있습니다.
