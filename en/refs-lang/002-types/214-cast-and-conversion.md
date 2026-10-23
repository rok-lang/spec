# 2.14. Conversions

This section defines the syntax that moves a value from one type to another. The semantics - which conversions are allowed and how their results are determined - are the responsibility of the standard library's conversion matrix; this document covers only the language syntax.

```
cast_expr   ::= expr "as" type_expr
into_expr   ::= expr "." "into" "(" ")"
try_into    ::= expr "." "try_into" ("::" "<" type_expr ">")? "(" ")"
```

## 2.14.1. Three forms

Following the principle of [The boundary between the language and the standard libraries](../000-meta/004-language-vs-standard-libraries.md), the language provides all three forms as syntax, and each desugars to a standard library trait method. The trait each form desugars to:

| Form               | Shape                  | Desugars to                 |
|--------------------|------------------------|-----------------------------|
| `as` cast          | `expr as Type`         | `Cast<Target>::cast`        |
| `.into()` call     | `expr.into()`          | `Target::from` (`Into::into`) |
| `.try_into()` call | `expr.try_into::<T>()` | `Target::try_from` (`TryInto::try_into`) |

The exact signatures of the three traits, and which implementations are registered for which type pairs, are defined in the standard library's conversion matrix. This document is responsible only for the language-level desugaring rules.

## 2.14.2. `as` casts

The first form is the `as` cast.

```rok
expr as Type
```

`as` is a [keyword](../001-lexical/105-keywords.md); the left side is any expression and the right side is a [type expression](201-type-expressions.md). It desugars to `<typeof(expr) as Cast<Target>>::cast(expr)`.

The `Cast<Target>` trait denotes a direct cast from the left-hand type to `Target`. It is generally used for conversions that may not preserve the value, such as bit truncation, sign extension and representation changes. Which type pairs implement `Cast` is decided by the standard library. Examples:

```rok
let x: i32 = 256i32;
let y: u8 = x as u8;          // Cast<u8>::cast(x)
let z: f64 = x as f64;        // Cast<f64>::cast(x)
```

`as` to `char` is allowed only for `u8 as char` (and `char as char`). Other integer, floating-point, `bool` and `c_char` → `char` casts are type errors, with the message ``only `u8` can be cast as `char`, not `u32`; use `char::from_u32(n)` (`char | none`) for a wider value``. `char as <integer>` is allowed. A payload-less `enum` casts to an integer type, giving its tag or value ([2.4.2](204-enum.md#casting-to-an-integer)).

`as` always succeeds for type pairs with a registered `Cast` impl. For a conversion that may fail, use `.try_into()` instead. Using `as` on a type pair without a `Cast` impl is a compile error.

## 2.14.3. `.into()` calls

Next, the `.into()` call.

```rok
expr.into()
```

This desugars to `Into::into::<Target>(expr)`, i.e. `Target::from(expr)`, and requires an implementation of `Target: From<typeof(expr)>`. The target type is inferred from context, such as the variable's declared type or the parameter type.

```rok
let x: i64 = 100i64;
let y: BigInt = x.into();       // BigInt::from(x)
```

Where the target type cannot be inferred, it can be given with turbofish.

```rok
let y = x.into::<BigInt>();
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
| `as`          | No                   | Yes                     | `Cast<T>`    |
| `.into()`     | No                   | No (value-preserving)   | `From<S>`    |
| `.try_into()` | Yes (throws an error) | No (on success)        | `TryFrom<S>` |

Choose by situation: use `as` to change bits or precision intentionally; use `.into()` when only a safe, lossless widening is needed; use `.try_into()` to handle the possibility of failure explicitly.

## 2.14.6. Context-based automatic conversion

An unsuffixed numeric literal is not converted; its type is determined by the expected type. The `1` in `let x: f64 = 1` is `f64` from the start. The exact rules are defined in [Numeric literals](../001-lexical/106-numeric-literals.md#163-unsuffixed-literals).

Automatic conversion of runtime variables, by contrast, is not allowed. One of the three forms above must always be written.

```rok
let a: i64 = 100i64;
let b: i32 = a;                 // ERROR - explicit conversion required
let b: i32 = a as i32;          // OK
let b: i32 = try! a.try_into::<i32>();     // OK - panics on failure
```

This explicitness prevents silent loss of precision.

## 2.14.7. Where the conversion traits live

`Cast<T>`, `From<U>` / `Into`, and `TryFrom<U>` / `TryInto` are all defined in the `rok` tier of the [standard library tiers](../000-meta/005-standard-libraries-tiers.md). The full matrix of which traits are implemented for which type pairs is defined in the standard library specification.

This document is responsible only for the fact that the language provides the three forms and that each desugars to the corresponding trait above.
