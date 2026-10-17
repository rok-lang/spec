# 2.18. task — M:N scheduler task API

The `rok::core::task` module provides a cooperative task API running on the M:N scheduler. Inspired by Rust Tokio and Go goroutines, tasks are implemented as **stackless coroutines** on a work-stealing scheduler.

Because `use rok::core::**;` applies implicitly, `task::spawn`, `task::race`, `Task<T>` and the rest are usable without any import; the module prefix `task::` is still kept. Where code should be explicitly marked as asynchronous, `use rok::core::task::*;` is recommended, to make the asynchronous flow visibly separate.

## 2.18.1. Core API

The signatures of the core API:

```rok
fn block_on<F: Future>(f: F) -> F::Output;              // sync entry point — attaches to the executor and runs f to completion
fn spawn<F: Future + Send>(f: F) -> Task<F::Output> where F::Output: Send;
fn spawn_local<F: Future>(f: F) -> Task<F::Output>;     // non-Send future — pinned to the current worker
fn spawn_blocking<T: Send>(f: fn() -> T + Send) -> Blocking<T>;   // blocking work on an OS thread outside the executor
fn workers() -> usize;                                  // pool size inside the executor, 1 outside
fn cpu_count() -> usize;                                // number of online CPUs — the pool size
async fn race<T>(tasks: List<Task<T>>) -> T;            // result of the first task to complete
fn select<F: Future>(futs: List<F>) -> SelectFuture<F>; // await — (index, result), does not spawn
fn id() -> TaskId;                                      // id of the current task — panics outside a task
fn try_id() -> TaskId | none;
async fn join<T>(tasks: List<Task<T>>) -> List<T>;      // after all complete
async fn timeout<T>(dur: Duration, task: Task<T>) throws (TimeoutError) -> T;   // throws in try await
async fn sleep(dur: Duration);
async fn sleep_until(when: Instant);
async fn yield_now();                                   // await — yields once to other ready tasks
fn registrations() -> usize;                            // diagnostic — parked wakers + pending timers of the running pool (implementation-defined)
fn readable(fd: i32) -> IoReady;                       // await — until fd becomes readable (refs-lang 7.7.2)
fn writable(fd: i32) -> IoReady;
```

### Example

```rok
use rok::core::task::*;

let t = task::spawn(async { compute_heavy() });
let result = await t;

let results = await task::join([
    task::spawn(async { fetch("a") }),
    task::spawn(async { fetch("b") }),
    task::spawn(async { fetch("c") }),
]);

let first = await task::race([
    task::spawn(async { source_a() }),
    task::spawn(async { source_b() }),
]);

await task::sleep(Duration::ms(100));
await task::yield_now();
```

`spawn` takes a future, not a closure, so it is passed an `async` block or the result of an `async fn` call, as in `task::spawn(async { ... })`. A future that is not `Send` runs on the current thread through `spawn_local`. `join` / `race` / `timeout` are `await`ed to obtain their results.

### `spawn_blocking` — blocking work

```rok
struct Blocking<T> { /* result slot */ }
impl<T> Future for Blocking<T> { type Output = T; ... }
```

