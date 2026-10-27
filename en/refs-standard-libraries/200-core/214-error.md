# 2.14. error - Error traits / types

The `rok::core::error` / `rok::core::panic` modules provide the authority for error handling (`ZeroError` lives beside `NonZero` in `rok::core::num::niche`). Rok has a single error model. A function that can throw an error writes `throws` in its signature, and the caller marks the call with `try` ([refs-lang 4.14](../../refs-lang/004-functions/414-throws.md)). `Result<T, E>` is an ordinary enum for **holding an error as a value**; it is not a means of propagating errors.

This chapter defines three of the 48 lang slots: `panic_fn` / `oom_fn` / `unreachable_fn`.

## 2.14.1. Model overview - `throws` to throw, `Result` to hold

```rok
// (a) throwing: throws and try
fn parse_int(s: str) throws (ParseError) -> i32 {
    if !s.is_digit() { throw ParseError { input: s, pos: 0, reason: "not a digit" }; }
    s.to_i32()
}

let n = try parse_int("42") catch e: ParseError {
    log(f"parse failed: {e:?}");
    0
};

// (b) holding: Result
let results: List<Result<i32, ParseError>> =
    inputs.map(|s| Result::catching(|| try parse_int(s)));

for r in results {
    match r {
        Ok(v) => total = total + v,
        Err(e) => failures.push(e),
    }
}
```

The choice follows a one-line rule: **use `throws` if the error must flow to the caller, and `Result` if the error must be kept to deal with later.**

| Situation                                        | Tool                    | Reason                                    |
|--------------------------------------------------|-------------------------|-------------------------------------------|
| Operation that can fail (I/O, parse, lookup)     | `throws`                | `try` at every call site makes the flow visible |
| API boundary / public                            | `throws (E)` / `throws` | error type exposed in the signature       |
| Collecting parallel results, passing results to callbacks | `Result<T, E>`  | keeps the error as a value                |
| Gathering several errors to report at once, caching results | `Result<T, E>` | stored as a value and handled later     |
| Ignore failure, only the value is needed         | `try?`                  | received as `T \| none`                   |
| Failure is a program bug                         | `try!`                  | panics on failure                         |

## 2.14.2. `Result<T, E>` - enum for holding

`Result<T, E>` is an ordinary enum in `rok::core::error`.

```rok
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

> [!NOTE]
> Rok has **no `Some` / `None` enum variants**. `Option<T>` itself **does not exist**;
> the sum type `T | none` is used in its place. `Result` is defined as a separate enum because **both branches carry meaningful data**.

### Core methods

The core methods of `Result` are:

```rok
impl<T, E> Result<T, E> {
    fn is_ok(self) -> bool;
    fn is_err(self) -> bool;
    fn ok(self) -> T | none;
    fn err(self) -> E | none;

    fn unwrap(self) -> T;                               // panics on Err
    fn unwrap_or(self, default: T) -> T;
    fn unwrap_or_else(self, f: fn(E) -> T) -> T;

    fn map<U>(self, f: fn(T) -> U) -> Result<U, E>;
    fn map_err<F>(self, g: fn(E) -> F) -> Result<T, F>;
    fn and_then<U>(self, f: fn(T) -> Result<U, E>) -> Result<U, E>;
    fn or_else<F>(self, g: fn(E) -> Result<T, F>) -> Result<T, F>;

    fn get(self) throws (E) -> T;                       // throws on Err, back into the error flow
    fn catching(f: fn() throws (E) -> T) -> Result<T, E>;   // captures the error flow as a value
}
```

`get` and `catching` bridge the two worlds. `catching` calls a throwing function and holds the outcome as `Ok` on success or `Err` on error; `get` extracts the held value and rethrows the error if it is `Err`.

### `Result` has no `?`

The postfix `?` of [refs-lang 3.14.10](../../refs-lang/003-expressions/314-error-control-flow.md#31410-the--operator---early-return-of-none) is only for `T | none`. Using `?` on a `Result` value makes the compiler emit the following diagnostic.

```rok
fn load(r: Result<Config, IoError>) -> Result<Config, IoError> {
    let cfg = r?;
    // ERROR: `?` is only for `T | none`. Use `throws` and `try` to propagate errors.
    ...
}
```

To propagate a held error to the caller, declare the function `throws` and write `try r.get()`.

```rok
fn load(r: Result<Config, IoError>) throws (IoError) -> Config {
    try r.get()
}
```

## 2.14.3. `Error` trait - the surface of an error

The surface of every error type is defined by the `Error` trait.

```rok
trait Error: Display + Debug {
    fn source(self) -> any Error | none {
        none                                  // default: no chain
    }
}
```

An error type implements this trait and uses `Display` for message output and `Debug` for debug display. An untyped `throws` is shorthand for `throws (any Error)`, so values thrown by an untyped throwing function must implement `Error`. To hold or pass errors regardless of type, use `any Error`, a [constrained `any`](../../refs-lang/002-types/202-any-slot.md) (`any Error | none` is `(any Error) | none`).

### Chain - `source()`

When wrapping an error, the cause chain can be expressed as follows.

```rok
struct ConfigError {
    msg: str;
    cause: any Error | none;
}

