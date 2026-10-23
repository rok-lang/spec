# 2.11. `const` generics

`const` generics take a value in a [generic](210-generics.md) parameter position. Integers, booleans and short expressions determined at compile time serve as parameters.

## 2.11.1. Motivation

They carry values determined at compile time - array lengths, matrix dimensions, embedded memory pool sizes - as part of the type.

```rok
struct Matrix<T, const M: usize, const N: usize> {
    data: [T; M * N];
}

fn dot<T: Num, const N: usize>(a: [T; N], b: [T; N]) -> T { ... }
```

Since the array length becomes part of the type, length checks at call time cost nothing. The compiler can also apply length-dependent optimizations such as SIMD and unrolling, and operations where dimension compatibility matters, such as matrix multiplication, can be verified at compile time.

## 2.11.2. Declaration form

```rok
const ParamName : Type
```

The `const` keyword is followed by the parameter name, `:`, and the type. The type must be one of the allowed `const` types described below.

Multiple parameters are separated by commas. Type parameters and `const` parameters may be mixed freely.

```rok
fn replicate<T, const N: usize>(value: T) -> [T; N] { ... }
struct Tagged<T, const TAG: u32> { value: T; }
```

## 2.11.3. Allowed types

The type of a `const` parameter is one of the integer types (`i8` – `i64`, `u8` – `u64`, `usize`, `isize`), `bool`, or `char`.

Floating-point types, sum types and user-defined structs cannot be `const` parameters. Floating-point types are excluded because of nondeterminism in their bit representation, and user-defined types because of consistency problems with equality comparison.

## 2.11.4. Arguments at instantiation

At a call or instantiation site, a `const` expression is passed for a `const` parameter.

```rok
let m: Matrix<f64, 3, 4> = ...;
let v: [i32; 16] = ...;
let arr = replicate::<f64, 8>(0.0f64);
```

The argument must be evaluable at compile time. Allowed are literals, `const` variables, `const` function calls, other `const` parameters, and simple arithmetic combinations of these. A runtime variable cannot be used as a `const` argument; doing so is a compile error.

## 2.11.5. Arithmetic in `const` expressions

Simple arithmetic between `const` parameters can be used inside types.

```rok
struct Matrix<T, const M: usize, const N: usize> {
    data: [T; M * N];
}

fn flatten<T, const M: usize, const N: usize>(
    m: Matrix<T, M, N>
) -> [T; M * N] { ... }
```

The allowed operators are `+`, `-`, `*`, `/`, `%`, and unary `-`. Comparisons and conditionals are not allowed in the current specification; only simple arithmetic. The type system does recognize that `M * N` and `N * M` are the same value (commutativity).

More complex `const` expressions, such as conditionals or recursion, are covered by the `const` function specification.

## 2.11.6. Parameter inference

A `const` argument may be omitted if it can be inferred from argument types, struct field types, or the expected type (`let` annotation, parameter type, return type).

```rok
fn replicate<T, const N: usize>(value: T) -> [T; N] { ... }

let arr: [i32; 8] = replicate(0i32);     // N = 8 inferred
let arr2 = replicate::<i32, 8>(0i32);    // explicit
```

Inference priority and disambiguation are defined in the expression specification.

### Repeat literal `[v; N]`

`[v; N]` is a `[T; N]` array repeating the value `v` `N` times. `N` is a `const` expression and may be a `const` parameter.

```rok
fn zeros<const N: usize>() -> [f64; N] { [0.0; N] }
let z: [f64; 4] = zeros();              // N = 4 - from the expected type
let w = [1u8; 16];                       // [u8; 16]
```

## 2.11.7. Monomorphization

`const` generics work the same way as monomorphization of [generics](210-generics.md). A separate compilation result is produced for each combination of `const` arguments.

```rok
let a = replicate(0i32);    // N = ? - determined at inference time
```

Calls with the same combination of `const` arguments share the result.

## 2.11.8. Equality

Two types whose `const` arguments have the same values are the same type.

```rok
let a: [i32; 4] = ...;
let b: [i32; 2 + 2] = ...;
// a and b have the same type
```

This equality is possible because `const` expression evaluation is deterministic.

## 2.11.9. Where they may be defined

`const` generics can be used with the same syntax in all of these positions: functions and methods; structs, `enum`s and `union`s; traits, `impl` blocks and type aliases.

This uniformity matches ordinary generics in [Generics](210-generics.md), so there are no extra rules to memorize.

## 2.11.10. Use as an index

A `const` parameter can be used directly in value position, like a plain `const` variable.

```rok
fn first<T, const N: usize>(arr: [T; N]) -> T {
    arr[0]
}

fn middle<T, const N: usize>(arr: [T; N]) -> T {
    arr[N / 2]
}
```

The exact rules for use in value position are defined in the expression specification.
