# 7.5. Concurrent mark

A full collection may run its mark phase concurrently with mutators. Minor collection, explicit `rok_gc_collect`, and evacuation are always stop-the-world ([7.3](703-gc-algorithm.md)).

## 7.5.1. When

| `ROK_GC_CONCURRENT` | Full collection                                    |
|---------------------|----------------------------------------------------|
| `auto` (default)    | Concurrent mark if old objects total 32 MiB or more |
| `always`            | Always concurrent mark                             |
| `off`               | Always STW                                         |

## 7.5.2. Phases

1. **Initial pause (STW).** Run one minor collection and take a snapshot of the roots at a safepoint. Set the global `MARKING` flag.
2. **Concurrent mark.** A single marker thread marks starting from the snapshot. Mutators keep running.
3. **Final pause (STW).** Drain the SATB buffers, rescan the roots, and sweep. Clear `MARKING`.

## 7.5.3. SATB barrier

Snapshot-at-the-beginning (Yuasa), per object. It reuses the generational write barrier ([7.4.2](704-barriers-and-roots.md#742-write-barrier-incremental-update)) as is.

- The slow path on the first write to an `UNLOGGED` object, when `MARKING` is set, records all of that object's current references in the SATB buffer (TLS `rok_tls_satb_cursor` / `rok_tls_satb_limit`, [8.3](../008-abi/803-tls-slots.md)).
- While recording, it sets the header's `SATB` bit. A competing store to the same object waits until that bit is cleared.
- So the barrier code the compiler emits is the same whether or not concurrent mark is in use.

## 7.5.4. Allocation and collection

- Allocation during mark is black at line granularity (objects in that line are treated as live). Objects allocated during mark are young after mark finishes.
- No minor collection happens during mark. If allocation demands a collection, the ongoing mark is finished.
- Evacuation happens only in STW full collections. If the heap is fragmented after a concurrent mark, the next full collection is done as STW with evacuation.