impl Error for ConfigError {
    fn source(self) -> any Error | none {
        self.cause
    }
}

impl Display for ConfigError {
    fn fmt(self, ref f: Formatter) throws (FmtError) {
        try f.write_str(self.msg)
    }
}
```

When a value is stored in `cause`, the compiler checks at compile time that its static type implements `Error`. With a chain built this way, the REPL or a logger can follow it and print a traceback of causes.

```rok
let cfg = try parse_config(text) catch e: ParseError {
    throw ConfigError { msg: "bad config", cause: e };
};
```

## 2.14.4. Built-in error types

The error types the standard library provides by default are defined as follows. Their fields are `pub`.

```rok
struct TypeError {
    expected: str;
    actual: str;
}

struct ValueError {
    msg: str;
}

struct IndexError {
    index: i64;
    len: usize;
}

struct OverflowError {
    op: str;          // "add" / "mul" / ...
    ty: str;          // "i32" / "i64" / ...
}

struct ParseError {
    input: str;
    pos: usize;
    reason: str;
}

struct ConversionError {
    msg: str;
}

struct ZeroError;      // rok::core::num::niche
```

Each type implements `Error + Display + Debug`, and its `Display` body provides a human-friendly message. Where each error is thrown:

| Type              | Thrown by                                                                  |
|-------------------|----------------------------------------------------------------------------|
| `TypeError`       | runtime cast failure, type check failure                                   |
| `ValueError`      | general invalid value (parsers / APIs, etc.)                               |
| `IndexError`      | out-of-range argument to a throwing API that takes an index (the bounds check of `xs[10]` panics) |
| `OverflowError`   | range exceeded in a checked conversion such as `try_into` (arithmetic overflow panics) |
| `ParseError`      | parse failure such as `str → number` (ROKON uses `RokonError`)             |
| `ConversionError` | default for `TryFrom::Error`; failure of a non-numeric conversion ([2.11.4](211-conversion.md#2114-tryfromu-trait---fallible-conversion)); `try_from` of `NonNan` / `Finite` |
| `ZeroError`       | 0 passed to `NonZero::try_from` ([2.2.2](202-non-zero.md#222-nonzerot---zero-forbidden)) |

### Fatal errors

The following types are classified as Fatal.

```rok
struct StackOverflowError;     // Fatal: cannot be caught
struct OutOfMemoryError;       // Fatal: cannot be caught
struct InternalError;          // Fatal: compiler-internal bug
```

A Fatal error is not a `throws` error and cannot be caught with `catch`. Like a panic, it prints a message to stderr and ends the process with exit code `101` ([refs-lang 3.14.8](../../refs-lang/003-expressions/314-error-control-flow.md#3148-panics-and-fatal-errors)).

## 2.14.5. `panic_fn` / `oom_fn` / `unreachable_fn` - compiler hooks

At the bottom of the error model are hook functions that the compiler calls directly.

```rok
mod panic {                         // rok::core::panic
    #[lang("panic_fn")]
    #[track_caller]
    fn panic(msg: str) -> ! {
        rok_panic_at(msg)            // prints to stderr with the caller's location, then exit code 101
    }

    #[lang("oom_fn")]
    fn oom() -> ! {
        rok_oom()                    // "fatal error: out of memory", exit code 101
    }

    #[lang("unreachable_fn")]
    #[track_caller]
    fn unreachable() -> ! {
        rok_panic_at("internal error: entered unreachable code")
    }
}
```

Each function has return type `-> !` (never) and never returns normally. `panic` and `unreachable` are `#[track_caller]`: the reported location is the first caller outside a `#[track_caller]` chain.

