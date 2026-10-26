# 4.14. `throws` signatures

This section defines the `throws` / `rethrows` forms of function signatures, which combine with `throw`, `try`, `catch` and `finally` from [error control flow](../003-expressions/314-error-control-flow.md).

> [!IMPORTANT]
> Rok has a single error model: `throws`. A function that can throw an error writes `throws` in its signature, and the caller marks the call with `try`. The standard library's [`Result<T, E>`](../../refs-standard-libraries/200-core/214-error.md) is used only when an error must be held as a value. The rule is one line: **throw it → `throws`; hold it → `Result`.**

## 4.14.1. Form

In `fn_decl` of [function declarations](401-function-declaration.md), `throws_clause?` sits after the parameter parentheses and before the return type arrow.

```
throws_clause   ::= "throws" ("(" type_expr ")")?
                  | "rethrows"
```

The full form of `fn_decl` is therefore:

```
fn_decl ::= visibility? fn_modifiers "fn" identifier "(" param_list? ")" throws_clause? return_type? block
```

Here `fn_modifiers` is `unsafe? async? yield?` as defined in [4.1 Function Declarations](401-function-declaration.md).

```rok
fn name(args) -> R                              // non-throwing
fn name(args) throws -> R                       // untyped - shorthand for throws (any Error)
fn name(args) throws (E) -> R                   // typed - single error type
fn name(args) throws (A | B | C) -> R           // typed - sum type of errors
fn name(f: fn(T) throws -> U) rethrows -> R     // throws only when the function argument throws
```

