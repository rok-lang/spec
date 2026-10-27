# 8.3. Mutator TLS 슬롯 레이아웃

각 OS thread (= scheduler worker = mutator) 는 thread-local storage (TLS) 의 정해진 자리에 런타임 정보를 가지게 됩니다. 컴파일러가 emit 하는 코드와 런타임이 동일한 TLS 슬롯을 읽고 쓰기 때문에, 본 문서에서 각 슬롯의 offset 과 타입을 정의합니다.

## 8.3.1. TLS 슬롯 구조

TLS 슬롯은 런타임이 정의하고 export 하는 하나의 `thread_local` 블록 `rok_tls` 이며, 각 슬롯은 그 안의 고정 offset 에 있습니다. 컴파일된 코드는 블록 주소에서 각 슬롯의 offset 으로 접근하며, Rok 함수는 그 주소를 호출자에게서 받습니다 (8.3.8).

`p` 는 바이트 단위의 포인터 폭입니다.

| Offset   | 슬롯                        | 타입 | 의미                                    |
|----------|-----------------------------|------|-----------------------------------------|
| 0        | `rok_tls_shadow_stack_top`  | ptr  | shadow stack 체인 머리                  |
| p        | `rok_tls_tlab_cursor`       | ptr  | TLAB 의 다음 할당 위치                  |
| 2p       | `rok_tls_tlab_limit`        | ptr  | TLAB 의 끝                              |
| 3p       | `rok_tls_wb_log_cursor`     | ptr  | write barrier log 의 위치               |
| 4p       | `rok_tls_wb_log_limit`      | ptr  | write barrier log 의 끝                 |
| 5p       | `rok_tls_satb_cursor`       | ptr  | SATB buffer 의 위치 (동시 mark 중)      |
| 6p       | `rok_tls_satb_limit`        | ptr  | SATB buffer 의 끝                       |
| 7p       | `rok_tls_task_worker`       | ptr  | 이 스레드의 executor worker (GC 객체, 없으면 0) |
| 8p       | `rok_tls_stack_guard`       | [usize; 2] | `[lo, hi)` - 이 범위의 fault 는 이 스레드의 스택 오버플로 (`[0, 0)` 이면 모름, [8.5.7](805-extern-symbols.md#스택-오버플로)) |
| 10p      | `rok_tls_worker_id`         | u64  | worker id                               |
| 10p+8    | `rok_tls_yield_flag`        | u8   | safepoint 에서 `yield` 해야 하는지      |
| 10p+9    | `rok_tls_mutator_state`     | u8   | mutator 상태                            |

**ABI 계약** 은 블록 심볼 `rok_tls` 와 각 슬롯의 offset, 타입, 의미입니다. 슬롯 이름은 가리키기 위한 것일 뿐 심볼이 아닙니다. 블록은 8 정렬이며 적어도 10p+10 바이트입니다. 마지막 ABI 슬롯 뒤에 구현이 정하는 슬롯이 올 수 있습니다.

**impl-defined 슬롯**: 런타임은 스케줄링, 통계, 프로파일링, 디버깅 등을 위한 thread-local 상태를 ABI 슬롯 뒤나 다른 곳에 더 둘 수 있습니다. 컴파일러와 외부 패키지는 이것을 참조해서는 안 됩니다.

> [!NOTE]
> Rok 의 런타임은 `no_safepoint` 를 10p+10 (`u8`, debug rokc 가 `rok_nosp_enter` / `rok_nosp_leave` 로 감싸는 `#[no_safepoint]` 호출의 깊이, [8.4.10](804-safepoint.md#8410-native-호출)), `task_lane` 을 10p+16, `static_inits` 를 11p+16, `held_locks` (이 스레드가 가진 lock. 세션 host 에서 사라진 스레드가 block 중에 가진 lock 은 해제됨) 를 12p+16, `thread_name` 을 13p+16 (`{ptr, len}`, 보고에 쓰는 스레드 이름. 이름이 없으면 null) 에 둡니다. 블록은 15p+16 바이트입니다.

## 8.3.2. 접근 메커니즘

각 OS 와 아키텍처는 TLS 접근에 대한 자체 메커니즘을 갖추고 있습니다.

| 플랫폼         | 메커니즘                              |
|----------------|---------------------------------------|
| Linux x86_64   | `%fs:offset` 또는 `%gs:offset`        |
| Linux aarch64  | `tpidr_el0`                           |
| macOS x86_64   | `%gs:offset`                          |
| Windows x86_64 | TEB (Thread Environment Block) offset |
| `wasm32-wasi-threads` | `__tls_base` global + offset   |
| 그 밖의 WASM (스레드 없음) | 블록은 일반 global         |

컴파일러는 타깃별로 적절한 명령어를 emit 하게 되며, LLVM 의 `__thread` 와 TLS 모델이 이를 자동으로 처리합니다.

## 8.3.3. 슬롯별 의미

### `rok_tls_yield_flag` (u8)

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 가 safepoint 시점에 set 하는 플래그이며, mutator 는 poll 단계에서 이를 검사합니다. bit 0 은 런타임의 정지 요청이며 런타임이 올리고 내립니다. 그 밖의 bit 는 embedding host (예: REPL 의 인터럽트) 의 것이며 런타임은 건드리지 않습니다. poll 은 플래그가 0 이 아니면 언제나 `rok_runtime_yield` 를 호출합니다.

자세한 메커니즘은 [배리어와 루트](../007-runtime/704-barriers-and-roots.md) 의 Safepoint Polling 항목에서 정의합니다.

### `rok_tls_tlab_cursor` (ptr)

[할당](../007-runtime/702-allocation.md) 의 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 에서 다음 할당이 일어날 위치를 가리킵니다.

```
allocate(size):
    tmp = tlab_cursor
    new_cursor = tmp + size
    if new_cursor > tlab_limit:
        slow path
    tlab_cursor = new_cursor
    return tmp
```

이 네 개의 명령어가 TLAB fast path 를 구성하며, 컴파일러는 이를 인라인으로 emit 하게 됩니다.

### `rok_tls_tlab_limit` (ptr)

TLAB 의 끝을 가리킵니다. cursor 가 이 값을 넘어서면 slow path 로 진입합니다.

### `rok_tls_shadow_stack_top` (ptr)

shadow stack 체인 머리입니다. 자세한 내용은 8.3.8 에서 정의합니다.

### `rok_tls_satb_cursor` / `rok_tls_satb_limit` (ptr)

동시 mark ([7.5](../007-runtime/705-concurrent-collection.md)) 중 write barrier slow path 가 기록하는 SATB buffer 입니다. 가득 차면 수집기에 넘깁니다. mark 밖에서는 쓰지 않습니다.

### `rok_tls_wb_log_cursor` / `rok_tls_wb_log_limit` (ptr)

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md) 의 write barrier log buffer 에 대한 cursor 와 limit 입니다.

```
write_barrier(p, q):
    if !needs_log: return
    if wb_log_cursor >= wb_log_limit: flush
    *wb_log_cursor++ = p
```

대부분의 동작은 non-atomic 이며 thread-local 로 처리됩니다.

두 슬롯은 스레드마다의 4 KiB log chunk 를 가리킵니다. 가득 찬 chunk 는 수집기가 비우는 lock-free stack 에 올립니다 ([7.4.2](../007-runtime/704-barriers-and-roots.md#742-write-barrier-incremental-update)).

### `rok_tls_mutator_state` (u8)

mutator 의 현재 상태를 나타냅니다.

| 값    | 의미                          |
|-------|-------------------------------|
| `0`   | RUNNING. 일반 실행            |
| `1`   | GC_COOPERATING. GC 와 협력 중 |
| `2`   | PARKED. sleep / wait 중       |
| `3` ~ | 예약                          |

이 값은 GC 사이클의 동기화나 safepoint 협력에 사용됩니다.

### `rok_tls_worker_id` (u64)

해당 thread 의 worker id (스케줄러 내부에서 사용) 를 나타내며, [work-stealing](../000-meta/007-glossary.md#work-stealing) 이나 통계 등에 활용됩니다.

## 8.3.4. 슬롯 폭과 정렬

각 슬롯은 자기 타입의 폭을 가지며, 그 폭에 맞춰 정렬된 offset 에 놓입니다.

## 8.3.5. 32-bit 와 64-bit 의 차이

포인터 슬롯은 p 바이트 (64-bit 에서 8, 32-bit 에서 4) 이므로, offset 은 표가 정한 대로 32-bit 와 64-bit 사이에 다릅니다.

## 8.3.6. TLS 슬롯의 초기화

worker 가 시작될 때 다음과 같이 초기화됩니다.

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

`rok_thread_init` 은 스케줄러가 worker 시작 시 호출하며, 이 함수는 [extern 심볼](805-extern-symbols.md) 의 일부에 해당합니다.

## 8.3.7. TLS 슬롯의 종료

worker 가 종료될 때는 다음 절차를 거칩니다.

```
rok_thread_shutdown():
  // log flush
  // shadow stack walk (마지막 mark 도 가능하면)
  // worker_id 반환
```

## 8.3.8. 컴파일러 emit 의 형식

블록은 런타임이 정의하고 export 하는 `thread_local` global 입니다. 컴파일러는 이를 외부 선언으로 참조하고, 블록 주소에서 슬롯을 offset 으로 load 합니다. Rok 함수는 그 주소를 호출자에게서 숨은 마지막 인자로 받으므로 (8.6.6) Rok 함수 사이의 호출은 TLS 를 읽지 않습니다. Rok 바깥에서 들어오는 코드는 주소를 직접 한 번 얻어 (`llvm.threadlocal.address`) 넘겨 줍니다: C `main`, `extern "C"` 정의의 C 진입점, `#[export_name]` Rok 함수, `any` dispatcher, lazy `static` 의 초기화 함수, 그리고 함수 값이 거치는 thunk (8.6.11) 입니다.

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

이후 LLVM 백엔드가 타깃별 TLS 명령어로 lower 해 주게 됩니다.

`rok_tls` 는 ELF 에서 initial-exec TLS 모델을 씁니다 (컴파일러가 접근 코드를 이 모델로 emit 하므로 링커의 relax 에 기대지 않음). 런타임은 항상 실행 파일에 정적으로 링크되므로 이 모델로 충분합니다. Mach-O 는 플랫폼의 TLV 디스크립터 방식을 씁니다. 툴체인이 TLS 모델을 고를 수 없는 구현은 general-dynamic 으로 대신해도 됩니다.

### shadow stack 머리

`rok_tls` 의 offset 0 슬롯 (`rok_tls_shadow_stack_top`) 은 [배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-emit-하는-shadow-stack) 의 [shadow stack](../000-meta/007-glossary.md#shadow-stack) 체인 머리, 즉 가장 안쪽 `ShadowFrame` 의 주소입니다 (없으면 null). 체인 머리는 이것 하나뿐입니다. 컴파일러가 emit 하는 코드가 함수를 지나는 각 경로의 첫 safepoint 앞에서 frame 을 link 하고, link 한 경로의 모든 `return` 직전에 unlink 합니다 (8.4.7).

LLVM 의 `llvm.gcroot` 와 `llvm_gc_root_chain` 은 쓰지 않습니다. `llvm_gc_root_chain` 은 프로세스 전역 하나뿐이라 thread 별 체인을 표현할 수 없기 때문입니다.

## 8.3.9. 안정성

8.3.1 표의 offset, 타입, 의미가 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 의 일부이며, 변경 시에는 [개요](README.md) 의 안정성 정책을 따릅니다. offset 을 바꾸는 것은 major `break` 이며, 새 슬롯은 기존 슬롯 뒤의 offset 을 받으므로 minor compatible 변경입니다. 기존 슬롯의 타입, 의미를 바꾸는 것도 major `break` 입니다.

impl-defined 슬롯은 ABI 의 일부가 아니므로 같은 major 안에서도 자유롭게 바뀔 수 있습니다.
