# rok standard library reference

This is the formal specification of the rok standard libraries. Where the language specification defines syntax and semantic mechanisms, the standard library specification supplies the meaning of the types, traits and functions those mechanisms need in order to actually work. It defines in one place which traits operators resolve to, which interfaces collections have, and on which guarantees async and I/O operate.

The language specification is collected separately in [`refs-lang/`](../refs-lang/README.md); this document covers only the specification. The actual implementation of the libraries is outside the scope of the spec.

> [!IMPORTANT]
> **Boundary between the language specification and the standard library specification**
>
> [refs-lang](../refs-lang/README.md) defines **syntax and semantic mechanisms** (e.g. which trait
> method the `+` token maps to). [refs-standard-libraries](README.md) (this document) defines
> **the signatures and meaning of those traits themselves** (e.g. which numeric conversion
> `Add::add` applies).
>
> Both specifications follow the same [notation conventions](../refs-lang/000-meta/008-notation.md).

---

## The 3-tier model

The standard libraries are not a single flat plane but are divided into three layers. Almost all systems code is written on top of safe abstractions, but implementing those abstractions eventually requires handling raw memory and FFI somewhere. Moreover, features that are meaningful only with a host environment, such as files or networking, must not exist at all on targets without such an environment, such as embedded. To expose these differences directly in the dependency graph, the libraries follow a 3-tier structure under the virtual namespace `rok::`: `rok::bare` / `rok::core` / `rok::std`.

```
rok::bare   (base, depends only on runtime extern "C")
  ↓
rok::core   (foundation, safe, all targets - uses rok::bare internally)
  ↓
rok::std   (target-dep - uses rok::core + rok::bare)
```

`rok::bare` exposes only raw capabilities at the bottom, and `rok::core` provides a safe foundation on top of it, identically on every target. `rok::std` adds host-dependent features on top of `rok::core`, completing the surface an ordinary application expects. Everyday code needs only `rok::core`; it reaches for `rok::std` the moment OS resources are needed, and imports `rok::bare` explicitly only when it must cross an abstraction boundary directly. The detailed dependency rules are covered in [0.1 Tier model](000-meta/001-tier-model.md).

| Tier        | Path           | Directory                          | Area                                           |
|-------------|----------------|------------------------------------|------------------------------------------------|
| `rok::bare` | `rok::bare::*` | [`100-bare/`](100-bare/README.md)  | `Ptr`, `ConstPtr`, `transmute`, raw alloc, pin |
| `rok::core` | `rok::core::*` | [`200-core/`](200-core/README.md)  | Numbers, operators, iter, collections, fmt, sync, task |
| `rok::std` | `rok::std::*` | [`300-std/`](300-std/README.md)    | fs, net, time, thread, io, env, process, path, crypto (random) |

---

## Chapters

### 0. Meta - [000-meta/](000-meta/README.md)

Library-wide policies and tables.

- [0.1. The 3-tier model](000-meta/001-tier-model.md)
- [0.2. Import policy (implicit `use rok::core::**;`)](000-meta/002-import.md)
- [0.3. Intrinsic mechanism - the compiler ↔ standard library boundary](000-meta/003-intrinsics.md)
- [0.4. `#[lang]` slot mapping table](000-meta/004-lang-slot-table.md)
- [0.5. Notation](000-meta/005-notation.md)

### 1. Unsafe - [100-bare/](100-bare/README.md)

Low-level / dangerous area. Explicit import required.

- 1.1. [Pointer family (`Ptr`/`ConstPtr`/`ManyPtr`/`CPtr` - Zig arity-encoded + Rust strict provenance)](100-bare/101-ptr.md)
- 1.2. [Cast operations (`ptr_cast`/`align_cast`/`const_cast`/`reinterpret`)](100-bare/102-cast.md)
- 1.3. [`MaybeUninit<T>` + `Pod` + raw mem ops](100-bare/103-mem.md)
- 1.4. [Raw allocation](100-bare/104-alloc.md)
- 1.5. [Volatile memory access](100-bare/105-volatile.md)
- 1.6. [`Pinned<T>` - GC pin (Go 1.21 Pinner model)](100-bare/106-gc-pin.md)
- 1.7. [FFI - `extern "C"` / `CStr` / `CString`](100-bare/107-ffi.md)
- 1.8. [WASM Component Model (`ffi.wit`)](100-bare/108-wit.md)
- 1.9. [`rok::bare::mlir` - direct access to MLIR dialects](100-bare/109-mlir.md)
- 1.10. [`rok::bare::mlir::plugin` - registering user-defined dialects](100-bare/110-mlir-plugin.md)

### 2. Rok - [200-core/](200-core/README.md)

Safe area. All targets. `use rok::core::**;` applied implicitly (no separate prelude module).

