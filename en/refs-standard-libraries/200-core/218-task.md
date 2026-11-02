# 2.18. task - M:N scheduler task API

The `rok::core::task` module provides a cooperative task API running on the M:N scheduler. Inspired by Rust Tokio and Go goroutines, tasks are implemented as **stackless coroutines** on a work-stealing scheduler.

The module `task` and `Task<T>` are in the [prelude](../000-meta/002-import.md), so `task::spawn`, `task::race`, `Task<T>` and the rest are usable without any import; the module prefix `task::` is still kept. Where code should be explicitly marked as asynchronous, `use rok::core::task::*;` is recommended, to make the asynchronous flow visibly separate.

## 2.18.1. Core API

The signatures of the core API:

```rok
fn block_on<F: Future>(f: F) -> F::Output;              // sync entry point: attaches to the executor and runs f to completion
fn try_block_on<T, E>(f: TryAsyncFuture<T, E>) throws (E) -> T;   // block_on for a throwing async body; its error is thrown here
fn spawn<F: Future + Send>(f: F) -> Task<F::Output> where F::Output: Send;
fn spawn_local<F: Future>(f: F) -> Task<F::Output>;     // non-Send future, pinned to the current worker (not an OS thread)
fn spawn_blocking<T: Send>(f: fn() -> T + Send) -> Task<T>;   // blocking work on an OS thread outside the executor
fn spawn_blocking_with_arg<V: Send, T: Send>(arg: V, f: fn(V) -> T + Send) -> Task<T>;   // spawn_blocking that hands arg to f
fn worker_count() -> usize;                               // pool size inside the executor, 1 outside
fn cpu_count() -> usize;                                // number of online CPUs (the pool size)
async fn race<I: IntoIterator<Item = Task<T>>, T>(tasks: I) -> T;   // result of the first task to complete; cancels the others
const fn select<I: IntoIterator<Item = F>, F: Future>(iter: I) -> SelectFuture<F>;   // await: (index, result), does not spawn
fn id() -> TaskId;                                      // id of the current task; panics outside a task
fn current_id() -> TaskId | none;                       // id of the current task; none outside a task
async fn join<I: IntoIterator<Item = Task<T>>, T>(tasks: I) -> List<T>;   // after all complete
async fn timeout<T>(dur: Duration, task: Task<T>) throws (TimeoutError) -> T;   // throws in try await; cancels the task
async fn sleep(dur: Duration);
async fn sleep_until(when: Instant);
async fn yield_now();                                   // await: yields once to other ready tasks
fn registrations() -> usize;                            // diagnostic: parked wakers + pending timers of the running pool (implementation-defined)
```

`sleep` and `timeout` measure their duration from the first poll. A hand-written `Future` waits through its `Waker` ([2.20](220-future.md)): it keeps `ctx.waker()`, returns `Pending`, and whoever changes the state calls `wake()` on it.

The futures and the error these functions return:

