# 3.4. `thread::` - OS threads (1:1 model)

The `rok::std::thread` module is the entry point for OS-level threads. As set out in [2.18 task](../200-core/218-task.md), it is a separate abstraction from tasks, which run on the M:N cooperative scheduler; this module provides the path for spawning native threads directly.

## 3.4.1. When to use thread vs. task

The following table indicates which to choose in which situation.

| Situation                      | Recommendation                              |
|--------------------------------|---------------------------------------------|
| General concurrency / async I/O | `task::spawn` (lightweight)                |
| CPU-bound computation          | `task::spawn` or `par::*` (work-stealing)   |
| Blocking C library calls       | `thread::spawn` (avoids blocking the task scheduler) |
| Where an OS thread API is required | `thread::spawn` (TLS, signal mask, etc.) |
| Embedded / no-task             | `thread::spawn`                             |

In short, tasks come first wherever they are light and expressive enough, and dropping down to threads is for places that must deal with OS resources directly.

## 3.4.2. API

The basic types and function signatures are as follows.

```rok
struct Thread { /* OS handle, not generic */ }
struct JoinHandle<T> { /* join result T */ }
struct ThreadId { /* opaque */ }       // Clone, Copy, PartialEq, Eq, Hash, Debug

struct JoinError { .. }                 // private fields (0.5.10)
impl JoinError {
    fn new(kind: JoinErrorKind) -> JoinError;
    fn kind(self) -> JoinErrorKind;
    fn os_code(self) -> i32 | none;      // the number of an Os kind
    fn message(self) -> str;             // as Display shows it
}
// implements Display, Debug, Error, PartialEq, Eq and Clone

#[non_exhaustive]
enum JoinErrorKind {
    Os(i32),                             // the OS's join error number (errno)
    Lost,                                // the thread ended with a failed REPL cell
    Taken,                               // already joined or detached
}                                        // Clone, Copy, PartialEq, Eq, Hash, Debug

fn spawn<T: Send>(f: fn() -> T + Send) -> JoinHandle<T>;   // = try! Builder::new().spawn(f)

struct Builder { /* opaque: name, stack size */ }

impl Builder {
    fn new() -> Builder;                 // no name, default stack
    fn name(self, name: str) -> Builder;
    fn stack_size(self, size: usize) -> Builder;   // bytes (default 8 MiB)
    fn spawn<T: Send>(self, f: fn() -> T + Send) throws (IoError) -> JoinHandle<T>;
}

impl<T> JoinHandle<T> {
    fn join(self) throws (JoinError) -> T;
    fn detach(self);                     // does not join; the thread runs to completion
    fn thread(self) -> Thread;
}

impl<T: Send> IntoFuture for JoinHandle<T> { ... }   // `await handle` joins and yields T

impl ThreadId {
    fn as_u64(self) -> u64;              // the runtime's id of the thread
}

impl Thread {
    fn id(self) -> ThreadId;
    fn name(self) -> str | none;        // name given with Builder::name (also set as the OS thread name), none if absent
    fn unpark(self);
}

// current thread
fn current() -> Thread;
fn yield_now();
fn sleep(dur: Duration);
fn park();                                     // blocks until unparked (may return without a reason)
fn park_timeout(dur: Duration);
fn panicking() -> bool;                        // true on a thread that panicked, while the process ends

// thread-local storage
struct ThreadLocal<T> { .. }            // Send + Sync for every T: one value per thread
fn local<T>(init: fn() -> T + Sync) -> ThreadLocal<T>;     // init is called on first use in each thread

impl<T> ThreadLocal<T> {
    fn with<R>(self, f: fn(T) -> R) -> R;    // f of this thread's value, lent to f (scoped)
    fn set(self, value: T);                  // replaces this thread's value
    fn replace(self, value: T) -> T;         // replaces it, returns the old one
    unsafe fn as_ptr(self) -> Ptr<T>;        // this thread's value, pinned until the thread ends
}
impl<T: Copy> ThreadLocal<T> {
    fn get(self) -> T;                       // a copy of this thread's value
}
impl<T: Default> ThreadLocal<T> {
    fn take(self) -> T;                      // the value, leaving T::default()
}
```

`Thread::unpark` sets that thread's park token and wakes it. A token is consumed by one `park`.

