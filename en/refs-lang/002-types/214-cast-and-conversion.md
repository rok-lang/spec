# 2.14. Conversions

This section defines the syntax that moves a value from one type to another. `as` is built into the language; `.into()` and `.try_into()` are ordinary method calls on standard library traits, whose impls form the standard library's conversion matrix.

```
cast_expr   ::= expr "as" type_expr
into_expr   ::= expr "." "into" ("::" "<" type_expr ">")? "(" ")"
try_into    ::= expr "." "try_into" ("::" "<" type_expr ">")? "(" ")"
```

## 2.14.1. Three forms

The three forms and what each one runs:

| Form               | Shape                  | Runs                        |
|--------------------|------------------------|-----------------------------|
| `as` cast          | `expr as Type`         | a built-in cast (no trait)  |
| `.into()` call     | `expr.into()`          | `Target::from` (`Into::into`) |
| `.try_into()` call | `expr.try_into::<T>()` | `Target::try_from` (`TryInto::try_into`) |

There is no `Cast` trait, and a user type cannot add `as` pairs. The exact signatures of `From` / `Into` and `TryFrom` / `TryInto`, and which type pairs implement them, are defined in the standard library's conversion matrix.

## 2.14.2. `as` casts

The first form is the `as` cast.

```rok
expr as Type
```

`as` is a [keyword](../001-lexical/105-keywords.md); the left side is any expression and the right side is a [type expression](201-type-expressions.md). The compiler lowers the cast directly; no trait method is called. It is used for conversions that may not preserve the value, such as bit truncation, sign extension and representation changes, and gives the same result as `.wrapping_into::<T>()` ([stdlib 2.11.5](../../refs-standard-libraries/200-core/211-conversion.md)).

```rok
let x: i32 = 256i32;
let y: u8 = x as u8;          // 0
let z: f64 = x as f64;        // 256.0
```

The allowed pairs are a fixed set:

- Any type to itself (a no-op).
- Between any two of the integer types, the floating-point types, `bool`, `char` and `c_char`. The one restriction is the target `char`: only `u8 as char` (and `char as char`) is allowed, since a wider value may be a surrogate or lie past `U+10FFFF`. Other integer, floating-point, `bool` and `c_char` → `char` casts are type errors, with the message ``only `u8` can be cast as `char`, not `u32`; use `char::from_u32(n)` (`char | none`) for a wider value``.
- A payload-less `enum` to an integer type, giving its tag or value ([2.4.2](204-enum.md#casting-to-an-integer)).
- To or from the arbitrary-precision std types `int` / `float`. A cast into one runs the lossless `From` impl, so a pair without one (`f64 as int`) is an error; a cast out of one runs the target's `WrappingFrom` impl.

Any other pair is a compile error: ``cannot cast `str` as `i32` (only numeric / bool / char casts)``. An allowed `as` always succeeds. For a conversion that may fail, use `.try_into()` instead.

## 2.14.3. `.into()` calls

Next, the `.into()` call.

```rok
expr.into()
```

This desugars to `Into::into::<Target>(expr)`, i.e. `Target::from(expr)`, and requires an implementation of `Target: From<typeof(expr)>`. The target type is inferred from context, such as the variable's declared type or the parameter type.

```rok
let x: i64 = 100i64;
let y: int = x.into();          // int::from(x)
```

Where the target type cannot be inferred, it can be given with turbofish.

```rok
let y = x.into::<int>();
```

`.into()` applies only where the standard library registers a value-preserving conversion, i.e. one without information loss. It is generally used to convert from a narrower type to a wider one.

## 2.14.4. `.try_into()` calls

Finally, the `.try_into()` call.

```rok
expr.try_into::<Type>()
```

This desugars to `Target::try_from(expr)` (`Target: TryFrom<typeof(expr)>`). `try_into` is a [`throws`](../004-functions/414-throws.md) method: on success it returns a value of the target type, and on failure it throws an error (the target's `TryFrom::Error`; for numeric conversions `OverflowError`, [stdlib 2.11.4](../../refs-standard-libraries/200-core/211-conversion.md#2114-tryfromu-trait---fallible-conversion)). The call must therefore be marked with `try`. It is used when the conversion can fail, such as moving a large integer to a smaller integer or a floating-point value to an integer.

```rok
let big: i64 = 10_000_000_000i64;
let n = try big.try_into::<i32>() catch e {
    ...
};
let m: i32 | none = try? big.try_into::<i32>();
```

## 2.14.5. Summary of the three forms

The three forms differ as follows.

| Form          | Can fail?            | Can lose information?   | Trait        |
|---------------|----------------------|-------------------------|--------------|
| `as`          | No                   | Yes                     | none (built-in) |
| `.into()`     | No                   | No (value-preserving)   | `From<S>`    |
| `.try_into()` | Yes (throws an error) | No (on success)        | `TryFrom<S>` |

Choose by situation: use `as` to change bits or precision intentionally; use `.into()` when only a safe, lossless widening is needed; use `.try_into()` to handle the possibility of failure explicitly.

## 2.14.6. Context-based automatic conversion

An unsuffixed numeric literal is not converted; its type is determined by the expected type. The `1` in `let x: f64 = 1` is `f64` from the start. The exact rules are defined in [Numeric literals](../001-lexical/106-numeric-literals.md#163-unsuffixed-literals).

Automatic conversion of runtime variables, by contrast, is not allowed. One of the three forms above must always be written.

```rok
let a: i64 = 100i64;
let b: i32 = a;                 // ERROR: explicit conversion required
let b: i32 = a as i32;          // OK
let b: i32 = try! a.try_into::<i32>();     // OK: panics on failure
```

This explicitness prevents silent loss of precision.

## 2.14.7. Where the conversion traits live

`From<U>` / `Into` and `TryFrom<U>` / `TryInto` are defined in `rok::core::convert` ([standard library tiers](../000-meta/005-standard-libraries-tiers.md)). The full matrix of which traits are implemented for which type pairs is defined in the standard library specification. `as` has no trait; its pairs are the fixed set in [2.14.2](#2142-as-casts).