`spawn_blocking(f)` runs `f` on an OS thread of the blocking pool rather than on an executor worker, and returns a future that completes with its result. It is used to wait inside a task on blocking syscalls that cannot be awaited by readiness (regular file I/O, etc., [stdlib 3.1.4a](../300-std/301-fs.md#314a-asynchronous-file-io-_async)).

- `f` starts **immediately on the call**, not at the first poll (if the pool is full, it is queued first).
- There is one blocking pool per process. It adds threads as needed (up to a limit) and ends threads that stay idle for a while. Jobs are processed in arrival order (FIFO). The limit is 512 threads; once reached, jobs queue up. A thread idle for 10 seconds ends. Each job is handed to one idle thread.
- Even if the `await`ing task is cancelled, `f` runs to completion; only its result is discarded.

```rok
let n = await task::spawn_blocking(|| heavy_checksum(path));
```

### `Send` check for `spawn`

`task::spawn` requires the result type to be `Send` as well (`where F::Output: Send`), and rejects a future whose **frame** is not `Send` ([2.13.9](213-marker.md#2139-send--sync--thread-boundary-markers)). This holds wherever the future came from: not only `async` blocks and `async fn` calls but also parameters, fields of structs / tuples / sums, results of helper functions, list elements, futures held by closures, and futures inside other coroutine frames are checked. A future's frame consists of:

- captured values;
- local variables and temporaries live across an `await`;
- other futures the frame holds (their frames too, recursively).

```rok
let c = Cell::new(0);                      // Cell: !Send
task::spawn(async { c.set(1); });          // ERROR — !Send capture
task::spawn_local(async { c.set(1); });    // OK — pinned to the current worker

fn make() -> AsyncFuture<()> { let c = Cell::new(0); async { c.set(1); } }
task::spawn(make());                       // ERROR — the frame of a future made by a helper is checked too
```

The same check applies wherever a closure is sent to another thread: `thread::spawn`, `thread::Builder::spawn`, `par`'s `Scope::spawn`, and `task::spawn_blocking`. If the closure holds a future whose frame is not `Send`, it is the error ``` `X` would move a future to another thread: … ```.

The check may approximate conservatively. An implementation may reject a future that is actually `Send`, but must never accept a future that passes a non-`Send` value to another thread. The check does not descend into:

- values whose type cannot hold a function value (they cannot hide a future's frame);
- values of types with an explicit `unsafe impl Send` (e.g. `Task<T>`); that type is responsible for safety.

`AsyncFuture<T>`, the result type of an `async fn`, is `Send` when `T: Send`; its throwing counterpart `TryAsyncFuture<T, E>` is `Send` when both `T` and `E` are `Send`. Captures and frames are examined separately by the check above.

### `select` — the first future to finish

```rok
fn select<F: Future>(futs: List<F>) -> SelectFuture<F>;   // Output = (usize, F::Output)

let (i, v) = await task::select([fetch_a(), fetch_b()]);
```

Unlike `race`, it does not spawn the futures; it runs them **inside** the waiting task.

- Futures are polled in list order; if two or more finish in the same poll, the lower index wins.
- When one finishes, all the others that have already started are cancelled. They are polled once more in the cancelled state so that `finally` / `using` run (2.18.4). A future never polled is never run.
- All wakeup registrations (timers, parking) made by the losing futures are released.
- If the waiting task is cancelled, the child futures are cancelled too.
- An empty list panics.
- The futures must have a single type. If the result types differ, wrap them: `async { Msg::A(await a) }`.

A timeout can also be built from `select` and `sleep`.

```rok
enum Ev { Line(str), Timeout }

let (_, ev) = await task::select([
    async { Ev::Line(await read_line()) },
    async { await task::sleep(Duration::sec(5)); Ev::Timeout },
]);
```

### `TaskId`

`task::id()` is the id of the running task; calling it outside a task panics (`try_id()` returns `none`). `Task::id()` is the id of that task. Ids come from a process-wide counter and are never reused.

## 2.18.2. `Task<T>` type

The `Task<T>` type is defined as follows.

```rok
struct Task<T> {
    // internal: coroutine frame on the GC heap + result slot
}

impl<T> Task<T> {
    fn cancel(self);                      // requests cancellation — ends at the next await (2.18.4)
    fn is_cancelled(self) -> bool;
    fn is_finished(self) -> bool;
    fn join(self) -> Join<T>;             // try await t.join() — CancellationError if cancelled without a result
    fn id(self) -> TaskId;
}

struct TaskId { /* u64 */ }
impl TaskId {
    fn as_u64(self) -> u64;
}
// TaskId: PartialEq / Eq / Hash / Display / Debug

struct CancellationError {}

impl<T> Future for Task<T> {              // `await t` — suspend until completion
    type Output = T;
    ...
}
```

### `IntoFuture` integration

`Task<T>` implements `Future<Output = T>`, so it can be awaited with `await t`, and it is covered by the `IntoFuture` blanket impl of [2.20 future](220-future.md), so it is awaited the same way as `Thread`. The authoritative definitions of the `IntoFuture` trait and the lang slot `future_trait` are in [2.20 future](220-future.md); this section follows them as is.

## 2.18.3. Scheduler model — work-stealing

A process has one executor. It starts on first use and consists of as many worker threads as there are CPUs (`cpu_count()`), a blocking lane, and an I/O driver. `block_on` attaches the calling thread to this executor as a guest worker and runs `f` to completion. Tasks still running when `block_on` returns keep running. `par::*` and `spawn_blocking` use the same executor.

A task created with `spawn` can run on any worker, so it must be `Future + Send` and its result must be `Send` too. A task from `spawn_local` is pinned to the worker that created it.

The scheduler skeleton is as follows.

```
Worker 0 ─┐      Worker 1 ─┐    ...    Worker N-1 ─┐
 Local     │      Local     │            Local      │
 deque     │      deque     │            deque      │
 (Chase-Lev)      (Chase-Lev)            (Chase-Lev)│
           └─ Global Injector Queue ──────┘
           └─ I/O Driver (portable) ──────┘
           └─ Timer (heap)  ───────────────┘
           └─ Parked Worker Stack   ──────┘
```

Key properties of this structure: the local queue is a lock-free Chase-Lev deque, and both the owning worker and thieves take the oldest task first (so a task that keeps re-waking itself does not starve the others, [refs-lang 7.6.2](../../refs-lang/007-runtime/706-scheduler.md#762-work-stealing-structure)). A busy worker processes expired timers every 32 tasks. For the I/O driver, one reactor thread per process waits on epoll / kqueue edge-triggered (io_uring / IOCP are future work, [refs-lang 7.7](../../refs-lang/007-runtime/707-io-and-timer.md)); the timer structure is implementation-defined (binary heap or timing wheel, [refs-lang 7.7.4](../../refs-lang/007-runtime/707-io-and-timer.md#774-timer-structure)).

Note that the standard library's authority is limited to the API signatures; the algorithm bodies belong to the runtime and standard library body, outside the spec.

## 2.18.4. Cooperative cancellation — `Task::cancel`

Cooperative cancellation is used as follows.

```rok
let t = task::spawn(async { long_running() });
// ... other work ...
t.cancel();              // cooperative signal — the task ends at a cancellation point
```

`cancel()` **requests** cancellation. The rules ([refs-lang 7.7.9](../../refs-lang/007-runtime/707-io-and-timer.md#779-cancellation)):

- Every `await` is a cancellation point. A cancelled task leaves its body at the next `await` as if by `return`, running the `finally` and `using` cleanups it passes, innermost first.
- Cancellation is not an error that can be `catch`ed inside the task. No `CancellationError` is thrown into the task.
- A blocking syscall is not a cancellation point. The task leaves at the next `await` after the call finishes.
- A task waiting in socket I/O (`*_async`), `readable` / `writable`, `sleep` / `timeout`, `lock_async`, or `recv_async` / `send_async` wakes when cancelled and leaves at the next `await`.
- When a task ends, all its wakeup registrations (I/O readiness, timers, parked waits) are released.
- If cancelled while waiting on `spawn_blocking` or the `fs::*_async` built on it, work already started runs to completion in the background and only the result is discarded. Side effects such as writes do happen. A result that finished in the meantime is dropped too: a cancelled poll returns `Pending`.
- A poll in which the awaited future handed over a value (a message, a lock guard) keeps that value. The cancellation check looks at "whether the awaited coroutine left because of cancellation"; the value stays and the task leaves at the next `await` ([refs-lang 7.7.9](../../refs-lang/007-runtime/707-io-and-timer.md#779-cancellation)).
- A task cancelled after passing its last cancellation point finishes normally and has a result. `await t` / `join` return that result.
- `CancellationError` is received by whoever waits on a task that ended without a result. Waiting on such a task with `try await t.join()` throws `CancellationError`; waiting with `await t` panics.

```rok
let t = task::spawn(async {
    using log = open_log();                      // dropped here on cancellation
    loop {
        await task::sleep(Duration::sec(1));     // cancellation point
        tick(log);
    }
});
t.cancel();
```

```rok
let r = try await t.join() catch _: CancellationError { fallback() };
```

## 2.18.5. `thread::*` — 1:1 OS threads

The separate `thread` module spawns **OS threads** directly rather than using the M:N model.

```rok
use rok::core::thread::*;

let t = thread::spawn { os_level_work() };
let r = t.join();              // OS-level synchronous wait
```

The two models compare as follows.

| Aspect                  | `task::spawn`              | `thread::spawn`         |
|-------------------------|----------------------------|-------------------------|
| Model                   | M:N (user-space scheduler) | 1:1 (OS thread)         |
| Cost                    | light (~1KB frame)         | heavy (~MB stack)       |
| Cooperative cancellation | ✓                         | ✗ (needs OS-level kill) |
| async integration       | ✓                          | △ (only the joiner awaits) |
| Used for                | general concurrency (default) | CPU-bound, blocking I/O |

Use `task::spawn` by default, and choose `thread::spawn` only when an OS thread is explicitly required.

## 2.18.6. `Duration` / `Instant`

`Duration` and `Instant`, for representing time, are defined as follows.

```rok
struct Duration { /* i64 in ns */ }
struct Instant { /* monotonic clock */ }

impl Duration {
    fn ns(n: i64) -> Duration;
    fn us(n: i64) -> Duration;
    fn ms(n: i64) -> Duration;
    fn sec(n: i64) -> Duration;
    fn min(n: i64) -> Duration;
    fn hours(n: i64) -> Duration;
}

impl Instant {
    fn now() -> Instant;
    fn elapsed(self) -> Duration;
}
```

Calls such as `await task::sleep(Duration::ms(100))` or `await task::sleep_until(t)` are guaranteed accurate monotonic time. Both types live in `rok::core::task::time` (2.18.7), and `rok::std::time` re-exports them.

## 2.18.7. Module mapping

The module where each item lives:

| Item                 | Module                                | Behavior                 |
|----------------------|---------------------------------------|--------------------------|
| `task::spawn`        | `rok::core::task` + scheduler runtime | scheduler push           |
| `task::race`         | `rok::core::task`                     | first-completed selector |
| `task::join`         | `rok::core::task`                     | join all                 |
| `task::timeout`      | `rok::core::task`                     | race + timer             |
| `Task<T>`            | `rok::core::task`                     | coroutine frame ref      |
| `Duration`/`Instant` | `rok::core::task::time`               | i64 ns + monotonic clock |
| `Future` (lang slot) | `rok::core::future`                   | poll-based               |
| `IntoFuture`         | `rok::core::future`                   | await entry point        |
