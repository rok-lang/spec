# 8.4. Safepoint protocol

For the [GC](../000-meta/007-glossary.md#gc-garbage-collector) to perform temporary STW (Stop-The-World) work, every mutator thread must have reached a safe point. This document defines the [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) protocol for that synchronization.

## 8.4.1. `yield` flag check: fast path

Mutators periodically check the [TLS slot](803-tls-slots.md) `rok_tls_yield_flag` (u8). The check proceeds as follows.

```
poll:
    flag = TLS[yield_flag]
    if flag != 0:
        call rok_runtime_yield()    // slow path
    // proceed normally
```

In LLVM IR:

```mlir
%flag = load i8, i8* @rok_tls_yield_flag
%check = icmp ne i8 %flag, 0
br i1 %check, label %slow, label %cont

slow:
  call void @rok_runtime_yield()
  br label %cont

cont:
  ; ... continue
```

This is a two-instruction fast path plus one branch, and the branch almost always evaluates to `false`.

## 8.4.2. Emission sites

The compiler emits safepoint polls at the following sites.

| Site                  | Reason                    |
|-----------------------|---------------------------|
| Function entry        | Can cooperate with the GC right after a call |
| Loop back-edge        | Prevents an infinite loop from blocking the GC |
| Right after a function call (optional) | Cooperate immediately if the call was long |

The detailed emission policy is up to the compiler; this specification guarantees only the following minimum frequency.

- Every loop back-edge has a poll, and every cycle of calls contains a poll.
- A function that can reach no safepoint (no loop, allocation, indirect call, lazy `static`, or call to a function that may reach one) needs none.

Together these bound the time from a yield request to the next safepoint.

## 8.4.3. `rok_runtime_yield`: slow path

The slow path signature is:

```
extern "C" {
    fn rok_runtime_yield();
}
```

A mutator calls this function to cooperate with the GC. Its effects are:

1. Sets `mutator_state` in the mutator's [Mutator TLS slot layout](803-tls-slots.md) to `GC_COOPERATING`.
2. Exposes the root information the GC needs, such as the [shadow stack](../000-meta/007-glossary.md#shadow-stack) walk.
3. Waits until the GC finishes.
4. When the GC finishes, sets `mutator_state` back to `RUNNING`.

`yield_flag` is cleared automatically while inside `rok_runtime_yield`, or is cleared by the GC.

## 8.4.4. Setting / clearing the `yield` flag

When the GC needs an STW pause:

```
gc_initiate_stw():
  for each mutator thread:
    atomic store TLS[yield_flag] = 1

  // wait until every mutator yields
  for each mutator thread:
    wait until mutator_state == GC_COOPERATING
```

When the GC finishes STW:

```
gc_finish_stw():
  for each mutator thread:
    atomic store TLS[yield_flag] = 0
  notify all (resume)
```

## 8.4.5. Fast path cost

The cost borne by a mutator is a single load of the `yield` flag and one branch.

In almost all cases the flag is 0, so the branch is not taken, and the branch predictor of modern CPUs handles it at nearly zero cost.

For that cost, every thread gets a deterministic cooperative synchronization mechanism.

## 8.4.6. STW phases

STW collection (minor, full with evacuation, `rok_gc_collect`) is a single pause. The thread that started the collection stops all other mutators (`GC_COOPERATING`, or `PARKED` if inside a blocking runtime call), finishes the collection and resumes them.

A full collection with concurrent mark has two pauses, an initial and a final one ([7.5.2](../007-runtime/705-concurrent-collection.md#752-phases)). In between, a single marker thread runs alongside the mutators.

## 8.4.7. Safepoint invariant

When a safepoint is reached, every live GC object variable of the mutator must be registered in the shadow stack.

The compiler guarantees this by linking the shadow stack frame from [Barriers and root tracking](../007-runtime/704-barriers-and-roots.md) directly at function entry. A local holding GC references at a safepoint is registered in the frame, and the registered word is kept current at every write. The compiler may also keep a private copy (in a register), since a collection never rewrites a root word: objects reached directly from roots are not moved ([7.4.1](../007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)).

## 8.4.8. Loop back-edge poll

If a long loop does not cooperate with the GC, STW waits forever. The compiler therefore emits a poll on every loop back-edge.

```
loop:
    body
    poll                              // back-edge poll
    branch loop
```

The cost of this poll is again two instructions and one branch, negligible compared to the loop body.

The poll stays on the back edge. Its slow path (the `rok_runtime_yield` call) may branch to a re-entry block placed before the loop header instead of returning into the edge.

The compiler detects every natural loop and emits a poll for each.

## 8.4.9. Inline functions and polls

When a small function is inlined, its entry poll may disappear with it. The inlined result is absorbed into the caller's body, however, so the caller's entry poll suffices.

When large-scale inlining happens, an LLVM pass adds extra polls automatically (an implementation decision).

## 8.4.10. Native calls

A thread inside C or a blocking runtime call cannot reach a safepoint, so it counts as `PARKED`: the GC treats it as already cooperating.

- The compiler brackets every `extern "C"` call: `PARKED` for the duration of the call, `RUNNING` after it, then a yield-flag check. The caller's argument roots stay in its shadow-stack frame. The fast path is inline; `rok_enter_native` / `rok_leave_native` are the slow paths ([8.5.2](805-extern-symbols.md#852-runtime-area)).
- Every `extern "C" fn` definition starts with `rok_native_reenter` and ends with `rok_native_resume`, so a C callback runs as a mutator.
- The runtime brackets its own blocking calls (I/O wait, `rok_sync_wait`, join, sleep) the same way.
- A call through an `extern "C"` declaration of a `rok_*` symbol ([8.5](805-extern-symbols.md), the runtime's reserved prefix) is not bracketed: the runtime function runs as a mutator.

```
call_extern_c:
    mutator_state = PARKED      // seq_cst
    c_function()
    mutator_state = RUNNING     // seq_cst
    if yield_flag:
        rok_leave_native()      // wait out a running collection
```

## 8.4.11. Direct calls from user code

User code may also (rarely) call `rok_runtime_yield()` directly.

```rok
extern "C" {
    fn rok_runtime_yield();
}

fn long_loop() {
    for i in 0..1_000_000 {
        ...
        if i % 1000 == 0 { rok_runtime_yield(); }
    }
}
```

`rok_runtime_yield` is a `rok_*` symbol, so the call is not bracketed (8.4.10) and the thread yields as a running mutator. The compiler already emits polls on loop back-edges, so users almost never need to call it directly. This pattern is used only in special situations such as inside inline assembly.

## 8.4.12. Stability

The ABI is the slot symbol `rok_tls_yield_flag` and its type (u8). The offset is not ABI ([8.3.1](803-tls-slots.md#831-tls-slot-structure)).

`rok_runtime_yield` follows a simple calling convention with no arguments and no return value.