```rok
struct SelectFuture<F> { /* ... */ }
impl<F: Future> Future for SelectFuture<F> { type Output = (usize, F::Output); ... }

struct Join<T> { /* ... */ }                        // Task::join, awaited with try await
impl<T> Join<T> {
    fn poll(ref self, ref ctx: Context) throws (CancellationError) -> Poll<T>;
}

struct TimeoutError;                                // task::timeout ran out; message() and Display "timed out"
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

`spawn` takes a future, not a closure, so it is passed an `async` block or the result of an `async fn` call, as in `task::spawn(async { ... })`. A future that is not `Send` runs on the current worker through `spawn_local`. `join` / `race` / `timeout` are `await`ed to obtain their results.

`join`, `race` and `select` take any `IntoIterator` of their tasks or futures, a list literal included.

`race` and `timeout` cancel their losers. Once `race` sees the first task finish (in list order, so a tie goes to the earlier task), it calls `cancel` ([2.18.4](#2184-cooperative-cancellation---taskcancel)) on every other task it was given; once the time of a `timeout` runs out, it cancels its task and throws `TimeoutError`. A loser is then a task cancelled as by `cancel`:

- If it passed a cancellation point, `is_cancelled()` is true, `try await t.join()` throws `CancellationError`, and plain `await t` panics with a message that names the cause: ``awaited a task cancelled by `task::race`; use `try await t.join()` `` (or `task::timeout`).
- If it had already finished, or finished without passing another cancellation point, `await t` and `join` return its own value. A loser never yields the result of the `race`.

Neither waits for the losers' cleanups: they return once cancellation is requested, as `select` does. Code that needs a loser's cleanups finished writes `try await loser.join()`, which completes after them. Cancelling the task that awaits `race` or `timeout` cancels the tasks given to it; a copy of such a `Task` held elsewhere sees that cancellation. `race` of no tasks panics.

### `spawn_blocking` - blocking work

`spawn_blocking(f)` runs `f` on an OS thread of the blocking pool rather than on an executor worker, and returns a `Task<T>` that completes with its result, the same handle `spawn` returns (2.18.2). It is used to wait inside a task on blocking syscalls that cannot be awaited by readiness (regular file I/O, etc., [stdlib 3.1.4a](../300-std/301-fs.md#314a-asynchronous-file-io-_async)).

- `f` starts **immediately on the call**, not at the first poll (if the pool is full, it is queued first).
- There is one blocking pool per process. It adds threads as needed (up to a limit) and ends threads that stay idle for a while. Jobs are processed in arrival order (FIFO). The limit is 512 threads; once reached, jobs queue up. A thread idle for 10 seconds ends. Each job is handed to one idle thread.
- `f` has no cancellation point. `cancel()` before a pool thread takes the job means `f` never runs, and `join` throws `CancellationError`; once `f` runs it finishes, and `await` / `join` return its result.
- If the `await`ing task is cancelled, it leaves its `await` as for any `Task`; the blocking task is not cancelled, and its result stays in the handle.

```rok
let n = await task::spawn_blocking(|| heavy_checksum(path));
```

`f` shares its plain captures with the code that made it, so they must be `Send + Sync` ([refs-lang 4.6.9](../../refs-lang/004-functions/406-capture-semantics.md#469-additional-constraints-in-parallel-and-concurrent-contexts)). A value only the blocking work needs (a collection, a `Command`) is given to it, in either of two forms:

- `[move x]` in `f`'s capture list moves `x` into the closure ([refs-lang 4.6.11](../../refs-lang/004-functions/406-capture-semantics.md#4611-move-capture-the-move-x-capture-list)).
- `spawn_blocking_with_arg(arg, f)` hands `arg` over as a channel send does and calls `f(arg)` on the pool thread: the thread gets a copy of every collection `arg` reaches, and using `arg` afterwards is a compile error ([2.13.9](213-marker.md#2139-send--sync---thread-boundary-markers)). `arg` needs only `Send`.

Either way the spawning code can no longer use the value. A closure that owns a moved capture is handed over too: calling it again after `spawn_blocking(f)` is a compile error.

```rok
let rows: List<str> = load();
let n = await task::spawn_blocking([move rows] || write_all(rows));
let m = await task::spawn_blocking_with_arg(load(), |rows: List<str>| write_all(rows));
```

### `Send` check for `spawn`

`task::spawn` requires the result type to be `Send` as well (`where F::Output: Send`), and rejects a future whose **frame** is not `Send` ([2.13.9](213-marker.md#2139-send--sync---thread-boundary-markers)). This holds wherever the future came from: not only `async` blocks and `async fn` calls but also parameters, fields of structs / tuples / sums, results of helper functions, list elements, futures held by closures, and futures inside other coroutine frames are checked. A future's frame consists of:

- captured values;
- local variables and temporaries live across an `await`;
- other futures the frame holds (their frames too, recursively).

```rok
let c = Cell::new(0);                      // Cell: !Send
task::spawn(async { c.set(1); });          // ERROR: !Send capture
task::spawn_local(async { c.set(1); });    // OK: pinned to the current worker

fn make() -> AsyncFuture<()> { let c = Cell::new(0); async { c.set(1); } }
task::spawn(make());                       // ERROR: the frame of a future made by a helper is checked too
```

The same check applies wherever a closure is sent to another thread: `thread::spawn`, `thread::Builder::spawn`, `par`'s `Scope::spawn`, and `task::spawn_blocking`. If the closure holds a future whose frame is not `Send`, it is the error ``` `X` would move a future to another thread: ... ```.

The check may approximate conservatively. An implementation may reject a future that is actually `Send`, but must never accept a future that passes a non-`Send` value to another thread. The check does not descend into:

- values whose type cannot hold a function value (they cannot hide a future's frame);
- values of types with an explicit `unsafe impl Send` (e.g. `Task<T>`); that type is responsible for safety.

`AsyncFuture<T>`, the result type of an `async fn`, is `Send` when `T: Send`; its throwing counterpart `TryAsyncFuture<T, E>` is `Send` when both `T` and `E` are `Send`. Captures and frames are examined separately by the check above. A frame whose captures are `Send` but not `Sync` (a list an `async fn` takes, a list or a future an `async` block captures) is accepted: `task::spawn` hands the future over like a channel send ([2.13.9](213-marker.md#2139-send--sync---thread-boundary-markers)). The task gets a frame of its own holding copies of what the future captured, made in one hand-off, so what the captures shared (one list twice, a list and a future over it) they still share; using the future or what it captured after the spawn is a compile error (`E0105`), as after a send. Only a future nothing has polled can be handed over: handing over one that has started (polled once by `select`, a user's `Future`, or by hand) or finished, or whose frame or a captured future's frame has a `[ref x]` capture, panics, and nothing is shared. A coroutine future sent through a channel, put in a `Mutex` or a `Frozen` is handed over the same way, a frame of its own each time. Each `await` of a task whose `Task` was handed on (`Frozen::get`, a channel, a lock, a `par` item) gets its own copy of the result; copies of a `Task` on one thread share it.

### `select` - the first future to finish

```rok
const fn select<I: IntoIterator<Item = F>, F: Future>(iter: I) -> SelectFuture<F>;   // Output = (usize, F::Output)

