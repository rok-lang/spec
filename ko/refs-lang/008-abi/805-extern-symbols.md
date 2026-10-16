# 8.5. Extern "C" 심볼

`librokr.a` 가 노출하는 [심볼](../000-meta/007-glossary.md#extern-c-심볼) 의 시그니처를 정의합니다. 컴파일러가 생성한 코드와 표준 라이브러리가 이 심볼을 부릅니다.

rok 쪽 선언은 런타임 ABI `extern "rok"` ([5.9.2](../005-modules/509-extern-block.md#592-abi-명시)) 이며, toolchain 이 `rok::core::rt` 에 `pub(pkg)` 로 둡니다. 아래 블록의 `str` 은 `(ptr, len)` 두 word, `fn` 값은 `RokFn` (8.5.0a), 그 밖의 값은 C ABI 와 같습니다. C 에서 부를 때는 이 펼친 형태를 씁니다.

> [!IMPORTANT]
> 모든 심볼은 `rok_` 접두사로 시작합니다. 이는 외부 라이브러리나 사용자 코드와의 심볼 충돌을 회피하기 위함입니다.

## 8.5.0. Thread-safety 분류

각 심볼의 정의 옆에는 **Thread-safety** 표기가 따라붙습니다. 이는 컴파일러가 emit 한 코드와 런타임 사이의 contract 의 일부로, 어떤 호출 컨텍스트에서 안전하게 호출할 수 있는지를 명시합니다. 사용 가능한 분류는 다음 6 가지입니다.

| 분류                  | 의미                                                                         |
|-----------------------|------------------------------------------------------------------------------|
| `thread-safe`         | 다중 thread 가 동시에 호출해도 안전 (런타임이 내부 동기화)                   |
| `caller-thread-only`  | 호출자 thread 의 컨텍스트에 묶임 — 다른 thread 가 그 호출자를 대신할 수 없음 |
| `process-init`        | 프로세스 시작 시점, mt 진입 전 단 한 번                                      |
| `process-shutdown`    | 프로세스 종료 시점, mt 종료 후 단 한 번                                      |
| `per-thread-init`     | 각 worker thread 의 시작 시점에 그 thread 가 자기 자신에 대해 한 번          |
| `per-thread-shutdown` | 각 worker thread 의 종료 시점에 그 thread 가 자기 자신에 대해 한 번          |

`caller-thread-only` 는 함수의 효과가 호출자의 TLS 슬롯이나 stack frame 에 결합되어 있어 다른 thread 에서 그 호출자를 대신할 수 없다는 뜻입니다. 함수 자체가 reentrant 하지 않다는 의미는 아니며, 다중 thread 가 각자 자기 호출자에 대해 동시에 호출하는 것은 허용됩니다.

## 8.5.0a. `RokFn` — 함수 값의 C ABI 형태

rok 함수 값 `{code, env}` ([4.4](../004-functions/404-function-type.md#함수-값의-표현)) 를 런타임 심볼에 넘길 때의 C 표현입니다.

```c
typedef struct {
    void *code;   // env 를 첫 인자로 받는 코드 포인터
    void *env;    // 캡처 환경 (이름 있는 함수는 NULL)
} RokFn;
```

`RokFn` 은 C 구조체로 **값 전달** 됩니다. 실제 레지스터 배치는 타깃 C ABI 의 16 바이트 구조체 규칙을 따릅니다.

## 8.5.1. GC 영역

### `rok_gc_alloc_slow`

[할당](../007-runtime/702-allocation.md) 의 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) slow path 에 해당합니다. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_gc_alloc_slow(size: usize, align: usize, type_id: u32) -> Ptr<u8>;
}
```

각 인자와 반환값의 의미는 다음과 같습니다.

- `size`: 객체의 총 크기로, 16 바이트의 헤더에 페이로드를 더한 뒤 정렬한 값입니다.
- `align`: 객체의 정렬 요구로, 보통 8 이며 `#[repr(align(N))]` 가 지정되면 더 커집니다.
- `type_id`: 객체의 타입 ID 입니다.
- 반환: 헤더 자리부터 시작하는 포인터를 돌려주며, type_id 와 gc_flags 는 함수가 채워둡니다.

null 을 반환하는 경우는 out of memory 와 같은 극단적 상황입니다. 실제로는 fatal error 를 던지게 되며, 이는 구현이 결정합니다.

### `rok_gc_alloc_bytes`

헤더 뒤에 `size` 바이트의 payload 를 가진 GC 객체를 할당합니다. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_gc_alloc_bytes(size: usize, traced: bool) -> str;
}
```

- `traced = false` 이면 type id `1` (GC 문자열), `true` 이면 type id `2` (word-traced buffer) 로 할당합니다 ([8.2.12](802-scan-table.md#8212-런타임-예약-type-id)).
- 반환값은 payload 를 가리키는 `(ptr, len)` 쌍 (`str` 의 표현, [8.8](808-lowering-types.md)) 입니다.

### `rok_alloc_raw` / `rok_realloc_raw`

GC 밖의 raw 메모리를 할당합니다 ([stdlib 1.4 alloc](../../refs-standard-libraries/100-bare/104-alloc.md)). **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_alloc_raw(size: usize, align: usize, zeroed: i32) -> usize;
    fn rok_realloc_raw(ptr: usize, old_size: usize, align: usize, new_size: usize) -> usize;
}
```

`zeroed` 가 0 이 아니면 할당한 메모리를 0 으로 채웁니다. `rok_alloc_raw` 는 `size` 가 0 이거나 `align` 이 2 의 거듭제곱이 아니거나 할당에 실패하면 null (`0`) 을 반환하며, abort 하지 않습니다. `rok_realloc_raw` 도 실패 시 null 을 반환합니다. 실패 처리는 호출자 (`rok::bare::alloc`) 가 합니다.

### `rok_gc_wb_slow`

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md) 의 write barrier slow path 에 해당합니다. **Thread-safety: `caller-thread-only`** (TLS 의 wb log buffer 에 기록).

```
extern "C" {
    fn rok_gc_wb_slow(obj: Ptr<u8>);
}
```

인자 `obj` 는 변경된 슬롯이 속한 객체의 포인터이며, 별도의 반환값은 없습니다. 효과로는 remembered set 에 obj 를 추가하고, log 가 차면 flush 합니다.

### `rok_gc_register_type`

[ScanTable](802-scan-table.md) 의 타입을 등록합니다. **Thread-safety: `process-init`**.

```
extern "C" {
    fn rok_gc_register_type(type_id: u32, scan_table: ConstPtr<u8>);
}
```

호출 시점은 프로세스 시작 시점이며, mt 진입 전이므로 동기화 불필요.

### `rok_gc_pin` / `rok_gc_unpin`

[FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 객체의 pinning 을 담당합니다 ([객체 레이아웃과 메모리 공간](../007-runtime/701-object-layout.md), [`extern` 블록 (FFI 선언)](../005-modules/509-extern-block.md) 참조). **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_gc_pin(obj: Ptr<u8>) -> Ptr<u8>;       // 고정된 객체 (같은 주소) 반환
    fn rok_gc_unpin(obj: Ptr<u8>);
}
```

`extern "C"` 함수나 blocking 런타임 호출에 넘긴 GC 포인터는 호출이 끝날 때까지 인자가 루트에 있으므로 따로 pin 하지 않아도 옮겨지지 않습니다 ([7.4.1](../007-runtime/704-barriers-and-roots.md#루트가-가리키는-객체는-옮기지-않음)). C 코드가 호출 뒤에도 포인터를 보관할 때만 pin 이 필요합니다. pin 은 객체에 `PINNED` 를 켜서 evacuation 대상에서 빼고 GC 가 살려 두게 하며, unpin 은 그것을 풉니다.

### `rok_gc_collect`

명시적인 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 트리거이며, 드물게 사용됩니다. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_gc_collect();
}
```

full 수집을 하고, 수집이 끝난 뒤 돌아옵니다.

## 8.5.2. Runtime 영역

### `rok_runtime_init`

런타임을 초기화합니다. **Thread-safety: `process-init`**.

```
extern "C" {
    fn rok_runtime_init();
}
```

호출 시점은 프로세스 시작 직후이자 `main()` 이전이며, 효과로는 worker 풀을 생성하고 GC 공간을 mmap 하며 I/O driver 를 초기화합니다. 컴파일러가 생성한 C-ABI `main` 은 이 호출 직후에 `rok.register_types` 를 호출합니다 ([8.2.13](802-scan-table.md#8213-등록-시점)).

런타임은 이 밖에 thread-local 데이터 심볼 `rok_tls_shadow_stack_top` 을 export 합니다 ([8.3.8](803-tls-slots.md#838-컴파일러-emit-의-형식)).

### `rok_runtime_shutdown`

런타임을 종료합니다. **Thread-safety: `process-shutdown`**.

```
extern "C" {
    fn rok_runtime_shutdown();
}
```

호출 시점은 `main()` 이 끝난 직후이며, 효과로는 worker 풀을 종료하고 미해제 자원을 정리합니다.

### `rok_exit`

프로세스를 즉시 종료합니다. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_exit(code: i32) -> !;
}
```

[`rok::std::env::exit`](../../refs-standard-libraries/300-std/306-env.md) 가 이 심볼을 부릅니다. `finally` 와 `Drop` 은 실행하지 않습니다 ([4.16.4](../004-functions/416-entry-point.md#4164-종료-코드와-미처리-예외)).

### 문자열 helper

`FromStr` 과 `str::bytes` 를 뒷받침하는 내부 helper 입니다 ([stdlib 2.15](../../refs-standard-libraries/200-core/215-string.md)). **Thread-safety: `thread-safe`**. `str` 인자는 `(ptr, len)` 으로 전달됩니다.

```
extern "C" {
    fn rok_str_byte(s: str, i: usize) -> u8;                         // i 번째 바이트 (경계 검사는 호출자)
    fn rok_str_int_status(s: str, signed: i32, bits: u32) -> i32;    // 0 ok, 1 empty, 2 invalid digit, 3 out of range
    fn rok_str_to_i64(s: str) -> i64;                                 // status 0 확인 후 호출
    fn rok_str_to_u64(s: str) -> u64;                                 // status 0 확인 후 호출
    fn rok_str_float_ok(s: str) -> i32;                               // inf, NaN, 지수 포함 — 1 이면 파싱 가능
    fn rok_str_to_f64(s: str) -> f64;                                 // float_ok 확인 후 호출
}
```

### `rok_runtime_yield`

[Safepoint 프로토콜](804-safepoint.md) 의 `yield` 함수, 즉 safepoint poll 의 slow path 입니다. **Thread-safety: `caller-thread-only`** (호출자 mutator 가 자기 yield_flag 를 polling 한 결과로 호출).

```
extern "C" {
    fn rok_runtime_yield();
}
```

### `rok_thread_init` / `rok_thread_shutdown`

worker thread 의 lifecycle 을 담당합니다. **Thread-safety: `rok_thread_init` 은 `per-thread-init`, `rok_thread_shutdown` 은 `per-thread-shutdown`**.

```
extern "C" {
    fn rok_thread_init();
    fn rok_thread_shutdown();
}
```

각 worker thread 의 시작 및 종료 시점에 호출되어, [TLS 슬롯](803-tls-slots.md) 의 TLS 를 초기화하거나 정리합니다.

## 8.5.3. Coroutine 영역

### `rok_coro_frame_alloc`

[코루틴](../007-runtime/708-coroutine.md) 의 frame 을 할당합니다. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_coro_frame_alloc(size: usize, type_id: u32) -> Ptr<u8>;
}
```

`rok_gc_alloc_slow` 와 거의 동일하지만, frame 전용으로 별도 처리가 가능합니다.

### `rok_coro_park` / `rok_coro_unpark`

컴파일러는 이 두 심볼을 emit 하지 않습니다 (`await` 는 `rok::core::task` 의 parking registry 를 씁니다). 런타임은 C 나 저수준 코드를 위해 제공합니다. `frame` (아무 GC 객체) 의 주소가 parking key 이며, `rok_coro_unpark(frame)` 은 그 key 에 park 한 waker 를 모두 깨웁니다. task executor 밖에서 부르면 abort 합니다.

**Thread-safety: `rok_coro_park` 는 `caller-thread-only`** (호출자 코루틴 자신을 park), **`rok_coro_unpark` 는 `thread-safe`** (다른 thread 에서 호출 가능).

```
extern "C" {
    fn rok_coro_park(frame: Ptr<u8>, waker: Ptr<u8>);
    fn rok_coro_unpark(frame: Ptr<u8>);
}
```

`rok_coro_park` 는 코루틴을 일시 중단시키고 waker 를 등록합니다. `rok_coro_unpark` 는 코루틴을 재개시키며 task 큐에 push 합니다.

여기서 `waker` 는 깨우는 함수와 컨텍스트의 묶음, 곧 런타임의 `Waker` 객체 ([2.20.2](../../refs-standard-libraries/200-core/220-future.md)) 를 가리키는 **불투명 포인터 (`Ptr<u8>`)** 입니다. C ABI 경계에서는 `Waker` 의 내부 표현을 노출하지 않으므로, [7.8.8](../007-runtime/708-coroutine.md) 을 비롯한 모든 자리에서 `waker` 인자의 타입은 `Ptr<u8>` 로 통일됩니다 (`rok_io_register`, `rok_timer_register` 도 같음). 포인터가 가리키는 객체의 정확한 레이아웃은 코루틴 구현이 결정합니다.

## 8.5.4. Scheduler 영역

task 런타임 (spawn, join, cancel, executor) 은 `rok::core::task` 의 rok 코드입니다 ([stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)). 런타임은 아래의 executor 슬롯, 스레드, parking 심볼만 제공합니다.

### `rok_task_worker_*` / `rok_time_now_ns`

현재 스레드의 executor worker ([stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)) 를 TLS 슬롯 `rok_tls_task_worker` ([8.3.1](803-tls-slots.md#831-tls-슬롯-구조)) 에 두고 꺼내는 함수와 시계입니다. **Thread-safety: `caller-thread-only`** (`rok_time_now_ns` 는 `thread-safe`).

```
extern "C" {
    fn rok_task_worker_present() -> i32;       // 이 스레드가 worker 이면 1
    fn rok_task_worker_get() -> usize;         // worker 객체 주소, 없으면 0
    fn rok_task_worker_set(w: usize);          // 살아 있는 GC 객체 또는 0. 설치하는 동안 pin
    fn rok_task_worker_clear();                // = rok_task_worker_set(0)
    fn rok_time_now_ns() -> i64;               // monotonic clock (ns)
}
```

### `rok_thread_spawn`

1:1 OS thread 를 생성합니다. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_thread_spawn(f: RokFn, stack_size: usize, id: u64) -> usize;
    fn rok_thread_new_id() -> u64;
    fn rok_thread_id() -> u64;                        // 현재 스레드의 worker id
    fn rok_thread_sleep_ns(ns: i64);
    fn rok_thread_yield_now();
}
```

`rok_thread_spawn` 은 rok 클로저 `f` (아래 `RokFn`, 값 전달) 를 OS 스레드에서 실행합니다 (`stack_size` 0 은 기본값). `id` 는 `rok_thread_new_id` 로 받은 값이며 새 스레드의 `rok_tls_worker_id` 가 됩니다. 반환값은 join handle 이고, 생성에 실패하면 0 입니다. 스레드가 끝날 때까지 `env` 는 GC root 입니다.

### `rok_task_hooks_install` / `rok_task_key`

`rok::core::task` 의 executor 가 런타임에 hook 을 설치합니다. 런타임의 I/O driver, timer, `rok_coro_park` / `rok_coro_unpark` 는 이 hook 으로 task 를 깨우고 멈춥니다. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_task_hooks_install(
        io: extern "C" fn(fd: i32, interest: u32, waker: Ptr<u8>),
        timer: extern "C" fn(deadline_ns: u64, waker: Ptr<u8>),
        park: extern "C" fn(key: Ptr<u8>, waker: Ptr<u8>),
        unpark: extern "C" fn(key: Ptr<u8>),
    );                                          // 처음 설치한 것이 이김
    unsafe fn rok_task_key(obj: Ptr<u8>) -> usize;   // GC 객체 주소를 park 키로
}
```

`rok_task_key` 는 주소를 키로 쓰므로 그 객체에 `NOMOVE` ([8.1.3](801-object-header-abi.md#813-gc_flags-offset-4-u32)) 를 켭니다. 그 객체는 이후 옮겨지지 않습니다. 호출 안에서만 쓰는 동일성 비교는 `buf_same` ([stdlib 2.9](../../refs-standard-libraries/200-core/209-collections.md)) 을 쓰며 아무것도 고정하지 않습니다.

### `rok_sync_wait` / `rok_sync_wake`

동기화 primitive (`Mutex`, `Condvar`, `Once`) 의 대기와 깨우기입니다. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_sync_wait(buf: Ptr<u8>, index: usize, expected: i32, timeout_ns: i64);   // GcBuf<i32> 원소가 expected 인 동안 대기. timeout_ns < 0 이면 무한
    fn rok_sync_wake(buf: Ptr<u8>, index: usize);                                    // 그 원소에서 대기 중인 스레드를 모두 깨움
}
```

`rok_sync_wait` 는 깨운 이유 없이 돌아올 수 있으므로 호출자가 조건을 다시 확인합니다. 대기하는 동안 그 스레드는 GC 에 대해 정지한 것으로 봅니다.

### `rok_thread_join`

**Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_thread_join(handle: usize) -> i32;          // 0 이면 성공
}
```

### `rok_thread_detach` / `rok_thread_park` / `rok_thread_unpark` / `rok_cpu_count`

**Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_thread_detach(handle: usize) -> i32;   // 끝나면 OS 자원 반환, 더 이상 join 불가. 0 이면 성공
    fn rok_thread_park(timeout_ns: i64);          // 토큰이 설 때까지 대기 (음수 = 무기한), 깨운 이유 없이 돌아올 수 있음
    fn rok_thread_unpark(id: u64);                // 스레드 id 의 토큰을 세우고 깨움 (이미 끝났으면 무시)
    fn rok_cpu_count() -> usize;                  // 온라인 CPU 수 (최소 1) — par 풀의 크기
}
```

### `rok_static_init_begin` / `rok_static_init_end`

`const` 가 아닌 초기값을 가진 `static` 의 지연 초기화 ([5.3](../005-modules/503-static.md)) 를 뒷받침합니다. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_static_init_begin(state: Ptr<u8>) -> i32;   // 1: 호출자가 초기화식 실행, 0: 이미 준비됨
    fn rok_static_init_end(state: Ptr<u8>);            // 초기화 완료 공개
}
```

`state` 는 그 `static` 의 상태 바이트 (0 새로, 1 진행 중, 2 준비됨) 입니다. 다른 스레드가 초기화 중이면 `rok_static_init_begin` 은 끝날 때까지 기다리며, 그동안 GC 에 대해 정지한 것으로 봅니다.

### `rok_gc_add_root`

```
extern "C" {
    fn rok_gc_add_root(word: ConstPtr<usize>);   // static 의 참조 word 를 프로세스 끝까지 GC root 로 등록
}
```

컴파일러가 만든 시작 코드가 GC 참조를 담는 모든 `static` 에 대해 호출합니다.

### `rok_os_*` — OS helper (내부, 변경 가능)

표준 라이브러리 `rok::std` (fs, io, env, process) 를 뒷받침하는 내부 심볼입니다. ABI 안정 대상이 아니며 바뀔 수 있으므로 사용자 코드는 직접 부르지 않습니다.

```
rok_os_strerror  rok_os_open      rok_os_close     rok_os_read      rok_os_read_all
rok_os_write     rok_os_write_all rok_os_fsync     rok_os_ftruncate rok_os_lseek
rok_os_dup       rok_os_fchmod    rok_os_flock     rok_os_copy_fd   rok_os_stat
rok_os_mkdir     rok_os_unlink    rok_os_rename    rok_os_link      rok_os_opendir
rok_os_readdir   rok_os_closedir  rok_os_getcwd    rok_os_chdir     rok_os_realpath
rok_os_stdin_line rok_os_stdin_read rok_os_flush_stdout
rok_os_argc      rok_os_arg       rok_os_env_at    rok_os_getenv    rok_os_setenv
rok_os_spawn     rok_os_waitpid   rok_os_kill      rok_os_getpid    rok_os_getppid
rok_os_abort
```

### `rok_deque_*` — work-stealing deque

task 스케줄러 ([7.6.2](../007-runtime/706-scheduler.md#762-work-stealing-구조)) 의 lock-free Chase-Lev deque 입니다. deque 와 그 ring 은 GC 객체이며 GC 가 추적합니다. 그래서 deque 안에만 있는 원소도 살아 있습니다. **Thread-safety:** `push` / `pop` 은 소유 thread 만, `steal` 과 `len` 은 어느 thread 든.

```
extern "C" {
    fn rok_deque_new() -> Ptr<u8>;                                   // 빈 deque (GC 객체)
    unsafe fn rok_deque_push(d: Ptr<u8>, item: usize);               // 소유자: 끝에 넣음 (GC 참조)
    unsafe fn rok_deque_pop(d: Ptr<u8>, out: Ptr<u8>, index: usize) -> i32;    // 소유자: 가장 새 원소를 out[index] 에, 있으면 1
    unsafe fn rok_deque_steal(d: Ptr<u8>, out: Ptr<u8>, index: usize) -> i32;  // 가장 오래된 원소를 out[index] 에, 있으면 1
    unsafe fn rok_deque_len(d: Ptr<u8>) -> usize;                    // 원소 수 (경합 중의 근사값)
}
```

`out` 은 GC 참조를 담는 `GcBuf` 이며 `index` 자리에 결과를 씁니다. 이 함수들 안에는 safepoint 가 없습니다.

## 8.5.5. I/O / Timer 영역

### `rok_io_register` / `rok_timer_register`

C 나 저수준 코드가 task 를 fd 준비나 시각에 깨우도록 등록합니다. 내부적으로는 reactor ([7.7](../007-runtime/707-io-and-timer.md)) 와 executor 의 timer 를 씁니다. task executor 밖에서 부르면 abort 합니다.

**Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_io_register(fd: i32, interest: u32, waker: Ptr<u8>);
    fn rok_timer_register(deadline: u64, waker: Ptr<u8>);
}
```

- `fd`: file descriptor 입니다.
- `interest`: `1` = 읽기, `2` = 쓰기 (둘을 OR 할 수 있음).
- `waker`: 원소 하나인 `GcBuf<Waker>` 입니다. 등록은 edge-triggered 이고 **한 번만** 깨웁니다 (가짜 깨움 가능). 이미 준비되어 있거나 감시할 수 없는 fd 면 곧바로 깨웁니다.
- `deadline`: `rok_time_now_ns` 시계 기준의 절대 시각 (ns) 입니다. 그 시각에 닿으면 waker 를 한 번 깨웁니다.

### `rok_io_*` — reactor (내부, 변경 가능)

[I/O 드라이버](../007-runtime/707-io-and-timer.md) 의 reactor 를 이루는 내부 심볼입니다. ABI 안정 대상이 아니며 바뀔 수 있습니다. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_io_poller() -> i32;                    // 프로세스의 kqueue / epoll fd (처음 부를 때 생성)
    fn rok_io_driver_claim() -> i32;              // 처음 부른 쪽만 1 — 그쪽이 reactor thread 를 띄움
    fn rok_io_watch(fd: i32) -> i32;              // fd 를 읽기 / 쓰기 edge-triggered 로 등록 (다시 걸어도 무해), 0 또는 -errno
    fn rok_io_poll(out: Ptr<u8>, max: usize) -> i32;   // 변화가 올 때까지 막힘 (GC 에는 park), 사건 수 또는 -errno
}
```

`rok_io_poll` 은 사건을 `GcBuf<i64>` 인 `out` 에 `fd * 2 + 방향` (읽기 0, 쓰기 1) 으로 최대 `max` 개 씁니다.

### `rok_net_*` — 소켓 helper (내부, 변경 가능)

표준 라이브러리 `rok::std::net` ([stdlib 3.2](../../refs-standard-libraries/300-std/302-net.md)) 을 뒷받침하는 내부 심볼입니다. ABI 안정 대상이 아니며 바뀔 수 있으므로 사용자 코드는 직접 부르지 않습니다. 소켓은 늘 non-blocking 으로 만들어지며, 실패는 `-errno` 로 돌려줍니다.

```
rok_net_socket     rok_net_bind       rok_net_listen     rok_net_connect
rok_net_sock_error rok_net_accept     rok_net_addr       rok_net_send
rok_net_send_to    rok_net_recv_from  rok_net_shutdown   rok_net_setopt
rok_net_multicast_v4 rok_net_wait     rok_net_resolve    rok_net_gai_strerror
```

`rok_net_wait` 는 blocking 호출이 준비를 기다리는 `poll(2)` 이며 (GC 에는 native 구간), `rok_net_resolve` 는 `getaddrinfo` 입니다.

## 8.5.6. 에러 전달 (심볼 없음)

[에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 `throw` / `try` / `catch` 에는 런타임 심볼이 없습니다. throwing 함수는 에러를 반환값의 일부로 돌려주고 ([8.6.5](806-calling-convention.md#865-throwing-함수의-반환-값-기반-에러-전달)), 호출 지점이 그 값을 검사해 분기합니다 ([8.9.6](809-lowering-control.md#896-try--catch--finally-값-기반-분기)). 따라서 에러를 던지거나 현재 에러를 꺼내기 위해 런타임을 호출하는 일은 없으며, 에러 처리를 위한 TLS 슬롯이나 jmp_buf 도 두지 않습니다.

## 8.5.7. Diagnostic 영역

### `rok_panic`

**Thread-safety: `thread-safe`** (어떤 thread 에서든 호출 가능, 호출 즉시 프로세스 종료).

```
extern "C" {
    fn rok_panic(msg: ConstPtr<u8>, msg_len: usize);   // 반환 안 함
}
```

회복 불가능한 panic 으로, fatal 입니다. 이 함수는 반환하지 않으며, 메시지를 stderr 에 출력한 뒤 프로세스를 종료 코드 `101` 로 종료시킵니다. `catch` 나 `finally` 는 실행되지 않습니다.

### `rok_panic_str`

**Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_panic_str(msg: str) -> !;          // msg 는 (ptr, len)
}
```

`rok_panic` 과 같으며 메시지를 `str` 로 받습니다. core 의 `panic(msg: str)` 과 컴파일러가 생성한 런타임 panic (경계 검사, overflow, 0 으로 나누기 등) 이 모두 이 심볼을 부릅니다.

### `rok_uncaught_error`

```
extern "rok" {
    fn rok_uncaught_error(what: str) -> !;    // 미처리 예외: what 을 stderr 에 출력하고 종료 코드 1
}
```

진입점 밖으로 전파된 에러를 처리합니다 ([4.16.4](../004-functions/416-entry-point.md#4164-종료-코드와-미처리-예외)). 선언은 core 에 있습니다.

### `rok_oom`

**Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_oom() -> !;
}
```

`fatal error: out of memory` 를 stderr 에 출력하고 종료 코드 `101` 로 종료합니다. 런타임 안의 할당 실패와 core 의 `oom()` 이 모두 이 심볼을 거칩니다. 런타임은 rok 함수를 이름으로 부를 수 없으므로, 할당 실패 시 `oom_fn` 슬롯의 함수를 부르지 않고 이 심볼을 직접 씁니다.

### 스택 오버플로

스레드의 스택이 넘치면 런타임은 stdout 을 (가능하면) flush 하고 stderr 에 다음을 출력한 뒤 abort (SIGABRT) 합니다. panic 과 달리 unwind 나 `finally` 실행은 없습니다.

```
thread '<name>' has overflowed its stack
```

`<name>` 은 `thread::Builder::name` 으로 준 스레드 이름입니다 (OS 스레드 이름으로도 설정됨, [7. 런타임](../007-runtime/README.md#런타임-오류-스택-오버플로)). main 스레드 (`rok_runtime_init`), `rok_thread_spawn` 스레드 (executor worker 포함, 깊은 `await` 사슬도), `rok_thread_init` 을 부른 C 스레드가 모두 대상입니다. 런타임은 각 스레드의 스택 아래 guard 범위를 TLS 슬롯 `rok_tls_stack_guard` ([8.3.1](803-tls-slots.md#831-tls-슬롯-구조)) 에 기록하고 대체 신호 스택 (`sigaltstack`) 을 둡니다. 그 범위 안의 SIGSEGV / SIGBUS 만 스택 오버플로로 보고, 그 밖의 fault 는 기본 동작을 따릅니다.

## 8.5.8. 심볼 명명 규약

심볼의 접두사별 영역은 다음 표와 같이 정리됩니다.

| Prefix          | 영역           |
|-----------------|----------------|
| `rok_gc_*`      | GC 관련        |
| `rok_runtime_*` | 런타임 일반    |
| `rok_thread_*`  | OS thread 관련 |
| `rok_coro_*`    | 코루틴 관련    |
| `rok_task_*`    | task 관련      |
| `rok_io_*`      | I/O            |
| `rok_timer_*`   | 타이머         |
| `rok_panic`     | 진단           |

이러한 접두사 규약을 통해 외부 라이브러리나 사용자 코드와의 심볼 충돌을 피합니다.

## 8.5.9. 함수 시그니처의 호환성

각 함수의 시그니처는 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 의 일부를 이룹니다. 변경 시의 정책은 다음과 같습니다.

- minor: 호환 가능한 변경만 허용되며, 인자 추가는 함수 wrapper 를 통해 처리합니다.
- major: 호환을 깨는 변경이 가능하며, 이때 ABI 버전 해시가 변환됩니다.

자세한 사항은 [개요](README.md) 의 안정성 정책을 참조합니다.

## 8.5.10. Calling convention

위 모든 함수의 calling convention 은 타깃 플랫폼의 C ABI 를 따릅니다. 예를 들어 System V x86_64, AAPCS aarch64, MS x64 Windows 등이 해당합니다.

## 8.5.11. 사용 예: TLAB fast path 의 slow 호출

```mlir
; LLVM IR (fast path 실패 시)
%cursor = load i8*, i8** @rok_tls_tlab_cursor
%new_cursor = getelementptr i8, i8* %cursor, i64 24
%limit = load i8*, i8** @rok_tls_tlab_limit
%check = icmp ugt i8* %new_cursor, %limit
br i1 %check, label %slow, label %fast

slow:
  %obj = call i8* @rok_gc_alloc_slow(i64 24, i64 8, i32 12345)
  br label %cont

fast:
  store i8* %new_cursor, i8** @rok_tls_tlab_cursor
  br label %cont

cont:
  %obj_phi = phi i8* [%obj_slow, %slow], [%cursor, %fast]
  ; ... 계속
```

위 예시는 fast path 와 slow path 가 결합된 LLVM IR 구조를 보여줍니다.

## 8.5.12. 보조 심볼 (부록, 비규범)

> [!IMPORTANT]
> 이 절은 **비규범 (non-normative)** 입니다. 아래 심볼은 참조 런타임 (`rokr`) 이 내보내는 보조 심볼입니다. ABI 본 표 (8.5.1 ~ 8.5.11) 의 일부가 아니며 안정성 약속이 없습니다.

시그니처는 본문과 같은 표기 (`Ptr<T>` = 가변 raw 포인터, `ConstPtr<T>` = 읽기 전용 raw 포인터) 를 따릅니다. 문자열 인자는 NUL 종료 C 문자열입니다.

```
extern "C" {
    // 메모리
    fn rok_free(ptr: Ptr<u8>);                             // null 무시
    fn rok_load_i32(ptr: ConstPtr<i32>) -> i32;
    fn rok_load_i64(ptr: ConstPtr<i64>) -> i64;
    fn rok_load_f64(ptr: ConstPtr<f64>) -> f64;
    fn rok_store_i32(ptr: Ptr<i32>, val: i32);
    fn rok_store_i64(ptr: Ptr<i64>, val: i64);
    fn rok_store_f64(ptr: Ptr<f64>, val: f64);

    // C 문자열
    fn rok_str_len(s: ConstPtr<u8>) -> usize;              // null → 0

    // GC
    fn rok_gc_collect_young();                             // minor 수집을 강제
```

다음 규칙이 적용됩니다.

- 사용자 코드와 표준 라이브러리는 이 심볼에 의존해서는 안 됩니다. 시그니처는 예고 없이 바뀌거나 제거될 수 있습니다.
- `rok_` prefix 를 공유하지만 8.5.8 의 영역 표에는 포함되지 않습니다. 이름 충돌을 피하기 위해 사양의 규범 심볼은 위 이름을 재사용하지 않습니다.
- `rok_panic` 은 이 부록에 속하지 않습니다. 시그니처는 [8.5.7](#857-diagnostic-영역) 의 `rok_panic(msg: ConstPtr<u8>, msg_len: usize)` 가 권위이며, 구현이 이와 다른 시그니처를 내보내는 것은 부적합 (non-conformance) 입니다.
