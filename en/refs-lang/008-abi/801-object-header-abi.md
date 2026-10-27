# 8.1. Object header ABI

Defines the bit layout of the 16-byte header from [Object layout and memory spaces](../007-runtime/701-object-layout.md). GC constants such as the header size, the `gc_flags` bits, the LOS threshold, the starting type id value and the runtime buffer type ids come from a single definition (`gc_abi`) shared by the compiler and the runtime.

## 8.1.1. Exact layout

The header has the following form.

```
+---------+---------+-----------------+----------------+
| offset  |  size   |  field          |  type          |
+---------+---------+-----------------+----------------+
|    0    |    4    |  type_id        |  u32           |
|    4    |    4    |  gc_flags       |  u32           |
|    8    |    8    |  gc_word        |  u64 / ptr     |
|   16    |    var  |  payload        |  per type_id   |
+---------+---------+-----------------+----------------+
```

> [!IMPORTANT]
> The total header size is 16 bytes, 8-byte aligned, identical on 32-bit and 64-bit targets.

Every object is at least 24 bytes (header plus one payload word, `MIN_OBJECT_SIZE` in `gc_abi`), so compiled code may load the first payload word of any object it holds a reference to.

## 8.1.2. `type_id` (offset 0, u32, little-endian)

`type_id` is the unique identifier of a type. It is a 32-bit unsigned value stored little-endian; the value `0` is reserved and denotes the null type or an invalid object. At process start the compiler calls `rok_gc_register_type` to register each type_id.

### type_id assignment

Within one build, the same type has the same id. Ids are not stable across builds, so they are not used for serialization.

### Reserved ranges

The reserved id ranges are as follows.

| Range          | Use                         |
|----------------|-----------------------------|
| `0`            | invalid / null              |
| `1`            | runtime (GC string, not traced) |
| `2`            | runtime (word-traced buffer) |
| `3`            | runtime (weak cell, `Weak<T>`) |
| `4` ~ `0xFFFF` | standard library / runtime (reserved) |
| `0x10000` ~    | user / other package types  |

The exact id mapping of standard library types is defined in the standard library specification. Ids the compiler assigns to user types start at `0x10000`.

## 8.1.3. `gc_flags` (offset 4, u32)

