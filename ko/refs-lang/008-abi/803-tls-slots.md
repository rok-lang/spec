# 8.3. Mutator TLS 슬롯 레이아웃

각 OS thread (= scheduler worker = mutator) 는 thread-local storage (TLS) 의 정해진 자리에 런타임 정보를 가지게 됩니다. 컴파일러가 emit 하는 코드와 런타임이 동일한 TLS 슬롯을 읽고 쓰기 때문에, 본 문서에서 정확한 슬롯 위치를 정의해 두려고 합니다.

## 8.3.1. TLS 블록 구조

TLS 블록은 OS thread 단위로 다음과 같은 형태를 가집니다.

```
TLS Block (per OS thread):
+---------+---------+----------------------+----------------+
| offset  |  size   |  field               |  type          |
+---------+---------+----------------------+----------------+
|    0    |    1    |  yield_flag          |  u8            |
|    8    |    8    |  tlab_cursor         |  ptr           |
|   16    |    8    |  tlab_limit          |  ptr           |
|   24    |    8    |  shadow_stack_top    |  ptr           |
|   32    |    8    |  wb_log_cursor       |  ptr           |
|   40    |    8    |  wb_log_limit        |  ptr           |
|   48    |    1    |  mutator_state       |  u8            |
|   56    |    8    |  worker_id           |  u64 / ptr     |
+---------+---------+----------------------+----------------+   ← 64 byte 경계
|   64+   |   var   |  extra slots         |  impl-defined  |
+---------+---------+----------------------+----------------+
```

TLS 블록은 두 영역으로 나뉘어 있습니다.

**ABI 안정 영역 (offset 0–63, 총 64 바이트)**
이 영역은 컴파일러와 런타임 사이의 계약 자리입니다. 동일한 major 버전에서 모든 컴파일러, 런타임, 패키지가 동일한 offset 을 참조하도록 보장합니다. 슬롯의 위치, 크기, 의미는 본 문서가 권위 자리를 차지합니다. 64 바이트라는 크기 자체는 우연이 아니라 의도된 설계로서, 대부분의 64-bit 아키텍처 (x86_64, ARM64) 의 cache line 크기와 일치합니다. 따라서 fast-path 가 자주 만지는 슬롯들이 단일 cache line 안에 모이게 되어, false sharing 을 피하고 single prefetch 만으로 hot 영역을 끌어올 수 있습니다.

**impl-defined 영역 (offset 64 이상)**
ABI 보장 밖 영역입니다. 런타임 구현이 자유롭게 사용하며, 통계, 프로파일링, 추가 캐시, 디버깅 같은 용도로 쓰이게 됩니다. 외부 패키지가 이 영역을 직접 접근해서는 안 되며, 같은 major 버전 안에서도 위치와 크기가 변경될 수 있습니다.

이 두 영역의 분리는 Bionic (Android) 의 TLS slot 정책과 비슷한 설계로서, ABI 약속과 구현 유연성을 동시에 확보하기 위한 장치입니다. 향후 ABI 안정 영역을 확장해야 한다면 8.3.9 의 절차에 따라 끝에 추가하는 minor compatible 변경으로 처리됩니다.

## 8.3.2. 접근 메커니즘

각 OS 와 아키텍처는 TLS 접근에 대한 자체 메커니즘을 갖추고 있습니다.

| 플랫폼         | 메커니즘                              |
|----------------|---------------------------------------|
| Linux x86_64   | `%fs:offset` 또는 `%gs:offset`        |
| Linux aarch64  | `tpidr_el0`                           |
| macOS x86_64   | `%gs:offset`                          |
| Windows x86_64 | TEB (Thread Environment Block) offset |
| WASM           | `__tls_base` global + offset          |

컴파일러는 타깃별로 적절한 명령어를 emit 하게 되며, LLVM 의 `__thread` 와 TLS 모델이 이를 자동으로 처리해 줍니다.

## 8.3.3. 슬롯별 의미

### `yield_flag` (offset 0, u8)

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 가 safepoint 시점에 set 하는 플래그이며, mutator 는 poll 단계에서 이를 검사합니다. 값이 `0` 이면 정상 실행 상태이고, `1` 이면 `yield` 해야 하는 상태로서 다음 poll 에서 `rok_runtime_yield` 를 호출하게 됩니다.

자세한 메커니즘은 [배리어와 루트](../007-runtime/704-barriers-and-roots.md) 의 Safepoint Polling 항목에서 정의하고 있습니다.

### `tlab_cursor` (offset 8, ptr)

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

### `tlab_limit` (offset 16, ptr)

TLAB 의 끝을 가리킵니다. cursor 가 이 값을 넘어서면 slow path 로 진입하게 됩니다.

### `shadow_stack_top` (offset 24, ptr)

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md) 의 [shadow stack](../000-meta/007-glossary.md#shadow-stack) 의 현재 top 을 나타냅니다. 함수 진입 시점에 gcroot 변수들을 위한 frame 을 push 하고, 함수 종료 시점에 pop 합니다.

컴파일러는 `llvm.gcroot` intrinsic 으로 emit 하고, LLVM 백엔드가 자동으로 emit 하는 코드가 이 슬롯을 읽고 쓰는 방식으로 동작합니다.

### `wb_log_cursor` / `wb_log_limit` (offset 32, 40, ptr)

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md) 의 write barrier log buffer 에 대한 cursor 와 limit 입니다.