`thread::spawn` is `Builder::spawn` with defaults behind `try!`: where `Builder::spawn` throws (no threads on the target, or the OS refuses a new thread), `thread::spawn` panics. Use `Builder::new().spawn(f)` to handle the error.

`await handle` ([2.20.5](../200-core/220-future.md#2205-intofuture---awaitable-values)) waits for the thread from inside a task: the join runs on the executor's blocking lane ([`task::spawn_blocking`](../200-core/218-task.md#2181-core-api)), so only the task waits. It yields `T`; an OS join failure (`JoinError`) panics, so call `join` to handle it.

### Panics and `JoinError`

A panic in any thread terminates the whole process with exit code 101. `join` therefore never receives another thread's panic, and `thread::panicking()` is true only for code that runs on the panicking thread while the process ends (an embedder's exit hook). `JoinError` represents OS-level join failures (`JoinErrorKind::Os`), or `JoinErrorKind::Lost` for a thread that ended with a failed REPL cell ([refs-lang 9.7.0](../../refs-lang/009-tools/907-jit-cells.md#970-rules-of-the-cell-session)), displayed as "thread join failed: the thread ended with a failed REPL cell".

`JoinErrorKind::Taken`: the handle, or a copy of it, was already joined or detached. A second `join` throws it; a second `detach` does nothing.

## 3.4.3. Examples

From the simplest spawn-join pattern to builders and multiple spawns:

```rok
use rok::std::thread;

let h = thread::spawn(|| {
    heavy_blocking_work()
});
let result = try h.join();

// builder
let h = try thread::Builder::new()
    .name("worker-1")
    .stack_size(4 * 1024 * 1024)
    .spawn(|| { ... });

// multiple
let handles: List<JoinHandle<i64>> = (0..10)
    .map(|i| thread::spawn(|| { compute(i) }))
    .collect();

let results: List<i64> = handles
    .into_iter()
    .map(|h| try! h.join())
    .collect();
```

## 3.4.4. Thread-local storage

When each thread needs its own independent storage, use `thread::local`.

```rok
let counter = thread::local(|| 0);

try thread::spawn(|| {
    counter.set(counter.get() + 1);         // only this thread's value changes
}).join();
```

The shape follows Rust's `LocalKey`. `with(f)` lends the current thread's value to `f` and returns what `f` returns. The value is a scoped view, tracked like a lock guard's value ([stdlib 2.16.3](../200-core/216-sync.md#2163-mutext---exclusive-lock)): returning it out of `f`, storing it past the call (a capture, a field, a `Cell`), sending it, or passing it to a callee that keeps it is compile error E0137, and so is giving `with` an `async` closure, whose future would hold the value across an `await` and could resume on another worker thread. Keep a `clone()`, or what `f` computes, instead.

```rok
static SEEN: ThreadLocal<List<str>> = thread::local(|| -> List<str> { [] });

SEEN.with(|l| l.push("a"));             // changes this thread's list in place
let n = SEEN.with(|l| l.len());          // what f computes comes out
let copy = SEEN.with(|l| l.clone());     // so does a copy
let all = SEEN.take();                   // the list itself, leaving []
```

The owned forms move values in and out: `set(v)` stores `v`, `replace(v)` stores `v` and returns the old value, `take()` returns the value and leaves `T::default()`. `get()` returns a copy and needs `T: Copy`; for any other `T` it is a compile error (use `with`, `replace` or `take`). `unsafe fn as_ptr()` points at the current thread's value; the slot stays pinned until the thread ends, and reads and writes through the pointer are the caller's responsibility.

`with` takes a closure that cannot `await`, so a thread's value is never held across an `await`, where the task may resume on another thread.

## 3.4.5. impl / target

Per-platform support:

| Item            | linux/macos/windows   | wasm32-wasip2 / browser     | wasm32-wasip1-threads |
|-----------------|-----------------------|---------------------------|---------------------|
| `spawn`         | ✓                     | panics (`Unsupported` from `Builder::spawn`) | ✓    |
| `Builder::spawn` | ✓                    | throws `Unsupported`      | ✓                   |
| `local`         | ✓                     | ✓ (one thread)            | ✓                   |
| `park`/`unpark` | futex / WaitOnAddress | ✓                         | ✓                   |

On `wasm32-wasip2` and in the browser there is one thread: `thread::current().name()` is `none`. On `wasm32-wasip1-threads`, `rok_cpu_count()` and the available parallelism are 4, since WASI cannot report a core count.