let (i, v) = await task::select([fetch_a(), fetch_b()]);
```

Unlike `race`, it does not spawn the futures; it runs them **inside** the waiting task.

- Futures are polled in list order; if two or more finish in the same poll, the lower index wins.
- When one finishes, all the others that have already started are cancelled. They are polled once more in the cancelled state so that `finally` / `using` run (2.18.4). A loser whose cleanups `await` finishes them in the background, as a task of its own; `select` does not wait for it. A future never polled is never run.
- All wakeup registrations (timers, parking) made by the losing futures are released.
- If the waiting task is cancelled, the child futures are cancelled too.
- No futures panics.
- The futures must have a single type. If the result types differ, wrap them: `async { Msg::A(await a) }`.

A timeout can also be built from `select` and `sleep`.

```rok
enum Ev { Line(str), Timeout }

let (_, ev) = await task::select([
    async { Ev::Line(await read_line()) },
    async { await task::sleep(Duration::secs(5)); Ev::Timeout },
]);
```

### `TaskId`

`task::id()` is the id of the running task; calling it outside a task panics with ``task::id: called outside a task; use `task::current_id()`, which is `none` there``. `Task::id()` is the id of that task. Ids come from a process-wide counter and are never reused.

## 2.18.2. `Task<T>` type

The `Task<T>` type is defined as follows.

```rok
struct Task<T> {
    // internal: coroutine frame on the GC heap + result slot
}

impl<T> Task<T> {
    fn cancel(self);                      // requests cancellation; ends at the next await (2.18.4)
    fn is_cancelled(self) -> bool;
    fn is_finished(self) -> bool;
    fn join(self) -> Join<T>;             // try await t.join(): CancellationError if cancelled without a result
    fn id(self) -> TaskId;
}

struct TaskId { /* u64 */ }
impl TaskId {
    fn as_u64(self) -> u64;
}
// TaskId: Clone / Copy / PartialEq / Eq / Hash / Display / Debug

struct CancellationError;           // message() and Display "task cancelled"

impl<T> Future for Task<T> {              // `await t`: suspend until completion
    type Output = T;
    ...
}
```

### `IntoFuture` integration

`Task<T>` implements `Future<Output = T>`, so it can be awaited with `await t`, and it is covered by the `IntoFuture` blanket impl of [2.20 future](220-future.md), so it is awaited the same way as `thread::JoinHandle` (which has its own `IntoFuture` impl, 3.4). The authoritative definitions of the `IntoFuture` trait and the lang slot `future_trait` are in [2.20 future](220-future.md); this section follows them as is.

## 2.18.3. Scheduler model - work-stealing

A process has one executor. It starts on first use and consists of as many worker threads as there are CPUs (`cpu_count()`), a blocking lane, and an I/O driver. `block_on` attaches the calling thread to this executor as a guest worker and runs `f` to completion. Tasks still running when `block_on` returns keep running. `block_on` returns only after the `spawn_local` tasks pinned to its thread are done too; they cannot move to another thread. `par::*` and `spawn_blocking` use the same executor.

A task created with `spawn` can run on any worker, so it must be `Future + Send` and its result must be `Send` too. A task from `spawn_local` is pinned to the worker that created it. It is pinned to the worker, not to an OS thread: a worker may continue on another OS thread ([refs-lang 7.6.7](../../refs-lang/007-runtime/706-scheduler.md#767-automatic-detection-of-blocking-syscalls)), so a `spawn_local` task should not rely on which OS thread runs it or on that thread's thread-local state. (The reference runtime never hands on a worker with live `spawn_local` tasks.) A `spawn_local` from a task that a `block_on` thread polls but does not own (a `spawn` task it picked up while waiting) is pinned to a pool worker instead, so `block_on` waits only for the locals of its own tasks. All the locals one such poll spawns go to the same pool worker, and they start only once that poll has ended, as on the worker that would have run them.

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

The executor's own types and functions (`TaskRef`, `Pool`, `Worker`, `wake_task`, `cancel_requested`, `unwinding`, `park`, `unpark`) are `pub(pkg)` in `rok::core`: no other package names them. The hooks an `await` calls (`cancel_point`, `cancel_left`, `cleanup_running`, `cleanup_done`) are `pub(pkg)` too; lowered code reaches them through their lang slots ([0.4.4](../000-meta/004-lang-slot-table.md)). `readable` / `writable` (the readiness futures the socket `*_async` methods await, [refs-lang 7.7.2](../../refs-lang/007-runtime/707-io-and-timer.md#772-io-wait-pattern)), `wake_io` (waking the readable and writable waiters of a descriptor after a close or shutdown) and `host_enter` / `host_leave` / `host_run` (the event-loop entry of WebAssembly components) are `#[internal]`: `rok::std` calls them, and a use from any other package is an error ([0.4](../000-meta/004-lang-slot-table.md)). `host_leave` outside `host_enter` panics and says so.

