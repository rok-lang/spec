# 2.2. Niche wrappers - `NonZero` / `NonNan` / `Finite` / `Positive`

`rok::core::num::niche` provides wrappers that restrict a value to a subset. Each wrapper is a newtype carrying `#[niche(..)]` ([refs-lang 2.7.4](../../refs-lang/002-types/207-niche-optimization.md#274-opt-in-niches-niche)), and the excluded values become the niche of `T | none`.

## 2.2.1. Size

A `T | none` with a niche has the size of `T` ([refs-lang 2.7.2](../../refs-lang/002-types/207-niche-optimization.md#272-automatic-niches)). `none` is represented by the niche value; the actual bits are chosen by the implementation.

| Type                     | `T \| none` size | Niche value of `none` |
|--------------------------|-----------------|-----------------------|
| `i32 \| none`             | 8 B (tag)       | none                  |
| `NonZero<i32> \| none`    | 4 B             | 0                     |
| `f64 \| none`             | 16 B (tag)      | none                  |
| `NonNan<f64> \| none`     | 8 B             | NaN                   |
| `Finite<f64> \| none`     | 8 B             | NaN, ±∞               |
| `Positive<i32> \| none`   | 4 B             | negative values       |
| `Positive<u32> \| none`   | 8 B (tag)       | none                  |

When the niche value is 0 (`NonZero`), the payload bits are unchanged, so `NonZero<T> | none` crosses the C ABI as an integer in which 0 is `none`.

## 2.2.2. `NonZero<T>` - zero forbidden

```rok
trait NicheInt { .. }            // implemented by the 10 integer types

#[niche(0)]
struct NonZero<T>(T);

impl<T: NicheInt> NonZero<T> {
    fn new(value: T) -> NonZero<T> | none;
    unsafe fn new_unchecked(value: T) -> NonZero<T>;   // UB if value == 0
    fn get(self) -> T;
}
```

Only integer types are accepted. Floating-point types use `NonNan` / `Finite`.

| Operation                   | Result                                            |
|-----------------------------|---------------------------------------------------|
| `NonZero<T> * NonZero<T>`   | `NonZero<T>`. Panics if the product wraps to 0    |
| `NonZero<T> + - / NonZero<T>` | `T`                                             |
| `T / NonZero<T>`            | `T`. No division by zero                          |

`TryFrom<T>` throws `ZeroError` on 0. `From<NonZero<T>> for T` is lossless.

```rok
let n: NonZero<i32> = NonZero::new(42) ?? panic("zero");
let nz: NonZero<i32> = try 7.try_into();
let v: i32 = nz.into();
```

## 2.2.3. `NonNan<F>` - NaN forbidden

```rok
trait NicheFloat { .. }          // implemented by f32, f64

#[niche(F::NAN)]
struct NonNan<F>(F);

impl<F: NicheFloat> NonNan<F> {
    fn new(value: F) -> NonNan<F> | none;
    unsafe fn new_unchecked(value: F) -> NonNan<F>;    // UB if NaN
    fn get(self) -> F;
}
```

- Implements `Eq`, `Ord`, `Hash`. `-0.0 == 0.0`, and their hashes are equal.
- The result of `+ - * /` is `F` (∞ − ∞ and the like may be NaN).
- `TryFrom<F>` throws `ConversionError` on NaN.

## 2.2.4. `Finite<F>` - NaN and ±∞ forbidden

```rok
#[niche(F::NAN, F::INFINITY, F::NEG_INFINITY)]
struct Finite<F>(F);

impl<F: NicheFloat> Finite<F> {
    fn new(value: F) -> Finite<F> | none;
    unsafe fn new_unchecked(value: F) -> Finite<F>;    // UB if NaN / ±∞
    fn get(self) -> F;
}
```

- Implements `Eq`, `Ord`, `Hash`. `-0.0 == 0.0`.
- `+ - *` yield `Result<Finite<F>, OverflowError>` (`Err` if the result is ±∞).
- `TryFrom<F>` throws `ConversionError` on NaN / ±∞.

## 2.2.5. `Positive<T>` - negative values forbidden

```rok
#[niche(T::MIN..=-1)]
struct Positive<T>(T);

impl<T: NicheInt> Positive<T> {
    fn new(value: T) -> Positive<T> | none;            // none if negative; 0 is included
    unsafe fn new_unchecked(value: T) -> Positive<T>;
    fn get(self) -> T;
}
```

The value is 0 or greater. An unsigned `T` has no negative values, so it has no niche.

## 2.2.6. `new_unchecked`

`new_unchecked` is an `unsafe fn`. Violating the invariant is undefined behavior, and `T | none` may read the value as `none`.

```rok
let n: i32 = compute();                                // the caller guarantees it is not 0
let nz: NonZero<i32> = unsafe { NonZero::new_unchecked(n) };
```

## 2.2.7. Validation

- In const position, `NonZero::new(0)` and the like are evaluated at compile time to `none`. With `?? panic(..)` attached it is a compile error ([refs-lang 5.2.4](../../refs-lang/005-modules/502-const.md#524-constant-expressions-and-the-evaluator)).
- At runtime, validation uses `new` (`T | none`) and `try_from` (throws).

## 2.2.8. Definition summary

| Type          | Bound        | `#[niche]`                                 |
|---------------|--------------|--------------------------------------------|
| `NonZero<T>`  | `NicheInt`   | `0`                                        |
| `Positive<T>` | `NicheInt`   | `T::MIN..=-1`                              |
| `NonNan<F>`   | `NicheFloat` | `F::NAN`                                   |
| `Finite<F>`   | `NicheFloat` | `F::NAN, F::INFINITY, F::NEG_INFINITY`     |

## 2.2.9. Comparison with other languages

| Aspect       | Rust                          | rok                                   |
|--------------|-------------------------------|---------------------------------------|
| `NonZero<T>` | `NonZero<T>` (integers)       | `NonZero<T: NicheInt>`                |
| `NonNan` / `Finite` | None (external crate)  | Built in                              |
| Automatic niche | `Option<Box<T>>` = `Box<T>` | `T \| none`                            |
| User niche   | Unstable attribute            | `#[niche(..)]`                         |
