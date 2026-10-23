# 7.2. Allocation

This document covers how the objects defined in [Object layout](701-object-layout.md) are placed in memory. The allocation algorithm is defined in two stages: a fast path and a slow path.

## 7.2.1. TLAB (Thread-Local Allocation Buffer)

Each scheduler worker (= OS thread) has its own TLAB. A TLAB consists of one [block](../000-meta/007-glossary.md#block) (or part of one) obtained from the [Immix](../000-meta/007-glossary.md#immix) space, and allocates quickly by bump pointer.

```
TLAB structure:
  cursor : ptr      // next allocation position
  limit  : ptr      // end of the TLAB
  block  : ptr      // start (for reuse)
```

While cursor is below limit, there is enough space. If cursor plus size exceeds limit, the [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) is exhausted and allocation falls to the slow path.

## 7.2.2. Fast path: 4 inline instructions

> [!IMPORTANT]
> Most allocations are handled by inline code. They finish in 4–5 machine instructions, with zero synchronization cost.

```
1. tmp = cursor
2. new_cursor = tmp + size
3. if new_cursor > limit → goto slow
4. cursor = new_cursor
   (return tmp)
```

This path completes in 4–5 machine instructions. The branch is almost always `false`, following the fast path, and since the TLAB itself is thread-local, synchronization cost is zero.

The fast path is inline code the compiler generates directly in LLVM IR; the runtime does not expose it as a function call.

## 7.2.3. Slow path: `rok_gc_alloc_slow`

Used when the TLAB is exhausted or the object is large ([LOS](../000-meta/007-glossary.md#los-large-object-space)).

```
extern "C" {
    fn rok_gc_alloc_slow(size: usize, align: usize, type_id: u32) -> Ptr<u8>;
}
```

This is an `extern "C"` function provided by the runtime; the compiler calls it when the fast path fails.

The slow path does the following:

1. If the object is 8 KiB or more, or its alignment is greater than 8 bytes, allocate it in the LOS (7.2.9).
2. An object larger than one line that does not fit in the remaining TLAB is allocated in its own line range.
3. Otherwise, refill the TLAB. If needed, run a [GC](../000-meta/007-glossary.md#gc-garbage-collector) collection first (7.2.8).
4. Initialize the header's `type_id` and `gc_flags`.

## 7.2.4. Object header initialization

The object header is initialized in the slow path; on the fast path too, the compiler fills in the header inline before use.

```
*(ptr + 0) = type_id
*(ptr + 4) = 0u32          // initial gc_flags
*(ptr + 8) = size          // gc_word - total object size (8.1.4)
```

The payload is already zero, and user code fills it with field values. The collector may keep its own bookkeeping before the header or in a side table.

## 7.2.5. Alignment

Every allocation is guaranteed 8-byte alignment. The TLAB cursor is kept at a multiple of 8 bytes, and object sizes are rounded up to a multiple of 8 bytes. Objects requiring larger alignment (`#[repr(align(N))]`) are handled on the slow path.

## 7.2.6. TLAB

A TLAB is a contiguous range of free lines within an Immix block. On refill, the range is zeroed before being handed out, so the fast path receives zero-filled memory.

## 7.2.7. TLABs and work-stealing

The work-stealing mechanism of the [scheduler](706-scheduler.md) lets one worker steal another worker's task.

A stolen task allocates from the stealing worker's TLAB, independent of the original worker's TLAB.

This model works because each TLAB is bound thread-locally to its OS thread.

## 7.2.8. GC trigger

The slow path starts a collection in these cases. The conditions and the choice between minor and full follow [7.3.2](703-gc-algorithm.md#732-sticky-mark-bit-generational-collection).

- When a new TLAB or block cannot be handed out.
- When the amount allocated since the last collection exceeds the next cycle's budget.

`rok_gc_collect` explicitly performs a full collection.

## 7.2.9. LOS allocation of large objects

Objects of 8 KiB or more, or requiring alignment greater than 8 bytes, bypass the TLAB and are allocated individually in the LOS. Placing a large object in a TLAB wastes the remaining space, and since LOS objects are not moved, bump allocation gives no benefit.

## 7.2.10. Pinned objects

During an `extern "C"` call that receives a GC pointer, no separate pinning is needed ([7.4.1](704-barriers-and-roots.md#objects-pointed-to-by-roots-are-not-moved)). If C keeps the pointer after the call, pin it with `rok_gc_pin`. Pinning only sets the `PINNED` bit; it does not copy the object. `rok_gc_unpin` releases it.

## 7.2.11. Allocation time vs heap usage

The fast path is constant time (~4 instructions). The average allocation cost is determined by the fast path.

## 7.2.12. Allocation elision

The optimizer may remove an allocation whose object never escapes and is never read past its stores; such an allocation is no safepoint and is not counted by the collector's statistics. An allocation whose size cannot be represented (`len * size + header` overflowing) always reaches the runtime's "allocation too large".
