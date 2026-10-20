# 3.4. `thread::` — OS threads (1:1 model)

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
struct Thread { /* OS handle — not generic */ }
struct JoinHandle<T> { /* join result T */ }
struct ThreadId { /* opaque */ }

struct JoinError { code: i32; }          // OS join error number (errno), or JOIN_LOST
const JOIN_LOST: i32 = -1;               // the thread ended with a failed REPL cell
// implements Display, Debug and Error

fn spawn<T: Send>(f: fn() -> T + Send) -> JoinHandle<T>;

struct Builder {
    name: str | none;
    stack_size: usize | none;
}

impl Builder {
    fn new() -> Builder;
    fn name(self, name: str) -> Builder;
    fn stack_size(self, size: usize) -> Builder;
    fn spawn<T: Send>(self, f: fn() -> T + Send) throws (IoError) -> JoinHandle<T>;
}

impl<T> JoinHandle<T> {
    fn join(self) throws (JoinError) -> T;
    fn detach(self);                     // does not join — the thread runs to completion
    fn thread(self) -> Thread;
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

// thread-local storage
fn local<T>(init: fn() -> T + Sync) -> ThreadLocal<T>;     // init is called on first use in each thread

impl<T> ThreadLocal<T> {
    fn get(self) -> T;                       // value of the current thread
    fn set(self, value: T);
    fn with<R>(self, f: fn(T) -> R) -> R;
}
```

`Thread::unpark` sets that thread's park token and wakes it. A token is consumed by one `park`.

### Panics and `JoinError`

A panic in any thread terminates the whole process with exit code 101. `join` therefore never receives another thread's panic, and `JoinError` represents OS-level join failures, or `JOIN_LOST` for a thread that ended with a failed REPL cell ([refs-lang 9.7.0](../../refs-lang/009-tools/907-jit-cells.md#970-rules-of-the-cell-session)), displayed as "thread join failed: the thread ended with a failed REPL cell".

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

## 3.4.5. impl / target

Per-platform support:

| Item            | linux/macos/windows   | wasm32-wasi               | wasm32-wasi-threads |
|-----------------|-----------------------|---------------------------|---------------------|
| `spawn`         | ✓                     | throws `Unsupported`      | ✓ (wasi-threads)    |
| `Builder`       | ✓                     | throws `Unsupported`      | partial             |
| `local`         | ✓                     | ✓ (one thread)            | partial             |
| `park`/`unpark` | futex / WaitOnAddress | ✗                         | partial             |

Desktop OSes have consistent behavior. `wasm32-wasi` has no threads; on `wasm32-wasi-threads` the rest will be filled in as the threads proposal progresses.
