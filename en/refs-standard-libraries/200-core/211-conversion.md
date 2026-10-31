# 2.11. Conversion traits

The `rok::core::convert` module defines the family of traits that move a value from one type to another. The central principle of Rok's conversion policy is to reject silent information loss. `.into()` therefore allows only lossless conversions; for a conversion that may lose information, the user must choose the policy explicitly.

The syntax of [refs-lang 2.14 Conversions](../../refs-lang/002-types/214-cast-and-conversion.md) dispatches to these traits.

## 2.11.1. Conversion policy - four kinds

Rok separates conversions into four kinds by **intent**. The user must state which kind applies each time.

| Kind         | Method                              | Meaning                                    | On failure                  |
|--------------|-------------------------------------|--------------------------------------------|-----------------------------|
| **lossless** | `.into::<T>()`                      | zero information loss; always succeeds     | compile / runtime error     |
| **fallible** | `.try_into::<T>()`                  | may lose information; caller handles it    | throws the target's `TryFrom::Error` |
| **saturate** | `.saturating_into::<T>()`           | out of range clamps to i32::MAX / MIN      | explicit saturate (user's choice) |
| **bit**      | `.wrapping_into::<T>()` or `as T`   | bit reinterpretation: modular arithmetic / bit pattern  | explicit wrap (user's choice) |

Each kind's method name states the intent by itself. Writing `.into()` means **lossless** intent; writing `.saturating_into()` means **saturate** intent. The compiler does not guess the meaning.

### Design rationale

A silent saturate or silent wrap alters a value without the user knowing, producing bugs that are hard to debug. Rok adopts both the lesson JS learned in introducing Number → BigInt and the lesson NumPy learned in moving from wrap to OverflowError.

## 2.11.2. `From<U>` trait - lossless conversion

```rok
trait From<U> {
    fn from(value: U) -> Self;
}
```

`From::from(u)` is a conversion that always succeeds with no information loss.

### Example

```rok
let n: i32 = 42;
let big: i64 = i64::from(n);     // i32 → i64 (always lossless)

let x: int = int::from(42i32);   // i32 → int (lossless)
```

### import

`From` is available automatically through the [implicit `use rok::core::**;`](../000-meta/002-import.md), so no separate `use` declaration is needed.

## 2.11.3. `Into` trait - mirror of `From`

```rok
trait Into {
    fn into<T: From<Self>>(self) -> T;
}

impl<S> Into for S {
    fn into<T: From<S>>(self) -> T { T::from(self) }
}
```

`Into` is a trait with no type parameter and a single generic method `into`. One blanket impl gives every type `into`, so `x.into()` is `T::from(x)`. The target type `T` is determined by the expected type (a `let` annotation, an argument position, etc.) or by `x.into::<T>()`. Users do not implement `Into`; they write only `From`.

A convertibility bound is written on the `From` side. There is no bound `T: Into<U>`.

```rok
fn widen<T, U: From<T>>(x: T) -> U { x.into() }      // OK (U: From<T>)
fn widen2<T: Into<U>, U>(x: T) -> U { x.into() }     // ERROR: Into takes no type argument
```

### Example

```rok
let n: i32 = 42;
let big: i64 = n.into();         // i32::into::<i64>() automatically, using From<i32> for i64
let big_typed: i64 = (42i32).into();
```

### Prefer `From` - design pattern

An explicit conversion can be written in either of two forms.

```rok
// form 1: From directly
let big = i64::from(42i32);

// form 2: into method
let big: i64 = (42i32).into();
```

The two forms are semantically equivalent. With `from` the target type is visible at the call site; with `into` the **annotation** determines the target type. Choose whichever suits readability or the pattern.

### Compile-time check of losslessness

Attempting a combination for which `From<U> for T` is not implemented is a compile error.

```rok
let big: i64 = 1000;
let small: i32 = big.into();   // compile error: no From<i64> for i32
                                //   (not lossless)
```

In that case one of `try_into`, `saturating_into` or `as` must be chosen explicitly.

```rok
let small: i32 = try big.try_into();       // throws, caller handles it
let small: i32 = big.saturating_into();    // explicit saturate
let small: i32 = big as i32;               // explicit wrap
```

## 2.11.4. `TryFrom<U>` trait - fallible conversion

```rok
trait TryFrom<U> {
    type Error = ConversionError;
    fn try_from(value: U) throws (Self::Error) -> Self;
}
```

`try_from` expresses in its signature that the conversion may fail. The error type is the associated type `Error`, whose default is [`ConversionError`](214-error.md) (an impl that omits it gets the default, [refs-lang 2.8.3](../../refs-lang/002-types/208-trait.md)). Numeric conversion impls (integer ↔ integer, floating-point → integer, `int` / `float` conversions, `TryFrom<u32> for char`) declare `type Error = OverflowError;`. Because the signature is `throws`, the user marks the call with `try` to propagate, handles it with `catch`, or receives `T | none` with `try?`.

```rok
let big: i64 = 9999999999;
let small_result: i32 = try i32::try_from(big);  // propagate with try
let small_catch: i32 = try i32::try_from(big) catch e: OverflowError { default };
let small_opt: i32 | none = try? i32::try_from(big);
```

### Automatic `TryInto` mirror

The same pattern as `From` ↔ `Into` applies.

```rok
trait TryInto {
    fn try_into<T: TryFrom<Self>>(self) throws (T::Error) -> T;
}
impl<S> TryInto for S { ... }            // = try T::try_from(self)
```

In the `OverflowError { op, ty }` ([2.14](214-error.md)) thrown by a numeric `try_from`, `op` is `"try_from"` and `ty` is the target type's name (`"i32"`, etc.).

```rok
let small: i32 = try big.try_into();     // = try i32::try_from(big)
```

## 2.11.5. `saturating_into` / `wrapping_into` - explicit policy

For a conversion that may lose information, the user states the value-altering policy explicitly. Two methods are provided for this.

```rok
trait SaturatingFrom<U> {
    fn saturating_from(value: U) -> Self;    // out of range: nearest end
}
trait SaturatingInto {
    fn saturating_into<T: SaturatingFrom<Self>>(self) -> T;
}

trait WrappingFrom<U> {
    fn wrapping_from(value: U) -> Self;      // keeps the low bits (= as)
}
trait WrappingInto {
    fn wrapping_into<T: WrappingFrom<Self>>(self) -> T;
}
```

The shape matches `From` / `Into`. The conversion itself is the `SaturatingFrom` / `WrappingFrom` impl; `saturating_into` / `wrapping_into` are generic methods supplied by a blanket impl.

### `saturating_into` - clamp to the range ends

```rok
let big: i64 = 9999999999;
let small: i32 = big.saturating_into();    //→ i32::MAX (= 2147483647)

let big_neg: i64 = -9999999999;
let small_neg: i32 = big_neg.saturating_into();  //→ i32::MIN
```

This policy is useful in DSP, signal processing, UI clamping and similar domains.

### `wrapping_into` - modular arithmetic

```rok
let big: i64 = 0x100000001;
let small: i32 = big.wrapping_into();      //→ 1 (low 32 bits)
```

It means the same as `as i32`, but expresses the intent as an explicit method call. This policy suits hashing and bit-masking domains.

## 2.11.6. `as T` - bit reinterpretation / numeric cast

The `as` keyword ([refs-lang 2.14](../../refs-lang/002-types/214-cast-and-conversion.md)) is the means of making a numeric / bit conversion **explicit**.

```rok
let n: i32 = 256;
let b: u8 = n as u8;             // low 8 bits = 0
let b2: u8 = 257 as u8;          // 1
let b_neg: u8 = -1i32 as u8;     // 255

let f: f64 = 1.7;
let i: i32 = f as i32;            // 1 (truncation, IEEE 754 → integer)
let big: i32 = 1e10 as i32;       // i32::MAX (saturates)
let nan: i32 = f64::NAN as i32;   // 0
let f2: f32 = 3.14f64 as f32;     // 3.14 → f32 (IEEE 754 narrowing)
```

`as` behaves as follows.

| Conversion                                         | Meaning                                            |
|----------------------------------------------------|----------------------------------------------------|
| integer → integer of the same width (signed/unsigned) | same bit pattern; only the sign interpretation changes  |
| narrow integer → wide integer                      | sign-extend (signed) or zero-extend (unsigned)     |
| wide integer → narrow integer                      | truncation (low bits only)                         |
| integer → floating-point                           | to IEEE 754 representation (may lose precision)    |
| floating-point → integer                           | truncate toward zero; out of range saturates to MIN/MAX, NaN → 0 (every optimization level) |
| floating-point → floating-point of same/other width | IEEE 754 round                                    |
| `u8` → `char`                                      | that code point (U+0000..=U+00FF)                  |
| `char` → integer                                   | code point (truncated for a narrow integer)        |

`as` to `char` works only from `u8`. Other integers / floating-point / `bool` / `c_char` are a type error; a wider value is converted with `char::from_u32(n)` (`char | none`) ([2.15.4](215-string.md)).

### Limits of `as`

`as` is limited to numeric and bit conversions. Conversion between arbitrary types is rejected.

```rok
struct Vec3 { x: f64; y: f64; z: f64; }
let v = Vec3 { x: 1.0, y: 2.0, z: 3.0 };
let s: str = v as str;     // compile error: `as` rejects user structs
```

Conversion between user types must be stated with the `From` / `Into` traits.

## 2.11.7. Literal-context auto-From - compile-time range check

For an annotated literal, a compile-time range check and a lossless conversion apply together.

```rok
let x: i32 = 42;             // OK: 42 is in i32 range
let y: i32 = 9999999999;     // compile error: literal outside i32 range
let z: f32 = 3.14;           // OK: round to nearest f32
let w: u8 = 256;             // compile error: 256 ∉ u8 0..=255
```

The rule for selecting an impl from a literal, as in `T::from(1)` / `1.into()`, follows [refs-lang 3.1.3](../../refs-lang/003-expressions/301-operator-dispatch.md#selecting-an-impl). This mechanism works because the literal is a value known at compile time. Converting a runtime value requires one of `from`, `into`, `try_into` or `as` to be written explicitly.

## 2.11.8. `From` impl matrix - between primitives

The `From<U> for T` impls provided by the standard library are tabulated below (lossless only).

### Integer → integer (signed)

| from \ to | i8 | i16 | i32 | i64           | isize         | int |
|-----------|----|-----|-----|---------------|---------------|-----|
| `i8`      |    | ✓   | ✓   | ✓             | ✓             | ✓   |
| `i16`     |    |     | ✓   | ✓             | ✓             | ✓   |
| `i32`     |    |     |     | ✓             | ✓ (target≥32) | ✓   |
| `i64`     |    |     |     |               | ✓ (target=64) | ✓   |
| `isize`   |    |     |     | ✓ (target≤64) |               | ✓   |

### Integer → integer (unsigned)

| from \ to | u8 | u16 | u32 | u64           | usize         | int (always lossless) |
|-----------|----|-----|-----|---------------|---------------|-----------------------|
| `u8`      |    | ✓   | ✓   | ✓             | ✓             | ✓                     |
| `u16`     |    |     | ✓   | ✓             | ✓             | ✓                     |
| `u32`     |    |     |     | ✓             | ✓ (target≥32) | ✓                     |
| `u64`     |    |     |     |               | ✓ (target=64) | ✓                     |
| `usize`   |    |     |     | ✓ (target≤64) |               | ✓                     |

`TryFrom<u64> for usize` and `TryFrom<i64> for isize` exist on every target, beside the `From` that only 64-bit targets have, so `usize::try_from(n)` builds everywhere; on a 64-bit target it never throws.

### Unsigned → signed (lossless only)

| from \ to | i8 | i16 | i32 | i64 | int |
|-----------|----|-----|-----|-----|-----|
| `u8`      |    | ✓   | ✓   | ✓   | ✓   |
| `u16`     |    |     | ✓   | ✓   | ✓   |
| `u32`     |    |     |     | ✓   | ✓   |
| `u64`     |    |     |     |     | ✓   |

A conversion from unsigned to a signed type of the same or smaller width (e.g. u32 → i32) is not guaranteed lossless, so `try_into` or `as` must be used.

### Integer → floating-point

| from \ to | f32 | f64                         |
|-----------|-----|-----------------------------|
| `i8`      | ✓   | ✓                           |
| `i16`     | ✓   | ✓                           |
| `i32`     |     | ✓                           |
| `i64`     |     | (no `From`: loses precision) |

f32 has a 24-bit mantissa and f64 a 53-bit mantissa, so losslessness holds only within that bit width. The unsigned integers have no `From` impl into a floating-point type; use `as`.

### Between floating-point types

| from \ to | f32 | f64 | float |
|-----------|-----|-----|-------|
| `f32`     |     | ✓   | ✓     |
| `f64`     |     |     | ✓     |

f64 → f32 is a narrowing conversion, so one of `try_into`, `saturating_into` or `as` must be used.

### int / float and fixed primitives

| from \ to | i32                   | i64             | f64                        |
|-----------|-----------------------|-----------------|----------------------------|
| `int`     | `try_into` only       | `try_into` only | `into` (rounds, exception) |
| `float`   | `try_into` (truncates) | `try_into`     | `into` (exception)         |

`From<int> for f64` and `From<float> for f64` are **documented exceptions** to "From is lossless" in 2.11.2. `int` rounds to the nearest f64, and a `float` promoted to an exponent outside f64 range cannot be represented as f64 and loses information. Where the loss must be visible, `to_f64()` or `as f64` is recommended.

`int → fixed` is not guaranteed lossless (the `int` may be outside the fixed range), so `try_into` or `saturating_into` must be used.

Conversely, `fixed → int` is always lossless (`From<i32> for int` and the rest are all implemented).

## 2.11.9. `From` impls - user types

Conversion between user types is defined by writing `impl From<U> for T` directly.

```rok
struct Celsius { degree: f64; }
struct Fahrenheit { degree: f64; }

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Fahrenheit {
        Fahrenheit { degree: c.degree * 9.0 / 5.0 + 32.0 }
    }
}

let c = Celsius { degree: 100.0 };
let f: Fahrenheit = c.into();   //→ Fahrenheit { degree: 212.0 }
```

The blanket `Into` routes `c.into()` to `Fahrenheit::from(c)`, so the user does not write `Into`.

### `try_from` follows the same pattern

```rok
impl TryFrom<str> for i32 {
    type Error = OverflowError;
    fn try_from(s: str) throws (Self::Error) -> i32 {
        // if s is an integer string within i32 range, that value; otherwise throw OverflowError
        ...
    }
}

let n: i32 = try "42".try_into();
```

Every fixed-width integer type implements `TryFrom<str>` this way, on top of its `FromStr`: empty text, an invalid digit or an out-of-range value throws `OverflowError { op: "try_from", ty }`. Floating-point types, `bool` and `char` provide only `FromStr`.

## 2.11.10. Automatic conversion of arithmetic results - none

Auto-From for arithmetic results ([refs-lang 2.14](../../refs-lang/002-types/214-cast-and-conversion.md)) applies only in **literal context**. The result of arithmetic on values is not converted automatically.

```rok
let a: i32 = 5;
let b: i32 = 3;
let c: i32 = a + b;          // OK: i32 + i32 = i32

let big: i32 = a + b * 1000000;
// → OK if a + b * 1000000 is within i32 range; otherwise (debug=panic / release=wrap)
// no silent saturate / silent promote

let auto: i64 = a + b;       // compile error: no silent promotion of an i32 result to i64
let auto: i64 = (a + b) as i64;  // explicit conversion OK
let auto: i64 = (a + b).into();  // explicit From<i32> for i64 (lossless)
```

## 2.11.11. Comparison with other languages - design rationale

| Language | Conversion model                                         | Difference from Rok                                                               |
|----------|----------------------------------------------------------|-----------------------------------------------------------------------------------|
| Rust     | `From`/`Into` lossless, `as` wrap, `try_into` Result     | closest to Rok; Rok adds explicit `saturating_into`/`wrapping_into` methods       |
| Swift    | all integer conversions trap by default, `.init(clamping:)` opt-in | Rok rejects trapping: debug=panic + release=wrap (fixed-width arithmetic), conversions are explicit  |
| Julia    | `Int32(big)` traps, `convert` function family            | Rok unifies on trait methods (`from`/`into`)                                      |
| Python   | `int(x)`/`float(x)` functions, always arbitrary precision  | Rok's `int` / `float` share the spirit; fixed primitives are a separate world   |
| C        | implicit conversion (`int` → `long` automatically)       | Rok explicitly rejects it, blocking silent type mixing                            |

From this comparison, Rok's key design decisions are: reject silent information loss (influenced by Rust); force expression of intent by making the four kinds (`into`, `try_into`, `saturating_into`, `as`) explicit; allow auto-From only in literal context and require explicit conversion for runtime values; and separate the auto-promotion of `int` / `float` from fixed primitives, a decision influenced by Python and Smalltalk.

## 2.11.12. Module mapping

| Trait               | lang slot              | Standard library location |
|---------------------|------------------------|---------------------------|
| `From<U>`           | (none, derivable)      | standard library          |
| `Into`              | (blanket impl)         | standard library          |
| `TryFrom<U>`        | (none)                 | standard library          |
| `TryInto`           | (blanket impl)         | standard library          |
| `SaturatingFrom<U>` | (none)                 | standard library          |
| `SaturatingInto`    | (blanket impl)         | standard library          |
| `WrappingFrom<U>`   | (none)                 | standard library          |
| `WrappingInto`      | (blanket impl)         | standard library          |

Every lossless conversion impl between primitives carries `#[intrinsic("rok.cast.FROM.TO")]`.

```rok
impl From<i32> for i64 {
    #[intrinsic("rok.cast.i32.i64")]
    fn from(value: i32) -> i64 { ... }
}
```

The numeric cast of `as T` uses the same intrinsic; the compiler chooses between the lossless and wrapping branches.
