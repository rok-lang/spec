# 2. Rok tier

The rok tier is the safe standard library: the practical surface of rok that builds on every target. Because about 90% of user code uses items from this tier, it is the layer that determines what rok looks like in everyday programming.

## Location

The specification documents live at:

- spec: `spec/en/refs-standard-libraries/200-core/`

## Dependencies

Upward, the rok tier depends on `unsafe` for internal use; downward, `std` and user code depend on it. The rok tier therefore sits at the center of the safe region, between unsafe and std.

## User import policy

`use rok::core::**;` is **applied implicitly**: the compiler prepends it to every user file. As a result, every pub item of rok (List, Dict, Add, Display, Iterator, Mutex, task, ...) is usable without an explicit import.

Narrow explicit imports are also allowed where needed.

```rok
use rok::core::collections::List;           // narrow, explicit
use rok::core::ops::{Add, Sub};
```

> [!NOTE]
> rok has **no separate `rok::core::prelude` module**. The `**` recursive glob itself serves
> as the prelude - no extra indirection.

The full policy is covered in [0.2 Import policy](../000-meta/002-import.md).

## Module index

The modules provided by the rok tier, grouped by area:

### Numbers / niche

- [2.1. Numeric types (`int`, `float`, primitive unification)](201-numbers.md)
- [2.2. Niche wrappers (`NonZero`, `NonNan`, `Finite`, `Positive`)](202-non-zero.md)

### Operator traits

- [2.3. Arithmetic / bitwise / logical (`Add`, `Sub`, `Mul`, `Div`, `Rem`, `Neg`, `Not`, `BitNot`, `BitAnd`, `BitOr`, `BitXor`, `Shl`, `Shr`, `LogicalAnd`, `LogicalOr`)](203-ops.md)
- [2.4. Compound assignment (`AddAssign`, `SubAssign`, ..., `ShrAssign`)](204-ops-assign.md)
- [2.5. Comparison (`PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`)](205-cmp.md)
- [2.6. Broadcast (`DotAdd`, `DotMul`, ..., `DotShr`)](206-broadcast.md)

### iter / stream

- [2.7. iter (`Iterator`, `IntoIterator`, `Range`, `RangeInclusive`)](207-iter.md)
- [2.8. `Stream` (async iterator)](208-stream.md)

### Collections

- [2.9. Collections (`List`, `Dict`, `Set`, `Tuple`, `Slice`, `OrderedDict`, `OrderedSet`)](209-collections.md)

### Formatting / conversion

- [2.10. fmt (`Display`, `Debug`, `Formatter`)](210-fmt.md)
- [2.11. Conversion (`From`, `Into`, `TryFrom`, `TryInto`, `saturating_into`, `wrapping_into`)](211-conversion.md)

### Meta / markers

- [2.12. Reflection (`Reflect`, `Inspect`)](212-reflect.md)
- [2.13. marker (`Clone`, `Copy`, `Drop`, `NoneSentinel`)](213-marker.md)

### Errors / strings

- [2.14. Errors (`Error`, `Result` (retained), `Option` (removed), `TypeError`, `ValueError`, `IndexError`, `OverflowError`)](214-error.md)
- [2.15. Strings (single mutable `str`, `char`, `c_char`)](215-string.md)

### Concurrency / async

- [2.16. Sync primitives (`Atomic`, `Mutex`, `RwLock`, `Cell`)](216-sync.md)
- [2.17. `Channel` - message passing](217-channel.md)
- [2.18. `task::` - M:N scheduler task API](218-task.md)
- [2.19. `par::` - parallel processing API](219-par.md)
- [2.20. `Future` / `await` integration](220-future.md)

### Cryptography

- [2.21. `crypto::` - hashes, MACs, AEADs, key exchange, signatures](221-crypto.md)

> [!NOTE]
> Scientific computing areas such as tensors, automatic differentiation, SIMD and GPU are not part of the rok tier; they are split out into **external libraries**. External libraries can access MLIR dialects directly through the [`rok::bare::mlir`](../100-bare/) module.

## Core design principles

The design of the rok tier follows four axes.

First, traits come first. Every semantic decision is made by trait method dispatch: `+` resolves to `Add::add`, and `[i]` indexing resolves to `Index::index`.

Second, conversions are explicit. A conversion that may lose information, such as `i64 → i32`, is rejected with a compile or runtime error, and policies such as saturate or wrap must be stated per operation (refs-standard-libraries 0.x - conversion policy).

Third, the tier aims to be **GC-friendly**. Collections and concurrency objects are managed by the GC, and `drop` of external resources is called only in `using` blocks.

Finally, **target independence** is guaranteed. Every item behaves identically on all Tier 1 platforms, and target-dependent items are split out into `std`.
