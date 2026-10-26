# 7.8. Coroutines

[Generators](../004-functions/411-generators.md), [async](../004-functions/412-async.md) and [streams](../004-functions/413-stream.md) use the same coroutine lowering.

## 7.8.1. State machine lowering

Rok's coroutines are stackless and are lowered to state machines built by the compiler itself. LLVM coroutine intrinsics (`llvm.coro.*`) are not used, because the compiler must know the frame layout to build the frame's ScanTable. There is no per-arch context-switch code; it is the same on all Tier 1 targets.

```
yield / async / async yield block
  → lifted into a body function (HIR, like a closure)
  → state machine (MIR): resume function + frame type
  → ordinary MLIR / LLVM
```

`yield fn` / `async fn` is an `fn` that wraps its body in such a block and returns a wrapper type.

## 7.8.2. Coroutine frame

```
Frame {
    state  : i32        // suspend point to continue from next
    locals : ...        // only locals live across suspend points
}
```

- A frame is an ordinary GC object, created by the allocation fast / slow path ([7.2](702-allocation.md)). Each type has a frame [ScanTable](../008-abi/802-scan-table.md).
- While suspended, the frame's GC references are tracked by the ScanTable; while the resume function runs, its locals are tracked by the shadow stack ([7.4.1](704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)).
- Frames of long-lived tasks become old through the sticky mark-bit ([7.3.2](703-gc-algorithm.md#732-sticky-mark-bit-generational-collection)).
- The call result (`GenIter<T>`, `AsyncFuture<T>`, `AsyncStream<T>` and their `Try*` variants) is a value wrapping the frame ([4.1.11](../004-functions/401-function-declaration.md#result-types-of-coroutine-functions)).

## 7.8.3. Generator

```
1. Call count() → allocate frame, return wrapper.
2. next() → resume function continues from state. At yield, saves the value and state and returns.
3. Next next() → continues from the saved state. When finished, none.
```

The caller drives progress (`for x in g { .. }`).

## 7.8.4. Async

```
1. Call fetch() → allocate frame, return Future.
2. poll → resume function proceeds. If an awaited future is Pending, saves state and returns Pending.
3. Wake (parking registry, I/O driver, timer) → the task re-enters the queue and a worker polls it.
```

The scheduler and runtime drive progress ([7.6](706-scheduler.md)).

## 7.8.5. Stream

`async yield` combines the two. Within the same frame, `yield` produces a value and `await` suspends with Pending.

## 7.8.6. Allocation elision

A frame is allocated on every coroutine call. There is no dedicated elision for a coroutine run to completion by its caller; a frame that never escapes may be removed like any allocation ([7.2.12](702-allocation.md#7212-allocation-elision)).

## 7.8.7. Lowering of `yield` / `await`

| Keyword        | Lowering                                                                  |
|----------------|---------------------------------------------------------------------------|
| `yield expr;`  | Save state + return value (Iterator) / return `Ready(value)` (Stream)     |
| `await expr;`  | Poll the target future. If Pending, save state + return `Pending`         |
| `return expr;` | End the state machine + return the result                                 |

## 7.8.8. Runtime symbols

Waiting and waking for `await` are handled by the parking registry of `rok::core::task` (hooks installed by `rok::core::task` via `rok_task_hooks_install`, [8.5.4](../008-abi/805-extern-symbols.md#854-scheduler-area)). The runtime's `rok_coro_frame_alloc` / `rok_coro_park` / `rok_coro_unpark` are for C and low-level code; the compiler does not use them ([8.5](../008-abi/805-extern-symbols.md)).

## 7.8.9. Stack depth

Nested `await`s proceed through native calls in which the outer resume function calls the inner future's poll. So the depth of `await` recursion is bound by the native stack, and very deep recursion causes a stack overflow ([Runtime error](README.md#runtime-error-stack-overflow)). If deep recursion is needed, split it into tasks.

## 7.8.10. Lifetime of a coroutine frame

A frame lives as long as it is reachable, like any other GC object. A running task is referenced by the task queue, a waiting task by the parking registry, and a waiter on the result by its own frame. When nothing references it, the GC reclaims it. There is no explicit deallocation.
