# 4. Functions overview

This chapter defines functions and their variants: closures, methods and coroutines. This document gives the big picture before the detailed rules.

## Contents

- [4.1. Function Declarations](401-function-declaration.md)
- [4.2. Parameters](402-parameters.md)
- [4.3. Return](403-return.md)
- [4.4. Function Types](404-function-type.md)
- [4.5. Closures](405-closure.md)
- [4.6. Capture Semantics](406-capture-semantics.md)
- [4.7. The `ref` Keyword](407-ref-keyword.md)
- [4.8. Trailing Closures](408-trailing-closure.md)
- [4.9. Methods](409-method.md)
- [4.10. Nested Functions](410-nested-functions.md)
- [4.11. Generators (`yield`)](411-generators.md)
- [4.12. Async (`async` and `await`)](412-async.md)
- [4.13. Streams (`async yield`)](413-stream.md)
- [4.14. `throws` Signatures](414-throws.md)
- [4.15. `unsafe` (Function Modifier and Block Expression)](415-unsafe.md)
- [4.16. Entry Point (`main.rok`, `fn main`, Top-Level Statements)](416-entry-point.md)

## The role of functions

A function is a unit of code that takes input values and produces a result value. A Rok function can have all of the following characteristics.

By naming: there are named functions written `fn name(...)` and anonymous closures written `|...|`. By type annotation: a function is static or gradual. If a parameter's type is written, it is static; if omitted, generic monomorphization happens at the call site ([`any` slot](../002-types/202-any-slot.md), [generics](../002-types/210-generics.md)). By execution model: synchronous, asynchronous and generator forms are all possible - plain functions, `async fn` (returns a Future), `yield fn` (returns an Iterator) and `async yield fn` (returns a Stream). By exception behavior: a function either can or cannot throw. Putting `throws` in the signature allows it to throw ([error control flow](../003-expressions/314-error-control-flow.md)). Finally, by ownership: a function is either a method, defined in an `impl` block and attached to a type, or a free function at module level.

## Two forms of function definition

Functions are defined in two forms.

### Named functions

The first is a function with a name.

```rok
fn area(r: f64) -> f64 {
    3.14159 * r * r
}
```

This form consists of the `fn` keyword, a name, parameters, a return type and a body. It may be defined at module level or inside an [`impl`](../002-types/209-impl.md) block.

### Anonymous closures

The second is a closure without a name.

```rok
|x: f64| -> f64 { 3.14159 * x * x }
|x| x * 2
|| compute()
```

This form encloses the parameter list in `|...|` followed by the body. It appears directly in expression position and may be assigned to a variable or passed directly as a function argument.

Both forms are handled by the same internal mechanism. The only differences are whether there is a name and whether an environment is captured when the closure is created.

## Calls

Both functions and closures are called with `()`.

```rok
let a = area(2.0f64);
let f = |x: f64| x * 2;
let b = f(3.0f64);
```

On a call, arguments are matched to parameters. The exact matching rules are defined in [Parameters](402-parameters.md).

## Return

The value of the last expression of the body becomes the function's return value. Function bodies follow the block rules of [block expressions](../003-expressions/309-block-expressions.md).

```rok
fn add(x: i32, y: i32) -> i32 { x + y }
fn early(x: i32) -> i32 {
    if x < 0 { return 0; }     // explicit return for early exit
    x * 2
}
```

The `return` keyword is therefore not required; the last expression is returned automatically. The detailed rules are in [Return](403-return.md).

## Functions and expressions

Functions and closures are values. They may be assigned to variables, passed as arguments to other functions, and stored in collections.

```rok
let f = |x| x * 2;
let v = list.map(f);
let g = if cond { f } else { |x| x + 1 };
```

Function types are defined in [Function Types](404-function-type.md) and consist of parameter types and a return type.

## Call dispatch

Following the [trait-as-type ban](../002-types/212-trait-as-type-banned.md), function call dispatch is resolved without vtable indirection.

A named function is called directly at a statically determined function address. A closure is a pair of a captured environment and a code pointer; a call is a direct call that passes the captured environment as the first argument. A generic function is resolved to a direct call at each call site through the monomorphization of [generics](../002-types/210-generics.md).

Function and closure calls are thus direct calls in principle and can be inlined. Their performance is equivalent to ordinary method calls.
