# 8.5. Extern "C" 심볼

`librokr.a`가 노출하는 [심볼](../000-meta/007-glossary.md#extern-c-심볼)의 시그니처를 정의합니다. 컴파일러가 생성한 코드와 표준 라이브러리가 이 심볼을 호출합니다.

Rok 쪽 선언은 런타임 ABI `extern "rok"`([5.9.2](../005-modules/509-extern-block.md#592-abi-명시))이며, toolchain이 `rok::core::rt`에 `pub(pkg)`로 둡니다. 아래 블록에서 `fn` 값은 `RokFn`(8.5.0a)이고, 그 밖의 값은 C ABI와 같습니다. `str`은 `struct { const uint8_t *ptr; size_t len; }`이며, `str`을 받거나 돌려주는 런타임 함수는 `RokFn`처럼 16바이트 구조체에 대한 플랫폼 C ABI를 따릅니다([8.6.3](806-calling-convention.md#863-struct-인자--반환-small-struct-rule)). ABI가 허용하면 레지스터로 값 전달하고, 레지스터와 스택에 나뉘지 않으며, Windows x64에서는 참조로 넘깁니다. 프로그램은 런타임 심볼을 직접 선언할 수 있습니다(`extern "rok"` / `extern "C"`). 그 선언은 여기서 정한 시그니처를 가져야 하며, 다르면 에러입니다.

> [!IMPORTANT]
> 모든 심볼은 `rok_` 접두사로 시작합니다. 외부 라이브러리나 사용자 코드와 심볼이 충돌하지 않게 하기 위해서입니다.

## 8.5.0. Thread-safety 분류

각 심볼 정의에는 **Thread-safety** 표기가 붙습니다. 이 표기는 컴파일러가 emit한 코드와 런타임 사이 contract의 일부이며, 어떤 호출 컨텍스트에서 안전하게 호출할 수 있는지 밝힙니다. 분류는 다음 6가지입니다.

| 분류                  | 의미                                                                         |
|-----------------------|------------------------------------------------------------------------------|
| `thread-safe`         | 여러 thread가 동시에 호출해도 안전 (런타임이 내부 동기화)                   |
| `caller-thread-only`  | 호출자 thread의 컨텍스트에 묶임. 다른 thread가 그 호출자를 대신할 수 없음   |
| `process-init`        | 프로세스 시작 시점, mt 진입 전 단 한 번                                      |
| `process-shutdown`    | 프로세스 종료 시점, mt 종료 후 단 한 번                                      |
| `per-thread-init`     | 각 worker thread가 시작할 때 그 thread가 자기 자신에 대해 한 번             |
| `per-thread-shutdown` | 각 worker thread가 끝날 때 그 thread가 자기 자신에 대해 한 번               |

`caller-thread-only`는 함수의 효과가 호출자의 TLS 슬롯이나 stack frame에 묶여 있어서, 다른 thread가 그 호출자를 대신할 수 없다는 뜻입니다. 함수가 reentrant하지 않다는 뜻은 아니며, 여러 thread가 각자 자기 자신을 위해 동시에 호출하는 것은 허용합니다.

## 8.5.0a. `RokFn` - 함수 값의 C ABI 형태

Rok 함수 값 `{code, env}`([4.4](../004-functions/404-function-type.md#함수-값의-표현))를 런타임 심볼에 넘길 때의 C 표현입니다.

```c
typedef struct {
    void *code;   // env 를 첫 인자로 받는 코드 포인터
    void *env;    // 캡처 환경 (이름 있는 함수는 NULL)
} RokFn;
```

`RokFn`은 C 구조체로 **값 전달**됩니다. 실제 레지스터 배치는 타깃 C ABI의 16바이트 구조체 규칙을 따릅니다.

## 8.5.1. GC 영역

### `rok_gc_alloc_slow`

[할당](../007-runtime/702-allocation.md)에서 다루는 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer)의 slow path입니다. **Thread-safety: `thread-safe`**.

```
extern "rok" {
    fn rok_gc_alloc_slow(size: usize, align: usize, type_id: u32) -> GcBuf<u8>;
}
```

인자와 반환값의 의미는 다음과 같습니다.

- `size`: 객체의 총 크기로, 16바이트 헤더에 payload를 더해 정렬한 값입니다.
- `align`: 객체의 정렬 요구입니다. 보통 8이며, `#[repr(align(N))]`를 지정하면 더 커집니다.
- `type_id`: 객체의 타입 ID입니다.
- 반환: 헤더 자리부터 시작하는 객체의 GC 참조(`GcBuf<u8>`, 런타임 ABI의 GC 포인터)를 돌려줍니다. type_id와 gc_flags는 함수가 채웁니다.

null을 반환하지 않습니다. 메모리 부족은 fatal(`rok_oom`)이며, `isize::MAX`를 넘는 요청은 fatal 에러 "allocation too large"(종료 코드 101)입니다.

### `rok_alloc_raw` / `rok_realloc_raw`

GC 밖의 raw 메모리를 할당합니다([stdlib 1.4 alloc](../../refs-standard-libraries/100-bare/104-alloc.md)). **Thread-safety: `thread-safe`**.

```
extern "rok" {
    fn rok_alloc_raw(size: usize, align: usize, zeroed: i32) -> usize;
    fn rok_realloc_raw(ptr: usize, old_size: usize, align: usize, new_size: usize) -> usize;
}
```

`zeroed`가 0이 아니면 할당한 메모리를 0으로 채웁니다. `rok_alloc_raw`는 `size`가 0이거나, `align`이 2의 거듭제곱이 아니거나, 할당에 실패하면 null(`0`)을 반환하며 abort하지 않습니다. `rok_realloc_raw`도 실패하면 null을 반환합니다. 실패 처리는 호출자(`rok::bare::alloc`)가 맡습니다.

### `rok_gc_wb_slow`

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md)의 write barrier slow path입니다. **Thread-safety: `caller-thread-only`**(TLS의 wb log buffer에 기록).

```
extern "rok" {
    fn rok_gc_wb_slow(obj: Ptr<u8>);
}
```

인자 `obj`는 변경된 슬롯이 속한 객체의 포인터이며, 반환값은 없습니다. remembered set에 obj를 추가하고, log가 차면 flush합니다.

### `rok_gc_register_type`

[ScanTable](802-scan-table.md)의 타입을 등록합니다. **Thread-safety: `process-init`**.

```
extern "rok" {
    fn rok_gc_register_type(type_id: u32, scan_table: ConstPtr<u8>);
}
```

프로세스 시작 시, mt 진입 전에 호출하므로 동기화가 필요 없습니다.

### `rok_gc_pin` / `rok_gc_unpin`

[FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 객체를 pin합니다([객체 레이아웃과 메모리 공간](../007-runtime/701-object-layout.md), [`extern` 블록 (FFI 선언)](../005-modules/509-extern-block.md) 참고). **Thread-safety: `thread-safe`**.

```
extern "rok" {
    fn rok_gc_pin(obj: Ptr<u8>) -> Ptr<u8>;       // 고정된 객체 (같은 주소) 반환
    fn rok_gc_unpin(obj: Ptr<u8>);
}
```

`extern "C"` 함수나 blocking 런타임 호출에 넘긴 GC 포인터는 호출이 끝날 때까지 인자가 루트에 있으므로, 따로 pin하지 않아도 옮겨지지 않습니다([7.4.1](../007-runtime/704-barriers-and-roots.md#루트가-가리키는-객체는-옮기지-않음)). C 코드가 호출 뒤에도 포인터를 보관할 때만 pin이 필요합니다. pin은 객체에 `PINNED`를 켜서 evacuation 대상에서 빼고 GC가 살려 두게 하며, unpin은 이를 풉니다.

### `rok_gc_collect`

명시적인 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 트리거이며, 드물게 씁니다. **Thread-safety: `thread-safe`**.

```
extern "rok" {
    fn rok_gc_collect();
}
```

full 수집을 하고, 수집이 끝나면 돌아옵니다.

### `rok_gc_secret_new` / `rok_gc_secret_wipe`

`rok::core::crypto::Secret`([stdlib 2.21.5](../../refs-standard-libraries/200-core/221-crypto.md#2215-보안-고려-사항))의 비밀 저장소입니다. **Thread-safety: `thread-safe`**.

```
extern "rok" {
    fn rok_gc_secret_new(len: usize) -> GcBuf<u8>;
    fn rok_gc_secret_wipe(obj: GcBuf<u8>);
}
```

`rok_gc_secret_new`는 `WIPE`([8.1.3](801-object-header-abi.md#813-gc_flags-offset-4-u32))를 켠 type id 1(참조 없음)의 LOS 객체를 할당합니다. 0으로 채운 payload는 `len`을 8의 배수로 올린 만큼의 바이트를 담습니다. 할당 전에 수집할 수 있으며, `isize::MAX`를 넘는 요청은 fatal error "allocation too large"입니다. `rok_gc_secret_wipe`는 payload 전체를 volatile store로 덮어쓰고 compiler fence를 둡니다. 할당도 block도 하지 않으므로 safepoint가 아닙니다([8.4](804-safepoint.md)). 수집기는 객체를 해제하기 전에 같은 방식으로 payload를 덮어씁니다.

## 8.5.2. Runtime 영역

### `rok_runtime_init`

런타임을 초기화합니다. **Thread-safety: `process-init`**.

```
extern "rok" {
    fn rok_runtime_init();
}
```

프로세스 시작 직후, `main()` 전에 호출합니다. worker 풀을 만들고, GC 공간을 mmap하고, I/O driver를 초기화합니다. 컴파일러가 생성한 C-ABI `main`은 이 호출 직후 `rok.register_types`를 호출합니다([8.2.13](802-scan-table.md#8213-등록-시점)).

런타임은 이 밖에 thread-local 블록 `rok_tls`를 export합니다([8.3.8](803-tls-slots.md#838-컴파일러-emit-의-형식)).

### `rok_runtime_shutdown`

런타임을 종료합니다. **Thread-safety: `process-shutdown`**.

```
extern "rok" {
    fn rok_runtime_shutdown();
}
```

`main()`이 끝난 직후 호출합니다. 다른 모든 Rok 스레드는 지금 실행 중인 런타임 / C 코드를 마친 뒤 다음 safepoint에서 영구히 멈춥니다. 그다음 worker 풀을 종료하고 남은 자원을 정리합니다.

### `rok_exit`

프로세스를 즉시 종료합니다. **Thread-safety: `thread-safe`**.

```
extern "rok" {
    fn rok_exit(code: i32) -> !;
}
```

[`rok::std::env::exit`](../../refs-standard-libraries/300-std/306-env.md)가 이 심볼을 호출합니다. `finally`와 `Drop`은 실행하지 않습니다([4.16.4](../004-functions/416-entry-point.md#4164-종료-코드와-미처리-예외)).

### 문자열 helper

`FromStr`과 `str::bytes`를 뒷받침하는 내부 helper입니다([stdlib 2.15](../../refs-standard-libraries/200-core/215-string.md)). **Thread-safety: `thread-safe`**. `str` 인자는 머리말에서 정한 구조체 형태입니다.

```
extern "rok" {
    fn rok_str_eq(a: str, b: str) -> i32;                            // 같으면 1, 아니면 0
    fn rok_str_byte(s: str, i: usize) -> u8;                         // i 번째 바이트 (경계 검사는 호출자)
    fn rok_str_slice(s: str, start: usize, end: usize, at: str) -> str;  // s[start:end]. 범위 / UTF-8 경계가 틀리면 at 위치로 panic
    fn rok_str_parse_int(s: str, radix: u32, signed: i32, bits: u32, out: GcBuf<i64>) -> i32;
                                        // 0 ok, 1 빈 문자열, 2 잘못된 자릿수, 3 범위 밖; 값의 비트는 out[0]
    fn rok_str_parse_float(s: str, out: GcBuf<f64>) -> i32;           // 0 ok, 2 잘못됨, 3 범위 초과 (유한한 숫자가 타입 범위를 넘음); 값은 out[0]
}
```

### `rok_runtime_yield`

[Safepoint 프로토콜](804-safepoint.md)의 `yield` 함수, 곧 safepoint poll의 slow path입니다. **Thread-safety: `caller-thread-only`**(호출자 mutator가 자기 yield_flag를 polling한 결과로 호출).

```
extern "rok" {
    fn rok_runtime_yield();
}
```

### `rok_enter_native` / `rok_leave_native`

native bracket의 slow path입니다([8.4.10](804-safepoint.md#8410-native-호출)). **Thread-safety: `caller-thread-only`**.

```
extern "rok" {
    fn rok_enter_native();                // C 로 들어감: 이 스레드는 PARKED
    fn rok_leave_native();                // C 에서 돌아옴: 진행 중인 수집을 기다림
}
```

컴파일러는 모든 `extern "C"` 호출 앞뒤에 fast path를 inline으로 냅니다. mutator 상태에 `PARKED` / `RUNNING`을 seq_cst로 쓰고 yield flag를 확인합니다. flag가 켜져 있을 때만 이 함수들을 호출합니다.

### `rok_native_reenter` / `rok_native_resume`

모든 `extern "C" fn` 정의의 입구와 출구이며, C callback이 mutator로 실행되게 합니다. **Thread-safety: `caller-thread-only`**.

```
extern "rok" {
    fn rok_native_reenter() -> u32;       // 스레드가 PARKED (C 안) 였으면 native 를 떠남; 그랬는지 반환
    fn rok_native_resume(parked: u32);    // reenter 가 떠났으면 다시 native 로
}
```

### `rok_nosp_enter` / `rok_nosp_leave`

`#[no_safepoint]` 런타임 함수 호출을 감싸는 검사용 bracket입니다([8.4.10](804-safepoint.md#8410-native-호출)). debug 빌드의 rokc만 냅니다. **Thread-safety: `caller-thread-only`**.

```
extern "rok" {
    fn rok_nosp_enter();                  // 이 스레드의 no_safepoint 깊이를 올림 (8.3.1)
    fn rok_nosp_leave();                  // 내림
}
```

깊이가 0이 아닌 동안 스레드가 수집을 위해 멈추려 하면 debug 런타임은 fatal 에러로 프로세스를 끝냅니다. release 런타임에서는 두 함수 모두 아무것도 하지 않습니다.

### `rok_thread_init` / `rok_thread_shutdown`

worker thread의 lifecycle을 담당합니다. **Thread-safety: `rok_thread_init`은 `per-thread-init`, `rok_thread_shutdown`은 `per-thread-shutdown`**.

```
extern "rok" {
    fn rok_thread_init();
    fn rok_thread_shutdown();
}
```

각 worker thread가 시작하고 끝날 때 호출되어 [TLS 슬롯](803-tls-slots.md)의 TLS를 초기화하거나 정리합니다.

### 내부 심볼 무리

다음 무리는 참조 런타임의 내부 helper이며 ABI 계약에 속하지 않습니다. `rok_fmt_*`, `rok_str_*` helper, `rok_char_*`, `rok_nat_*`, `rok_stdio_*`, `rok_time_*`, `rok_gc_stats_*`, `rok_buf_*`, `rok_cstr_len`.

### safepoint 가 아닌 함수

Rok 쪽에서는 `#[no_safepoint]`가 붙은 `extern "rok"` 블록에 선언합니다([5.9.2](../005-modules/509-extern-block.md#592-abi-명시)).

다음 런타임 함수는 할당도 block도 하지 않음이 보장되므로, 이들의 호출은 safepoint가 아닙니다([8.4.2](804-safepoint.md#842-emit-위치), [8.9.12](809-lowering-control.md#8912-shadow-stack-frame)). `rok_str_len`, `rok_str_byte`, `rok_str_cmp`, `rok_str_eq`, `rok_str_eq_at`, `rok_str_find`, `rok_str_rfind`, `rok_str_char_count`, `rok_str_hash`, `rok_str_bytes_buf`, `rok_buf_copy`, `rok_char_*` 속성 / 대소문자 함수, `rok_fmt_len`, `rok_f64_*`, `rok_f32_*`.

## 8.5.3. Coroutine 영역

### `rok_coro_frame_alloc`

[코루틴](../007-runtime/708-coroutine.md)의 frame을 할당합니다. **Thread-safety: `thread-safe`**.

```
extern "rok" {
    fn rok_coro_frame_alloc(size: usize, type_id: u32) -> GcBuf<u8>;
}
```

`rok_gc_alloc_slow`와 거의 같지만, frame 전용 처리를 따로 할 수 있습니다.

### `rok_coro_park` / `rok_coro_unpark`

컴파일러는 이 두 심볼을 emit하지 않습니다(`await`는 `rok::core::task`의 parking registry를 씁니다). 런타임은 C나 저수준 코드를 위해 이 심볼을 제공합니다. `frame`(아무 GC 객체)의 주소가 parking key이며, `rok_coro_unpark(frame)`은 그 key에 park한 waker를 모두 깨웁니다. task executor 밖에서 호출하면 abort합니다.

**Thread-safety: `rok_coro_park`는 `caller-thread-only`**(호출한 코루틴 자신을 park), **`rok_coro_unpark`는 `thread-safe`**(다른 thread에서 호출 가능).

```
extern "rok" {
    fn rok_coro_park(frame: Ptr<u8>, waker: Ptr<u8>);
    fn rok_coro_unpark(frame: Ptr<u8>);
}
```

`rok_coro_park`는 코루틴을 일시 중단하고 waker를 등록합니다. `rok_coro_unpark`는 코루틴을 task 큐에 push해서 재개합니다.

여기서 `waker`는 깨우는 함수와 컨텍스트의 묶음, 곧 런타임의 `Waker` 객체([2.20.2](../../refs-standard-libraries/200-core/220-future.md))를 가리키는 **불투명 포인터**(`Ptr<u8>`)입니다. C ABI 경계에서는 `Waker`의 내부 표현을 노출하지 않으므로, [7.8.8](../007-runtime/708-coroutine.md)을 비롯한 모든 곳에서 `waker` 인자의 타입은 `Ptr<u8>`입니다(`rok_io_register`, `rok_timer_register`도 같음). 포인터가 가리키는 객체의 정확한 레이아웃은 코루틴 구현이 정합니다.

## 8.5.4. Scheduler 영역

task 런타임(spawn, join, cancel, executor)은 `rok::core::task`의 Rok 코드입니다([stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)). 런타임은 아래의 executor 슬롯, 스레드, parking 심볼만 제공합니다.

### `rok_task_worker_*` / `rok_time_now_ns`

현재 스레드의 executor worker([stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md))를 TLS 슬롯 `rok_tls_task_worker`([8.3.1](803-tls-slots.md#831-tls-슬롯-구조))에 넣고 꺼내는 함수, 그리고 시계입니다. **Thread-safety: `caller-thread-only`**(`rok_time_now_ns`는 `thread-safe`).

```
extern "rok" {
    fn rok_task_worker_present() -> i32;       // 이 스레드가 worker 이면 1
    fn rok_task_worker_get() -> usize;         // worker 객체 주소, 없으면 0
    fn rok_task_worker_set(w: usize);          // 살아 있는 GC 객체 또는 0. 설치하는 동안 pin
    fn rok_task_worker_clear();                // = rok_task_worker_set(0)
    fn rok_time_now_ns() -> i64;               // monotonic clock (ns)
}
```

### `rok_thread_spawn`

1:1 OS thread를 만듭니다. **Thread-safety: `thread-safe`**.

```
extern "rok" {
    fn rok_thread_spawn(f: RokFn, stack_size: usize, id: u64) -> usize;
    fn rok_thread_new_id() -> u64;
    fn rok_thread_id() -> u64;                        // 현재 스레드의 worker id
    fn rok_thread_sleep_ns(ns: i64);
    fn rok_thread_yield_now();
}
```

`rok_thread_spawn`은 Rok 클로저 `f`(아래 `RokFn`, 값 전달)를 OS 스레드에서 실행합니다(`stack_size` 0은 기본값). `id`는 `rok_thread_new_id`로 받은 값이며, 새 스레드의 `rok_tls_worker_id`가 됩니다. 반환값은 join handle이고, 생성에 실패하면 0입니다. 스레드가 끝날 때까지 `env`는 GC root입니다.

### `rok_task_hooks_install` / `rok_task_key`

`rok::core::task`의 executor가 런타임에 hook을 설치합니다. 런타임의 I/O driver, timer, `rok_coro_park` / `rok_coro_unpark`는 이 hook으로 task를 깨우고 멈춥니다. **Thread-safety: `thread-safe`**.

```
extern "rok" {
    fn rok_task_hooks_install(
        io: extern "C" fn(fd: i32, interest: u32, waker: Ptr<u8>),
        timer: extern "C" fn(deadline_ns: u64, waker: Ptr<u8>),
        park: extern "C" fn(key: Ptr<u8>, waker: Ptr<u8>),
        unpark: extern "C" fn(key: Ptr<u8>),
        restart: extern "C" fn(),               // 실패한 REPL 셀과 함께 사라진 executor 작업을 재시작
    );                                          // 처음 설치한 것이 이김
    unsafe fn rok_task_key(obj: Ptr<u8>) -> usize;   // GC 객체 주소를 park 키로
}
```

`rok_task_key`는 주소를 키로 쓰므로 그 객체에 `NOMOVE`([8.1.3](801-object-header-abi.md#813-gc_flags-offset-4-u32))를 켭니다. 그 객체는 그 뒤로 옮겨지지 않습니다. 호출 안에서만 쓰는 동일성 비교에는 `buf_same`([stdlib 2.9](../../refs-standard-libraries/200-core/209-collections.md))을 쓰며, 이 함수는 아무것도 고정하지 않습니다.

### `rok_sync_wait` / `rok_sync_wake`

동기화 primitive(`Mutex`, `Condvar`, `Once`)의 대기와 깨우기입니다. **Thread-safety: `thread-safe`**.

```
extern "rok" {
    fn rok_sync_wait(buf: Ptr<u8>, index: usize, expected: i32, timeout_ns: i64);   // GcBuf<i32> 원소가 expected 인 동안 대기. timeout_ns < 0 이면 무한
    fn rok_sync_wake(buf: Ptr<u8>, index: usize);                                    // 그 원소에서 대기 중인 스레드를 모두 깨움
}
```

`rok_sync_wait`는 깨울 이유가 없어도 돌아올 수 있으므로 호출자가 조건을 다시 확인합니다. 대기하는 동안 그 스레드는 GC에 대해 정지한 것으로 봅니다.

### `rok_thread_join`

**Thread-safety: `thread-safe`**.

```
extern "rok" {
    fn rok_thread_join(handle: usize) -> i32;          // 0 이면 성공
}
```

### `rok_thread_detach` / `rok_cpu_count`

**Thread-safety: `thread-safe`**.

```
extern "rok" {
    fn rok_thread_detach(handle: usize) -> i32;   // 끝나면 OS 자원 반환, 더 이상 join 불가. 0 이면 성공
    fn rok_cpu_count() -> usize;                  // 온라인 CPU 수 (최소 1), par 풀의 크기
}
```

### `rok_static_init_begin` / `rok_static_init_end`

초기값이 `const`가 아닌 `static`의 지연 초기화([5.3](../005-modules/503-static.md))를 뒷받침합니다. **Thread-safety: `thread-safe`**.

```
extern "rok" {
    fn rok_static_init_begin(state: Ptr<u8>, at: ConstPtr<u8>, at_len: usize) -> i32;   // 1: 호출자가 초기화식 실행, 0: 이미 준비됨
    fn rok_static_init_end(state: Ptr<u8>);            // 초기화 완료 공개
}
```

`state`는 그 `static`의 상태 바이트(0 새로, 1 진행 중, 2 준비됨)입니다. 다른 스레드가 초기화 중이면 `rok_static_init_begin`은 끝날 때까지 기다리며, 그동안 이 스레드는 GC에 대해 정지한 것으로 봅니다. `at` / `at_len`은 그 static을 읽는 코드의 `path:line:col`이며, 자기 static을 읽는 초기화식은 그 자리에서 panic합니다([4.14.13](../004-functions/414-throws.md#41413-panic-과-fatal-에러)).

### `rok_gc_add_root`

```
extern "rok" {
    fn rok_gc_add_root(word: ConstPtr<usize>);   // static 의 참조 word 를 프로세스 끝까지 GC root 로 등록
}
```

컴파일러가 만든 시작 코드가 GC 참조를 담는 모든 `static`에 대해 호출합니다.

### `rok_os_*` - OS helper (내부, 변경 가능)

표준 라이브러리 `rok::std`(fs, io, env, process)를 뒷받침하는 내부 심볼입니다. ABI 안정성 대상이 아니며 바뀔 수 있으므로 사용자 코드는 직접 호출하지 않습니다.

```
rok_os_errno_kind     rok_os_strerror       rok_os_open           rok_os_close
rok_os_read           rok_os_write          rok_os_poll_read      rok_os_fsync
rok_os_ftruncate      rok_os_lseek          rok_os_dup            rok_os_fchmod
rok_os_mmap           rok_os_munmap         rok_os_mmap_copy      rok_os_flock
rok_os_copy_fd        rok_os_clone          rok_os_stat           rok_os_mkdir
rok_os_unlink         rok_os_rename         rok_os_link           rok_os_opendir
rok_os_readdir        rok_os_closedir       rok_os_getcwd         rok_os_chdir
rok_os_realpath       rok_os_current_exe    rok_os_argc           rok_os_arg
rok_os_env_at         rok_os_getenv         rok_os_setenv         rok_os_spawn
rok_os_waitpid        rok_os_signal_install rok_os_raise          rok_os_kill
rok_os_pidfd_open     rok_os_pidfd_signal   rok_os_getpid         rok_os_getppid
rok_os_abort          rok_os_random         rok_os_trust_roots
```

### `rok_stdio_*` / `rok_stdin_read` - 표준 스트림 (내부, 변경 가능)

`print` / `eprint`와 표준 입력을 뒷받침하는 내부 심볼입니다. ABI 안정성 대상이 아니며 바뀔 수 있습니다. 바이트 영역은 `GcBuf<u8>`의 `(buf, off, len)`이며, 실패는 `-errno`로 돌려줍니다.

```
extern "rok" {
    fn rok_stdio_write(fd: i32, s: str, newline: i32) -> i32;                       // s (newline != 0 이면 이어서 "\n") 를 fd 1 또는 2 로; 0 또는 -errno
    unsafe fn rok_stdio_write_buf(fd: i32, buf: Ptr<u8>, off: usize, len: usize) -> i32;   // buf 의 off 부터 len 바이트를 fd 1 또는 2 로
    fn rok_stdio_flush() -> i32;                                                     // stdout 버퍼를 내보냄; 0 또는 -errno
    unsafe fn rok_stdin_read(buf: Ptr<u8>, off: usize, len: usize, delim: i32) -> isize;
        // stdin 에서 최대 len 바이트를 buf 의 off 위치로 읽고, 첫 delim 바이트 뒤에서 멈춤 (delim < 0: 멈추지 않음);
        // 읽은 수 (입력 끝이면 0) 또는 -errno
}
```

### `rok_deque_*` - work-stealing deque

task 스케줄러([7.6.2](../007-runtime/706-scheduler.md#762-work-stealing-구조))의 lock-free Chase-Lev deque입니다. deque와 그 ring은 GC가 추적하는 GC 객체이므로, deque 안에만 있는 원소도 살아 있습니다. **Thread-safety:** `push` / `pop`은 소유 thread만, `steal`과 `len`은 어느 thread든 호출할 수 있습니다.

```
extern "rok" {
    fn rok_deque_new() -> GcBuf<u8>;                                 // 빈 deque (GC 객체)
    unsafe fn rok_deque_push(d: Ptr<u8>, item: usize);               // 소유자: 끝에 넣음 (GC 참조)
    unsafe fn rok_deque_pop(d: Ptr<u8>, out: Ptr<u8>, index: usize) -> i32;    // 소유자: 가장 새 원소를 out[index] 에, 있으면 1
    unsafe fn rok_deque_steal(d: Ptr<u8>, out: Ptr<u8>, index: usize) -> i32;  // 가장 오래된 원소를 out[index] 에, 있으면 1
    unsafe fn rok_deque_len(d: Ptr<u8>) -> usize;                    // 원소 수 (경합 중의 근사값)
}
```

`out`은 GC 참조를 담는 `GcBuf`이며, 결과를 `index` 자리에 씁니다. 이 함수들 안에는 safepoint가 없습니다.

## 8.5.5. I/O / Timer 영역

### `rok_io_register` / `rok_timer_register`

C나 저수준 코드가 fd가 준비될 때나 특정 시각에 task를 깨우도록 등록합니다. 내부적으로는 reactor([7.7](../007-runtime/707-io-and-timer.md))와 executor의 timer를 씁니다. task executor 밖에서 호출하면 abort합니다.

**Thread-safety: `thread-safe`**.

```
extern "rok" {
    fn rok_io_register(fd: i32, interest: u32, waker: Ptr<u8>);
    fn rok_timer_register(deadline: u64, waker: Ptr<u8>);
}
```

- `fd`: file descriptor입니다.
- `interest`: `1` = 읽기, `2` = 쓰기(둘을 OR할 수 있음).
- `waker`: 원소가 하나인 `GcBuf<Waker>`입니다. 등록은 edge-triggered이고 **한 번만** 깨웁니다(가짜 깨움 가능). fd가 이미 준비되어 있거나 감시할 수 없으면 곧바로 깨웁니다.
- `deadline`: `rok_time_now_ns` 시계 기준의 절대 시각(ns)입니다. 그 시각이 되면 waker를 한 번 깨웁니다.

### `rok_io_*` - reactor (내부, 변경 가능)

[I/O 드라이버](../007-runtime/707-io-and-timer.md)의 reactor를 이루는 내부 심볼입니다. ABI 안정성 대상이 아니며 바뀔 수 있습니다. **Thread-safety: `thread-safe`**.

```
extern "rok" {
    fn rok_io_poller() -> i32;                    // 프로세스의 kqueue / epoll fd (처음 부를 때 생성)
    fn rok_io_watch(fd: i32, interest: u32) -> i32;   // fd 를 edge-triggered 로 등록 (다시 걸어도 무해); interest 1 읽기, 2 쓰기; 0 또는 -errno
    fn rok_io_poll(out: Ptr<u8>, max: usize) -> i32;   // 변화가 올 때까지 막힘 (GC 에는 park), 사건 수 또는 -errno
    fn rok_io_armed() -> i32;                     // WASI 전용: 걸린 watch 가 있으면 1
    fn rok_io_wait(out: Ptr<u8>, max: usize, timeout_ns: i64) -> i32;   // WASI 전용: 시간 제한이 있는 rok_io_poll (< 0: 제한 없음); 0 = 시간 초과
}
```

`rok_io_poll`은 사건을 `GcBuf<i64>`인 `out`에 `fd * 2 + 방향`(읽기 0, 쓰기 1) 형태로 최대 `max`개 씁니다.

### `rok_net_*` - 소켓 helper (내부, 변경 가능)

표준 라이브러리 `rok::std::net`([stdlib 3.2](../../refs-standard-libraries/300-std/302-net.md))을 뒷받침하는 내부 심볼입니다. ABI 안정성 대상이 아니며 바뀔 수 있으므로 사용자 코드는 직접 호출하지 않습니다. 소켓은 항상 non-blocking으로 만들며, 실패는 `-errno`로 돌려줍니다.

```
rok_net_socket        rok_net_bind          rok_net_listen        rok_net_connect
rok_net_sock_error    rok_net_accept        rok_net_addr          rok_net_listeners
rok_net_send          rok_net_recv          rok_net_send_to       rok_net_recv_from
rok_net_shutdown      rok_net_setopt        rok_net_multicast_v4  rok_net_wait
rok_net_resolve       rok_net_unix          rok_net_unix_pair     rok_net_unix_path
rok_net_send_fd       rok_net_recv_fd       rok_net_sendfile      rok_net_send_zerocopy
rok_net_splice
```

`rok_net_wait`는 blocking 호출이 준비를 기다리는 `poll(2)`이며(GC에는 native 구간), `rok_net_resolve`는 `getaddrinfo`입니다. 그 실패 코드 `e`는 `-(100000 + |e|)`로 돌려주며, `rok_os_strerror`와 `rok_os_errno_kind`는 이 값을 errno처럼 받습니다. `rok_net_unix*`, `rok_net_send_fd` / `rok_net_recv_fd`, `rok_net_sendfile`, `rok_net_send_zerocopy`, `rok_net_splice`는 Windows에서 export하지 않습니다.

## 8.5.6. 에러 전달 (심볼 없음)

[에러 제어 흐름](../003-expressions/314-error-control-flow.md)의 `throw` / `try` / `catch`에는 런타임 심볼이 없습니다. throwing 함수는 에러를 반환값의 일부로 돌려주고([8.6.5](806-calling-convention.md#865-throwing-함수의-반환-값-기반-에러-전달)), 호출 지점이 그 값을 검사해 분기합니다([8.9.6](809-lowering-control.md#896-try--catch--finally-값-기반-분기)). 따라서 에러를 던지거나 현재 에러를 꺼내려고 런타임을 호출하는 일은 없으며, 에러 처리용 TLS 슬롯이나 jmp_buf도 없습니다.

## 8.5.7. Diagnostic 영역

아래 종료 코드(`101` 등)는 `wasm32-wasi`에서 host에 단순한 실패로 전달됩니다([4.16.4](../004-functions/416-entry-point.md#4164-종료-코드와-미처리-예외)).

### `rok_panic`

**Thread-safety: `thread-safe`**(어느 thread에서든 호출 가능, 호출하면 즉시 프로세스 종료).

```
extern "rok" {
    fn rok_panic(msg: ConstPtr<u8>, msg_len: usize);   // 반환 안 함
}
```

복구할 수 없는 panic이며, fatal입니다. 이 함수는 반환하지 않습니다. 메시지를 stderr에 출력한 뒤 종료 코드 `101`로 프로세스를 종료합니다. `catch`나 `finally`는 실행되지 않습니다.

### `rok_panic_str`

**Thread-safety: `thread-safe`**.

```
extern "rok" {
    fn rok_panic_str(msg: str) -> !;          // msg 는 (ptr, len)
}
```

`rok_panic`과 같지만 메시지를 `str`로 받습니다. 위치를 출력하지 않으며, 런타임 내부 panic에만 씁니다.

### `rok_panic_at`

**Thread-safety: `thread-safe`**.

```
extern "rok" {
    #[track_caller]
    fn rok_panic_at(msg: str) -> !;
}
```

선언이 `#[track_caller]`([6.2.12](../006-meta-system/602-built-in-attributes.md#6212-track_caller-호출-위치-전달))이므로, 심볼은 호출자의 위치를 숨은 마지막 `str` 인자(`"path:line:col"`)로 받습니다: `rok_panic_at(msg, at)`.

[4.14.13](../004-functions/414-throws.md#41413-panic-과-fatal-에러)의 형식으로 출력하고 종료 코드 `101`로 끝냅니다. core의 `panic`과 컴파일러가 생성한 panic(경계 검사, overflow, 0으로 나누기 등)이 이 심볼을 호출합니다.

### `rok_uncaught_error` / `rok_uncaught_error_at`

```
extern "rok" {
    fn rok_uncaught_error(what: str) -> !;              // 미처리 예외: what 을 stderr 에 출력하고 종료 코드 1
    fn rok_uncaught_error_at(what: str, at: str) -> !;  // 그 뒤에 "  --> <at>" 줄
}
```

진입점 밖으로 전파된 에러를 처리합니다([4.16.4](../004-functions/416-entry-point.md#4164-종료-코드와-미처리-예외)). 선언은 core에 있습니다.

### `rok_oom`

**Thread-safety: `thread-safe`**.

```
extern "rok" {
    fn rok_oom() -> !;
}
```

`fatal error: out of memory`를 stderr에 출력하고 종료 코드 `101`로 종료합니다. 런타임 안의 할당 실패와 core의 `oom()` 모두 이 심볼을 거칩니다. 런타임은 Rok 함수를 이름으로 호출할 수 없으므로, 할당에 실패하면 `oom_fn` 슬롯의 함수를 호출하지 않고 이 심볼을 직접 씁니다.

### 스택 오버플로

스레드의 스택이 넘치면 런타임은 (가능하면) stdout을 flush하고, stderr에 다음을 출력한 뒤 abort(SIGABRT)합니다. panic과 달리 unwind나 `finally` 실행은 없습니다.

```
thread '<name>' has overflowed its stack
```

`<name>`은 `thread::Builder::name`으로 준 스레드 이름입니다(OS 스레드 이름으로도 설정됨, [7. 런타임](../007-runtime/README.md#런타임-오류-스택-오버플로)). main 스레드(`rok_runtime_init`), `rok_thread_spawn` 스레드(executor worker 포함, 깊은 `await` 사슬도), `rok_thread_init`을 호출한 C 스레드가 모두 대상입니다. 런타임은 각 스레드의 스택 아래 guard 범위를 TLS 슬롯 `rok_tls_stack_guard`([8.3.1](803-tls-slots.md#831-tls-슬롯-구조))에 기록하고, 대체 신호 스택(`sigaltstack`)을 둡니다. 그 범위 안의 SIGSEGV / SIGBUS만 스택 오버플로로 보고하고, 그 밖의 fault는 기본 동작을 따릅니다. wasm에서는 모듈이 trap을 잡을 수 없으며 host가 보고합니다(함께 제공되는 JS host는 위 메시지를 출력).

## 8.5.8. 심볼 명명 규약

접두사별 영역은 다음과 같습니다.

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

이 접두사 규약으로 외부 라이브러리나 사용자 코드와의 심볼 충돌을 피합니다. 8.5.1 ~ 8.5.7의 표가 규범적인 런타임 심볼이며, 런타임은 구현이 정하는 `rok_*` 심볼(예: `rok_fmt_*`, `rok_os_*`, `rok_stdio_*`, `rok_static_init_*`)을 더 내보낼 수 있습니다.

## 8.5.9. 함수 시그니처의 호환성

각 함수의 시그니처는 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface)의 일부입니다. 변경 정책은 다음과 같습니다.

- minor: 호환되는 변경만 허용하며, 인자 추가는 함수 wrapper로 처리합니다.
- major: 호환성을 깨는 변경이 가능하며, 이때 ABI 버전 해시가 바뀝니다.

자세한 내용은 [개요](README.md)의 안정성 정책을 참고하세요.

## 8.5.10. Calling convention

위 모든 함수의 calling convention은 타깃 플랫폼의 C ABI(예: System V x86_64, AAPCS aarch64, MS x64 Windows)를 따릅니다.

## 8.5.11. 사용 예: TLAB fast path 의 slow 호출

```mlir
; LLVM IR (fast path 실패 시)
%t = call ptr @llvm.threadlocal.address(ptr @rok_tls)
%cursor_p = getelementptr i8, ptr %t, i64 8          ; tlab_cursor at p
%cursor = load i8*, i8** %cursor_p
%new_cursor = getelementptr i8, i8* %cursor, i64 24
%limit_p = getelementptr i8, ptr %t, i64 16          ; tlab_limit at 2p
%limit = load i8*, i8** %limit_p
%check = icmp ugt i8* %new_cursor, %limit
br i1 %check, label %slow, label %fast

slow:
  %obj = call i8* @rok_gc_alloc_slow(i64 24, i64 8, i32 12345)
  br label %cont

fast:
  store i8* %new_cursor, i8** %cursor_p
  br label %cont

cont:
  %obj_phi = phi i8* [%obj_slow, %slow], [%cursor, %fast]
  ; ... 계속
```

위 예시는 fast path와 slow path를 결합한 LLVM IR 구조를 보여 줍니다.

## 8.5.12. 보조 심볼 (부록, 비규범)

> [!IMPORTANT]
> 이 절은 **비규범**(non-normative)입니다. 아래 심볼은 참조 런타임(`rokr`)이 내보내는 보조 심볼입니다. ABI 본 표(8.5.1 ~ 8.5.11)에 속하지 않으며 안정성 약속도 없습니다. 런타임은 구현이 정하는 `rok_*` 심볼을 더 내보낼 수 있습니다(8.5.8).

시그니처는 본문과 같은 표기(`Ptr<T>` = 가변 raw 포인터, `ConstPtr<T>` = 읽기 전용 raw 포인터)를 따릅니다. 문자열 인자는 NUL로 끝나는 C 문자열입니다.

```
extern "rok" {
    // 메모리
    fn rok_free(ptr: Ptr<u8>);                             // null 무시

    // C 문자열
    fn rok_cstr_len(s: ConstPtr<u8>) -> usize;             // strlen; null → 0

    // GC
    fn rok_gc_collect_young();                             // minor 수집을 강제
    fn rok_gc_weak_new(target: Ptr<u8>) -> Ptr<u8>;       // target 을 가리키는 weak cell (type id 3)
    fn rok_gc_weak_get(cell: Ptr<u8>) -> Ptr<u8>;         // 대상, 해제되었으면 null

    // 프로세스 종료
    fn rok_set_exit_hook(hook: extern "C" fn(i32) | null);
        // panic, 처리되지 않은 에러, exit, (Windows) stack overflow 가 프로세스를
        // 끝내기 전에, 끝나는 스레드에서 종료 상태와 함께 호출; hook 이 반환하면
        // 프로세스가 끝남

    // REPL host (세션 저장 / 복원)
    fn rok_session_hold(timeout_ns: i64) -> i32;          // 조용한 순간에 world 를 멈춤: 실행 중인 poll / blocking job 이
                                                           //   없고 lock 안에서 멈춘 스레드가 없을 때. 0: timeout_ns 안에
                                                           //   못 함 (요청은 남음)
    fn rok_session_unhold();                               // 대기 중이거나 얻은 hold 를 거둠
    fn rok_session_resume();                               // hold 뒤에 world 를 재개
    fn rok_session_enter();                                // poll / blocking job 둘레의 executor 스레드 gate
    fn rok_session_leave();
    fn rok_session_pass();
    fn rok_lock_notes() -> i32;                            // 세션 host 에서 1
    fn rok_lock_note(word: GcBuf<i32>, kind: i32) -> i32;  // kind: 0 Mutex, 1 read, 2 write, 3 실행 중인 Once; 1 = 기록함
    fn rok_lock_drop(word: GcBuf<i32>);
    fn rok_world_threads(out: Ptr<u64>, cap: usize) -> usize;   // world 정지 중: 다른 mutator 의 OS id (Windows)
    fn rok_session_restored(lost: ConstPtr<u64>, n: usize);     // 세션을 되돌림. n == SIZE_MAX: 다른 스레드는 모두 사라짐
    fn rok_thread_lost(worker_id: u64) -> i32;             // 실패한 셀 뒤에 그 스레드가 사라졌으면 1
    fn rok_io_quiesce();                                   // Windows: 메모리를 되돌리기 전에 진행 중인 I/O 취소
    fn rok_watch_writes();                                 // Windows: heap 을 write-watch. 런타임 초기화 전에 호출
    fn rok_stdio_line_buffered();                          // 터미널에서처럼 줄바꿈마다 stdout 을 flush
    fn rok_set_stdin_hook(hook: extern "C" fn(Ptr<u8>, usize) -> isize | null);
        // stdin 을 hook 으로 읽음 (null: fd 0): 읽은 수, 입력의 끝이면 0,
        // 또는 -errno. 매 호출 전에 stdout 을 flush
    static rok_page_epoch: u64;                            // Windows: watch 하는 page 가 쓰기 없이 바뀌거나
                                                           //   watch 하는 block 이 해제되면 증가
}
```

다음 규칙을 적용합니다.

- 사용자 코드와 표준 라이브러리는 이 심볼에 의존하면 안 됩니다. 시그니처는 예고 없이 바뀌거나 제거될 수 있습니다.
- `rok_` prefix를 공유하지만 8.5.8의 영역 표에는 들어가지 않습니다. 이름 충돌을 피하려고, 사양의 규범 심볼은 위 이름을 다시 쓰지 않습니다.
- `rok_panic`은 이 부록에 속하지 않습니다. 시그니처는 [8.5.7](#857-diagnostic-영역)의 `rok_panic(msg: ConstPtr<u8>, msg_len: usize)`가 권위이며, 구현이 이와 다른 시그니처를 내보내면 부적합(non-conformance)입니다.
