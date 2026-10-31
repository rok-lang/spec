# 7.4. Barriers and root tracking

This document defines the [GC](../000-meta/007-glossary.md#gc-garbage-collector)'s mark starting points (roots) and the recording of old → young references (write barrier).

> [!NOTE]
> This section is informative: it describes the reference runtime's GC, which may change in any release. Only the object layout fixed as ABI in [8.1](../008-abi/801-object-header-abi.md) is normative.

## 7.4.1. Root tracking: a shadow stack emitted directly by the compiler

### Choosing a shadow stack

The two most common root-tracking approaches are:

| Approach                    | Advantage                     | Disadvantage                        |
|-----------------------------|-------------------------------|-------------------------------------|
| LLVM statepoint             | Precise stackmaps             | Supports only x86_64 / aarch64 / ppc |
| Shadow stack (direct emit)  | Same code on every target     | Extra stack memory and push/pop     |

Tier 1 of [Platform targets](../000-meta/006-platform-targets.md) includes 32-bit ARM (`armv7`), 32-bit x86 (`i686`) and WASM. LLVM statepoints do not support these, so the only approach applicable to all of Tier 1 is a [shadow stack](../000-meta/007-glossary.md#shadow-stack).

LLVM's `gc "shadow-stack"` strategy (`llvm.gcroot`) is not used. Its chain head, `llvm_gc_root_chain`, is a single process-wide global, so a per-thread shadow stack is impossible with it.

> [!IMPORTANT]
> Rok's root tracking is a shadow stack whose frame push / pop the compiler emits directly. The chain head is the slot at offset 0 of `rok_tls` (`rok_tls_shadow_stack_top`), one per thread ([8.3.8](../008-abi/803-tls-slots.md#838-form-of-compiler-emission)). It works the same way on all Tier 1 targets (including 32-bit ARM / x86 / WASM).

### Structure of the shadow stack

Each OS thread has its own shadow stack: a linked list, via `prev`, of frames placed on each function's machine stack.

```
rok_tls_shadow_stack_top ──► frame N      (current function)
                             ├── prev ──► frame N-1 (caller) ──► ... ──► null
                             ├── count = 2
                             ├── slots[0]  ──► &word (local x)      ──► GC object
                             └── slots[1]  ──► &word (spill slot)   ──► GC object
```

```
ShadowFrame {
    prev:  ptr          // caller's frame (null if none)
    count: usize        // number of slots (pointer width)
    slots: [ptr; count] // addresses of words holding GC references
}
```

`slots` holds not GC reference values but **addresses of words holding GC references**. Such a word is a local variable's stack slot or a spill slot; words inside array elements or `struct` fields are pointed to the same way. So assigning to a variable needs no separate frame update.

### Compiler emit rules

A function needs a frame only if it holds GC references and can reach a safepoint ([8.4.2](../008-abi/804-safepoint.md#842-emission-sites)). Calls to functions that never return (panics) and to runtime functions that neither allocate nor block ([8.5.2](../008-abi/805-extern-symbols.md#functions-that-are-not-safepoints)) do not count. For such functions, the compiler directly emits the following code:

Only locals live across a safepoint need a root.

1. On entry, zero the GC reference words.
2. Reserve a `ShadowFrame` on the machine stack. Before the first safepoint on each path through the function, record the addresses of those words in `slots`, save the current `rok_tls_shadow_stack_top` in `prev`, then set the head to this frame (link). The link may come earlier, up to the function's entry; where paths join, either all of them have linked or none has. A path that reaches no safepoint need not link.
3. Immediately before every `return` on a path that linked the frame, restore the head to `prev` (unlink). An error return is also an ordinary `return`, so it takes the same path. A panic terminates the process, so no unlink is needed.
4. A local live across back-edge polls only ([8.4.8](../008-abi/804-safepoint.md#848-loop-back-edge-poll)), and across no other safepoint, may instead be registered on the poll's slow path alone: in a second frame whose words the slow path writes (null where the local is dead) just before it links that frame and calls the runtime, and which it unlinks right after. Such a word is written nowhere else. A frame of rule 2 not yet linked on that path is linked and unlinked with it. This rests on one rule: a thread stops for a collection only at a safepoint the compiler counts (a call, an allocation, an entry poll) or at a back-edge poll. A runtime function that is not a safepoint ([8.5.2](../008-abi/805-extern-symbols.md#functions-that-are-not-safepoints)) must therefore never wait for a stop: it neither allocates, nor blocks, nor polls.

When a variable is no longer used (by liveness), the word its root slot points to may be cleared to null. This does not change meaning as long as that word is not read again, and keeps the frame from holding dead objects alive.

When marking, the GC starts from every thread's chain head, walks the frames, and marks the value of the word each `slots[i]` points to as a root.

### Objects pointed to by roots are not moved

Objects pointed to directly by roots (shadow stack, `static`s, registered thread roots) are not moved in that collection. So a GC pointer passed to a blocking runtime call or an `extern "C"` function stays in place **as long as the argument is in the roots until the call ends**. If C code keeps the pointer after the call ends, it must be pinned with `rok_gc_pin` ([8.5.1](../008-abi/805-extern-symbols.md#851-gc-area)).

A coroutine frame is a GC heap object ([7.8](708-coroutine.md)), so the variables in it are tracked through the frame object's scan table, not the shadow stack.

### Contract with the collector

Roots (the shadow stack of this section) and the write barrier ([7.4.2](#742-write-barrier-incremental-update)) are emitted as specified here. The collector requires nothing else from compiler output.

### Coexistence with error propagation

Errors of [Error control flow](../003-expressions/314-error-control-flow.md) are delivered through the return value of a throwing function ([8.6.5](../008-abi/806-calling-convention.md#865-return-of-throwing-functions-value-based-error-propagation)). When an error propagates, each frame still exits through the ordinary return path and pops its shadow stack entry, so the shadow stack always matches the actual call stack without any special adjustment for error handling.

## 7.4.2. Write barrier: incremental update

### Why a write barrier is needed

Minor collection does not enter old objects ([7.3.2](703-gc-algorithm.md#732-sticky-mark-bit-generational-collection)). When an old object comes to point to a young object, the write barrier informs the collector. Otherwise that young object would be reclaimed.

### Object-remembering

The barrier checks **only the header of the destination object `p`**. A surviving (old) object has the `UNLOGGED` bit ([8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags-offset-4-u32)) set along with `STICKY`; on the first write to that object, the whole object is recorded in the remembered set.

```
*p = q;
                                // ↓ barrier (may be before the store)
if (p.gc_flags & UNLOGGED) {
    rok_gc_wb_slow(p);          // clear UNLOGGED and put p in the remembered set
}
```

- The header of the stored value `q` (`q.sticky` etc.) is **not read**. `q` may be an inline integer in an `any` slot or an interior pointer of a `str`, so reading it as an object header is unsafe.
- The barrier may be placed before the store.
- The slow path `rok_gc_wb_slow(obj)` clears `UNLOGGED` with an atomic `fetch_and`. During concurrent mark, it also records the object's current references in the SATB buffer ([7.5.3](705-concurrent-collection.md#753-satb-barrier)). So even when several threads write concurrently, only one thread records the object. The slow path must not be a safepoint (a collection must not start inside it).
- A young GC treats objects in the remembered set as roots and marks the young objects they point to. After the collection, `UNLOGGED` is set again on the recorded objects.

### Card marking

Old objects of 8 KiB or more are recorded per 512-byte **card** instead of as a whole object. Such objects have the `CARDED` bit ([8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags-offset-4-u32)) set instead of `UNLOGGED`.

- Such an object has a card table (1 byte per 512 B), allocated after the object. The pointer to that card table is in the word immediately before the header.
- The inline fast path checks `UNLOGGED | CARDED` in `p`'s header at once (one load, one branch).
- Order: if `UNLOGGED`, call the slow path first; then, if `CARDED`, mark the cards. During concurrent mark ([7.5](705-concurrent-collection.md)), an old LOS object may have both bits.
- If `CARDED`, the barrier marks dirty, inline and without a call, the cards the written slot spans, i.e. from `(slot - p) >> 9` to `(slot + len - 1 - p) >> 9`. So the barrier takes the slot address and the size of the stored value (`len`).
- If `UNLOGGED`, it calls `rok_gc_wb_slow(p)` as above (unchanged).
- Minor collection scans only the dirty cards of `CARDED` objects and cleans those cards.

Stores that need no barrier:

| Store                               | Reason                                              |
|-------------------------------------|-----------------------------------------------------|
| Initializing a just-allocated object | No GC between allocation and store, so still young |
| Writing to a `static`               | Statics are roots                                   |
| Raw `Ptr<T>` memory                 | Not tracked by the GC                               |
| Atomics (non-reference values)      | Not GC references                                   |

### Remembered set

Each thread has a 4 KiB log chunk (TLS slots `rok_tls_wb_log_cursor` / `rok_tls_wb_log_limit`). Full chunks, and all chunks at collection or thread exit, are pushed onto a lock-free stack. The collector drains that stack and uses the recorded objects as roots.

### Inline cost

The fast path is one load of `p`'s header and one branch (`UNLOGGED | CARDED`); usually both bits are clear (a young object or already recorded), so it passes straight through.

### Compiler emit

The compiler emits the fast path inline; only the slow path calls a runtime function ([8.5](../008-abi/805-extern-symbols.md)).

```
extern "C" {
    fn rok_gc_wb_slow(obj: Ptr<u8>);   // clear UNLOGGED and add to the remembered set (slow path, not a safepoint)
}
```

## 7.4.3. Safepoint polling

A collection proceeds while all mutator threads are stopped at safepoints. The compiler emits a safepoint poll in every function and loop (8.4.2's exceptions aside) ([8.4](../008-abi/804-safepoint.md)).

```
poll:
    if tls.yield_flag {
        call rok_runtime_yield()       // signal arrival at a GC-safe point
    }
```

`rok_tls_yield_flag` is normally 0. When the collector sets it, each thread stops at its next poll.

## 7.4.4. Walking the shadow stack

When the GC starts marking, it does the following:

```
for each thread T:
    frame = T.rok_tls_shadow_stack_top
    while frame != null:
        for i in 0..frame.count:
            ref = *frame.slots[i]          // slot is the address of a word
            if ref != null:
                mark(ref)
        frame = frame.prev
```

It walks each thread's shadow stack and adds every live GC object variable to the mark queue.

After that, the mark algorithm ([GC algorithm](703-gc-algorithm.md)) starts from those objects and follows their children.

## 7.4.5. Key TLS slots

Each thread has the following TLS slots. Each slot is one `thread_local` global exported by the runtime.

| Slot                        | Meaning                                                                                   |
|-----------------------------|-------------------------------------------------------------------------------------------|
| `rok_tls_yield_flag`        | Whether to `yield` at a safepoint                                                         |
| `rok_tls_tlab_cursor`       | Next allocation position in the [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) |
| `rok_tls_tlab_limit`        | End of the TLAB                                                                           |
| `rok_tls_shadow_stack_top`  | Shadow stack chain head (innermost frame)                                                 |
| `rok_tls_wb_log_cursor`     | Position in the write barrier log                                                         |
| `rok_tls_mutator_state`     | Current mutator state (running / GC-cooperating)                                          |

Detailed TLS slot definitions are in [TLS slots](../008-abi/803-tls-slots.md).