`throws` and `rethrows` are contextual keywords, keywords only after the parameter parenthesis `)` ([1.5.4](../001-lexical/105-keywords.md#154-contextual-keywords)), and are placed before the return type arrow. The optional parentheses after `throws` give the type of errors that can be thrown.

## 4.14.2. Non-Throwing functions

```rok
fn add(x: i32, y: i32) -> i32 { x + y }
```

In a function without `throws`, no error can leave the function. If an error from a `throw` in the body or from a call marked with `try` is not fully handled by a `catch` inside the function, it is a compile error. Callers call this function without `try`.

```rok
fn bad() -> i32 {
    throw ValueError { msg: "nope" };   // ERROR - error leaves a non-throwing function
}
```

This rule forces the signature to state whether a function can throw, which blocks silent error propagation.

## 4.14.3. Untyped `throws`

```rok
fn parse(s: str) throws -> i32 { ... }
```

Untyped `throws` is shorthand for `throws (any Error)`. The function may throw an error of any type implementing the standard library's [`Error`](../../refs-standard-libraries/200-core/214-error.md) trait, and the thrown value is delivered as an `any Error` value, a [constrained `any`](../002-types/202-any-slot.md). The caller narrows to concrete types with type patterns in `catch`, or catches everything with `catch e: any Error` or a catch-all.

The untyped form is recommended when the set of error kinds is not closed to the caller, as in public APIs.

## 4.14.4. Typed `throws`: single types and sum types

```rok
fn fetch_user(id: i64) throws (NotFound) -> User { ... }
fn parse_or_load(s: str) throws (ParseError | IOError | TimeoutError) -> Data { ... }
```

The parentheses may state the exact type that can be thrown, and sum types (`A | B | C`) may be used as-is.

The typed form suits internal or library APIs, embedded environments, and cases where the set of error kinds is closed. The compiler can then check whether the caller's `catch` arms cover all error types; if they do, the expression is treated as no longer throwing.

```rok
fn caller() -> Data {             // non-throwing - all three types handled
    try parse_or_load(input)
    catch e: ParseError { ... }
    catch e: IOError { ... }
    catch e: TimeoutError { ... }
}
```

## 4.14.5. Agreement between `throws` and the body

Every error that can leave the function must be compatible with the signature's `throws`. Errors originate from `throw` statements in the body and from throwing calls marked with `try`; errors handled by a `catch` inside the function are excluded from the check.

```rok
fn f() throws (A) -> i32 {
    throw A_instance;             // OK
    throw B_instance;             // ERROR - only A allowed
}
```

The compatibility rules: for typed `throws (E)`, every originating error type must be a member of `E`. For untyped `throws`, every originating error type must implement `Error`, and the value is delivered as `any Error`.

There is no automatic conversion between error types. To propagate a callee's error as a different type, wrap it in a `catch` and throw again.

```rok
fn load_config(path: str) throws (ConfigError) -> Config {
    try parse_toml(read_text(path))
    catch e: IOError { throw ConfigError::io(e); }
    catch e: ParseError { throw ConfigError::syntax(e); }
}
```

## 4.14.6. Caller obligations

An expression calling a `throws` function must be inside an expression covered by `try`. Calling a throwing function without `try` is a compile error. `try` covers the entire expression to its right, so `try a(b())` covers both the `a` and `b` calls, and `try { a(); b(); }` covers all calls in the block ([3.14.2](../003-expressions/314-error-control-flow.md#3142-try-marking-call-sites)).

An error from a call marked with `try` is handled by the nearest `catch`; an unhandled error propagates to the enclosing function, whose `throws` must accept that error type (4.14.5).

```rok
fn caller() {                     // non-throwing
    parse(s);                     // ERROR - throwing call not marked with try
    try parse(s);                 // ERROR - error propagates but caller is not throws
}

fn caller2() throws {             // propagates
    try parse(s);                 // OK
}

fn caller3() {                    // catches
    try parse(s) catch _ {
        ...
    };
}

fn caller4() -> i32 {             // discards the error or turns it into a panic
    let a = (try? parse(s)) ?? 0; // try? - none on error
    let b = try! parse(t);        // try! - panic on error
    a + b
}
```

Thus every point where an error can flow is visible in the code as `try`, and how far an error propagates is visible in the function signature's `throws`.

## 4.14.7. `rethrows`

```rok
fn map<U>(self, f: fn(T) throws -> U) rethrows -> List<U> {
    let out: List<U> = [];
    for x in self {
        out.push(try f(x));
    }
    out
}
```

`rethrows` is the form for higher-order functions that take function-typed parameters. A `rethrows` function throws only when a function or closure passed as an argument throws.

- If every argument passed to a throwing function-typed parameter at the call site is non-throwing, the call is non-throwing and is made without `try`.
- If any of those arguments is throwing, the call is throwing, and the thrown error type is the error type of those arguments. The caller marks it with `try`.
- In the body of a `rethrows` function, the only errors that can leave the function are those from calls of the function-typed parameters. If the body throws its own error directly, it is a compile error.

```rok
let lens = names.map(|s| s.len());              // non-throwing closure - no try
let nums = try names.map(|s| try parse(s));     // throwing closure - try required
```

When a `rethrows` function is used as a value, the expected type (a function type) determines which instance is used. Without an expected type, the error parameter is set to `any Error`.

## 4.14.8. `throws` in function types

`throws` also attaches to [function types](404-function-type.md).

```rok
fn(T) -> U                    // non-throwing
fn(T) throws -> U             // untyped - throws (any Error)
fn(T) throws (E) -> U         // typed
```

A non-throwing function value may be used in a throwing function type position with the same parameter and return types, and a `throws (A)` function value may be used in a `throws (A | B)` position or (if `A` implements `Error`) an untyped `throws` position. The reverse direction is not allowed. Details are in [4.4](404-function-type.md).

## 4.14.9. Combining `throws` with `async` and `yield`

All three modifiers may be combined on the same function.

```rok
async fn fetch(url: str) throws (NetworkError) -> Data { ... }

yield fn lines(s: str) throws (ParseError) -> str { ... }

async yield fn read_paginated(url: str) throws (NetworkError) -> Page { ... }
```

`throws` is a keyword in signature position and combines orthogonally with `async` or `yield`.

## 4.14.10. Effect of `throws` on sum type signatures

```rok
async fn fetch_data() throws (NetworkError) -> Data { ... }
```

One might ask whether calling this function yields a Future holding `Data | NetworkError`.

The answer is **`TryAsyncFuture<Data, NetworkError>`** ([4.1.11](401-function-declaration.md#result-types-of-coroutine-functions)). The error is not mixed into the result type; it is delivered through the error channel at the `await` point. The side that `await`s therefore marks it with `try`. `TryAsyncFuture` does not implement the plain `Future` trait.

```rok
let f = fetch_data();              // f: TryAsyncFuture<Data, NetworkError>
let d = try await f catch e: NetworkError {
    ...
};
```

This decision keeps the result from turning into an `enum` with two variants, result and error.

`yield` and `async yield` follow the same pattern. A generator with `throws (E)` is `TryGenIter<T, E>` and a stream is `TryAsyncStream<T, E>`; the error is delivered through the error channel at the `next` point.

### `try for`

A `for` over an iterator whose `next` throws (`TryGenIter`, etc.) is a throwing expression and must be covered with `try`. The same applies to `for await`.

```rok
try for line in lines("a.txt") {         // IoError from each next propagates
    process(line);
}
try for await ev in events() { handle(ev); }
for line in lines("a.txt") { ... }       // ERROR - no try
```

## 4.14.11. `throws` on methods

Like ordinary [methods](409-method.md), methods may have `throws` signatures.

```rok
impl Database {
    fn query(self, sql: str) throws (SqlError) -> ResultSet { ... }
}
```

The caller's obligations and propagation rules apply exactly as for free functions.

The `throws` of an impl method implementing a trait must be the same as the `throws` of the trait method. If the impl adds `throws` or widens the error type, it is a compile error.

Operator trait methods (`Add::add`, etc.) are declared without `throws`, so operators do not throw. Operations that can fail are provided as named methods or return a `Result` ([stdlib 2.3](../../refs-standard-libraries/200-core/203-ops.md)).

## 4.14.12. Value-Based implementation model

A throwing function returns its result as a value. The return value is a pair of a success flag and a payload holding either the success value or the error value. The error channel is distinguished by the signature, so the success type and the error type may be the same type. A `try` at the call site checks the flag and branches; on error, it goes to the nearest `catch` or returns the error as the enclosing function's error result.

In this model, the cost of the no-error path is a single branch, and entering `try` or `catch` costs nothing. No stack unwinding and no runtime error-delivery function are used. The exact return value layout is defined in [8.6.5](../008-abi/806-calling-convention.md#865-return-of-throwing-functions-value-based-error-propagation) and the lowering in [8.9.6](../008-abi/809-lowering-control.md#896-try--catch--finally-value-based-branching).

## 4.14.13. Panics and fatal errors

A panic is not a `throws` error. Panics - the standard library's `panic_fn`, a failed `try!`, division by zero, integer overflow, a failed bounds check - and the fatal errors of [error control flow](../003-expressions/314-error-control-flow.md) are not part of any `throws` signature and cannot be caught with `catch`. The runtime writes the following to standard error (stderr) and terminates the process with exit code `101`.

```
thread '<name>' panicked at <path>:<line>:<col>:
<message>
```

- `<name>` is the thread name: `main` for the main thread, `<unnamed>` for unnamed threads.
- The location is the failing expression. Inside a chain of [`#[track_caller]`](../006-meta-system/602-built-in-attributes.md#6212-track_caller-passing-the-call-site) functions, it is the first call location outside that chain.
- Backtraces are not defined.