## 2.18.4. Cooperative cancellation - `Task::cancel`

Cooperative cancellation is used as follows.

```rok
let t = task::spawn(async { long_running() });
// ... other work ...
t.cancel();              // cooperative signal; the task ends at a cancellation point
```

`cancel()` **requests** cancellation. The rules ([refs-lang 7.7.9](../../refs-lang/007-runtime/707-io-and-timer.md#779-cancellation)):

- Every `await` is a cancellation point. A cancelled task leaves its body at the next `await` as if by `return`, running the `finally` and `using` cleanups it passes, innermost first.
- An `await` inside those cleanups is not a cancellation point: it completes, and the task is polled until its cleanups end. A cleanup that never ends keeps the cancelled task alive; bound it with `task::timeout`.
- Cancellation is not an error that can be `catch`ed inside the task. No `CancellationError` is thrown into the task.
- A blocking syscall is not a cancellation point. The task leaves at the next `await` after the call finishes.
- A task waiting in socket I/O (`*_async`), `sleep` / `timeout`, `lock_async`, or `recv_async` / `send_async` wakes when cancelled and leaves at the next `await`.
- When a task ends, all its wakeup registrations (I/O readiness, timers, parked waits) are released.
- A task cancelled before its first poll never runs: no part of its body runs, and it ends without a result.
- If cancelled while waiting on `spawn_blocking` or the `fs::*_async` built on it, work already started runs to completion in the background. Side effects such as writes do happen. A result that finished before the task polls again is kept by that poll, as below.
- A poll in which the awaited future handed over a value (a message, a lock guard) keeps that value. The cancellation check looks at "whether the awaited coroutine left because of cancellation"; the value stays and the task leaves at the next `await` ([refs-lang 7.7.9](../../refs-lang/007-runtime/707-io-and-timer.md#779-cancellation)).
- A task cancelled after passing its last cancellation point finishes normally and has a result. `await t` / `join` return that result.
- `CancellationError` is received by whoever waits on a task that ended without a result. Waiting on such a task with `try await t.join()` throws `CancellationError`; waiting with `await t` panics.

```rok
let t = task::spawn(async {
    using log = open_log();                      // dropped here on cancellation
    loop {
        await task::sleep(Duration::secs(1));     // cancellation point
        tick(log);
    }
});
t.cancel();
```

```rok
let r = try await t.join() catch _: CancellationError { fallback() };
```

## 2.18.5. `thread::*` - 1:1 OS threads

The separate `thread` module spawns **OS threads** directly rather than using the M:N model.

```rok
use rok::std::thread;

let t = thread::spawn { os_level_work() };
let r = try t.join();          // OS-level synchronous wait
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

The time types the task functions take are `rok::core::time::Duration` (a signed `i64` nanosecond span) and `rok::core::time::Instant` (a monotonic clock reading), defined in [3.3](../300-std/303-time.md) (3.3.2, 3.3.5). They live beside `task`, not in it: `sleep`, `sleep_until` and `timeout` stay in `task` because they suspend a task.

Calls such as `await task::sleep(Duration::ms(100))` or `await task::sleep_until(t)` are guaranteed accurate monotonic time.

## 2.18.7. Module mapping

The module where each item lives:

| Item                 | Module                                | Behavior                 |
|----------------------|---------------------------------------|--------------------------|
| `task::spawn`        | `rok::core::task` + scheduler runtime | scheduler push           |
| `task::race`         | `rok::core::task`                     | first-completed selector |
| `task::join`         | `rok::core::task`                     | join all                 |
| `task::timeout`      | `rok::core::task`                     | race + timer             |
| `Task<T>`            | `rok::core::task`                     | coroutine frame ref      |
| `Duration`/`Instant` | `rok::core::time`                     | i64 ns + monotonic clock |
| `Future` (lang slot) | `rok::core::future`                   | poll-based               |
| `IntoFuture`         | `rok::core::future`                   | await entry point        |
