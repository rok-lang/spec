# 7.7. I/O driver and timers

Workers of the [scheduler](706-scheduler.md) use two mechanisms to handle tasks waiting on I/O and on timers efficiently.

## 7.7.1. I/O multiplexing

Each process has **one reactor thread** that waits on the OS readiness multiplexer and wakes the tasks waiting on ready fds.

| OS / environment      | Mechanism                           |
|-----------------------|-------------------------------------|
| Linux                 | epoll (`EPOLLET`, edge-triggered)   |
| BSD (FreeBSD / macOS) | kqueue (`EV_CLEAR`, edge-triggered) |
| Windows               | IOCP + AFD poll (readiness emulation) |
| WASI (`wasm32-wasi`)  | `wasi:io/poll`                      |
| WASI (`wasm32-wasi-threads`) | `poll_oneoff`                |

The driver for the target is compiled at build time, through the `target_os` branches of [Conditional compilation](../006-meta-system/603-cfg.md).

The reactor thread starts on the first async I/O wait.

On `wasm32-wasi` (a preview 2 component) the driver is `wasi:io/poll`, and `std::net` works through `wasi:sockets` when the host grants network access. On `wasm32-wasi-threads` (preview 1) `poll_oneoff` watches the sockets the host passes in.

## 7.7.2. I/O wait pattern

An I/O wait proceeds as follows:

```
1. The task tries read on a non-blocking socket → WouldBlock.
2. The task does await task::readable(fd) - (re)registers fd with the reactor and parks.
3. The worker runs another task.
4. When the reactor receives a readiness edge for fd, it unparks the task parked on that fd.
5. Some worker takes that task and retries the read.
```

This flow combines with `await` in `async` `fn` (see [Async](../004-functions/412-async.md)). Registration is edge-triggered, so it is re-armed for every wait. The `task::readable(fd)` / `task::writable(fd)` futures check the fd once more after parking so as not to miss an edge that passed in the meantime. A woken task may find the fd not yet ready, so the caller retries the operation.

```rok
fn readable(fd: i32) -> IoReady;     // rok::core::task - Future<Output = ()>
fn writable(fd: i32) -> IoReady;
```

Blocking calls (`TcpStream::read` etc.) do not use this path; they block the calling OS thread. I/O that suspends the task is provided by the standard library's `*_async` methods ([stdlib 3.2.0](../../refs-standard-libraries/300-std/302-net.md#320-common-rules)).

Regular files cannot be waited on with the readiness model (they are always ready under epoll / kqueue). So async I/O on regular files (`fs::*_async`, [stdlib 3.1.4a](../../refs-standard-libraries/300-std/301-fs.md#314a-asynchronous-file-io-_async)) runs the blocking syscall on another OS thread (`task::spawn_blocking`) and suspends the task meanwhile.

## 7.7.3. I/O driver abstraction

Each driver implements a common interface.

```
trait IODriver {
    fn register(ref self, fd: Fd, interest: IoInterest) throws (IoError);   // edge-triggered
    fn poll(ref self, timeout: Duration) -> List<Event>;
}
```

This interface is conceptual; the exact signatures are decided by the runtime implementation. The driver holds no wakers. The task to wake is found in the parking table of `rok::core::task` by a parking key determined by the fd and direction (read / write). For runtime symbols, see [8.5.5](../008-abi/805-extern-symbols.md#855-io--timer-area).

## 7.7.4. Timer structure

Waits of `task::sleep(d)` / `task::timeout(d, ..)` are kept in a binary heap ordered by deadline. When the deadline passes, the task is woken. Timers of cancelled tasks remain in the heap and are discarded on expiry or cleanup. Sleeps whose deadlines have passed by the time timers are processed wake in deadline order, even when several threads process timers at once; a sleep whose deadline has passed at its first poll completes then.

Insertion and expiry are `O(log n)`.

## 7.7.5. Timer precision

The default tick is 1 ms (millisecond). When finer, microsecond-level timing is needed, the library provides separate mechanisms such as busy-waiting or hardware timers.

Millisecond granularity is sufficient for most `async` code, timeouts and sleeps.

## 7.7.6. Integrating timers and I/O

The worker's main `loop` is as follows. For I/O readiness, the reactor thread returns tasks to the queue via the parking table, so workers do not poll the driver directly.

```rok
loop {
    // 1. Process ready tasks from the queue (including tasks woken by the reactor)
    while let task: Task = queue.next_ready() {        // stops on none
        execute(task);
    }

    // 2. Return tasks of expired timers to the queue
    timers.fire_due();

    // 3. If there is nothing to do, park until the next timer (woken by unpark)
    park_until(timers.next_deadline());
}
```

The reactor thread's `loop` is:

```rok
loop {
    let events = io_driver.poll();                     // blocks - a native section for the GC
    for event in events {
        task::unpark(parking_key(event.fd, event.direction));
    }
}
```

## 7.7.7. Park / unpark

When a worker's queue is completely empty and I/O is idle, the worker is parked in an OS sleep.

- Park uses `pthread_cond_wait` or an OS-specific sleep mechanism.
- Unpark is triggered by another worker or an I/O event.

CPU usage is zero while parked, so many idle workers cost nothing extra.

## 7.7.8. Dynamic adjustment of worker count

By default, N is fixed to the number of CPU cores.

When enabled (`ROK_DYNAMIC_WORKERS=1`), an excessively idle worker parks and pushes wake the others first; it is taken back when every active worker is busy. A worker blocked in a call is replaced in every mode ([7.6.7](706-scheduler.md#767-automatic-detection-of-blocking-syscalls)).

## 7.7.9. Cancellation

`Task::cancel` **requests** cancellation ([stdlib 2.18.4](../../refs-standard-libraries/200-core/218-task.md#2184-cooperative-cancellation---taskcancel)). The runtime sets a flag and wakes the task.

- Every `await` is a cancellation point. `await` checks the flag on entry and just before suspending.
- The check after the awaited future returns `Ready` is not "is the flag set" but "**did the awaited coroutine leave because of cancellation**". So a value the future handed over (a received message, an acquired lock guard) is preserved, and the task leaves at the next `await`.
- A cancelled task leaves its body at the next `await` as if by `return`. `finally` blocks passed through and `drop`s of `using` run from the innermost outward. Nested `async fn` calls are cleaned up starting from the innermost coroutine.
- This is not an error that can be `catch`ed inside the task. A non-throwing async body ends by the same path. The result is discarded.
- A task cancelled after its last cancellation point finishes normally and has a result (`await` / `join` return that result). `CancellationError` applies only to tasks that ended without a result.
- Blocking syscalls (`TcpStream::read` etc.) are not cancellation points. The task leaves at the next `await` after the call finishes.
- A task waiting in socket I/O, `readable` / `writable`, `sleep` / `timeout`, `lock_async`, or `recv_async` / `send_async` is woken when cancelled and leaves at the next `await`.
- When a task finishes, all its wake registrations (I/O readiness waits, timers, wakers in the parking table) are released. The number of registrations in the running pool can be inspected with the diagnostic `task::registrations()`; the exact meaning of that value is implementation-defined.

After `task::timeout` expires or `task::race` decides a winner, the losing tasks keep running. To stop them, call `cancel`. `task::select` does not spawn, so it cancels the losing futures itself ([stdlib 2.18.1](../../refs-standard-libraries/200-core/218-task.md#select---the-first-future-to-finish)).

## 7.7.10. Comparison with the reactor pattern

Rok uses a traditional reactor structure: a single separate reactor thread waits on I/O and meets the worker threads only through the parking table. The price is that an I/O wakeup crosses threads (one unpark).