```
write_barrier(p, q):
    if !needs_log: return
    if wb_log_cursor >= wb_log_limit: flush
    *wb_log_cursor++ = p
```

대부분의 동작은 non-atomic 이며 thread-local 로 처리됩니다.

### `mutator_state` (offset 48, u8)

mutator 의 현재 상태를 나타냅니다.

| 값    | 의미                          |
|-------|-------------------------------|
| `0`   | RUNNING. 일반 실행            |
| `1`   | GC_COOPERATING. GC 와 협력 중 |
| `2`   | PARKED. sleep / wait 중       |
| `3` ~ | 예약                          |

이 값은 GC 사이클의 동기화나 safepoint 협력에 사용됩니다.

### `worker_id` (offset 56, u64)

해당 thread 의 worker id (스케줄러 내부에서 사용) 를 나타내며, [work-stealing](../000-meta/007-glossary.md#work-stealing) 이나 통계 등에 활용됩니다.

## 8.3.4. 슬롯 폭과 정렬

대부분의 슬롯은 8 바이트 정렬을 따르고 있습니다. `yield_flag` 나 `mutator_state` 처럼 1 바이트짜리 슬롯도 정렬을 위해 8 바이트 자리에 패딩을 포함하여 배치됩니다. 이러한 정렬 덕분에 fast path 에서 single load 만으로 슬롯에 접근할 수 있게 됩니다.

## 8.3.5. 32-bit 와 64-bit 의 차이

포인터 슬롯의 폭이 서로 다릅니다.

| 슬롯               | 64-bit   | 32-bit              |
|--------------------|----------|---------------------|
| `tlab_cursor`      | 8 바이트 | 4 바이트 (자리는 8) |
| `tlab_limit`       | 8 바이트 | 4 바이트 (자리는 8) |
| `shadow_stack_top` | 8 바이트 | 4 바이트 (자리는 8) |
| ...                | ...      | ...                 |

32-bit 환경에서 포인터는 4 바이트이지만, 슬롯 자리는 항상 8 바이트로 잡혀 있고 상위 4 바이트는 패딩으로 채워집니다. 이러한 패딩은 64-bit 와 32-bit 양쪽에서 동일한 offset 을 유지하기 위한 장치입니다.

## 8.3.6. TLS 블록의 초기화

worker 가 시작될 때 다음과 같이 초기화됩니다.

```
rok_thread_init():
  yield_flag = 0
  tlab_cursor = NULL                  // slow path 가 첫 할당 시 채움
  tlab_limit = NULL
  shadow_stack_top = shadow_stack_base
  wb_log_cursor = wb_log_base
  wb_log_limit = wb_log_base + LOG_SIZE
  mutator_state = RUNNING
  worker_id = (allocated)
```

`rok_thread_init` 은 스케줄러가 worker 시작 시 호출하며, 이 함수는 [extern 심볼](805-extern-symbols.md) 의 일부에 해당합니다.

## 8.3.7. TLS 블록의 종료

worker 가 종료될 때는 다음 절차를 거칩니다.

```
rok_thread_shutdown():
  // log flush
  // shadow stack walk (마지막 mark 도 가능하면)
  // worker_id 반환
```

## 8.3.8. 컴파일러 emit 의 형식

각 TLS 슬롯에 대한 접근은 LLVM IR 의 `@thread_local` global 로 모델링됩니다. 컴파일러는 다음과 같은 형태로 코드를 생성합니다.

```mlir
; LLVM IR
@rok_tls_yield_flag = thread_local global i8 0
@rok_tls_tlab_cursor = thread_local global i8* null
...

; safepoint poll
%flag = load i8, i8* @rok_tls_yield_flag
%check = icmp ne i8 %flag, 0
br i1 %check, label %slow, label %cont
```

이후 LLVM 백엔드가 타깃별 TLS 명령어로 lower 해 주게 됩니다.

## 8.3.9. 안정성

본 TLS 레이아웃 중 **ABI 안정 영역 (offset 0–63)** 만이 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 의 일부에 해당하며, 변경 시에는 [개요](README.md) 의 안정성 정책을 따릅니다. 새 슬롯을 추가하는 경우 기존 offset 을 유지한 채 안정 영역의 끝에 추가하면 minor compatible 한 변경이 되고 (필요하다면 64 → 128 바이트로 안정 영역 자체를 확장하는 것도 같은 방식이 됩니다), 기존 슬롯의 offset 이나 size 를 변경하는 것은 major `break` 에 해당합니다.

**impl-defined 영역 (offset 64 이상)** 은 ABI 의 일부가 아니므로 안정성 보장 대상에서 제외됩니다. 같은 major 안에서도 minor 또는 patch 사이에 자유롭게 변경 가능하며, 외부 패키지가 이 영역을 직접 접근해서는 안 됩니다.

이러한 보장 덕분에 동일한 major version 의 컴파일러, 런타임, 사용자 패키지가 모두 안정 영역에 한해 호환되어 동작할 수 있습니다.
