# 2.11. `const` generics

`const` generics take a value in a [generic](210-generics.md) parameter position. Integers, booleans and short expressions determined at compile time serve as parameters.

## 2.11.1. Motivation

They carry values determined at compile time (array lengths, matrix dimensions, embedded memory pool sizes) as part of the type.

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

The type of a `const` parameter is one of the integer types (`i8` to `i64`, `u8` to `u64`, `usize`, `isize`), `bool`, `char`, or `c_char`.

Floating-point types, `str`, sum types and user-defined structs cannot be `const` parameters; such a parameter is an error at its declaration. Floating-point types are excluded because of nondeterminism in their bit representation, and user-defined types because of consistency problems with equality comparison.

## 2.11.4. Arguments at instantiation

At a call or instantiation site, a `const` expression is passed for a `const` parameter.

```rok
let m: Matrix<f64, 3, 4> = ...;
let v: [i32; 16] = ...;
let arr = replicate::<f64, 8>(0.0f64);
```

The argument must be evaluable at compile time. Allowed are literals, `const` variables, `const` function calls, other `const` parameters, and simple arithmetic combinations of these. A runtime variable cannot be used as a `const` argument; doing so is a compile error. The argument's type must be the parameter's type: for `const X: usize`, `S<true>` and `S<-1>` are errors.

In a generic argument list, a bare argument is a literal, a negated literal (`-1`) or a name (`N`, a `const` variable). Anything else, arithmetic included, is wrapped in braces:

```rok
fn grow<const N: usize>(b: Buf<N>) -> Buf<{ N + 1 }> { ... }
let b: Buf<{ 2 * 8 }> = ...;
```

An array length (`[T; M * N]`) is already an expression position and takes no braces.

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

`[v; N]` is a `[T; N]` array of `N` elements, each a copy of `v`. `N` is a `const` expression and may be a `const` parameter.

`v` is evaluated exactly once, before any element is written, so its side effects happen once (also when `N` is 0). Each element is then a value of its own:

- If `T` is `Copy`, or a value type with nothing in it that refers to shared storage (numbers, `bool`, `char`, `str`, `none`, functions, and arrays, tuples, structs and enums of these), every element is a bitwise copy of `v` and no `clone` is called.
- Otherwise elements `0` to `N - 2` are `v.clone()`, called in index order, and the last element is `v` itself. A `List` element, an array of lists or a struct that holds a `Dict` gives `N` separate collections, so writing through one element leaves the others unchanged.
- If `T` is not `Clone`, the literal is an error (E0069) whose help writes `List::from_fn(N, |_| ..)`, which builds each element with its own value. A type parameter needs a `T: Clone` bound.
- If `T` holds a value with one owner (a must-close type such as `File`, or a `Drop` type, at any array depth), the literal is an error with the same help: `N` elements cannot share one owner.

```rok
fn zeros<const N: usize>() -> [f64; N] { [0.0; N] }
let z: [f64; 4] = zeros();              // N = 4, from the expected type
let w = [1u8; 16];                       // [u8; 16]

let g = [[0; 3]; 2];
g[0][1] = 5;                             // g[1] is still [0, 0, 0]
let rows = [List::<i32>::new(); 3];
rows[0].push(1);                         // rows[1] and rows[2] stay empty
let fs = [try File::open("a"); 2];       // ERROR: a `File` has one owner (E0069)
```

## 2.11.7. Monomorphization

`const` generics work the same way as monomorphization of [generics](210-generics.md). A separate compilation result is produced for each combination of `const` arguments.

```rok
let a = replicate(0i32);    // N = ?, determined at inference time
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
