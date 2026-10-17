# 7. Runtime overview

This block defines the implementation specification of the rok runtime system ([GC](../000-meta/007-glossary.md#gc-garbage-collector), task scheduler, coroutines). The system works identically under both engines ([JIT](../000-meta/007-glossary.md#jit-just-in-time) / [AOT](../000-meta/007-glossary.md#aot-ahead-of-time)) of [Execution engines and the user surface](../000-meta/001-execution-modes.md).

## Contents

The sections of this chapter are:

- [7.1. Object layout and memory spaces](701-object-layout.md)
- [7.2. Allocation](702-allocation.md)
- [7.3. GC algorithm](703-gc-algorithm.md)
- [7.4. Barriers and root tracking](704-barriers-and-roots.md)
- [7.5. Concurrent mark](705-concurrent-collection.md)
- [7.6. Scheduler](706-scheduler.md)
- [7.7. I/O driver and timers](707-io-and-timer.md)
- [7.8. Coroutines](708-coroutine.md)

## Key decisions

The decisions for the runtime's core components are summarized as:

```
GC          in-house Immix + Sticky Mark-Bit generations
Scheduler   M:N work-stealing
Coroutine   compiler-generated state machine (stackless)
```

All three are implemented in-house, with zero dependencies on external libraries.

## Zero-dependency principle

The runtime does not depend on external libraries. Specifically, it does not use:

- external GC libraries (Boehm GC, MMTk, etc.);
- external async runtimes;
- external coroutine libraries.

Platform system interfaces (`libc`, OS headers, etc.) do not count as dependencies, since they are an essential part of the OS interface.

## M:N model

The scheduler follows an M:N hybrid model.

```
OS Thread (worker) x N  ──── rok task / coroutine x M
                                   (typically M >> N)
```

The default N is the number of CPU cores and can be adjusted by runtime configuration. One OS thread performs several roles: Scheduler Worker, Mutator, I/O Driver and GC Worker. Currently, however, the I/O driver is handled by a single separate reactor thread ([7.7](707-io-and-timer.md)).

With this integration, all runtime activity is handled in one thread pool without the cost of separate threads.

## Single .a output

The runtime is distributed as a single staticlib, `librokr.a`. Building a user package links `librokr.a`, producing an executable binary with no further external dependencies.

## Runtime error: stack overflow

When a thread's stack overflows, the runtime flushes stdout (if possible), prints the following to stderr, and aborts (SIGABRT). Unlike a panic, there is no unwinding and `finally` does not run.

```
thread '<name>' has overflowed its stack
```

`<name>` is the thread's name: the name given with `thread::Builder::name` ([stdlib 3.4](../../refs-standard-libraries/300-std/304-thread.md)), which is also set as the OS thread name. The main thread is `main`, and an unnamed thread is `<unnamed>`. This applies to the main thread, `thread::spawn` threads, and executor workers (including deep `await` chains). The detection mechanism (guard range and `sigaltstack`) and TLS slots are in [8.5.7](../008-abi/805-extern-symbols.md#stack-overflow).

## Interface with the compiler

The runtime communicates with compiler-generated code through a fixed [ABI](../000-meta/007-glossary.md#abi-application-binary-interface). The interface consists of three elements:

- The object layout (16-byte header + payload).
- `extern "C"` functions such as `rok_gc_alloc_slow`, `rok_runtime_yield`, `rok_thread_spawn`.
- Coroutines are compiler-generated state machines and do not use `llvm.coro.*` ([7.8](708-coroutine.md)). For GC roots, the compiler emits shadow stack frames directly, without intrinsics ([7.4.1](704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)).

This ABI is defined in detail in [8. Runtime ABI / MLIR lowering](../008-abi/README.md); this chapter covers only internal behavior.
