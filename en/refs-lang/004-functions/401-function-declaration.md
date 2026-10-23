# 4.1. Function declarations

This section defines the declaration form of named functions, which begin with the `fn` keyword ([keywords](../001-lexical/105-keywords.md)).

## 4.1.1. Form

The grammar of a function declaration is:

```
fn_decl       ::= visibility? fn_modifiers "fn" identifier "(" param_list? ")" throws_clause? return_type? block
fn_modifiers  ::= "unsafe"? "async"? "yield"?
return_type   ::= "->" type_expr
param_list    ::= param ("," param)* ","?
```

Here `visibility?` is one of `pub`, `pub(pkg)`, `pub(sub)` from [visibility](../005-modules/506-visibility.md), or omitted (module-private). `fn_modifiers` is the position for the three modifiers [unsafe](415-unsafe.md), [async](412-async.md) and [yield (generators)](411-generators.md), each written **zero or one time in a fixed order** (the detailed rules are in [4.1.11 Order and combination of fn modifiers](#4111-order-and-combination-of-fn-modifiers)). The meaning of `throws_clause?` is defined in [throws](414-throws.md); this document does not cover the meaning of visibility, the modifiers or the throws keyword themselves.

Informally, the overall shape is:

```rok
fn name(parameters) -> ReturnType {
    body
}
```

The components mean:

- It begins with the `fn` keyword.
- The function name follows the form of an [identifier](../001-lexical/104-identifiers.md).
- A parenthesized parameter list follows; the detailed rules are defined in [Parameters](402-parameters.md).
- The return type is optional: a [type expression](../002-types/201-type-expressions.md) after the `->` token ([punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md)).
- The body is a [block expression](../003-expressions/309-block-expressions.md).

## 4.1.2. The simplest examples

The simplest function declarations look like this:

```rok
fn greet() {
    print("hello");
}

fn add(x: i32, y: i32) -> i32 {
    x + y
}
```

## 4.1.3. Omitting the return type

The `->` part of the return type may be omitted, with the following meaning:

- The return type is inferred from the body; the type of the last expression of the body becomes the return type.
- If it cannot be inferred or is ambiguous, it is a compile error.

In the following example the return type is omitted, but `i32` is inferred from the type of the last expression.

```rok
fn double(x: i32) {              // return type omitted
    x * 2                         // inferred: i32
}
// the above is equivalent to fn double(x: i32) -> i32
```

If the body produces no value (the last expression ends with a terminator), the return type is unit (`()`).

```rok
fn print_msg(s: str) {
    print(s);                     // terminator at the end — returns unit
}
```

If the return type is written but the function body does not match it, it is a compile error.

```rok
fn bad() -> i32 {
    "hello"                       // ERROR — str is not i32
}
```

## 4.1.4. Omitting parameter types

A parameter without a type is treated as an implicit generic ([`any` slot](../002-types/202-any-slot.md), [generics](../002-types/210-generics.md)).

```rok
fn id(x) {                        // x: T (generic)
    x
}
// monomorphized by the type at the call site
let a = id(42i32);                // id<i32>
let b = id("hi");                 // id<str>
```

If a parameter is meant to be a dynamic slot, write `any` explicitly.

```rok
fn dyn_id(x: any) {               // x is an any slot (runtime dispatch)
    x
}
```

## 4.1.5. Body

A function body follows the rules of [block expressions](../003-expressions/309-block-expressions.md) and has these properties:

- It contains zero or more statements.
- It may end with zero or one final expression (without a terminator), whose value becomes the function's return value.
- It may return explicitly at the end with `return` ([`return`](403-return.md)).

## 4.1.6. Empty body

A function with an empty body returns unit.

```rok
fn nothing() { }                  // empty body — returns unit
```

## 4.1.7. Return type and the last expression of the body

In a function with an explicit return type, a last expression without a terminator becomes the return value.

```rok
fn f() -> i32 {
    42i32                         // last expression is the return value
}
```

In the example above, `42i32` has no terminator, so it becomes the function's return value. If it is written with a terminator (`42i32;`), the value is discarded and the body returns unit, which conflicts with the declared `-> i32` and is a compile error.

```rok
fn f() -> i32 {
    42i32;                        // ERROR — terminator discards the value
                                  // (does not match return type i32)
}
```

## 4.1.8. Explicit `return` in a function body

Inside the body, the `return` keyword returns explicitly; details are defined in [`return`](403-return.md).

```rok
fn early(x: i32) -> i32 {
    if x < 0 { return -1; }
    x * 2
}
```

## 4.1.9. Visibility

A visibility keyword (`pub`, `pub(pkg)`, `pub(sub)`) on a function controls how far it is exposed.

```rok
pub fn area(r: f64) -> f64 { ... }              // callable from external packages
pub(pkg) fn helper(x: i32) -> i32 { ... }       // only within the same package
fn _internal(s: str) { ... }                     // module-private (default)
```

The exact meaning of each visibility level is defined in [visibility](../005-modules/506-visibility.md).

## 4.1.10. No duplicate definitions in the same module

Declaring two functions with the same name in the same module is a compile error. rok has no overloading.

```rok
fn add(x: i32, y: i32) -> i32 { x + y }
fn add(x: f64, y: f64) -> f64 { x + y }   // ERROR — duplicate
```

To vary behavior by type, combine traits and generics ([`trait`](../002-types/208-trait.md)).

Functions, `const`s and `static`s of a module share one value namespace: a `const`, `static` or function with the name of another of them in the same module is also a compile error. A tuple or unit struct's constructor is in that namespace too ([2.3.7](../002-types/203-struct.md#237-no-duplicate-definitions-in-the-same-module)): `struct Point(i32, i32);` beside `fn Point`, `const Point` or `static Point` is a duplicate definition. Different arities, `async`, or type parameters do not make two functions distinct. To accept different argument counts, give one function default values for its trailing parameters ([4.2.3](402-parameters.md#423-default-values)).

## 4.1.11. Order and combination of fn modifiers

Function modifiers (`unsafe`, `async`, `yield`) **must be written in this order**:

```
unsafe → async → yield
```

Violating this order is a syntax error. Writing the same modifier twice is also impossible at the grammar level (the `?` in `unsafe?` allows only zero or one).

```rok
unsafe async yield fn ok() -> i32 { ... }   // OK — prescribed order
async unsafe fn bad1() { ... }           // ERROR — async before unsafe
yield async fn bad2() { ... }            // ERROR — yield before async
unsafe unsafe fn bad3() { ... }          // ERROR — duplicate unsafe
```

This rule guarantees that functions with the same meaning are always written the same way, which simplifies formatters, linters and readability.

### The 8 possible combinations

All three modifiers are optional, so there are 8 possible combinations, defined as follows.

| Combination             | Call result                            | Meaning              | Defined in                         |
|-------------------------|----------------------------------------|----------------------|------------------------------------|
| `fn`                    | `R`                                    | plain function       | [4.1](401-function-declaration.md) |
| `unsafe fn`             | `R`                                    | unsafe function      | [4.15](415-unsafe.md)              |
| `async fn`              | `AsyncFuture<R>`                   | async function       | [4.12](412-async.md)               |
| `yield fn`              | `GenIter<R>`                       | generator            | [4.11](411-generators.md)          |
| `async yield fn`        | `AsyncStream<R>`                   | async stream         | [4.13](413-stream.md)              |
| `unsafe async fn`       | `AsyncFuture<R>`                   | unsafe async function | (sum of each modifier)            |
| `unsafe yield fn`       | `GenIter<R>`                       | unsafe generator     | (sum of each modifier)             |
| `unsafe async yield fn` | `AsyncStream<R>`                   | unsafe async stream  | (sum of each modifier)             |

Combinations with `unsafe` are simple: the whole function body becomes an unsafe scope (as defined in 4.15), and the result type and coroutine behavior follow the definitions of the other modifiers (`async` / `yield`). That is, `unsafe` only adds permission to dereference raw pointers and call `unsafe fn` inside the body.

### Result types of coroutine functions

Calling a function (or block, or closure) marked `async` / `yield` yields a standard library type. The declaration's `-> R` gives the element / result type `T`.

| Form             | Result type       | Implemented trait      | With `throws (E)`         |
|------------------|-------------------|------------------------|---------------------------|
| `yield fn`       | `GenIter<T>`      | `Iterator<Item = T>`   | `TryGenIter<T, E>`        |
| `async fn`       | `AsyncFuture<T>`  | `Future<Output = T>`   | `TryAsyncFuture<T, E>`    |
| `async yield fn` | `AsyncStream<T>`  | `Stream<Item = T>`     | `TryAsyncStream<T, E>`    |

The `Try*` variants throw the error at the next progress point. For example, `next` of `TryGenIter<T, E>` is `next(ref self) throws (E) -> T | none`, and `TryAsyncFuture` throws at `try await`. These variants do not implement the plain `Iterator` / `Future` / `Stream` traits ([4.14.10](414-throws.md#41410-effect-of-throws-on-sum-type-signatures)).

```rok
async fn fetch_user(id: i64) -> User { ... }
let f: AsyncFuture<User> = fetch_user(42i64);

yield fn lines(path: str) throws (IoError) -> str { ... }
let it: TryGenIter<str, IoError> = lines("a.txt");
```

### Closure and block forms

The same ordering rule applies to the closure / block forms of `async` and `yield`. For example, a closure that produces a Stream is always written `async yield ||`; `yield async ||` is a syntax error. See each modifier's document for the closure / block forms.

## 4.1.12. Evaluation order of parameters, return and body

The evaluation order of a call is:

1. Evaluate the argument expressions (left to right).
2. Match parameters and begin evaluating the function body.
3. Evaluate the body; on completion, the value of the last expression or of `return` is the result.
4. Deliver the result to the call site.

The detailed evaluation order is defined in the parameter evaluation order section of [Parameters](402-parameters.md).
