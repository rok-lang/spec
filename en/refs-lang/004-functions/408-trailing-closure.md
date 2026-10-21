# 4.8. Trailing closures

When the last argument of a function or method call is a closure, the closure may be moved outside the parentheses. This form is called a trailing closure.

## 4.8.1. Form

The grammar of trailing closures is:

```
trailing_call   ::= primary "(" arg_list? ")" trailing_closure
                  | primary trailing_closure                    // 0-arg call
trailing_closure ::= ("|" closure_params? "|" ("->" type)?)? block
```

```rok
function(args) |params| { body }
function(args) { body }            // zero-parameter shorthand
```

The closure directly follows the call parentheses, and the body is a block from [block expressions](../003-expressions/309-block-expressions.md).

### Zero-Parameter shorthand: `{ body }`

A closure without parameters may be shortened from `|| { body }` to `{ body }`.

```rok
let h = thread::spawn { heavy_work() };
// == thread::spawn(|| { heavy_work() })

with_lock(mutex) { critical_section() };
```

This shorthand has meaning only for a `{ ... }` directly after a function call. A `{ ... }` anywhere else is an ordinary block expression from [block expressions](../003-expressions/309-block-expressions.md).

### One or more parameters

With one or more parameters, write:

```rok
let doubled = xs.map |x| { x * 2 };
let pos = xs.filter |x| { x > 0 };
let opened = with_file(path) |f| {
    let data = f.read();
    process(data)
};
pairs.for_each |k, v| { print(k, v) };
```

The pipes (`|...|`) go outside the parentheses, directly before the body. The body is always a block: always `|x| { body }`, never a one-line `|x| body`.

Trailing closures use block bodies by default for readability and consistency.

### Rules for the parenthesis-free form

Attaching a closure without call parentheses, as in `xs.map |x| { .. }`, is a trailing closure only under these conditions:

- It is immediately preceded by a path (`thread::spawn`, `f`) or a member name (`.map`).
- It is followed by `|params|`, an optional `-> R`, and `{`. Without this shape, `|` is bitwise OR.
- `||` is always logical OR. A closure without parameters uses the `{ body }` shorthand.
- Inside the head (condition and scrutinee) of `if` / `while` / `match`, parenthesis-free trailing closures and the `{ body }` shorthand are disabled; a `{` there opens the body block. If a closure is needed in the head, use parentheses (`if xs.any(|x| { x > 0 }) { .. }`).

```rok
let ys = xs.map |x| -> i64 { x as i64 * 2 };   // explicit return type
let z = a | b;                                  // bitwise OR — not followed by the `|params| {` shape
if ok || retry { .. }                           // logical OR
```

## 4.8.2. Free choice with explicit parentheses

Trailing closures are permitted, not required.

```rok
xs.map(|x| x * 2);                // explicit parentheses — OK
xs.map |x| { x * 2 };              // trailing — OK
```

Teams and users may choose freely according to their style.

## 4.8.3. When the closure is not the last argument

A trailing closure can fill only the last argument position. If the closure is a middle argument, explicit parentheses must be used.

```rok
fn process(input: str, transform: fn(str) -> str, output: str) -> str { ... }

process("in", |s| { s.to_upper() }, "out");           // explicit parentheses (middle argument)
process("in", "out") |s| { s.to_upper() };            // ERROR — trailing only in the last position
```

## 4.8.4. Calls with zero arguments

When the call has no other arguments, the empty parentheses may also be omitted.

```rok
thread::spawn { heavy() };        // same as thread::spawn() { heavy() }
                                   // == thread::spawn(|| { heavy() })
```

When the only parameter is a closure, as with `thread::spawn`, this is the most natural call form.

## 4.8.5. Nesting

Trailing closures may be nested freely.

```rok
par::scope |s| {
    s.spawn { f() };
    s.spawn { g() };
};
```

The same shorthand rule (`{ body }` ↔ `|| { body }`) applies to inner closures.

## 4.8.6. Resolving parser ambiguity

Trailing closures introduce an ambiguity where some code could be read as either a block expression or a trailing closure. The following rules resolve it.

| Position                                                                          | Meaning                                                              |
|-----------------------------------------------------------------------------------|----------------------------------------------------------------------|
| `{ ... }` after a control keyword (`if`, `else`, `for`, `while`, `loop`, `match`) | control body                                                         |
| `{ ... }` directly after a function call                                          | trailing closure (0-arg)                                             |
| `\|params\| { ... }` directly after a function call                               | trailing closure (1+ arg)                                            |
| `{ ... }` in an ordinary expression position such as `x = { ... };`               | block expression (evaluated immediately) — a 0-arg closure needs explicit `\|\| { ... }` |
| `\|...\| { ... }` inside an expression                                            | ordinary closure expression                                          |

In short: a `{ ... }` or `|...| { ... }` directly after a function call (`(args)`), with or without whitespace, is a trailing closure.

## 4.8.7. A lone `{}`: ordinary block vs 0-Arg closure

```rok
let x = { 42 };                   // block expression (value 42)
let f = { 42 };                   // ??
```

These two lines have the same form; which meaning takes precedence?

The rule: a `{ body }` appearing alone in expression position is a block expression.

If a 0-arg closure is intended, write `|| { body }` explicitly.

```rok
let f = || { 42 };                // explicit 0-arg closure
let r = f();                       // call
```

That is, only a `{ body }` directly after a function call is recognized as a trailing closure.

## 4.8.8. Closures in statement-like positions such as par `for`

For parallel loops, transactions, context managers and the like, trailing closures produce DSL-friendly forms that read like keywords.

```rok
par::for_each(xs) |x| { ... }     // looks like par for
with_lock(mutex) { ... }           // looks like a with-statement
retry(3) { ... }                   // looks like a retry keyword
```

This is consistent with the principle of minimizing reserved keywords in [keywords](../001-lexical/105-keywords.md). Combining library functions with trailing closures yields patterns that read like control flow.

## 4.8.9. Combining with named arguments

```rok
fn run(name: str = "task", body: fn() -> ()) { ... }

run("compute") { ... };           // positional + trailing OK
run(name: "compute") { ... };     // named + trailing OK
```

Positional arguments, named arguments and trailing closures may be combined freely.

## 4.8.10. Summary

Trailing closures allow DSL-like expressions at the specification level without additional keywords. The key point is that users can build patterns that look like language keywords directly as libraries.
