# 8.2. ScanTable serialization format

The mark phase of the [GC](../000-meta/007-glossary.md#gc-garbage-collector) must follow each object's child pointers, so it needs data that says at which offsets child pointers live. This data is called the ScanTable; this document defines its serialization format.

## 8.2.1. Purpose

The GC mark loop works as follows.

```
GC mark loop:
  for each live object O:
    type_id = O.type_id
    scan_table = lookup(type_id)
    for each (offset, kind) in scan_table:
      child = *(O + offset)
      mark(child)
```

Here `lookup(type_id)` uses what the compiler registered at process start through `rok_gc_register_type` from the [object header ABI](801-object-header-abi.md).

The stack (shadow stack) and `static` roots are scanned with the same byte-offset-based walk. A single root slot is treated like one ScanTable entry, and an `any` root also covers its payload word.

## 8.2.2. Registration call

Registration is done by calling the following external function.

```
extern "C" {
    fn rok_gc_register_type(type_id: u32, scan_table: ConstPtr<u8>);
}
```

- `type_id`: the ID of the type being registered.
- `scan_table`: pointer to the start of the serialized ScanTable.

The compiler emits ScanTable memory into .rodata, so it is valid for the lifetime of the process. The runtime only reads this area.

## 8.2.3. Serialization format

A ScanTable is an 8-byte header followed by an array of entries.

```
ScanTable Header (8 bytes, 4-byte aligned):
  offset  0:  total_size       u32         // total object size (16 + payload)
  offset  4:  version          u16  = 1
  offset  6:  num_entries      u16         // number of child pointer entries

Entry array (8 bytes each):
  offset  8 + 8*i + 0:  field_offset  u32  // offset within the object
  offset  8 + 8*i + 4:  field_kind    u32  // kind of child pointer
```

The total size is 8 + 8 × num_entries, and every field is naturally aligned.

### Header

The header fields are as follows.

| Field         | Type | Meaning                                          |
|---------------|------|--------------------------------------------------|
| `version`     | u16  | Format version (bits 0 ~ 14, currently = 1). bit 15 = array table |
| `total_size`  | u32  | Total object size (16-byte header + aligned payload) |
| `num_entries` | u16  | Number of child pointer entries                  |

If bit 15 of `version` is 1, the table is an **array table**. In that case `total_size` is the element stride, each entry's `field_offset` is relative to the element start, and the entries are applied repeatedly to every element across the whole payload. The compiler writes element tables for GC buffers in this form.

### Entry

Each entry designates one child pointer slot in the object.

| Field          | Type | Meaning                          |
|----------------|------|----------------------------------|
| `field_offset` | u32  | Offset of the slot (from the object start) |
| `field_kind`   | u32  | Kind of the slot                 |

### `field_kind` values

The values `field_kind` may take and their meanings are as follows.

| Value | Meaning                                                        |
|-----|------------------------------------------------------------------|
| 0   | `STRONG_REF`: ordinary GC object pointer (followed by mark)      |
| 1   | `WEAK_REF`: weak reference (not followed by mark, cleared on collection) |
| 2   | `INTERIOR_REF`: points inside an object (adjusted on forwarding) |
| 3   | `ANY_SLOT`: dynamic slot of the form `tag + payload`. Marked after checking the tag |
| 4   | `SUM_PAYLOAD`: tag of a sum type and per-variant references (8.2.6). bits 8..31 hold the descriptor location |
| 5   | `AMBIGUOUS_WORD`: a word that may or may not be a reference. The pointed-to object is kept alive but not moved (type id `2` buffer, untagged `union`) |
| 6 ~ | reserved / additional kinds                                      |

How each kind is actually marked is up to the GC implementation; this specification defines only which kinds exist and their encoding.

## 8.2.4. Encoding example: `struct Point { x: f64; y: f64; }`

This struct has no child pointers (all fields are primitive floating-point types).

```
ScanTable {
  version      = 1
  total_size   = 32        // 16 header + 8 + 8
  num_entries  = 0
}
```

The ScanTable is therefore 8 bytes, header only.

## 8.2.5. Encoding example: `struct Pair { a: List<i32>; b: List<f64>; }`

This one has two child pointers.

```
ScanTable {
  version      = 1
  total_size   = 32        // 16 header + 8 (List ptr) + 8
  num_entries  = 2

  entry[0]: field_offset = 16, field_kind = STRONG_REF (0)
  entry[1]: field_offset = 24, field_kind = STRONG_REF (0)
}
```

The total size is 8 + 16 = 24 bytes.

## 8.2.6. Encoding example: `enum` variants

```
enum Result {
    Ok(i32),
    Err(str),
    Pending,
}
```

An object of this type consists of the tag (4 bytes), padding (4 bytes) and the payload of the largest variant (8 bytes, a str pointer), 32 bytes in total.

References sit at different places in each variant, so a single `SUM_PAYLOAD` entry and its descriptor are used.

- The entry's `field_offset` is the offset of the tag (u32).
- The low byte of `field_kind` is `4`; bits 8..31 are the location of the descriptor (in u32 words from the table start).
- The descriptor holds `n` followed by `{tag, num_entries, entries_index}` for each variant that has references.
- A variant's entry is `{offset from the tag, field_kind}`. A nested sum is again written as `SUM_PAYLOAD`.
- Variants whose tag is not in the descriptor have no references.

```
ScanTable {
  version = 1, total_size = 40, num_entries = 1
  entry[0]: field_offset = 16, field_kind = 4 | (desc << 8)    // tag at offset 16
  desc:     n = 1
            { tag = 1 (Err), num_entries = 1, entries_index = e }
  e:        { offset = 8, field_kind = STRONG_REF }             // str ptr of Err
}
```

References in enum / sum type payloads are precise, so objects reached through a payload may also be moved by evacuation. `AMBIGUOUS_WORD` is used only for type id `2` buffers and for the storage of user `union`s, which have no tag and cannot be interpreted.

## 8.2.7. ANY slots

An `any` slot ([`any` slot](../002-types/202-any-slot.md)) can hold a value of any type in the form `tag + payload`.

```
ANY slot layout.
  offset  0:            tag       u32
  offset  ptr-aligned:  payload   usize (numbers/bool/char up to pointer width stored directly, everything else as a box ptr)
```

The corresponding ScanTable entry is as follows.

```
field_offset = (offset of the slot)
field_kind = ANY_SLOT (3)
```

Whether the payload is a box reference is carried in the tag.

- If **bit 31** of the tag is 1, the payload is a box reference; if 0, it is the value itself.
- Type comparisons on the tag use the value with bit 31 masked off.

The GC can therefore scan the payload precisely and rewrite the address of a moved box without any extra table. Tables need not be merged across separately compiled packages either.

## 8.2.8. WEAK_REF

A weak reference does not keep an object alive.

- The mark phase does not follow a weak `ref`.
- After sweep, if the object a weak `ref` pointed to was collected, the weak `ref` slot is cleared to 0.

This mechanism is useful for caches, observer patterns and the like; the exact library interface is defined in the standard library specification.

## 8.2.9. INTERIOR_REF

An INTERIOR_REF points to a location inside an object (not its start).

- It arises from slices, partial views and the like.
- On forwarding, it is adjusted by the relative offset between the old and new object.

The exact handling is determined by the GC implementation.

## 8.2.10. Endianness of the serialization

All fields are little-endian. rok targets little-endian platforms first (all Tier 1 targets in [Platform targets](../000-meta/006-platform-targets.md) are little-endian), and the serialization format follows suit.

When big-endian target support is added, it will be decided at that time whether serialization endianness stays fixed to little or follows the host endianness.

## 8.2.11. Compatibility of changes

Compatibility of the ScanTable format is handled by the version field.

- Within the same major version, compatibility is handled by the version field alone.
- Across major versions, a separate .a is required.

## 8.2.12. Runtime-reserved type ids

The following two type ids are defined directly by the runtime; the compiler does not register ScanTables for them ([8.1.2](801-object-header-abi.md#812-type_id-offset-0-u32-little-endian)).

| type id | Meaning           | scan                                                    |
|---------|-------------------|---------------------------------------------------------|
| `1`     | GC string         | payload is bytes and is not scanned                     |
| `2`     | word-traced buffer | every word of the payload is scanned as an ambiguous word (`AMBIGUOUS_WORD`) |

Both are allocated with `rok_gc_alloc_slow` with type id `1` or `2` ([8.5.1](805-extern-symbols.md#851-gc-area)). The standard library's [`GcBuf<T>`](../../refs-standard-libraries/200-core/209-collections.md#2912-gcbuft--raw-gc-buffer) sits on top of this.

## 8.2.13. Registration time

Registration is done in bulk at process start.

```
process_start:
  for each registered type:
    rok_gc_register_type(type_id, &scan_table)
  ↓
  user main()
```

The compiler emits a function `rok.register_types` that collects all registration calls. The C-ABI `main` the compiler generates calls this function immediately after `rok_runtime_init`, and then enters the user entry point.

```
main(argc, argv):                // compiler-generated, C ABI
  rok_runtime_init()
  rok.register_types()           // collection of rok_gc_register_type calls
  → user entry point (4.16)
```

The slow path of GC object allocation is `rok_gc_alloc_slow(size, align, type_id)` ([7.2.3](../007-runtime/702-allocation.md#723-slow-path-rok_gc_alloc_slow)), and type ids assigned by the compiler start at `0x10000` ([8.1.2](801-object-header-abi.md#812-type_id-offset-0-u32-little-endian)).
