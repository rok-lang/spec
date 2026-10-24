# 9.4. ROKHIR format

This section defines the interface file `<name>.rokhir` of a prebuilt library package. An [ROKHIR](../000-meta/007-glossary.md#rokhir-rok-hir-bytecode) file sits next to that package's `lib<name>.a` ([Package archive](903-package-archive.md)) and is not a member of the archive.

## 9.4.1. Motivation

Monomorphization in the [AOT build pipeline](902-aot-pipeline.md) [specializes](../000-meta/007-glossary.md#specialize) by the concrete types at the call site. So when another package calls a [generic](../000-meta/007-glossary.md#generic) `fn` of a package, specialization must happen at that call site. The object code of [concrete fns](../000-meta/007-glossary.md#concrete-fn--generic-fn) alone is not enough, so a separate interface is provided for downstream to read.

The input of monomorphization is the AST. The ROKHIR body is therefore **stripped package source**, which the downstream parser reads.

## 9.4.2. No external libraries

Under the zero-dependency principle no serialization library is used either; ROKHIR is designed from scratch.

## 9.4.3. Overall layout

```
ROKHIR file (little-endian):
+-----------------------------------+
|  Header                           |  (16 bytes fixed)
|    magic       "ROKH"    (4 bytes)  |
|    version     u32  (= 6)         |
|    flags       u32                |
|    body_offset u32                |
+-----------------------------------+
|  Producer string        str       |  "rokc <version> (<compiler id>)"
+-----------------------------------+
|  Meta                             |
|    name                 str       |  mount name (--extern <name>=)
|    hash                 str       |  package hash (0.3.3)
|    source dir           str       |
|    gc type offset       varint    |
|    gc type count        varint    |
|    features  varint n, n × str    |
+-----------------------------------+
|  Body (at body_offset)            |
|    file count           varint    |
|    file × n: path str, text str   |
+-----------------------------------+
|  Summaries                        |
|    count                varint    |
|    n × (function str, summary str) |
+-----------------------------------+
```

`str` is UTF-8 bytes preceded by a PrefixVarInt length (9.4.5). There are no string / type / item sections.

## 9.4.4. Header and producer

| Field         | Size | Meaning                                                      |
|---------------|------|--------------------------------------------------------------|
| `magic`       | 4    | `"ROKH"`                                                       |
| `version`     | u32  | Format version (`6`). Bumped when the layout or body content changes: 2 added the `features` meta, 3 the `.wit` files of the body, 4 inlines `include_str!` / `include_bytes!` data, 5 adds the hand-off summaries, 6 adds the summaries' exit kinds |
| `flags`       | u32  | Reserved (0)                                                 |
| `body_offset` | u32  | Start offset of the body                                     |

The producer string is `rokc <version> (<compiler id>)`. The compiler id is the size and modification time of that `rokc` executable (`<size>-<mtime>`, hexadecimal).

## 9.4.5. VarInt encoding

The same PrefixVarInt as MLIR bytecode, a variant of LEB128. The number of leading 1 bits of the first byte is the number of following bytes, and the value occupies the remaining bits in big-endian order.

```
1-byte int  : 0xxxxxxx                    (0 ~ 127)
2-byte int  : 10xxxxxx xxxxxxxx           (0 ~ 16383)
3-byte int  : 110xxxxx xxxxxxxx xxxxxxxx  (0 ~ 2097151)
...
```

## 9.4.6. Meta

| Field            | Meaning                                                                           |
|------------------|-----------------------------------------------------------------------------------|
| name             | The package's mount name. It appears in symbols, so it must equal `--extern <name>=` |
| hash             | Package hash ([0.3.3](../000-meta/003-source-units.md#033-package-identifier)). The value used in symbol mangling. For the standard library package `rok` it is a `pkg=hash;...` list, one entry per tier |
| source dir       | The directory holding the package source at build time. The base for file paths in the body; downstream reads the body, not the disk |
| gc type offset / count | This package's GC type id range `[offset, offset + count)` (9.4.8)          |
| features         | The `feature` cfg values the package was built with. Downstream parses the body with the build's other cfg and these features |

## 9.4.7. Body

The body holds **every source file** of the package as (path, text) pairs. Each file is stripped as follows.

- The bodies of concrete functions are cut down to `;`. Newlines are kept, so diagnostic line numbers match the original. These functions are linked as archive symbols.
- Bodies that downstream must monomorphize or inline are kept: generic functions, `async` / `yield` functions, `#[inline]` functions, `extern "C"` definitions, default methods of traits, and methods of generic `impl`s.
- Declarations such as signatures, types, traits, `impl`s and `use`s are left as is.
- `include_str!` / `include_bytes!` in the kept text are replaced by the literal they read (line breaks kept), so downstream reads no included file and a moved library still works.

After the source files come the `.wit` files the package's `ffi::wit` items read ([stdlib 1.8](../../refs-standard-libraries/100-bare/108-wit.md)), with their `deps/`, whole. Their paths are relative to the source dir and may leave it through `..`. Downstream reads WIT from the body, not the disk.

After the body come the hand-off summaries ([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers)) of the package's functions. A summary is lines of tab-separated fields:

- `g <root> <mutex|rwlock|channel> <line> <function> <file>`: hands `<root>` over;
- `s <root> <root>`: one object;
- `i <root> <root>`: the first going away takes the second along;
- `t <guard parameter> <root>`: tied to that guard.
- `x <tag|*> [<root>]`: on the exits whose returned value has that tag (`*`: not known; a `bool`'s 0 / 1, a sum or enum variant, a carrier's 1 for an error, 0 for a value, 2 + the value's tag), `<root>` is among those handed over; with no `x` lines every exit hands over all `g` roots.

A root is a parameter index or `r` (the return value), then a `.`-separated field path (a tuple index has bit 31 set). An entry named `#fn` (`#fne`: a closure, environment first) followed by U+001F-separated parameter type keys joins the summaries of the functions used as values with those parameter types.

## 9.4.8. Rules for separate compilation

A prebuilt library cannot see the whole program at once, so packages are reconciled by the following rules.

- **`any` tag.** The tag of an `any` slot is a 31-bit hash of the type name, so separately built packages use the same tag for the same type.
- **`any` seed.** So that the program knows the types a library puts into `any`, the library appends a small rok function using those types to the end of the body. The program's closed-world `any` dispatcher includes these types too.
- **Dispatcher symbol.** The `any` dispatcher is the symbol `rok.any.<hash>`, defined weak in libraries and strong in the program. At link time the program's definition wins.
- **Sum member order.** Terms of a sum are ordered by a structural key, not by internal type numbers ([2.6.4](../002-types/206-sum-types.md#canonical-order)), so tags agree across packages.
- **GC type id.** Each package has its own GC type id range (9.4.6). It is an error for the ranges of two packages in one program to overlap.
- **Method symbols.** Methods of dependency packages are mangled by the stable path `<module>::<type>.<method>[__by_<trait>]` ([9.5](905-symbol-mangling.md)).

## 9.4.9. Reading and rejection

`--extern name=<name>.rokhir` or `--extern name=lib<name>.a` each locate the sibling file ([9.2.3](902-aot-pipeline.md#923-dependency-resolution)). Downstream parses the body as a source overlay to obtain declarations, declares stripped functions as prebuilt symbols, monomorphizes generics in place, and then links the archive.

If any of the following differs, the ROKHIR is rejected with a "rebuild" error.

- `version`
- producer (an ROKHIR made by a different `rokc`)
- name (the `--extern` name versus the ROKHIR's mount name)

## 9.4.10. ROKHIR inspection

`rokc rokhir-text <file>` prints the ROKHIR's version, producer, meta (including `features:`), the paths and text of the interface files, and the summaries ([9.9.2](909-debug-tools.md#992-rokhir-inspection)). `rokc inspect` ([9.9.14](909-debug-tools.md#9914-package-inspection)) shows a summary together with the archive.

## 9.4.11. Stability

Only ROKHIR made by the same `rokc` is read (9.4.9). When the format changes, `version` is bumped, and a different version requires a rebuild. The stability policy of the [Overview](../008-abi/README.md) applies.

The standard library is not distributed as ROKHIR ([9.3.6](903-package-archive.md#936-standard-library)).
