# 8.3. Mutator TLS 슬롯 레이아웃

각 OS thread (= scheduler worker = mutator) 는 thread-local storage (TLS) 의 정해진 자리에 런타임 정보를 가지게 됩니다. 컴파일러가 emit 하는 코드와 런타임이 동일한 TLS 슬롯을 읽고 쓰기 때문에, 본 문서에서 각 슬롯의 심볼 이름과 타입을 정의합니다.

## 8.3.1. TLS 슬롯 구조

TLS 슬롯은 하나의 블록이 아니라, 슬롯마다 런타임이 export 하는 `thread_local` global 하나입니다. 컴파일러는 고정 offset 이 아니라 심볼 이름으로 슬롯에 접근합니다.

| 심볼                        | 타입 | 의미                                    |
|-----------------------------|------|-----------------------------------------|
| `rok_tls_yield_flag`        | u8   | safepoint 에서 `yield` 해야 하는지     |
| `rok_tls_tlab_cursor`       | ptr  | TLAB 의 다음 할당 위치                  |
| `rok_tls_tlab_limit`        | ptr  | TLAB 의 끝                              |
| `rok_tls_shadow_stack_top`  | ptr  | shadow stack 체인 머리                  |
| `rok_tls_wb_log_cursor`     | ptr  | write barrier log 의 위치               |
| `rok_tls_wb_log_limit`      | ptr  | write barrier log 의 끝                 |
| `rok_tls_satb_cursor`       | ptr  | SATB buffer 의 위치 (동시 mark 중)      |
| `rok_tls_satb_limit`        | ptr  | SATB buffer 의 끝                       |
| `rok_tls_mutator_state`     | u8   | mutator 상태                            |
| `rok_tls_worker_id`         | u64  | worker id                               |
| `rok_tls_task_worker`       | ptr  | 이 스레드의 executor worker (GC 객체, 없으면 0)           |
| `rok_tls_stack_guard`       | [usize; 2] | `[lo, hi)` — 이 범위의 fault 는 이 스레드의 스택 오버플로 (`[0, 0)` 이면 모름, [8.5.7](805-extern-symbols.md#스택-오버플로)) |

**ABI 계약** 은 위 표의 심볼 이름, 타입, 의미입니다. 슬롯 사이의 배치와 offset 은 링커와 LLVM 의 TLS 모델이 정하며 ABI 에 속하지 않습니다.

**impl-defined 슬롯**: 런타임은 통계, 프로파일링, 디버깅 등을 위해 위 표에 없는 thread-local 변수를 자유롭게 둘 수 있습니다. 컴파일러와 외부 패키지는 이것을 참조해서는 안 됩니다.

## 8.3.2. 접근 메커니즘

각 OS 와 아키텍처는 TLS 접근에 대한 자체 메커니즘을 갖추고 있습니다.

| 플랫폼         | 메커니즘                              |
|----------------|---------------------------------------|
| Linux x86_64   | `%fs:offset` 또는 `%gs:offset`        |
| Linux aarch64  | `tpidr_el0`                           |
| macOS x86_64   | `%gs:offset`                          |
| Windows x86_64 | TEB (Thread Environment Block) offset |
| WASM           | `__tls_base` global + offset          |

컴파일러는 타깃별로 적절한 명령어를 emit 하게 되며, LLVM 의 `__thread` 와 TLS 모델이 이를 자동으로 처리합니다.

## 8.3.3. 슬롯별 의미

### `rok_tls_yield_flag` (u8)

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 가 safepoint 시점에 set 하는 플래그이며, mutator 는 poll 단계에서 이를 검사합니다. 값이 `0` 이면 정상 실행 상태이고, `1` 이면 `yield` 해야 하는 상태로서 다음 poll 에서 `rok_runtime_yield` 를 호출합니다.

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

각 슬롯은 자기 타입의 폭과 정렬을 따릅니다. 슬롯마다 별개의 global 이므로 패딩을 맞출 필요가 없습니다.

## 8.3.5. 32-bit 와 64-bit 의 차이

ptr 타입 슬롯은 타깃의 포인터 폭 (64-bit 에서 8 바이트, 32-bit 에서 4 바이트) 을 가집니다. 심볼로 접근하므로 두 환경 사이에 맞춰야 할 offset 이 없습니다.

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

각 TLS 슬롯은 런타임이 정의하고 export 하는 `thread_local` global 입니다. 컴파일러는 이를 외부 선언으로 참조하며, 다음과 같은 형태로 코드를 생성합니다.

```mlir
; LLVM IR — 정의는 런타임, 컴파일러 산출물은 external 선언
@rok_tls_yield_flag = external thread_local global i8
@rok_tls_tlab_cursor = external thread_local global ptr
...

; safepoint poll
%flag = load i8, i8* @rok_tls_yield_flag
%check = icmp ne i8 %flag, 0
br i1 %check, label %slow, label %cont
```

이후 LLVM 백엔드가 타깃별 TLS 명령어로 lower 해 주게 됩니다.

`rok_tls_*` 슬롯은 ELF 에서 initial-exec TLS 모델을 씁니다 (컴파일러가 접근 코드를 이 모델로 emit 하므로 링커의 relax 에 기대지 않음). 런타임은 항상 실행 파일에 정적으로 링크되므로 이 모델로 충분합니다. Mach-O 는 플랫폼의 TLV 디스크립터 방식을 씁니다. 툴체인이 TLS 모델을 고를 수 없는 구현은 general-dynamic 으로 대신해도 됩니다.

### `rok_tls_shadow_stack_top`

```
@rok_tls_shadow_stack_top = thread_local global ptr null     ; 런타임 정의, export
```

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-emit-하는-shadow-stack) 의 [shadow stack](../000-meta/007-glossary.md#shadow-stack) 체인 머리, 즉 가장 안쪽 `ShadowFrame` 의 주소입니다 (없으면 null). 런타임이 export 하는 thread-local 심볼이며, 체인 머리는 이것 하나뿐입니다. 컴파일러가 emit 하는 함수 진입 코드가 frame 을 link 하고 모든 `return` 직전에 unlink 합니다.

LLVM 의 `llvm.gcroot` 와 `llvm_gc_root_chain` 은 쓰지 않습니다. `llvm_gc_root_chain` 은 프로세스 전역 하나뿐이라 thread 별 체인을 표현할 수 없기 때문입니다.

## 8.3.9. 안정성

8.3.1 표의 심볼 이름, 타입, 의미가 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 의 일부이며, 변경 시에는 [개요](README.md) 의 안정성 정책을 따릅니다. 새 심볼을 추가하는 것은 minor compatible 변경이고, 기존 심볼의 이름이나 타입, 의미를 바꾸는 것은 major `break` 에 해당합니다.

impl-defined 슬롯은 ABI 의 일부가 아니므로 같은 major 안에서도 자유롭게 바뀔 수 있습니다.