- 2.1. [Numeric types (`int`, `float`, auto-promote)](200-core/201-numbers.md)
- 2.2. [Niche wrappers (`NonZero`, `NonNan`, `Finite`, `Positive`)](200-core/202-non-zero.md)
- 2.3. [Arithmetic / bitwise traits (`Add`, `Sub`, ..., `Shl`, `Shr`)](200-core/203-ops.md)
- 2.4. [Compound assignment traits (`AddAssign`, ...)](200-core/204-ops-assign.md)
- 2.5. [Comparison traits (`PartialEq`, `Eq`, `Ord`, `PartialOrd`, `Hash`)](200-core/205-cmp.md)
- 2.6. [Broadcast traits (`DotAdd`, `DotMul`, ...)](200-core/206-broadcast.md)
- 2.7. [Iter traits (`Iterator`, `IntoIterator`)](200-core/207-iter.md)
- 2.8. [`Stream` (async iterator)](200-core/208-stream.md)
- 2.9. [Collections (`List`, `Dict`, `Set`, `Tuple`, `Slice`)](200-core/209-collections.md)
- 2.10. [fmt (`Display`, `Debug`, `Formatter`)](200-core/210-fmt.md)
- 2.11. [Conversion (`From`, `Into`, `TryFrom`, `saturating_into`, ...)](200-core/211-conversion.md)
- 2.12. [Reflection (`Reflect`, `Inspect`)](200-core/212-reflect.md)
- 2.13. [Marker traits (`Clone`, `Copy`)](200-core/213-marker.md)
- 2.14. [Error types (`TypeError`, `ValueError`, ...)](200-core/214-error.md)
- 2.15. [Strings (`str` single mutable, `char`, `c_char`)](200-core/215-string.md)
- 2.16. [Sync primitives (`Atomic`, `Mutex`, `RwLock`, `Cell`)](200-core/216-sync.md)
- 2.17. [`Channel`](200-core/217-channel.md)
- 2.18. [`task::` API](200-core/218-task.md)
- 2.19. [`par::` API](200-core/219-par.md)
- 2.20. [`Future` / `await` integration](200-core/220-future.md)
- 2.21. [`crypto::` - hashes, MACs, AEADs, key exchange, signatures](200-core/221-crypto.md)

> [!NOTE]
> Scientific computing / AI areas such as tensors / automatic differentiation / SIMD / GPU are not part of the standard libraries and are split out into external libraries. External libraries can access MLIR dialects directly through the [`rok::bare::mlir`](100-bare/109-mlir.md) module.

### 3. Std - [300-std/](300-std/README.md)

Target-dependent area. Explicit import.

- 3.1. [`fs::` - file system (atomic ops, Dir capability, reflink)](300-std/301-fs.md)
- 3.2. [`net::` - TCP/UDP/Unix + TLS 1.3 (HTTP is external)](300-std/302-net.md)
- 3.3. [`time::` - `Instant` / `SystemTime` / `Duration` / `BootTime`](300-std/303-time.md)
- 3.4. [`thread::` - OS threads (1:1, separate from tasks)](300-std/304-thread.md)
- 3.5. [`io::` - `Reader` / `Writer` / stdin/out/err (sync only - no async traits)](300-std/305-io.md)
- 3.6. [`env::` - environment variables / args / directories](300-std/306-env.md)
- 3.7. [`process::` - `Command` / pidfd / kill_on_drop](300-std/307-process.md)
- 3.8. [`path::` - UTF-8 `Path` + `/` operator (Python pathlib style)](300-std/308-path.md)
- 3.9. [`rokon::` - the ROKON data format (`.rokon`)](300-std/309-rokon.md)
- 3.10. [`crypto::` - the OS random source](300-std/310-crypto.md)

---

## How to read

### First reading

On a first approach, it is recommended to follow Chapters 0-6 of [`refs-lang`](../refs-lang/README.md) to understand the language's mechanisms, and then read this document on top of that. This document gives meaning to the mechanisms refs-lang defines; without a feel for those mechanisms, the traits and signatures tend to read as if floating in the air. Reading the two documents side by side, moving back and forth, is the most efficient approach.

### As a reference

Each item is identified by a stable section number (`<chapter>.<section>`). The number is used identically in internal links and external references, so it points reliably even if files move.

```
refs-standard-libraries 2.3        →  rok ops (Add/Sub/...)
refs-standard-libraries 1.1        →  unsafe Ptr<T>
refs-standard-libraries 3.5.2      →  std io (specific subsection)
```

### Search

To find a specific trait or function quickly, the lightest approach is to search with `ripgrep`, narrowing the directory.

```bash
rg "trait Add" spec/en/refs-standard-libraries/200-core/
rg "fn into" spec/en/refs-standard-libraries/
```

---

## Directory / file naming

This document follows a different directory convention from `refs-lang`, because the two specifications have different shapes: refs-lang is a flat structure in which chapters 0 through 9 all have equal authority, whereas refs-standard-libraries is a hierarchical structure in which the three tiers act as the key information in cross-references. This structural difference is reflected directly in the number of digits in directory names.

| Specification           | Prefix                 | Reason                                                          |
|-------------------------|------------------------|-----------------------------------------------------------------|
| refs-lang               | One digit (`002-`)     | 10 chapters (0-9), all with equal authority - flat shape        |
| refs-standard-libraries | Three digits (`200-`)  | 3 tiers × N modules - the tier is key cross-reference information |

The concrete file naming rules are as follows. Tier directories have prefixes in units of 100, and module files within them start with a three-digit prefix combining one tier digit and two module digits.

| Location | Form                                   | Example                           |
|----------|----------------------------------------|-----------------------------------|
| Tier     | `<TNN>-<descriptive-name>/`            | `100-bare/`, `200-core/`          |
| Module   | `<tier><module>-<name>.md`             | `203-ops.md`, `211-conversion.md` |
|          | (one tier digit + two module digits)   |                                   |
| README   | `README.md` in each tier / chapter     | TOC + overview                    |

Although tier information is encoded in the file path itself, the stable section number (`2.3.1`, etc.) is the formal notation when citing this document externally. File paths serve only as an auxiliary notation for navigation convenience in IDEs and GitHub.
