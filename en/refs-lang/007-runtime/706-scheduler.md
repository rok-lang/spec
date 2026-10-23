# 7.6. Scheduler

rok's scheduler is M:N [work-stealing](../000-meta/007-glossary.md#work-stealing); this document defines its structure.

## 7.6.1. Unified worker model

A process has one executor, started on first use. Workers are as many OS threads as there are CPUs, plus a blocking lane and an I/O driver. `task::block_on` attaches the calling thread as a guest worker; remaining tasks keep running after it returns. `par::*` and `spawn_blocking` use the same executor ([stdlib 2.18.3](../../refs-standard-libraries/200-core/218-task.md#2183-scheduler-model---work-stealing)). Each worker combines these roles:

```
worker = {
    scheduler   // task queue (work-stealing)
    mutator     // TLAB + shadow stack (701 / 702 / 704)
}
```

Workers do not double as the I/O driver; a single reactor thread per process handles it ([7.7.1](707-io-and-timer.md#771-io-multiplexing)). STW collection is run by the thread that started the collection, and concurrent mark by the marker thread (7.6.10).

## 7.6.2. Work-stealing structure

Each worker has a local deque (lock-free Chase-Lev deque, [8.5.4 `rok_deque_*`](../008-abi/805-extern-symbols.md#rok_deque_---work-stealing-deque)).

```
Worker:
  ├── local_deque : growable ring    (this worker's tasks, GC-traced)
  └── ...
```

A newly woken or spawned task goes to the bottom of that worker's deque. **Both the owning worker and thieves take the oldest task (top) first.** If the owner took the newest task first, a task that keeps re-waking itself (such as one looping on `await task::yield_now()`) could starve the tasks queued behind it.

This is a work-stealing model in which a worker usually runs the tasks it spawned, preserving cache locality, while idle workers steal work, providing load balancing. An implementation may add "most recent task first" optimizations such as a LIFO slot, as long as the starvation above cannot occur.

## 7.6.3. Global injector queue

Besides each worker's local queue, there is a separate global injector queue.

```
Global injector:
  - receives tasks spawned from outside (e.g. extern callback).
  - when a worker's local queue is empty - pull from global.
```

This queue is the entry point for tasks arriving from outside the worker pool.

## 7.6.4. Steal algorithm

When a worker's local deque is empty, it proceeds in this order:

```
1. Try to pull from the global injector.
2. Try to steal from the top of another worker's local deque. Victims start at a per-worker pseudo-random index, then go round-robin.
3. If all fail - sleep / park (blocking sync).
```

Steal tries up to N-1 workers; if all are empty, the worker switches to the parked state.

## 7.6.5. Keywords vs library functions

Following the principle of minimizing reserved keywords in [Keywords](../001-lexical/105-keywords.md), scheduler functions are all provided as library functions.

| Library function        | Meaning                                                                  |
|-------------------------|--------------------------------------------------------------------------|
| `task::spawn(fut)`      | Push the future onto the queue as a new task. Result `Task<T>`.          |
| `task::yield_now()`     | Voluntary yield (`await`). Lets other ready tasks run once.              |
| `task::sleep(d)`        | Move to the timer queue.                                                 |
| `task::race([...])`     | The task that finishes first (`List<Task<T>>`, await).                   |
| `task::join([...])`     | Until all finish (`List<Task<T>>`, await).                               |
| `task::timeout(d, t)`   | Time limit. If exceeded, `TimeoutError` at `try await`.                  |
| `thread::spawn(f)`      | Create a new OS thread (1:1).                                            |
| `par::for_each(xs, f)`  | Parallel for-each ([block](../000-meta/007-glossary.md#block) on join). |
| `par::scope { ... }`    | Parallel scope (trailing closure).                                       |

Since these are not keywords, identifiers such as `spawn` or `race` can be used freely in user code.

## 7.6.6. 1:1 OS threads: `thread::spawn`

Most tasks run M:N as mapped by the scheduler, but for blocking work or external library calls a dedicated OS thread is sometimes the natural fit. In that case `thread::spawn` creates a real OS thread mapped 1:1.

Internally it directly calls Unix `pthread_create`, Windows `CreateThread`, etc., and runs in a pool separate from ordinary tasks. The result is returned as `JoinHandle<T>`, a type distinct from Future.

## 7.6.7. Automatic detection of blocking syscalls

The work-stealing model has the problem that when a task calls a blocking syscall, the worker itself stalls. This is solved by automatic offloading to a blocking pool.

The compiler and runtime automatically send known blocking syscalls to a blocking pool worker, which is a separate OS thread and so does not hinder other tasks' progress. The detailed mechanism is left to the scheduler implementation.

> [!NOTE]
> In the reference runtime, a pool thread whose task stays in a blocking call hands its worker to a new thread. Tasks spawned with `spawn_local` pin their worker, which is never handed on.

## 7.6.8. Thread-per-core option

By default there are N workers and tasks move freely between threads. With the thread-per-core option enabled, each task is pinned to a fixed worker and workers do not steal from each other; cache locality is maximized, suiting NUMA environments and datacenter workloads.

It is toggled with the environment variable `ROK_THREAD_PER_CORE=1`; `ROK_WORKERS=<n>` sets N (default: the online CPUs).

> [!NOTE]
> A task is pinned to a worker; a worker whose task blocks may continue on another OS thread ([7.6.7](#767-automatic-detection-of-blocking-syscalls)), so thread identity is not stable for a pinned task.

## 7.6.9. Priority / fairness

The default is priority-less FIFO + work-stealing. Both the owning worker and thieves process the oldest task in the deque first (7.6.2).

A busy worker does not defer timers either. Every 32 tasks executed, it processes expired timers and returns their tasks to the queue. So `task::sleep` / `task::timeout` wake on time even on a worker that always has work.

Explicit priorities are out of scope for this specification and are covered by a separate surface of the standard library.

## 7.6.10. Integrated GC worker role

STW collection is run by the thread that started it through allocation or `rok_gc_collect`, while other workers stop and wait at safepoints ([7.4.3](704-barriers-and-roots.md#743-safepoint-polling)). After the collection, workers return to running tasks. Concurrent mark is done by a single separate marker thread ([7.5](705-concurrent-collection.md)).

## 7.6.11. Scheduler and coroutines

The scheduler uses the stackless coroutines of [Coroutines](708-coroutine.md); each task's state machine is stored in a coroutine frame.

When a task hits `await` or `yield`, only the coroutine frame goes back into the queue and the worker continues with another task; when the task resumes, a different worker may take the frame and continue it. This model lets millions of tasks be handled efficiently on N workers.

## 7.6.12. Spawn cost

`task::spawn` completes by merely allocating a stackless coroutine frame. The frame's size depends on the task's local variables, roughly 64–256 bytes, and it is allocated on the [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) fast path, so the cost is near zero.

As a result, even millions of concurrent tasks are incomparably lighter than OS threads.

## 7.6.13. Overall flow

```
user code:    task::spawn(async { compute() })
  ↓
compiler:     create coroutine frame + push to queue
  ↓
worker:       pop task from deque/lifo and execute
  ↓ (on await / yield)
worker:       requeue frame, run another task
  ↓ (some worker)
resume:       pop frame and continue execution
  ↓ (on normal termination)
result:       delivered as Output of Task<T> (Task<T>: Future<Output = T>)
```
