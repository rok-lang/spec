# 7.1. Object layout and memory spaces

Every object managed by the [GC](../000-meta/007-glossary.md#gc-garbage-collector) starts with a fixed header. This document defines the header format and the four memory spaces an object can reside in.

> [!NOTE]
> This section is informative, except the 16-byte header, which is fixed as ABI in [8.1](../008-abi/801-object-header-abi.md). The memory spaces describe the reference runtime and may change in any release.

## 7.1.1. Byte header

The first 16 bytes of every GC object are the header.

```
offset  0 :  type_id    u32       // type identifier
offset  4 :  gc_flags   u32       // GC state bits
offset  8 :  gc_word    u64       // total object size (new address during evacuation)
offset 16 :  payload ...           // the object's actual fields
```

Both the header and the payload are 8-byte aligned. The layout is the same on 32-bit and 64-bit, so the header is 16 bytes even on the 32-bit targets of the Tier 1 platform matrix.

### `type_id` (offset 0, 4 bytes)

The type's unique identifier, assigned per type by the compiler.

It has several uses. In dynamic dispatch it serves as the tag of the [`any` slot](../002-types/202-any-slot.md); in GC field scanning it locates fields through the `type_id → field layout` mapping; and in diagnostics it is used to show a user-friendly type name.

A `type_id` is stable for the lifetime of the process; the same type always has the same id.

### `gc_flags` (offset 4, 4 bytes)

Bits representing GC state.

| Bit         | Meaning                                                                                 |
|-------------|-----------------------------------------------------------------------------------------|
| `mark`      | Whether reached in the current GC cycle                                                 |
| `sticky`    | [sticky mark-bit](../000-meta/007-glossary.md#sticky-mark-bit) (generation tracking)    |
| `pinned`    | Marked as not movable, e.g. for [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) |
| `forwarded` | Already evacuated to a new location (gc_word is the forwarding pointer)                 |
| `large`     | [LOS](../000-meta/007-glossary.md#los-large-object-space) (Large Object Space) object   |
| `immortal`  | Permanently alive (never collected)                                                     |
| ...         | Additional bits are decided by the GC implementation                                    |

The exact bit placement is decided by the GC implementation; the table only lists the kinds of bits semantically required.

### `gc_word` (offset 8, 8 bytes)

The object's total size (including the header). For an object moved during evacuation, the `forwarded` bit is set and this slot points to the new location ([8.1.4](../008-abi/801-object-header-abi.md#814-gc_word-offset-8-u64-or-ptr)).

### `payload` (offset 16+)

The region holding the object's fields. Field order and alignment are decided by the compiler; this applies to [`enum`](../002-types/204-enum.md) and [`union`](../002-types/205-union.md) alike. A [`struct`](../002-types/203-struct.md) value is not a heap object of its own; stored in a heap object (list buffer, closure environment, coroutine frame) it is laid out inline here ([8.8.3](../008-abi/808-lowering-types.md#883-representation-of-struct)).

If an attribute such as `#[repr(C)]` is attached, the corresponding representation rules apply ([Built-in attributes](../006-meta-system/602-built-in-attributes.md)).

## 7.1.2. Memory spaces

The GC places each object in one of the following four spaces.

### 1. Immortal

`const`s, literal instances, static objects created at boot, and the like go in this space.

Allocation is bump-pointer, the fastest, and there is no collection, so objects live for the lifetime of the process. When the `immortal` bit of `gc_flags` is set, the GC simply skips the object.

Results of `const` expressions of [`const`](../005-modules/502-const.md) and initial values of [`static`](../005-modules/503-static.md) reside here.

### 2. Immix

The space where most ordinary objects live, using the Immix mark-region algorithm.

It is divided into 32 KiB [blocks](../000-meta/007-glossary.md#block) and 128 B lines. Mark-sweep is the default, and a full collection may move objects out of fragmented blocks (opportunistic evacuation). Generations are separated by the sticky mark-bit ([7.3](703-gc-algorithm.md)).

### 3. Large Object Space (LOS)

Objects of 8 KiB or more, or requiring alignment greater than 8 bytes, and secret buffers (`rok_gc_secret_new`, [8.5.1](../008-abi/805-extern-symbols.md#rok_gc_secret_new--rok_gc_secret_wipe)), are allocated individually in the LOS, with the `LARGE` bit of `gc_flags` set. The LOS is collected by mark-sweep and does not move objects. Old LOS objects have a card table ([7.4.2](704-barriers-and-roots.md#742-write-barrier-incremental-update)). A secret buffer has the `WIPE` bit ([8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags-offset-4-u32)): the sweep that frees it overwrites its payload with zeros first.

### 4. Pinned objects (FFI pinned)

A GC pointer passed to an FFI function of an [`extern` block](../005-modules/509-extern-block.md) is not moved until the call ends, because the argument is in the roots ([7.4.1](704-barriers-and-roots.md#objects-pointed-to-by-roots-are-not-moved)). If foreign code needs to keep the address after the call, pin it with `rok_gc_pin`. A pinned object has the `PINNED` bit of `gc_flags` set and stays in place. `rok_gc_unpin` releases the pin.

Pinned objects are still collected by mark-sweep, but never evacuated.

## 7.1.3. Movement between spaces

```
TLAB (allocation) → Immix
Immix → Immix         (evacuation in full collection)
Immix / LOS ↔ pinned  (pin / unpin, in place)
Immortal              (no movement)
```

Objects do not move between spaces. Only evacuation changes an address.

## 7.1.4. Motivation for the header width

The 16-byte header is the result of a time/space tradeoff.

For small objects, such as a 4-byte i32, a 16-byte header hurts memory efficiency. But 16 bytes hold type_id, flags and forwarding all at once, simplifying the algorithms, and the layout stays the same on 32-bit and 64-bit, simplifying cross-platform code too.

This tradeoff is a reasonable choice under the assumption that most objects are 4 bytes or larger.

## 7.1.5. Registering type_id: `rok_gc_register_type`

Each type's field layout must be registered with the GC.

```
rok_gc_register_type(type_id, scan_table)
```

`type_id` is the type's identifier, and `scan_table` holds the offsets of the fields that are pointers to GC objects, plus additional metadata.

When marking an object, the GC uses its `type_id` to find the `scan_table` and follows the child pointers through it.

The compiler performs this registration at process start through `extern "C"` calls; the registration ABI is defined in [Extern "C" symbols](../008-abi/805-extern-symbols.md).

## 7.1.6. Computing the total object size

```
size = 16  (header) + payload_size  (aligned size of the fields)
```

The compiler knows `payload_size` through the `type_id → size` table. So for an ordinary Immix object the size is determined by a single `type_id` lookup.

For LOS objects the size is read directly from the header's `gc_word`.
