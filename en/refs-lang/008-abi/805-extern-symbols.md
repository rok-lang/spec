# 8.5. Extern "C" symbols

Defines the signatures of the [symbols](../000-meta/007-glossary.md#extern-c-symbol) exposed by `librokr.a`. Compiler-generated code and the standard library call these symbols.

On the rok side they are declared with the runtime ABI `extern "rok"` ([5.9.2](../005-modules/509-extern-block.md#592-abi-specifiers)), which the toolchain places in `rok::core::rt` as `pub(pkg)`. In the blocks below a `fn` value is `RokFn` (8.5.0a) and all other values are as in the C ABI. `str` is passed by value as `struct { const uint8_t *ptr; size_t len; }` following the target's 16-byte struct rule (like `RokFn`), never split between registers and the stack.

> [!IMPORTANT]
> Every symbol starts with the `rok_` prefix, to avoid symbol collisions with external libraries and user code.

## 8.5.0. Thread-safety classes

Each symbol definition carries a **Thread-safety** annotation. It is part of the contract between compiler-emitted code and the runtime and states in which calling contexts the symbol may be called safely. There are six classes.

| Class                 | Meaning                                                                      |
|-----------------------|------------------------------------------------------------------------------|
| `thread-safe`         | Safe to call concurrently from multiple threads (the runtime synchronizes internally) |
| `caller-thread-only`  | Bound to the calling thread's context — another thread cannot act on behalf of that caller |
| `process-init`        | Exactly once at process start, before entering mt                           |
| `process-shutdown`    | Exactly once at process exit, after mt ends                                  |
| `per-thread-init`     | Once at the start of each worker thread, by that thread for itself           |
| `per-thread-shutdown` | Once at the end of each worker thread, by that thread for itself             |

`caller-thread-only` means the function's effects are tied to the caller's TLS slots or stack frame, so another thread cannot act on behalf of that caller. It does not mean the function is non-reentrant; multiple threads may call it concurrently, each for itself.

## 8.5.0a. `RokFn` — C ABI form of function values

The C representation used when passing a rok function value `{code, env}` ([4.4](../004-functions/404-function-type.md#representation-of-function-values)) to a runtime symbol.

```c
typedef struct {
    void *code;   // code pointer that takes env as its first argument
    void *env;    // capture environment (NULL for named functions)
} RokFn;
```

`RokFn` is **passed by value** as a C struct. The actual register assignment follows the target C ABI's rules for 16-byte structs.

## 8.5.1. GC area

### `rok_gc_alloc_slow`

The [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) slow path from [Allocation](../007-runtime/702-allocation.md). **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_gc_alloc_slow(size: usize, align: usize, type_id: u32) -> Ptr<u8>;
}
```

The arguments and return value mean:

- `size`: the total object size: the 16-byte header plus the payload, aligned.
- `align`: the object's alignment requirement, usually 8, larger when `#[repr(align(N))]` is specified.
- `type_id`: the object's type ID.
- Returns: a pointer starting at the header; the function fills in type_id and gc_flags.

It never returns null. Out of memory is fatal (`rok_oom`), and a request above `isize::MAX` is the fatal error "allocation too large" (exit 101).

### `rok_alloc_raw` / `rok_realloc_raw`

Allocates raw memory outside the GC ([stdlib 1.4 alloc](../../refs-standard-libraries/100-bare/104-alloc.md)). **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_alloc_raw(size: usize, align: usize, zeroed: i32) -> usize;
    fn rok_realloc_raw(ptr: usize, old_size: usize, align: usize, new_size: usize) -> usize;
}
```

If `zeroed` is nonzero, the allocated memory is zero-filled. `rok_alloc_raw` returns null (`0`) if `size` is 0, if `align` is not a power of two, or if allocation fails; it does not abort. `rok_realloc_raw` also returns null on failure. Failure handling is the caller's (`rok::bare::alloc`) responsibility.

### `rok_gc_wb_slow`

The write barrier slow path from [Barriers and root tracking](../007-runtime/704-barriers-and-roots.md). **Thread-safety: `caller-thread-only`** (records into the wb log buffer in TLS).

```
extern "C" {
    fn rok_gc_wb_slow(obj: Ptr<u8>);
}
```

The argument `obj` is a pointer to the object containing the modified slot; there is no return value. Effect: adds obj to the remembered set and flushes when the log is full.

### `rok_gc_register_type`

Registers a type's [ScanTable](802-scan-table.md). **Thread-safety: `process-init`**.

```
extern "C" {
    fn rok_gc_register_type(type_id: u32, scan_table: ConstPtr<u8>);
}
```

Called at process start, before entering mt, so no synchronization is needed.

### `rok_gc_pin` / `rok_gc_unpin`

Pinning of [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) objects (see [Object layout and memory spaces](../007-runtime/701-object-layout.md) and [`extern` blocks (FFI declarations)](../005-modules/509-extern-block.md)). **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_gc_pin(obj: Ptr<u8>) -> Ptr<u8>;       // returns the pinned object (same address)
    fn rok_gc_unpin(obj: Ptr<u8>);
}
```

A GC pointer passed to an `extern "C"` function or a blocking runtime call is not moved even without pinning, because the argument stays in a root until the call ends ([7.4.1](../007-runtime/704-barriers-and-roots.md#objects-pointed-to-by-roots-are-not-moved)). Pinning is needed only when C code keeps the pointer after the call. Pin sets `PINNED` on the object, excluding it from evacuation and keeping it alive; unpin releases it.

### `rok_gc_collect`

An explicit [GC](../000-meta/007-glossary.md#gc-garbage-collector) trigger, rarely used. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_gc_collect();
}
```

Performs a full collection and returns after it finishes.

## 8.5.2. Runtime area

### `rok_runtime_init`

Initializes the runtime. **Thread-safety: `process-init`**.

```
extern "C" {
    fn rok_runtime_init();
}
```

Called right after process start, before `main()`. Effects: creates the worker pool, mmaps the GC spaces and initializes the I/O driver. The compiler-generated C-ABI `main` calls `rok.register_types` right after this call ([8.2.13](802-scan-table.md#8213-registration-time)).

The runtime additionally exports the thread-local data symbol `rok_tls_shadow_stack_top` ([8.3.8](803-tls-slots.md#838-form-of-compiler-emission)).

### `rok_runtime_shutdown`

Shuts down the runtime. **Thread-safety: `process-shutdown`**.

```
extern "C" {
    fn rok_runtime_shutdown();
}
```

Called right after `main()` ends. Every other rok thread stops for good at its next safepoint, after finishing any runtime or C code it is in. Then the worker pool shuts down and outstanding resources are released.

### `rok_exit`

Terminates the process immediately. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_exit(code: i32) -> !;
}
```

[`rok::std::env::exit`](../../refs-standard-libraries/300-std/306-env.md) calls this symbol. `finally` and `Drop` are not run ([4.16.4](../004-functions/416-entry-point.md#4164-exit-codes-and-unhandled-exceptions)).

### String helpers

Internal helpers backing `FromStr` and `str::bytes` ([stdlib 2.15](../../refs-standard-libraries/200-core/215-string.md)). **Thread-safety: `thread-safe`**. `str` arguments use the struct form of the preamble.

```
extern "C" {
    fn rok_str_byte(s: str, i: usize) -> u8;                         // i-th byte (bounds check is the caller's)
    fn rok_str_slice(s: str, start: usize, end: usize, at: str) -> str;  // s[start:end]. panics at location at if the range / UTF-8 boundary is wrong
    fn rok_str_parse_int(s: str, radix: u32, signed: i32, bits: u32, out: GcBuf<i64>) -> i32;
                                        // 0 ok, 1 empty, 2 invalid digit, 3 out of range; value bits in out[0]
    fn rok_str_parse_float(s: str, out: GcBuf<f64>) -> i32;           // 0 ok, 2 invalid; value in out[0]
}
```

### `rok_runtime_yield`

The `yield` function of the [Safepoint protocol](804-safepoint.md), i.e. the slow path of the safepoint poll. **Thread-safety: `caller-thread-only`** (called by the calling mutator as a result of polling its own yield_flag).

```
extern "C" {
    fn rok_runtime_yield();
}
```

### `rok_enter_native` / `rok_leave_native`

Slow paths of the native bracket ([8.4.10](804-safepoint.md#8410-native-calls)). **Thread-safety: `caller-thread-only`**.

```
extern "C" {
    fn rok_enter_native();                // entering C: this thread counts as PARKED
    fn rok_leave_native();                // back from C: wait out a running collection
}
```

The compiler inlines the fast path around every `extern "C"` call: a seq_cst store of `PARKED` / `RUNNING` to the mutator state, then a check of the yield flag. Only when the flag is set does it call these.

### `rok_native_reenter` / `rok_native_resume`

The entry and exit of every `extern "C" fn` definition, so that a C callback runs as a mutator. **Thread-safety: `caller-thread-only`**.

```
extern "C" {
    fn rok_native_reenter() -> u32;       // if the thread was PARKED (inside C), leave native; returns whether it was
    fn rok_native_resume(parked: u32);    // re-enter native if reenter left it
}
```

### `rok_thread_init` / `rok_thread_shutdown`

Worker thread lifecycle. **Thread-safety: `rok_thread_init` is `per-thread-init`, `rok_thread_shutdown` is `per-thread-shutdown`**.

```
extern "C" {
    fn rok_thread_init();
    fn rok_thread_shutdown();
}
```

Called at the start and end of each worker thread to initialize or tear down the TLS of the [TLS slots](803-tls-slots.md).

### Internal symbol families

These families are runtime-internal helpers of the reference runtime and are not part of the ABI contract: `rok_fmt_*`, `rok_str_*` helpers, `rok_char_*`, `rok_nat_*`, `rok_stdio_*`, `rok_time_*`, `rok_gc_stats_*`, `rok_buf_*`, `rok_cstr_len`.

### Functions that are not safepoints

The following runtime functions are guaranteed never to allocate and never to block, so a call to them is not a safepoint ([8.4.2](804-safepoint.md#842-emission-sites), [8.9.12](809-lowering-control.md#8912-shadow-stack-frame)): `rok_str_len`, `rok_str_byte`, `rok_str_cmp`, `rok_str_eq_at`, `rok_str_find`, `rok_str_rfind`, `rok_str_char_count`, `rok_str_hash`, `rok_str_bytes_buf`, `rok_buf_copy`, the `rok_char_*` property and case functions, `rok_fmt_len`, `rok_f64_*`, `rok_f32_*`.

## 8.5.3. Coroutine area

### `rok_coro_frame_alloc`

Allocates a [coroutine](../007-runtime/708-coroutine.md) frame. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_coro_frame_alloc(size: usize, type_id: u32) -> Ptr<u8>;
}
```

Nearly identical to `rok_gc_alloc_slow`, but allows frame-specific handling.

### `rok_coro_park` / `rok_coro_unpark`

The compiler does not emit these two symbols (`await` uses the parking registry of `rok::core::task`). The runtime provides them for C and low-level code. The address of `frame` (any GC object) is the parking key, and `rok_coro_unpark(frame)` wakes every waker parked on that key. Calling them outside a task executor aborts.

**Thread-safety: `rok_coro_park` is `caller-thread-only`** (parks the calling coroutine itself), **`rok_coro_unpark` is `thread-safe`** (may be called from another thread).

```
extern "C" {
    fn rok_coro_park(frame: Ptr<u8>, waker: Ptr<u8>);
    fn rok_coro_unpark(frame: Ptr<u8>);
}
```

`rok_coro_park` suspends the coroutine and registers the waker. `rok_coro_unpark` resumes the coroutine by pushing it onto the task queue.

Here `waker` is an **opaque pointer (`Ptr<u8>`)** to a bundle of wake function and context, i.e. the runtime's `Waker` object ([2.20.2](../../refs-standard-libraries/200-core/220-future.md)). The internal representation of `Waker` is not exposed across the C ABI boundary, so the `waker` argument has type `Ptr<u8>` everywhere, including [7.8.8](../007-runtime/708-coroutine.md) (likewise for `rok_io_register` and `rok_timer_register`). The exact layout of the pointed-to object is determined by the coroutine implementation.

## 8.5.4. Scheduler area

The task runtime (spawn, join, cancel, executor) is rok code in `rok::core::task` ([stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)). The runtime provides only the executor slot, thread and parking symbols below.

### `rok_task_worker_*` / `rok_time_now_ns`

Functions that store and retrieve the current thread's executor worker ([stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)) in the TLS slot `rok_tls_task_worker` ([8.3.1](803-tls-slots.md#831-tls-slot-structure)), plus a clock. **Thread-safety: `caller-thread-only`** (`rok_time_now_ns` is `thread-safe`).

```
extern "C" {
    fn rok_task_worker_present() -> i32;       // 1 if this thread is a worker
    fn rok_task_worker_get() -> usize;         // worker object address, 0 if none
    fn rok_task_worker_set(w: usize);          // a live GC object or 0. Pinned while installed
    fn rok_task_worker_clear();                // = rok_task_worker_set(0)
    fn rok_time_now_ns() -> i64;               // monotonic clock (ns)
}
```

### `rok_thread_spawn`

Creates a 1:1 OS thread. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_thread_spawn(f: RokFn, stack_size: usize, id: u64) -> usize;
    fn rok_thread_new_id() -> u64;
    fn rok_thread_id() -> u64;                        // worker id of the current thread
    fn rok_thread_sleep_ns(ns: i64);
    fn rok_thread_yield_now();
}
```

`rok_thread_spawn` runs the rok closure `f` (a `RokFn`, passed by value) on an OS thread (`stack_size` 0 means the default). `id` is a value obtained from `rok_thread_new_id` and becomes the new thread's `rok_tls_worker_id`. The return value is a join handle, or 0 if creation fails. `env` is a GC root until the thread finishes.

### `rok_task_hooks_install` / `rok_task_key`

The executor of `rok::core::task` installs hooks into the runtime. The runtime's I/O driver, timer, and `rok_coro_park` / `rok_coro_unpark` use these hooks to wake and suspend tasks. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_task_hooks_install(
        io: extern "C" fn(fd: i32, interest: u32, waker: Ptr<u8>),
        timer: extern "C" fn(deadline_ns: u64, waker: Ptr<u8>),
        park: extern "C" fn(key: Ptr<u8>, waker: Ptr<u8>),
        unpark: extern "C" fn(key: Ptr<u8>),
    );                                          // the first installation wins
    unsafe fn rok_task_key(obj: Ptr<u8>) -> usize;   // GC object address as a park key
}
```

`rok_task_key` uses the address as a key, so it sets `NOMOVE` ([8.1.3](801-object-header-abi.md#813-gc_flags-offset-4-u32)) on the object. The object is never moved afterwards. Identity comparisons used only within a call use `buf_same` ([stdlib 2.9](../../refs-standard-libraries/200-core/209-collections.md)), which pins nothing.

### `rok_sync_wait` / `rok_sync_wake`

Waiting and waking for synchronization primitives (`Mutex`, `Condvar`, `Once`). **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_sync_wait(buf: Ptr<u8>, index: usize, expected: i32, timeout_ns: i64);   // wait while the GcBuf<i32> element equals expected. timeout_ns < 0 means forever
    fn rok_sync_wake(buf: Ptr<u8>, index: usize);                                    // wake all threads waiting on that element
}
```

`rok_sync_wait` may return without a reason to wake, so the caller re-checks its condition. While waiting, the thread is considered stopped with respect to the GC.

### `rok_thread_join`

**Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_thread_join(handle: usize) -> i32;          // 0 on success
}
```

### `rok_thread_detach` / `rok_cpu_count`

**Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_thread_detach(handle: usize) -> i32;   // OS resources are released when it ends; can no longer be joined. 0 on success
    fn rok_cpu_count() -> usize;                  // number of online CPUs (at least 1) — size of the par pool
}
```

### `rok_static_init_begin` / `rok_static_init_end`

Back the lazy initialization of `static`s whose initializer is not `const` ([5.3](../005-modules/503-static.md)). **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_static_init_begin(state: Ptr<u8>) -> i32;   // 1: caller runs the initializer, 0: already ready
    fn rok_static_init_end(state: Ptr<u8>);            // publish initialization completion
}
```

`state` is the state byte of that `static` (0 new, 1 in progress, 2 ready). If another thread is initializing, `rok_static_init_begin` waits until it finishes, during which the thread is considered stopped with respect to the GC.

### `rok_gc_add_root`

```
extern "C" {
    fn rok_gc_add_root(word: ConstPtr<usize>);   // register a static's reference word as a GC root until process exit
}
```

Startup code generated by the compiler calls this for every `static` that holds a GC reference.

### `rok_os_*` — OS helpers (internal, subject to change)

Internal symbols backing the standard library `rok::std` (fs, io, env, process). They are not covered by ABI stability and may change, so user code does not call them directly.

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

The lock-free Chase-Lev deque of the task scheduler ([7.6.2](../007-runtime/706-scheduler.md#762-work-stealing-structure)). The deque and its ring are GC objects traced by the GC, so elements that exist only inside a deque are kept alive. **Thread-safety:** `push` / `pop` only from the owning thread; `steal` and `len` from any thread.

```
extern "C" {
    fn rok_deque_new() -> Ptr<u8>;                                   // empty deque (GC object)
    unsafe fn rok_deque_push(d: Ptr<u8>, item: usize);               // owner: push at the end (GC reference)
    unsafe fn rok_deque_pop(d: Ptr<u8>, out: Ptr<u8>, index: usize) -> i32;    // owner: newest element into out[index], 1 if any
    unsafe fn rok_deque_steal(d: Ptr<u8>, out: Ptr<u8>, index: usize) -> i32;  // oldest element into out[index], 1 if any
    unsafe fn rok_deque_len(d: Ptr<u8>) -> usize;                    // element count (approximate under contention)
}
```

`out` is a `GcBuf` holding GC references; the result is written at `index`. These functions contain no safepoints.

## 8.5.5. I/O / Timer area

### `rok_io_register` / `rok_timer_register`

Let C or low-level code register a task to be woken when an fd is ready or at a point in time. Internally they use the reactor ([7.7](../007-runtime/707-io-and-timer.md)) and the executor's timer. Calling them outside a task executor aborts.

**Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_io_register(fd: i32, interest: u32, waker: Ptr<u8>);
    fn rok_timer_register(deadline: u64, waker: Ptr<u8>);
}
```

- `fd`: the file descriptor.
- `interest`: `1` = read, `2` = write (the two may be ORed).
- `waker`: a single-element `GcBuf<Waker>`. Registration is edge-triggered and wakes **only once** (spurious wakeups are possible). If the fd is already ready or cannot be watched, it wakes immediately.
- `deadline`: an absolute time (ns) on the `rok_time_now_ns` clock. When that time is reached, the waker is woken once.

### `rok_io_*` — reactor (internal, subject to change)

Internal symbols making up the reactor of the [I/O driver](../007-runtime/707-io-and-timer.md). They are not covered by ABI stability and may change. **Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_io_poller() -> i32;                    // the process's kqueue / epoll fd (created on first call)
    fn rok_io_watch(fd: i32) -> i32;              // register fd for read / write edge-triggered (re-registering is harmless), 0 or -errno
    fn rok_io_poll(out: Ptr<u8>, max: usize) -> i32;   // blocks until something changes (parked for the GC), event count or -errno
}
```

`rok_io_poll` writes up to `max` events into `out`, a `GcBuf<i64>`, as `fd * 2 + direction` (read 0, write 1).

### `rok_net_*` — socket helpers (internal, subject to change)

Internal symbols backing the standard library `rok::std::net` ([stdlib 3.2](../../refs-standard-libraries/300-std/302-net.md)). They are not covered by ABI stability and may change, so user code does not call them directly. Sockets are always created non-blocking, and failures are returned as `-errno`.

```
rok_net_socket     rok_net_bind       rok_net_listen     rok_net_connect
rok_net_sock_error rok_net_accept     rok_net_addr       rok_net_send
rok_net_send_to    rok_net_recv_from  rok_net_shutdown   rok_net_setopt
rok_net_multicast_v4 rok_net_wait     rok_net_resolve    rok_net_gai_strerror
```

`rok_net_wait` is a `poll(2)` with which a blocking call waits for readiness (a native section for the GC); `rok_net_resolve` is `getaddrinfo`.

## 8.5.6. Error propagation (no symbols)

`throw` / `try` / `catch` from [Error control flow](../003-expressions/314-error-control-flow.md) have no runtime symbols. A throwing function returns the error as part of its return value ([8.6.5](806-calling-convention.md#865-return-of-throwing-functions-value-based-error-propagation)), and the call site checks that value and branches ([8.9.6](809-lowering-control.md#896-try--catch--finally-value-based-branching)). There is therefore no runtime call to throw an error or fetch the current error, and there are no TLS slots or jmp_buf for error handling.

## 8.5.7. Diagnostic area

### `rok_panic`

**Thread-safety: `thread-safe`** (callable from any thread; the process exits immediately).

```
extern "C" {
    fn rok_panic(msg: ConstPtr<u8>, msg_len: usize);   // does not return
}
```

An unrecoverable panic; fatal. This function does not return: it prints the message to stderr and terminates the process with exit code `101`. `catch` and `finally` are not run.

### `rok_panic_str`

**Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_panic_str(msg: str) -> !;          // msg is (ptr, len)
}
```

Same as `rok_panic` but takes the message as a `str`. It prints no location and is used only for panics internal to the runtime.

### `rok_panic_at`

**Thread-safety: `thread-safe`**.

```
extern "rok" {
    fn rok_panic_at(msg: str, at: str) -> !;  // at = "path:line:col"
}
```

Prints in the format of [4.14.13](../004-functions/414-throws.md#41413-panics-and-fatal-errors) and exits with code `101`. Core's `panic` and compiler-generated panics (bounds checks, overflow, division by zero, etc.) call this symbol.

### `rok_uncaught_error` / `rok_uncaught_error_at`

```
extern "rok" {
    fn rok_uncaught_error(what: str) -> !;              // uncaught exception: print what to stderr, exit code 1
    fn rok_uncaught_error_at(what: str, at: str) -> !;  // followed by a "  --> <at>" line
}
```

Handle errors propagated out of the entry point ([4.16.4](../004-functions/416-entry-point.md#4164-exit-codes-and-unhandled-exceptions)). The declarations live in core.

### `rok_oom`

**Thread-safety: `thread-safe`**.

```
extern "C" {
    fn rok_oom() -> !;
}
```

Prints `fatal error: out of memory` to stderr and exits with code `101`. Allocation failures inside the runtime and core's `oom()` both go through this symbol. The runtime cannot call rok functions by name, so on allocation failure it uses this symbol directly instead of calling the function in the `oom_fn` slot.

### Stack overflow

When a thread's stack overflows, the runtime flushes stdout (if possible), prints the following to stderr and aborts (SIGABRT). Unlike a panic, there is no unwinding and no `finally` execution.

```
thread '<name>' has overflowed its stack
```

`<name>` is the thread name given with `thread::Builder::name` (also set as the OS thread name, [7. Runtime](../007-runtime/README.md#runtime-error-stack-overflow)). This covers the main thread (`rok_runtime_init`), `rok_thread_spawn` threads (including executor workers, and deep `await` chains), and C threads that called `rok_thread_init`. The runtime records the guard range below each thread's stack in the TLS slot `rok_tls_stack_guard` ([8.3.1](803-tls-slots.md#831-tls-slot-structure)) and installs an alternate signal stack (`sigaltstack`). Only SIGSEGV / SIGBUS within that range is reported as a stack overflow; other faults follow the default behavior.

## 8.5.8. Symbol naming conventions

The area of each symbol prefix is summarized below.

| Prefix          | Area           |
|-----------------|----------------|
| `rok_gc_*`      | GC             |
| `rok_runtime_*` | Runtime in general |
| `rok_thread_*`  | OS threads     |
| `rok_coro_*`    | Coroutines     |
| `rok_task_*`    | Tasks          |
| `rok_io_*`      | I/O            |
| `rok_timer_*`   | Timers         |
| `rok_panic`     | Diagnostics    |

This prefix convention avoids symbol collisions with external libraries and user code.

## 8.5.9. Compatibility of function signatures

Each function signature is part of the [ABI](../000-meta/007-glossary.md#abi-application-binary-interface). The change policy is:

- minor: only compatible changes are allowed; adding arguments is handled through function wrappers.
- major: breaking changes are possible, and the ABI version hash changes.

See the stability policy in the [overview](README.md) for details.

## 8.5.10. Calling convention

All functions above use the target platform's C ABI calling convention, e.g. System V x86_64, AAPCS aarch64, MS x64 Windows.

## 8.5.11. Example: slow call from the TLAB fast path

```mlir
; LLVM IR (when the fast path fails)
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
  ; ... continue
```

The example shows the LLVM IR structure combining the fast path and the slow path.

## 8.5.12. Auxiliary symbols (appendix, non-normative)

> [!IMPORTANT]
> This section is **non-normative**. The symbols below are auxiliary symbols exported by the reference runtime (`rokr`). They are not part of the main ABI tables (8.5.1 ~ 8.5.11) and carry no stability promise.

Signatures use the same notation as the main text (`Ptr<T>` = mutable raw pointer, `ConstPtr<T>` = read-only raw pointer). String arguments are NUL-terminated C strings.

```
extern "C" {
    // memory
    fn rok_free(ptr: Ptr<u8>);                             // null ignored
    fn rok_load_i32(ptr: ConstPtr<i32>) -> i32;
    fn rok_load_i64(ptr: ConstPtr<i64>) -> i64;
    fn rok_load_f64(ptr: ConstPtr<f64>) -> f64;
    fn rok_store_i32(ptr: Ptr<i32>, val: i32);
    fn rok_store_i64(ptr: Ptr<i64>, val: i64);
    fn rok_store_f64(ptr: Ptr<f64>, val: f64);

    // C strings
    fn rok_str_len(s: ConstPtr<u8>) -> usize;              // null → 0

    // GC
    fn rok_gc_collect_young();                             // force a minor collection
}
```

The following rules apply.

- User code and the standard library must not depend on these symbols. Their signatures may change or be removed without notice.
- They share the `rok_` prefix but are not included in the area table of 8.5.8. To avoid name collisions, normative symbols of the specification do not reuse these names.
- `rok_panic` does not belong to this appendix. The signature `rok_panic(msg: ConstPtr<u8>, msg_len: usize)` in [8.5.7](#857-diagnostic-area) is authoritative, and an implementation exporting a different signature is non-conformant.
