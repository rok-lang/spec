# 4.4. Function types

Functions and closures are values, and those values have types. This document covers in detail the function type form of [type expressions](../002-types/201-type-expressions.md).

## 4.4.1. Form

A function type has this shape:

```rok
fn(ParamTypes) -> ReturnType
fn(i32, i32) -> i32
fn(str) -> bool
fn() -> ()
```

The `fn` keyword is followed by a parenthesized list of parameter types, then `->` and the return type. Parameter names are not written; only types. If the return type is omitted it is unit, so `fn(i32, i32)` is treated the same as `fn(i32, i32) -> ()`.

The type of a function that can throw writes [`throws`](414-throws.md) in the same position as in a function declaration.

```rok
fn(str) throws -> i32               // untyped - throws (any Error)
fn(str) throws (ParseError) -> i32  // typed
fn(Path) throws (IOError)           // return type omitted - unit
```

`unsafe fn(..) -> R` is the type of a function whose call requires `unsafe`. `unsafe fn` and C functions have this type as values. A safe `fn` value may be used where an `unsafe fn` is expected, but not the reverse ([4.15](415-unsafe.md#unsafe-fn-types)).

`throws` is part of the function type, so calling a value of a throwing function type must also be marked with `try`. `rethrows` is used only in function declarations, never in function types.

## 4.4.2. Types of function and closure values

Functions and closures are both expressed with the same form of function type.

```rok
fn add(x: i32, y: i32) -> i32 { x + y }
let f = add;                                    // f: fn(i32, i32) -> i32

let g = |x: i32| -> i32 { x * 2 };              // g: fn(i32) -> i32
let h: fn(i32) -> i32 = |x| x + 1;              // explicit type
```

Since functions and closures have the same function type, either fits wherever a function type is required.

## 4.4.3. Assigning to variables

A function or closure may be assigned to a variable with an explicit function type.

```rok
let f: fn(i32) -> i32 = |x| x * 2;
let g: fn(str, i32) -> str = format;
```

Both functions and closures may be assigned to a function-typed slot.

## 4.4.4. As function arguments and return values

Function types are mainly used as arguments and return values of other functions.

```rok
fn apply(f: fn(i32) -> i32, x: i32) -> i32 { f(x) }
fn make_adder(n: i32) -> fn(i32) -> i32 { |x| x + n }
```

The first pattern is a higher-order function taking a function as an argument; the second returns a function as its result. These two patterns are the core uses of function types.

## 4.4.5. Functions in collections

Like any other type, a function type may be the element type of a collection.

```rok
let handlers: List<fn(Event) -> ()> = [...];
let dispatch: Dict<str, fn(Request) -> Response> = {...};
```

Storing function types in collections lets behaviors be handled as data.

## 4.4.6. Closures with captures

Closures with captures are expressed with the same function type. To the user they present the same interface.

```rok
let n = 10;
let f: fn(i32) -> i32 = |x| x + n;     // captures n, but the type is fn(i32) -> i32
```

Captures are attached internally to the closure object and used automatically at call time. User code therefore need not care whether there are captures, and captures do not appear in the type.

Rust's distinction between the `Fn`, `FnMut` and `FnOnce` traits does not exist in rok. Captures are by value by default, which reduces semantic complexity. The detailed capture semantics are defined in [Closures](405-closure.md).

### `Send` / `Sync` bounds

Appending `+ Send` / `+ Sync` to a function type records in the type that the value may cross thread boundaries ([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers)).

```rok
fn(i32) -> i32 + Send
fn() + Send + Sync
```

- When a closure literal appears in a `+ Send` position, every captured value must be `Send + Sync`: a capture is a handle the creator keeps using, so both threads can reach it. A `fn(..) + Send` value is therefore also `Sync`.
- Spawn functions (`task::spawn`, `thread::spawn`, `par::*`) take `+ Send` closures and futures, so the spawned code cannot capture a non-`Sync` value.
- To use an `fn`-typed variable, the variable's type must already have the same bound. A `fn() -> T` value does not fit a `fn() -> T + Send` position.
- Named functions have no captures and are therefore always `Send + Sync`.

```rok
let c: Cell<i32> = Cell::new(0);
let f: fn() + Send = || print("hi");     // OK - no captures
let g: fn() + Send = || c.set(1);        // ERROR - Cell is not Send
let h: fn() = || print("hi");
let k: fn() + Send = h;                  // ERROR - h's type has no + Send
```

## 4.4.7. Gradual parameters and returns

Generic parameters in a function type are not directly expressible in this notation. A generic function's value is monomorphized at the use site into a function value with a concrete type.

```rok
fn id<T>(x: T) -> T { x }

let f = id;                       // ERROR - id itself is generic; T is not determined
let f: fn(i32) -> i32 = id;       // OK - monomorphized to i32
let g = id::<i32>;                // OK - monomorphized with turbofish
```

To put a dynamic slot in a function type, write `any` for a parameter or the return.

```rok
fn(any) -> any
```

This is the type of a function that accepts any type and can return any type. Note that this incurs dynamic dispatch cost.

## 4.4.8. Types of divergent functions

The type of a divergent function, covered in [Return](403-return.md), writes `!` in the return type position (`fn() -> !`). A `fn(A) -> !` value may be used in any `fn(A) -> R` position with the same parameters, and a throwing `fn(A) throws (E) -> !` fits `fn(A) throws (E) -> R`.

```rok
let f: fn() -> i32 = || panic("never");     // fn() -> ! value → fn() -> i32 position
```

### Representation of function values

A function value is a `{code, env}` pair: `code` is a code pointer and `env` is the capture environment. For a named function's value, `env` is null. A function-typed parameter of an `extern "C"` declaration has type `extern "C" fn(..)`, a C function pointer without `env`, and is not convertible to or from Rok function values ([5.9.4](../005-modules/509-extern-block.md#c-function-pointer-types)). Method paths (`S::new`, `i32::add`) are also function values ([3.5.3](../003-expressions/305-member-access-and-method-call.md#method-path-values)).

## 4.4.9. Subtyping between function types (none)

> [!IMPORTANT]
> Function type subtyping, as found in languages like Rust or TypeScript, does not exist in rok. A function type is not automatically converted to a more general or more specific function type. Parameter and return types must match exactly.

```rok
let f: fn(i32 | f64) -> i32 = |x: i32| x;     // ERROR - parameter type does not match exactly
```

Parameter and return types within a function type must match exactly; they are not widened to sum types.

The only exceptions are the divergent return of 4.4.8 and the error channel. If parameter and return types are the same, a function value that throws fewer errors may be used where a function type that can throw more is expected. This conversion (non-throwing → throwing, narrower error → wider error) may be implemented by a compiler-generated adapter.

```rok
fn parse_typed(s: str) throws (ParseError) -> i32 { ... }

let p: fn(str) throws (ParseError) -> i32 = |s| s.len() as i32;   // OK - non-throwing → throwing
let q: fn(str) throws -> i32 = parse_typed;   // OK - throws (ParseError) → throws (any Error)
let r: fn(str) -> i32 = parse_typed;          // ERROR - throwing → non-throwing
```

A `throws (A)` function value may be used in a `throws (A | B)` position, and, if `A` implements `Error`, in an untyped `throws` position.

This conversion applies to all function values, including function-typed variables and closures with captures. The compiler creates an adapter `{code, env}` wrapping the original value.

```rok
let g: fn(str) throws (ParseError) -> i32 = parse_typed;
let h: fn(str) throws -> i32 = g;             // OK - wrapped in an adapter
```

The motivation is explicitness. Rok's policy is that function conversions are written by the user in the explicit form of [conversion](../002-types/214-cast-and-conversion.md).

## 4.4.10. Evaluation of function types

A function type is a type that produces no value. It is only a specification of what shape of function is accepted and has no evaluation behavior of its own.

Calling a function follows the ordinary call rules (`f(...)`) described in [member access and method calls](../003-expressions/305-member-access-and-method-call.md).

## 4.4.11. Absence of parameter names

Function types contain no parameter names. A definition (`fn name(x: i32)`) has names, but the type notation (`fn(i32)`) keeps only positions.

```rok
fn(i32, i32) -> i32                // OK
fn(x: i32, y: i32) -> i32          // ERROR - function types have no names
```

Named arguments at a call ([Parameters](402-parameters.md)) refer to the names in the function definition. Named arguments therefore do not work when calling through a function type; calls through a value accept only positional arguments.

```rok
fn make_user(name: str, age: i32) -> User { ... }
let f: fn(str, i32) -> User = make_user;

make_user(name: "A", age: 30i32);    // named OK - uses the definition's names
f("A", 30i32);                        // positional OK
f(name: "A", age: 30i32);             // ERROR - the type has no names
```
