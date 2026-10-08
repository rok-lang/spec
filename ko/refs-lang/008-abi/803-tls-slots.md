# 8.3. Mutator TLS 슬롯 레이아웃

각 OS thread (= scheduler worker = mutator) 는 thread-local storage (TLS) 의 정해진 자리에
런타임 정보를 갖는다. 컴파일러 emit 코드와 런타임이 같은 TLS 슬롯을 읽고 쓰므로 본 문서가
정확한 슬롯 위치를 정의한다.

## 8.3.1. TLS 블록 구조

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
|   64+   |   var   |  추가 슬롯           |  구현 결정     |
+---------+---------+----------------------+----------------+
```

총 고정 64 바이트 (정렬 / 패딩 포함).

## 8.3.2. 접근 메커니즘

각 OS / 아키텍처가 TLS 접근에 자체 메커니즘을 가진다.

| 플랫폼         | 메커니즘                              |
| -------------- | ------------------------------------- |
| Linux x86_64   | `%fs:offset` 또는 `%gs:offset`        |
| Linux aarch64  | `tpidr_el0`                           |
| macOS x86_64   | `%gs:offset`                          |
| Windows x86_64 | TEB (Thread Environment Block) offset |
| WASM           | `__tls_base` global + offset          |

컴파일러는 타깃 별 적절한 명령어를 emit 한다. LLVM 의 `__thread` / TLS 모델이 자동 처리한다.

## 8.3.3. 슬롯 별 의미

### `yield_flag` (offset 0, u8)

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 가 safepoint 시 set 한다. mutator 가 poll 에서 검사한다.

- `0`: 정상 실행.
- `1`: `yield` 해야 함 (다음 poll 에서 `nok_runtime_yield` 호출).

자세한 메커니즘은 [배리어와 루트](../007-runtime/704-barriers-and-roots.md) 의 Safepoint Polling 에서 정의한다.

### `tlab_cursor` (offset 8, ptr)

[할당](../007-runtime/702-allocation.md) 의 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 의 다음 할당 위치.

```
allocate(size):
    tmp = tlab_cursor
    new_cursor = tmp + size
    if new_cursor > tlab_limit:
        slow path
    tlab_cursor = new_cursor
    return tmp
```

이 4 명령어가 TLAB fast path. 컴파일러가 인라인 emit 한다.

### `tlab_limit` (offset 16, ptr)

TLAB 의 끝. cursor 가 이를 넘으면 slow path.

### `shadow_stack_top` (offset 24, ptr)

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md) 의 [shadow stack](../000-meta/007-glossary.md#shadow-stack) 의 현재 top.

함수 진입 시 gcroot 변수들을 위한 frame push. 함수 종료 시 pop.

컴파일러는 `llvm.gcroot` intrinsic 으로 emit 하고, LLVM 백엔드가 자동 emit 하는 코드가 이 슬롯을
읽고 쓴다.

### `wb_log_cursor` / `wb_log_limit` (offset 32, 40, ptr)

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md) 의 write barrier log buffer 의 cursor / limit.

```
write_barrier(p, q):
    if !needs_log: return
    if wb_log_cursor >= wb_log_limit: flush
    *wb_log_cursor++ = p
```

대부분 non-atomic, thread-local.

### `mutator_state` (offset 48, u8)

mutator 의 현재 상태.

| 값    | 의미                          |
| ----- | ----------------------------- |
| `0`   | RUNNING. 일반 실행            |
| `1`   | GC_COOPERATING. GC 와 협력 중 |
| `2`   | PARKED. sleep / wait 중       |
| `3` ~ | 예약                          |

GC 사이클의 동기화 / safepoint 협력에 사용한다.

### `worker_id` (offset 56, u64)

이 thread 의 worker id (스케줄러 내부). [work-stealing](../000-meta/007-glossary.md#work-stealing) 이나 통계 등에 사용한다.

## 8.3.4. 슬롯 폭 / 정렬

- 8 바이트 정렬의 슬롯이 대부분.
- `yield_flag`, `mutator_state` 같이 1 바이트 슬롯도 정렬을 위해 8 바이트 자리에 padding
  포함.

이 정렬 덕분에 fast path 의 single load 가 가능하다.

## 8.3.5. 32-bit vs 64-bit 차이

포인터 슬롯의 폭이 다르다.

| 슬롯               | 64-bit   | 32-bit              |
| ------------------ | -------- | ------------------- |
| `tlab_cursor`      | 8 바이트 | 4 바이트 (자리는 8) |
| `tlab_limit`       | 8 바이트 | 4 바이트 (자리는 8) |
| `shadow_stack_top` | 8 바이트 | 4 바이트 (자리는 8) |
| ...                | ...      | ...                 |

32-bit 에서 포인터는 4 바이트지만, 슬롯은 항상 8 바이트 자리. 상위 4 바이트는 패딩.

이 패딩은 64-bit 와 32-bit 양쪽에서 같은 offset 을 유지하기 위함이다.

## 8.3.6. TLS 블록의 초기화

worker 시작 시:

```
nok_thread_init():
  yield_flag = 0
  tlab_cursor = NULL                  // slow path 가 첫 할당 시 채움
  tlab_limit = NULL
  shadow_stack_top = shadow_stack_base
  wb_log_cursor = wb_log_base
  wb_log_limit = wb_log_base + LOG_SIZE
  mutator_state = RUNNING
  worker_id = (allocated)
```

`nok_thread_init` 은 스케줄러가 worker 시작 시 호출한다. 본 함수는 [extern 심볼](805-extern-symbols.md) 의 일부.

## 8.3.7. TLS 블록의 종료

worker 종료 시:

```
nok_thread_shutdown():
  // log flush
  // shadow stack walk (마지막 mark 도 가능하면)
  // worker_id 반환
```

## 8.3.8. 컴파일러 emit 의 형식

각 TLS 슬롯 접근은 LLVM IR 의 `@thread_local` global 으로 모델링한다. 컴파일러는:

```mlir
; LLVM IR
@nok_tls_yield_flag = thread_local global i8 0
@nok_tls_tlab_cursor = thread_local global i8* null
...

; safepoint poll
%flag = load i8, i8* @nok_tls_yield_flag
%check = icmp ne i8 %flag, 0
br i1 %check, label %slow, label %cont
```

LLVM 백엔드가 타깃 별 TLS 명령어로 lower 한다.

## 8.3.9. 안정성

본 TLS 레이아웃은 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 의 일부. 변경 시 [개요](README.md) 의 안정성 정책을 따른다.

- 새 슬롯 추가: 기존 offset 유지하면서 끝에만 추가하면 minor compatible.
- 기존 슬롯 offset / size 변경: major `break`.

이 보장으로 같은 major version 의 컴파일러, 런타임, 사용자 패키지가 모두 호환된다.
