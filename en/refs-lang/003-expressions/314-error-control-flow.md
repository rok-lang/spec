# 3.14. Error control flow

`throw`, `try`, `catch`, and `finally` are the control flow for raising errors, marking the points where errors may flow, and receiving and handling them. Postfix `?` returns the `none` of a `T | none` value early to the caller.

```
throw_stmt     ::= "throw" expr ";"
try_expr       ::= ("try" | "try?" | "try!") expr
catch_expr     ::= expr catch_clause+ finally_clause?
                 | expr finally_clause
catch_clause   ::= "catch" pattern? block            // struct patterns also allowed: catch E { f, .. } { .. }
finally_clause ::= "finally" block
question_expr  ::= expr "?"
```

`catch` and `finally` are contextual keywords, keywords only in the position of a postfix clause after an expression or block ([1.5.4](../001-lexical/105-keywords.md#154-contextual-keywords)). The signatures of functions that can throw (`throws`, `rethrows`) are defined in [`throws` signatures](../004-functions/414-throws.md). This document covers control flow at the expression and statement level.

## 3.14.1. `throw`: raising an error

```rok
throw err;
```

A statement of the `throw` keyword followed by any expression; the expression's result value is thrown as the error value. A thrown error is handled by the nearest `catch` expression enclosing the statement; if unhandled, it becomes the error result of the enclosing function. The enclosing function's `throws` signature must then accept that error type ([4.14.5](../004-functions/414-throws.md#4145-agreement-between-throws-and-the-body)).

Flow does not continue after a `throw` statement, so a block ending in a `throw` statement is divergent.

```rok
let x = if cond { 42i32 } else { throw ValueError { msg: "not allowed" }; };
// x: i32 (else branch is divergent)
```

## 3.14.2. `try`: marking call sites

```rok
let n = try parse(s);
```

`try` is a prefix expression operator. A call to a throwing function (a function with a `throws` signature) must be inside an expression covered by `try`; otherwise it is a compile error. This rule makes every point where an error may flow visible in the code.

The value and type of `try e` are those of `e`. If a throwing call inside `e` raises an error, the nearest `catch` handles it; if there is no `catch` or none matches, it propagates to the enclosing function.

### Coverage

`try` covers the entire expression to its right. One `try` can mark several calls together.

```rok
try a(b());                 // covers both a and b
try f().g().h();            // covers the whole chain
try {                       // a block is also an expression, so every call in it is covered
    let cfg = load(path);
    apply(cfg);
};
```

The operand of `try` extends rightward to where the expression ends, stopping before a `catch` / `finally` clause, the statement terminator `;`, a closing bracket, or a comma. So `try a + b()` is `try (a + b())`, and the `catch` of `try parse(s) catch _ { 0 }` attaches to the whole `try parse(s)`. The exact binding rules are defined in [precedence and associativity](302-precedence-and-associativity.md).

When `try` appears as the right operand of a binary operator or in function argument position, it covers only that operand.

```rok
let total = base + try fetch_extra();
log(try describe(item));
```

### With `await`

Errors of an `async` throwing function are delivered at the `await` point, so the `await` expression is covered by `try` ([4.14.10](../004-functions/414-throws.md#41410-effect-of-throws-on-sum-type-signatures)). Errors of a generator are likewise received at the `next` point.

```rok
let data = try await fetch(url);
```

## 3.14.3. `try?` and `try!`

The two variants of `try` do not propagate the error to the enclosing function; they convert it in place into another form.

| Form     | Result type when `e: T`  | On error                          |
|----------|--------------------------|-----------------------------------|
| `try e`  | `T`                      | nearest `catch`, or propagate     |
| `try? e` | `T \| none`              | discard the error and yield `none` |
| `try! e` | `T`                      | panic (3.14.8)                    |

```rok
let port = (try? env::var("PORT")) ?? "8080";
let cfg = try! load_builtin_config();      // failure is a program bug
```

The `?` / `!` of `try?` / `try!` must immediately follow `try`. `try !x` is `try (!x)`. Because `try?` and `try!` do not propagate errors, the enclosing function need not be `throws`. Their coverage is the same as `try`. To chain `??` after the result of `try?`, close the range with parentheses as in the example above. Unparenthesized, `try? f() ?? d` is `try? (f() ?? d)`: an error gives `none`, not `d`. The compiler warns about that form (lint `try_coalesce_precedence`); write `(try? f()) ?? d` or `try? (f() ?? d)`.

## 3.14.4. `catch`: postfix handling expression

```rok
let n = try parse(s) catch e: ParseError { 0 } catch _ { -1 };
```

A `catch` clause is a postfix clause attached to the expression on its left. If an error arises while evaluating the left expression, the `catch` clauses are checked top to bottom and the block of the first matching clause is evaluated. If the left expression completes normally, no `catch` block runs. Errors from the left expression are those produced by throwing calls marked with `try` and by `throw` statements.

If no clause matches, the error propagates again, and the enclosing function must `throws` that error type.

### Block form

The form where `try` covers a block and `catch` / `finally` attach to that expression is interpreted by the same rules.

```rok
try {
    body
} catch e {
    handler
}
```

### Pattern and type matching

```rok
try fetch(url)
catch e: TimeoutError { retry() }
catch HttpError { code, msg } { fail(code, msg) }
catch e: any Error { log(f"{e}"); fallback() }
```

Any pattern form of [patterns](311-patterns.md) may follow `catch`. `e: TimeoutError` is a type-check pattern, and inside the matched branch `e` is narrowed to that type. `catch e: A | B` binds the caught error as the sum type `A | B` (a subset of the error sum type). Errors of untyped `throws` are [`any Error`](../002-types/202-any-slot.md) values, so to receive every error under one name, use `catch e: any Error` or `catch e`. Writing the bare trait name, `catch e: Error`, is a compile error per the [trait-as-type ban](../002-types/212-trait-as-type-banned.md).

`catch { ... }` with the pattern omitted is the same as `catch _ { ... }`.

### Exhaustiveness

If the `catch` clauses cover every error type the left expression can throw, the `catch` expression no longer throws. For a call with typed `throws (A | B)`, covering both `A` and `B` suffices; for untyped `throws`, there must be a clause accepting every value, such as `catch e`, `catch _`, or `catch e: any Error`.

```rok
fn parse_or_zero(s: str) -> i32 {          // non-throwing
    try parse(s) catch _ { 0 }             // covers every error
}
```

### Warnings for unreachable `try` and `catch`

The compiler warns in the following cases:

- a `try` that covers no throwing call
- a `catch` clause or branch that no error can reach

Clauses that trigger the warning are still type-checked.

## 3.14.5. `finally`

```rok
try {
    body
} finally {
    cleanup
}
```

It can also be combined with `catch`:

```rok
try {
    body
} catch e: IOError {
    handler
} finally {
    cleanup
}
```

The `finally` block runs in every case of leaving the left expression: normal completion, handling by `catch`, propagation of an unhandled error, and exit via `return` / `break` / `continue`. This makes it suitable for releasing resources, releasing locks, and restoring temporary state. If a `catch` block throws a new error or an error is thrown inside the `finally` block, the new error takes precedence.

Panics and fatal errors (3.14.8) terminate the process, so `finally` does not run.

## 3.14.6. Value and type of the expression

`try`, `catch`, and `finally` are all expressions.

```rok
let x = try parse(input) catch _ { default };
```

The value of a `catch` expression is the value of the left expression if it completes normally, or the value of the handler block if an error is caught. The left expression and every handler block must have the same type (the branch result rules of the [expressions overview](README.md)); a handler block that ends in `throw` or `return` is divergent and is excluded from the type check. The `finally` block produces no value and does not affect the expression's result.

## 3.14.7. Thrown values and function signatures

The `err` of `throw err` is any expression. A function with typed `throws (E)` may throw values corresponding to the members of `E`; a function with untyped `throws` is shorthand for `throws (any Error)` and may throw values implementing the standard library `Error` trait.

```rok
fn parse(s: str) throws -> i32 { ... }              // untyped, any Error
fn parse(s: str) throws (TypeError) -> i32 { ... }  // single type
fn parse(s: str) throws (A | B | C) -> i32 { ... }  // sum type
```

No error can leave a function whose signature lacks `throws`. When the `T` of `throws (T)` is a sum type, it is the closed set of errors the function can throw, used by callers to write `catch` clauses and check exhaustiveness.

Function signatures and the meaning of `throws` are defined in detail in [`throws` signatures](../004-functions/414-throws.md).

## 3.14.8. Panics and fatal errors

Unlike `throws` errors, the following are not delivered as values and cannot be caught with `catch`.

- Panic. This includes calls to the standard library `panic_fn`, failure of `try!`, division by zero, integer overflow, bounds-check failure, and so on.
- Fatal. States from which execution cannot continue, such as `StackOverflowError`, `OutOfMemoryError`, and `InternalError`.

In both cases the runtime prints a message to standard error (stderr) and terminates the process with exit code `101`. The exact list of situations that are panics or fatal errors is defined by the standard library.

A failed `try!` panics at the position of the `try!`. Its message shows the error the way an uncaught error out of `main` does ([4.16.4](../004-functions/416-entry-point.md): its `Display`, else its `Debug`, else its type name), then the position the error was raised at:

```text
thread 'main' panicked at src/main.rok:12:13:
try! failed: no such key `port`
  raised at src/main.rok:12:18
```

## 3.14.9. Evaluation order

`catch` expressions and `finally` are evaluated in this order:

1. Evaluate the left expression.
2. If the left expression raises an error, find the matching `catch` clause and evaluate its block.
3. If there is a `finally` block, evaluate it however the left expression was exited.
4. If no `catch` clause matches, run `finally` and then propagate the error outward.

When a `throw` statement is evaluated, its expression is evaluated first, and then the error is thrown.

## 3.14.10. The `?` operator - early return of `none`

`?` is a postfix unary operator that unwraps a `T | none` value and, if it is `none`, immediately returns `none` from the enclosing function.

```rok
fn first_word_len(text: str | none) -> usize | none {
    let t = text?;                      // none → return none immediately
    let w = t.split(" ").next()?;
    w.len()
}
```

In `e?` with `e: T | none`, the value of the expression is `T`. If `e` is `none`, the enclosing function does `return none`. The enclosing function's return type must therefore include a `none` member (`_ | none`); otherwise it is a compile error. Being postfix, it can be placed naturally at the end of a method chain.

### Desugaring rule

`?` is expanded directly by the compiler without going through a trait.

```
expr?
```

expands to:

```rok
match expr {
    none => return none,
    v => v,
}
```

### Scope of application

`?` applies only to `T | none`. It is not a means of propagating errors and cannot be used on standard library `Result` values or throwing calls. Error propagation is marked with `throws` signatures and `try`.

```rok
fn load() -> Result<Config, IOError> {
    let raw = read_result(path)?;     // ERROR: `?` is only for `T | none`.
                                      //   Use `throws` and `try` for error propagation.
    ...
}
```

A `?` inside a closure body operates on the closure's return type and is unrelated to the outer function's return type.

```rok
let lens = users.map(|u| -> usize | none { let nick = u.nickname?; nick.len() });
```

To fill `none` with a substitute value or pass member access through, use `?.` and `??` of [optional chaining and coalescing](308-optional-chaining-and-coalescing.md).
