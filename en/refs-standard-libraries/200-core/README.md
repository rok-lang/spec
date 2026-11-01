# 2. Core tier

The core tier (`rok::core`) is the safe standard library: the practical surface of Rok that builds on every target. Because about 90% of user code uses items from this tier, it is the layer that determines what Rok looks like in everyday programming.

## Location

The specification documents live at:

- spec: `spec/en/refs-standard-libraries/200-core/`

## Dependencies

Upward, the core tier depends on `rok::bare` for internal use; downward, `rok::std` and user code depend on it. The core tier therefore sits at the center of the safe region, between `rok::bare` and `rok::std`.

## User import policy

`use rok::core::prelude::*;` is **applied implicitly**: the compiler prepends it to every user file. The prelude's names (List, Dict, Display, Iterator, Mutex, task, ...) are usable without an explicit import; the rest of core goes through a prelude module (`ops::Add`, `task::sleep`) or a `use`.

Narrow explicit imports are also allowed where needed.

```rok
use rok::core::collections::List;           // narrow, explicit
use rok::core::ops::{Add, Sub};           // operator traits are not in the prelude
```

> [!NOTE]
> The prelude's list is normative and holds no enum variants: a bare variant resolves from the
> expected type ([refs-lang 2.4.8](../../refs-lang/002-types/204-enum.md#248-bare-variants-from-the-expected-type)).

The full policy is covered in [0.2 Import policy](../000-meta/002-import.md).

## Module index

The modules provided by the core tier, grouped by area. Each entry ends with the `rok::core` modules it specifies.

### Numbers / niche

- [2.1. Numeric types (fixed-width, `Int`, `Float`)](201-numbers.md): `num`
- [2.2. Niche wrappers (`NonZero`, `NonNan`, `Finite`, `Positive`)](202-non-zero.md): `num`

### Operator traits

- [2.3. Arithmetic / bitwise / logical (`Add`, `Sub`, `Mul`, `Div`, `Rem`, `Neg`, `Not`, `BitNot`, `BitAnd`, `BitOr`, `BitXor`, `Shl`, `Shr`, `LogicalAnd`, `LogicalOr`, `Index`)](203-ops.md): `ops`, `index`
- [2.4. Compound assignment (`AddAssign`, `SubAssign`, ..., `ShrAssign`)](204-ops-assign.md): `ops`
- [2.5. Comparison (`PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`, `Hasher`)](205-cmp.md): `cmp`, `hash`
- [2.6. Broadcast (`DotAdd`, `DotMul`, ..., `DotShr`)](206-broadcast.md): `ops`

### iter / stream

- [2.7. iter (`Iterator`, `IntoIterator`, `Range`, `RangeInclusive`)](207-iter.md): `iter`, `range`
- [2.8. `Stream` (async iterator)](208-stream.md): `stream`

### Collections

- [2.9. Collections (`List`, `Dict`, `Set`, `Slice`, `Deque`, `BinaryHeap`, arrays, tuples)](209-collections.md): `collections`, `array`, `tuple`

### Formatting / conversion

- [2.10. fmt (`Display`, `Debug`, `Formatter`, `print`)](210-fmt.md): `fmt`, `print`
- [2.11. Conversion (`From`, `Into`, `TryFrom`, `TryInto`, `saturating_into`, `wrapping_into`)](211-conversion.md): `convert`

### Meta / markers

- [2.12. Reflection (`Reflect`, `Inspect`)](212-reflect.md): `reflect`
- [2.13. marker (`Clone`, `Copy`, `Drop`, `NoneSentinel`)](213-marker.md): `marker`

### Errors / strings

- [2.14. Errors (`Error`, `Result`, `TypeError`, `ValueError`, `OverflowError`, `panic`)](214-error.md): `error`, `panic`
- [2.15. Strings (`str`, `StrBuf`, `char`, `c_char`)](215-string.md): `str`

### Concurrency / async

- [2.16. Sync primitives (`Atomic`, `Mutex`, `RwLock`, `Cell`)](216-sync.md): `sync`
- [2.17. `Channel` - message passing](217-channel.md): `channel`
- [2.18. `task::` - M:N scheduler task API](218-task.md): `task`
- `time` - `Duration` / `Instant`, specified with the std clocks in [3.3](../300-std/303-time.md)
- [2.19. `par::` - parallel processing API](219-par.md): `par`
- [2.20. `Future` / `await` integration](220-future.md): `future`

### Cryptography

- [2.21. `crypto::` - hashes, MACs, AEADs, key exchange, signatures](221-crypto.md): `crypto`

### Encoding

- [2.22. `encoding::` - hex and base64](222-encoding.md): `encoding`

### I/O

- [2.23. io (`Read`, `BufRead`, `Write`, `Seek`, `AsyncRead`, `AsyncWrite`, `BufReader`, `Cursor`, stdio)](223-io.md): `io`

### Benchmarks / optimizer hints

- `bench` - `Bench`, the handle a `#[bench]` fn receives. The measurement is specified in [refs-lang 9.9.11](../../refs-lang/009-tools/909-debug-tools.md#9911-benchmarks).

  ```rok
  pub struct Bench { /* private fields */ }   // Debug: `Bench { name: "parse_1kb", .. }`
  impl Bench {
      pub fn iter<T>(ref self, f: fn() -> T);  // calibrate, warm up, sample and report f
      pub fn set_bytes(ref self, n: i64);      // bytes per call: adds a throughput column
      pub fn set_items(ref self, n: i64);      // items per call: adds a throughput column
  }
  ```
- `hint` - `black_box<T>(x: T) -> T`: returns `x` unchanged, but the optimizer cannot see through it, so neither the computation that produced `x` nor a use of the result is folded away. Benchmarks wrap the inputs and result of a timed region with it.

> [!NOTE]
> Scientific computing areas such as tensors, automatic differentiation, SIMD and GPU are not part of the core tier; they are split out into **external libraries**. External libraries reach the compiler's backend through the backend interface, planned for Rok 0.2.2.

## Core design principles

The design of the core tier follows four axes.

First, traits come first. Every semantic decision is made by trait method dispatch: `+` resolves to `Add::add`, and `[i]` indexing resolves to `Index::index`.

Second, conversions are explicit. A conversion that may lose information, such as `i64 → i32`, is rejected with a compile or runtime error, and policies such as saturate or wrap must be stated per operation ([2.11](211-conversion.md)).

Third, the tier aims to be **GC-friendly**. Collections and concurrency objects are managed by the GC. A `Drop` resource is affine: it is released where its last owner leaves scope or at the end of its `using` block ([2.13.4](213-marker.md)).

Finally, **target independence** is guaranteed. Every item behaves identically on all Tier 1 platforms, and target-dependent items are split out into `std`.
