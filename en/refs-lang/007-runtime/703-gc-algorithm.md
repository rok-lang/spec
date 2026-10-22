# 7.3. GC algorithm

rok's GC is a collector that layers sticky mark-bit generational collection[^whippet] on top of [Immix](../000-meta/007-glossary.md#immix) mark-region[^immix]. Minor collection and evacuation are stop-the-world, with mutators stopped at safepoints ([8.4](../008-abi/804-safepoint.md)). The mark phase of a full collection may run concurrently ([7.5](705-concurrent-collection.md)). Roots and the write barrier are defined in [7.4](704-barriers-and-roots.md).

## 7.3.1. Immix mark-region

### Blocks and lines

```
block (32 KiB) = line (128 B) × 256
```

[Blocks](../000-meta/007-glossary.md#block) are obtained from the OS (mmap). Objects may cross line boundaries. Mark marks the object together with the lines it spans, and sweep reclaims unmarked lines. Small gaps between live objects are reused at line granularity.

### Sweep: block classification

| Class      | State                         | Handling                         |
|------------|-------------------------------|----------------------------------|
| clean      | All lines empty               | To the free pool                 |
| recyclable | Only some lines live          | Free lines reused for allocation |
| full       | Nearly all lines live         | Left as is                       |

Sweep does not zero dead objects, except secret buffers (`WIPE`, [7.1.2](701-object-layout.md#712-memory-spaces)). The allocator zeroes lines when it fills a TLAB.

### Opportunistic evacuation

A full collection may move live objects out of fragmented blocks into other blocks.

- Candidates are chosen from the line state left by the previous sweep (blocks at most half full). Objects are moved only when a fragmentation threshold is exceeded.
- A forwarding pointer is left at a moved object's old location, and all precise references are updated to the new location within the same collection.
- Objects not moved: objects pointed to directly by roots ([7.4.1](704-barriers-and-roots.md#objects-pointed-to-by-roots-are-not-moved)), objects pointed to by ambiguous words (`AMBIGUOUS_WORD`: type id `2` buffers, untagged `union`s), `PINNED` / `NOMOVE` objects, and LOS objects.
- Free blocks beyond the next cycle's allocation budget are returned to the OS.
- For testing, `ROK_GC_EVAC=always` evacuates on every full collection ([9.9.8](../009-tools/909-debug-tools.md#998-memory--gc-tools)).

## 7.3.2. Sticky mark-bit generational collection

There is no separate nursery space; generations are distinguished by the header's `STICKY` bit ([8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags-offset-4-u32)). An object that survives a collection gets `STICKY` set and becomes old. Objects stay in place; there is no promotion copy.

| Collection    | Mark starting points                             | Scope                                         |
|---------------|--------------------------------------------------|-----------------------------------------------|
| minor (young) | Roots, remembered set, dirty cards ([7.4.2](704-barriers-and-roots.md#742-write-barrier-incremental-update)) | Does not enter old objects. Sweep covers only lines handed out since the last collection |
| full (old)    | Roots                                            | Clears all `STICKY` and does a full mark / sweep |

- Young lines are parsed only in blocks reached by an object lookup.
- A full collection happens when the bytes of old objects reach `max(4 MiB, live after the previous full collection × 2)`, or when `rok_gc_collect` is called.

## 7.3.3. Mark procedure

1. Start from the roots ([7.4.1](704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)).
2. For each reached object, set `MARK` in the header, mark the lines it spans, and follow its children through the [ScanTable](../008-abi/802-scan-table.md) of its `type_id`.
3. Finish when the queue is empty.

ScanTables are registered with `rok_gc_register_type` ([8.5.1](../008-abi/805-extern-symbols.md#851-gc-area)).

## References

[^immix]: Blackburn & McKinley, _Immix: A Mark-Region Garbage Collector with Space Efficiency, Fast Collection, and Mutator Performance_, PLDI 2008.

[^whippet]: Wingo, Whippet: a sub-toolkit for embeddable garbage collection. <https://wingolog.org/tags/whippet>