The `gc_flags` field holds the exact positions of the [GC](../000-meta/007-glossary.md#gc-garbage-collector) state bits, laid out as follows.

```
bit  0  : MARK       (mark for the current GC cycle)
bit  1  : STICKY     (sticky mark-bit, old generation)
bit  2  : PINNED     (must not move, FFI)
bit  3  : FORWARDED  (after evacuation; gc_word is the new location)
bit  4  : LARGE      (LOS object; gc_word is the size)
bit  5  : reserved   (unused)
bit  6 ~ 31 : reserved / GC implementation-defined
```

The following bits are used from the implementation-reserved range ([7.4.2](../007-runtime/704-barriers-and-roots.md#742-write-barrier-incremental-update)).

| Bit  | Name       | Meaning                                                                                    |
|------|------------|--------------------------------------------------------------------------------------------|
| 6    | `UNLOGGED` | Set together with `STICKY` on objects that survive a collection. Cleared when the write barrier first records the object in the remembered set |
| 7    | `CARDED`   | Set instead of `UNLOGGED` on old objects of 8 KiB or more. The barrier marks a 512 B card dirty. The card table pointer is the word immediately before the header |
| 8    | `NOMOVE`   | An object whose address is used as a number. Never moved, regardless of lifetime. Set by `rok_task_key` ([8.5](805-extern-symbols.md)) |
| 9    | `NOEVAC`   | Collector-internal, for one full collection: an ambiguous word (a root, or a word of a runtime word buffer, type id `2`) refers to the object, so that collection's evacuation leaves it in place ([7.3.1](../007-runtime/703-gc-algorithm.md#opportunistic-evacuation)). Set before evacuation on such objects in candidate blocks, cleared when evacuation visits them |
| 10   | `SATB`     | Set while the barrier records this object's references during concurrent mark. Competing stores wait until it is cleared ([7.5.3](../007-runtime/705-concurrent-collection.md#753-satb-barrier)) |
| 11   | `SCANNED`  | This object has been scanned in concurrent mark |
| 12   | `WIPE`     | A secret buffer (`rok_gc_secret_new`, [8.5.1](805-extern-symbols.md#rok_gc_secret_new--rok_gc_secret_wipe)). The sweep that frees it zeroes its payload first. Never cleared |
| 13 ~ 28 | `VISIT_STAMP` | Used by a `par` visit ([stdlib 2.19.1](../../refs-standard-libraries/200-core/219-par.md#2191-core-api)): a 16-bit epoch of the last visit that reached the object. The collector keeps these bits |
| 29   | `VISIT_OTHERS` | `par` visit: reached by more than one item, each through slices |
| 30   | `VISIT_SLICED` | `par` visit: reached only through slices so far |
| 31   | `VISIT_CUR` | `par` visit: reached by the item being visited |

### Atomicity requirements

In STW collection, `MARK` / `STICKY` / `FORWARDED` are non-atomic. `MARK` / `SATB` / `SCANNED`, which the marker and mutators touch together during concurrent mark, and clearing `UNLOGGED`, which mutators race on, are atomic ([7.4.2](../007-runtime/704-barriers-and-roots.md#742-write-barrier-incremental-update)).

## 8.1.4. `gc_word` (offset 8, u64 or ptr)

`gc_word` is the total size of the object (including the 16-byte header, a multiple of 8). It applies to both Immix objects and LOS objects and is written inline by the allocation fast path ([8.9.9](809-lowering-control.md#899-gc-alloc-fast--slow-path)).

Only during evacuation in a full collection is the `FORWARDED` bit of a moved object set, with `gc_word` pointing to the new address. All references are fixed up within the same collection ([7.3.1](../007-runtime/703-gc-algorithm.md#731-immix-mark-region)) and the old location is then discarded, so mutators never observe forwarding.

Objects inside a block never have `type_id` 0. The collector uses 0 to mark free space.

### Collector-private information

The collector may keep its own bookkeeping before the header (at addresses lower than the object address) or in a separate side table. The compiler and other code see only the header and payload starting at the object address; everything outside that belongs to the collector implementation.

## 8.1.5. `payload` (offset 16+)

The compiler determines the payload area according to the field layout registered for the `type_id`. The format is as follows.

- For [`struct`](../002-types/203-struct.md) / [`enum`](../002-types/204-enum.md) / [`union`](../002-types/205-union.md), the layout is at the compiler's discretion.
- With `#[repr(C)]`, the C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) is followed.
- With `#[repr(packed)]`, alignment is ignored.

In the GC mark phase, child pointer offsets are found through the `type_id → ScanTable` mapping (the format is defined in [ScanTable serialization format](802-scan-table.md)).

## 8.1.6. Alignment requirements

The alignment requirement at each location is as follows.

| Location                 | Alignment                           |
|--------------------------|-------------------------------------|
| Object start (header)    | 8 bytes                             |
| payload start (offset 16) | 8 bytes (automatic, since the header is 16 bytes) |
| Each field               | The alignment of its type           |

If a field inside `payload` requires alignment greater than 8 bytes (e.g. `#[repr(align(16))]`), the payload start is aligned to that amount: the header sits 16 bytes before an aligned address, and the gap before it belongs to the collector ([8.1.4](#814-gc_word-offset-8-u64-or-ptr)).

## 8.1.7. Total size of header + payload

The total size of header plus payload is computed as follows.

```
total_size = 16 + align_up(payload_size, max(8, payload_align))
```

where `align_up(x, a) = (x + a - 1) & ~(a - 1)`.

This size is the increment of the [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) bump pointer (see [Allocation](../007-runtime/702-allocation.md)).

## 8.1.8. Example: i32 object

The layout of a GC object holding one i32 is as follows.

```
offset  0:  type_id = ID_I32     (4 bytes)
offset  4:  gc_flags = 0          (4 bytes)
offset  8:  gc_word = 24          (total size, 8 bytes)
offset 16:  i32 value             (4 bytes)
offset 20:  padding               (4 bytes, for 8-byte alignment)
total: 24 bytes
```

A single i32 takes 24 bytes in total: a 16-byte header, a 4-byte value and 4 bytes of padding.

This overhead looks large for small objects, but most objects have larger payloads (composite structures), so the ratio shrinks.

## 8.1.9. Example: large object (buffer of List<i64>)

An example list buffer with a 16 KB payload:

```
offset    0:  type_id = ID_BUFFER_I64  (4 bytes)
offset    4:  gc_flags = LARGE          (4 bytes)
offset    8:  gc_word = 16400            (total size, 8 bytes)
offset   16:  i64[2048]                 (16384 bytes)
total: 16 + 16384 = 16400 bytes
```

The `LARGE` bit is set, so this is a LOS object. `gc_word` is the total size (8.1.4).

## 8.1.10. Compatibility when the header changes

This header layout is the most central part of the ABI. Changes follow these rules.

- It must not change in a minor / patch version (that would break compatibility).
- It may change in a major version, in which case the ABI version hash changes with it.

Because of this stability, user packages, the standard library and the runtime that share the same major version are all compatible.