### Where panics occur

The compiler calls `panic_fn` in the following places. In every case the panic cannot be caught with `catch`, and `finally` and `Drop` do not run.

- The error path of `try!`: the message is the error value's `Display` (or `Debug` if there is none)
- Division by zero, integer overflow, bounds check failure
- Explicit panic calls in the standard library, such as `Result::unwrap` on `Err`

`throw` is not a panic. `throw` only produces the error result of a throwing function ([refs-lang 8.9.7](../../refs-lang/008-abi/809-lowering-control.md#897-throw)) and does not go through `panic_fn`.

### OOM handling

When a GC allocation fails, OOM is handled as follows. The runtime cannot call a Rok function by name, so it calls the runtime symbol `rok_oom` directly instead of `oom()` ([refs-lang 8.5.7](../../refs-lang/008-abi/805-extern-symbols.md#857-diagnostic-area)).

```
rok_gc_alloc_slow:
    if !try_collect_and_retry() {
        rok_oom();                       // "fatal error: out of memory", exit 101
    }
```

core's `oom()` calls the same `rok_oom`, so both paths produce the same output and exit code.

OOM is Fatal and is not caught.

### Using `unreachable`

`unreachable` marks a branch that cannot be reached.

```rok
match (x: i32 | f64) {
    v: i32 => process_int(v),
    v: f64 => process_float(v),
    // the compiler confirms every variant is handled; no unreachable position
}

// or explicitly.
fn require_positive(x: i32) -> u32 {
    if x < 0 { rok::core::panic::unreachable() }     // guaranteed by the caller
    x as u32
}
```

The compiler inserts `unreachable` automatically in these places:
- dead arms after the `match` exhaustiveness check
- positions where the else of `if let ... else` is guaranteed to be divergent

### `assert` / `assert_eq` / `assert_ne`

```rok
fn assert(cond: bool, msg: str = "") -> ();
fn assert_eq<T: PartialEq + Debug>(left: T, right: T, msg: str = "") -> ();
fn assert_ne<T: PartialEq + Debug>(left: T, right: T, msg: str = "") -> ();
```

They are usable anywhere through the core prelude. If the condition is false they panic (exit code 101). The message is as follows; if `msg` is not empty, `: msg` is appended.

```
assertion failed[: msg]
assertion `left == right` failed[: msg]
  left: <Debug of left>
 right: <Debug of right>
```

`assert_ne` uses `left != right`; everything else is the same.

## 2.14.6. `try` / `catch` / `throw` / `throws` / `finally`

The syntax is governed by [refs-lang 3.14 Error control flow](../../refs-lang/003-expressions/314-error-control-flow.md) and [refs-lang 4.14 `throws` signatures](../../refs-lang/004-functions/414-throws.md); this section covers only standard library conventions.

### Signature model

Throwing behavior in a function signature takes one of five forms.

```rok
fn foo() -> T                      // non-throwing: never throws
fn foo() throws -> T               // untyped: throws (any Error)
fn foo() throws (E) -> T           // typed single: only E
fn foo() throws (A | B | C) -> T   // typed sum: only A | B | C
fn foo(f: fn() throws -> T) rethrows -> T   // throws only when f throws
```

Untyped `throws` is the recommended default for public APIs because it keeps the API specification light. `throws (E)` suits library-internal APIs, embedded environments, or cases where the catch side needs to branch on the error. The typed sum `throws (A | B | C)` extends Swift SE-0413 typed throws to sum types. Standard library higher-order functions that take function parameters use `rethrows`, so that a call passing a non-throwing closure can be written without `try`.

### `try` / `catch` form

`catch` is a postfix expression; when it catches an error, the value of the handler block becomes the value of the whole expression.

```rok
let result = try compute() catch e: TypeError {
    log(f"type error: {e}");
    default_value
};
```

### `finally`

A `finally` block can be used for resource cleanup and the like.

```rok
let f = try File::open(path);
try {
    process(f);
} catch e: IoError {
    log(f"io error: {e}");
} finally {
    f.close();      // runs on both normal completion and error
}
```

The execution semantics of `finally`:
- It runs on normal completion.
- It runs on completion after a catch.
- When an uncaught error propagates, it runs and then the error propagates.
- It runs when leaving via `return` / `break` / `continue`.
- On a panic or Fatal error it does not run, because the process ends immediately.

An error thrown inside `finally` replaces the current error.

### `throw` and branch types

`throw value;` is a statement, and a block ending in `throw` is divergent. The branch result type therefore reflects only the other branches' types. For example, the result of `if c { x } else { throw e; }` has the type of `x`.

## 2.14.7. Converting between `Result` and `throws`

The two tools can be converted into each other explicitly.

### Throwing call → `Result<T, E>`

`Result::catching` calls a throwing closure and holds the outcome. It can also be built directly with a `catch` expression.

```rok
let r: Result<i32, ParseError> = Result::catching(|| try parse_int(s));
let r: Result<i32, ParseError> = try Ok(parse_int(s)) catch e { Err(e) };
```

### `Result<T, E>` → throw

`get` rethrows an `Err`. The enclosing function's `throws` must accept that error type.

```rok
fn first_ok(rs: List<Result<i32, ParseError>>) throws (ParseError) -> i32 {
    try rs[0].get()
}
```

### Extracting as `T | none`

To discard the error and keep only the value, use `try?` on a throwing call and `ok()` / `err()` on a held `Result`.

```rok
let a: i32 | none = try? parse_int("42");
let r: Result<i32, ParseError> = Result::catching(|| try parse_int("42"));
if let Ok(v) = r { use(v); }
if let n: i32 = r.ok() { use(n); }     // extracted as T | none
```

## 2.14.8. Module mapping

The traits each error type implements:

| Type                 | Display | Debug         | Error | Notes                          |
|----------------------|---------|---------------|-------|--------------------------------|
| `TypeError`          | ✓       | ✓             | ✓     |                                |
| `ValueError`         | ✓       | ✓             | ✓     |                                |
| `IndexError`         | ✓       | ✓             | ✓     |                                |
| `OverflowError`      | ✓       | ✓             | ✓     | range exceeded in a checked conversion |
| `ParseError`         | ✓       | ✓             | ✓     |                                |
| `ConversionError`    | ✓       | ✓             | ✓     | default `TryFrom::Error`       |
| `ZeroError`          | ✓       | ✓             | ✓     | in `rok::core::num::niche`     |
| `IoError`            | ✓       | ✓             | ✓     | std tier (target-dependent)    |
| `StackOverflowError` | ✓       | ✓             | ✓     | Fatal: cannot be caught        |
| `OutOfMemoryError`   | ✓       | ✓             | ✓     | Fatal: cannot be caught        |
| `InternalError`      | ✓       | ✓             | ✓     | Fatal: on a compiler bug       |
| `Result<T, E>`       | ✗       | ✓ (T,E:Debug) | ✗     | enum for holding               |
| `any Error`          | ✓       | ✓             | ✓     | constrained any, can chain     |

`Display` provides a user-friendly message; for example, `IndexError { index: 10, len: 3 }` reads `"index 10 out of bounds (len 3)"`. `Debug` shows the structure as is. An `any Error` value calls the `Display` / `Debug` / `source` of its actual type through the runtime tag.

## 2.14.9. Comparison with other languages

Compared with other major languages, Rok's error model sits as follows.

| Aspect           | Rust           | Swift                     | Java            | Rok                                       |
|------------------|----------------|---------------------------|-----------------|-------------------------------------------|
| Base model       | `Result<T, E>` | `throws`                  | `throws`        | `throws`; `Result` is for holding         |
| Call-site marker | `?`            | `try`                     | **none**        | `try` / `try?` / `try!`                   |
| typed throws     | **none**       | `throws(E)` (single type) | `throws E1, E2` | `throws (E)` / `throws (A \| B)`          |
| `?` operator     | `Result?`      | optional chaining         | **none**        | only for `T \| none`                      |
| Option type      | `Option<T>`    | `Optional<T>`             | `Optional<T>`   | **none**: uses `T \| none`                |
| Implementation   | return value   | return value (error register) | stack unwinding | return value `{ is_err, payload }`    |
| panic            | `panic!`       | `fatalError`              | `Error`         | `panic_fn`: cannot be caught, exit code 101  |

Rok differs in three main ways. First, there is no `Option`; the sum type `T | none` takes its place, removing one type. Second, error propagation is unified under `throws` and `try`, and `Result` serves only to hold errors as values. Third, the typed sum throws `throws (A | B | C)` is more expressive than Swift's typed throws, which accepts only a single type.
