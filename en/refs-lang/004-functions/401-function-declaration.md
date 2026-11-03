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
- The return type is optional: a [type expression](../002-types/201-type-expressions.md), or an opaque `some Tr` ([4.1.13](#4113-opaque-return-types-some-trait)), after the `->` token ([punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md)).
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

The `->` part of the return type may be omitted. What that means depends on the function's [visibility](../005-modules/506-visibility.md).

On a module-private or `pub(sub)` function:

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
    print(s);                     // terminator at the end: returns unit
}
```

On a `pub` or `pub(pkg)` function, other modules call it, so its signature is written out and a body edit cannot change it. A missing `->` there means `()`, whatever the body does, also when the body never returns. If the body gives a value (its last expression or a `return v` has a type other than `()`), it is a compile error, and the error names the type to write.

```rok
pub fn log(s: str) {              // returns ()
    print(s);
}

pub fn double(x: i32) {           // ERROR: the body gives i32; write `-> i32`
    x * 2
}

pub fn double2(x: i32) -> i32 {   // OK
    x * 2
}
```

Methods of an inherent `impl` follow the same rule by their own visibility. A method of a trait `impl` writes its `->` as the trait's declaration does, since the return type must match it ([2.9.1](../002-types/209-impl.md)); only its untyped parameters take their types from the trait ([4.2.2](402-parameters.md#422-omitting-the-type)).

If the return type is written but the function body does not match it, it is a compile error.

```rok
fn bad() -> i32 {
    "hello"                       // ERROR: str is not i32
}
```

## 4.1.4. Omitting parameter types

On a module-private or `pub(sub)` function, a parameter without a type is treated as an implicit generic ([`any` slot](../002-types/202-any-slot.md), [generics](../002-types/210-generics.md)).

```rok
fn id(x) {                        // x: T (generic)
    x
}
// monomorphized by the type at the call site
let a = id(42i32);                // id<i32>
let b = id("hi");                 // id<str>
```

A `pub` or `pub(pkg)` function writes the type of every parameter; an untyped one is a compile error ([4.2.2](402-parameters.md#422-omitting-the-type)).

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
fn nothing() { }                  // empty body: returns unit
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
    42i32;                        // ERROR: terminator discards the value
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

Declaring two functions with the same name in the same module is a compile error. Rok has no overloading.

```rok
fn add(x: i32, y: i32) -> i32 { x + y }
fn add(x: f64, y: f64) -> f64 { x + y }   // ERROR: duplicate
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
unsafe async yield fn ok() -> i32 { ... }   // OK: prescribed order
async unsafe fn bad1() { ... }           // ERROR: async before unsafe
yield async fn bad2() { ... }            // ERROR: yield before async
unsafe unsafe fn bad3() { ... }          // ERROR: duplicate unsafe
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

Combinations with `unsafe` are simple: the whole function body becomes an unsafe scope (as defined in 4.15), and the result type and coroutine behavior follow the definitions of the other modifiers (`async` / `yield`). That is, `unsafe` only adds, inside the body, the operations an `unsafe` block allows ([4.15.2](415-unsafe.md#4152-unsafe------block-expression)): calling `unsafe fn`, dereferencing raw pointers, reading `union` fields and using a value after it was handed over.

### Result types of coroutine functions

Calling a function (or block, or closure) marked `async` / `yield` yields a standard library type. The declaration's `-> R` gives the element / result type `T`.

| Form             | Result type       | Implemented trait      | With `throws (E)`         |
|------------------|-------------------|------------------------|---------------------------|
| `yield fn`       | `GenIter<T>`      | `Iterator<Item = T>`   | `TryGenIter<T, E>`        |
| `async fn`       | `AsyncFuture<T>`  | `Future<Output = T>`   | `TryAsyncFuture<T, E>`    |
| `async yield fn` | `AsyncStream<T>`  | `Stream<Item = T>`     | `TryAsyncStream<T, E>`    |

These types live in `iter` (`GenIter`, `TryGenIter`), `future` (`AsyncFuture`, `TryAsyncFuture`) and `stream` (`AsyncStream`, `TryAsyncStream`) and are not in the prelude: name them through their module (`future::AsyncFuture<T>`) or a `use`. The `Try*` variants throw the error at the next progress point. For example, `next` of `TryGenIter<T, E>` is `next(ref self) throws (E) -> T | none`, and `TryAsyncFuture` throws at `try await`. These variants do not implement the plain `Iterator` / `Future` / `Stream` traits ([4.14.10](414-throws.md#41410-effect-of-throws-on-sum-type-signatures)); `TryGenIter` implements `TryIterator` and `TryAsyncStream` implements `TryStream` ([stdlib 2.7.13](../../refs-standard-libraries/200-core/207-iter.md#2713-tryiterator---fallible-iteration), [2.8.11](../../refs-standard-libraries/200-core/208-stream.md#2811-trystream---fallible-streams)).

```rok
async fn fetch_user(id: i64) -> User { ... }
let f: future::AsyncFuture<User> = fetch_user(42i64);

yield fn lines(path: str) throws (IoError) -> str { ... }
let it: iter::TryGenIter<str, IoError> = lines("a.txt");
```

### Closure and block forms

The same ordering rule applies to the closure / block forms of `async` and `yield`. For example, a closure that produces a Stream is always written `async yield ||`; `yield async ||` is not a closure form: it reads as `yield` applied to an `async ||` closure, which is an error (E0059, `yield` outside a generator) whose help gives `async yield ||`. See each modifier's document for the closure / block forms.

## 4.1.12. Evaluation order of parameters, return and body

The evaluation order of a call is:

1. Evaluate the argument expressions (left to right).
2. Match parameters and begin evaluating the function body.
3. Evaluate the body; on completion, the value of the last expression or of `return` is the result.
4. Deliver the result to the call site.

The detailed evaluation order is defined in the parameter evaluation order section of [Parameters](402-parameters.md).

## 4.1.13. Opaque return types: `some Trait`

```
opaque_ret ::= "some" (bound | "(" bound ("+" bound)* ")")
bound      ::= type_path | fn_type
```

A function's return type (after `->`, also after `throws (E) ->`) may be `some Tr`: one concrete type that has `Tr`, chosen by the body and hidden from callers. `Tr` is a trait, with associated type bindings as written in bounds (`some Iterator<Item = str>`), or a function type (`some fn(i32) -> i32`). Several bounds go in parentheses: `some (Iterator<Item = i32> + Clone)`.

```rok
pub fn evens(n: i32) -> some Iterator<Item = i32> {
    (0..n).filter(|x| x % 2 == 0)
}

fn adder(k: i32) -> some fn(i32) -> i32 {
    |x| x + k                     // x: i32, from the bound
}

let total = evens(10).map(|x| x * x).sum();
```

- The concrete type is the type of the body's value, inferred as in [4.1.3](#413-omitting-the-return-type), also on a `pub` function. Calls are static: no tag, no boxing, as with [generics](../002-types/210-generics.md).
- One concrete type per function: return paths of different types are an error (`the return paths of `f` have different types ..`), help `` to return several types, write `-> any Tr` ``.
- The concrete type must have every bound: otherwise `` `f` returns `ListIter<i32>`, which is not `Iterator<Item = str>` `` at the bound.
- Callers see only the bounds: they may call the bounds' methods (and their supertraits'), iterate it when the bound is an iterator, pass it where a generic parameter with those bounds is expected, and call it when the bound is a function type. Naming or using the concrete type is an error: `` no method `contains` on `some Iterator<Item = i32>`: callers see only its bounds ``; `` `let b` expects `Range<i32>`, found `some Iterator<Item = i32>` ``.
- The same function always returns the same type, so two calls' results have one type. Results of different functions are different types.
- `some` is a function's return type, a parameter's type ([4.1.14](#4114-some-parameters)) or the right side of a named opaque type ([4.1.15](#4115-named-opaque-types-type-x--some-trait)). Elsewhere (a `let` annotation, a field, a type argument, a closure parameter) it is an error that says what to write: no annotation for a variable, a generic `<T: Tr>` for a field or a type argument, `any Tr` for a value of several types.
- Across packages the function's body travels with the package's interface, so the dependent build knows the concrete type; its users still see only the bounds.
- There is no `Fn` trait: a function bound is the function type `fn(A) -> R`.

## 4.1.14. `some` parameters

A parameter's type may be `some Tr` (the bounds as in [4.1.13](#4113-opaque-return-types-some-trait)): an anonymous type parameter. `fn show(x: some Display)` is `fn show<T: Display>(x: T)` with `T` unnamed.

```rok
fn show(x: some Display) -> str { f"<{x}>" }
fn both(a: some Display, b: some Display) -> str { f"{a}{b}" }
fn count(it: some Iterator<Item = i32>) -> usize { it.count() }
fn twice(f: some fn(i32) -> i32, x: i32) -> i32 { f(f(x)) }

let s = both(1, 2.5);            // a: i64, b: f64
let n = count([1, 2, 3].iter());
let t = twice(|x| x * 3, 2);     // x: i32, from the bound
```

- Each call has one static type for the parameter, the argument's, as with [generics](../002-types/210-generics.md): no tag, no boxing. The argument must have every bound.
- Each `some` parameter is its own type parameter: `both(1, 2.5)` gives `a` and `b` different types.
- The body sees only the bounds, as for a named parameter `T: Tr`.
- `some fn(A) -> R` takes the function value (a function or a closure), and the closure's parameters take their types from it.
- Free functions, methods and trait methods may have `some` parameters, alongside named type parameters.
- The type has no name: `::<..>` at a call gives only the named type parameters. Writing more is an error with the help that a `some` parameter's type is the argument's: pass the argument, or declare `<T: Tr>` to choose `T` with `::<..>`.

`any Tr` and `some Tr` are a pair. `any Tr` is a dynamic slot: it holds a value of any type with `Tr`, a different one each time, and calls dispatch at run time ([2.2](../002-types/202-any-slot.md)). `some Tr` is one static type, hidden: the callee's body (for a return) or the call's argument (for a parameter) fixes it, and calls dispatch statically.

## 4.1.15. Named opaque types: `type X = some Trait`

```
opaque_alias ::= visibility? "type" identifier "=" opaque_ret ";"
```

`type X = some Tr;` names one concrete type that has `Tr` (the bounds as in [4.1.13](#4113-opaque-return-types-some-trait)). Unlike `-> some Tr`, the type has a name, so it can be written wherever a type goes: a field, a `let` annotation, a type argument (`List<X>`), a parameter and a return type.

```rok
mod nums {
    pub type Evens = some Iterator<Item = i64>;

    pub fn evens(n: i64) -> Evens {        // defines Evens
        (0..n).filter(|x| x % 2 == 0)
    }

    pub fn first(e: Evens) -> i64 | none { e.next() }
}

struct Shelf { evens: nums::Evens; more: List<nums::Evens>; }

fn total(e: nums::Evens) -> i64 { e.sum() }

let s = Shelf { evens: nums::evens(10), more: [nums::evens(4)] };
println(total(s.evens));                  // 20
```

- The defining functions are the free functions of X's module whose return type is `X` (written `X` or `self::X`). Each one's concrete type is the type of its body's value, inferred as for `-> some Tr`, and must have every bound. There must be at least one, and they must all give the same type: none is `` `X` has no defining function `` (E0146), help ``a function in this module that returns `X` gives its type``; two that differ is `` `g` returns `B`, but `X` is `A` (from `f`) `` (E0146), help to return the same type or to call `f`.
- A defining function has no type parameters, and `X` has none: a generic one is an error (E0146) whose help writes `fn f<T>(..) -> some Tr`, a hidden type per call ([4.1.13](#4113-opaque-return-types-some-trait)).
- Methods, and functions that return `List<X>` or `X | none`, do not define `X`; they use it.
- Inside X's module (not its submodules), `X` is its concrete type: the module's functions call its methods, read its fields and build it like any value of that type.
- Outside X's module, `X` is a type of its own that has only its bounds: a value is stored, passed, returned and kept in `List<X>` as `X`, and used through the bounds' methods (and their supertraits'), as a `-> some Tr` result is. The concrete type is hidden: a method or field the bounds do not give is `` no method `len` on `Evens`: outside `nums`, only its bounds are visible ``; a value of `X` where another type is expected, or a value of another type where `X` is expected (a function outside the module that returns `X` from its own body), is `` `let n` expects `i64`, found `X`: outside `m`, `X` is opaque `` (E0147), help to keep the value as `X`, get one from X's defining functions, or move the code into X's module.
- With a function type as bound (`type Step = some fn(i64) -> i64;`), a value is only callable, so the type stays its concrete type everywhere.
- Calls and storage are static: a field of type `X` holds the concrete type inline, with no tag and no boxing.
- `pub type X = some Tr;` exports the name. Across packages the defining functions' bodies travel with the package's interface ([4.1.13](#4113-opaque-return-types-some-trait)), so the dependent build lays out `X`; dependents see only the bounds.
- `type X = any Tr;` is a plain type alias ([2.6.9](../002-types/206-sum-types.md#269-combining-with-type-aliases)) of the dynamic slot `any Tr`: each value of `X` may have a different type and calls dispatch at run time. `type X = some Tr;` is one static type, fixed by its module.
