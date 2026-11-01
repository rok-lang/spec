# 4.3. Return

This section defines the rules by which a function body produces its result value.

## 4.3.1. Core rule: `return` is not required

A function body is a block as defined in [block expressions](../003-expressions/309-block-expressions.md), and the value of the body's last expression becomes the function's return value.

```rok
fn add(x: i32, y: i32) -> i32 {
    x + y                         // returned automatically
}
```

The `return` keyword is used only for early exit; returning a single value on the normal path is idiomatically expressed with the last expression.

## 4.3.2. Explicit `return`

An explicit return has this grammar:

```
return_expr     ::= "return" expr?
```

```rok
return expr;
return;
```

`return expr;` exits the function returning the value of `expr`; `return;` exits without a value (that is, returning unit). A `return` expression is divergent: evaluation leaves the function at that point.

```rok
fn early(x: i32) -> i32 {
    if x < 0 { return 0; }        // early exit
    x * 2
}

fn validate(s: str) -> bool {
    if s.len() == 0 { return false; }
    if s.starts_with("#") { return false; }
    true
}
```

## 4.3.3. Agreement between return type and body

The return type defined in [function declarations](401-function-declaration.md) must agree on every return path of the body. A return path is the type of the body's last expression and the type of `expr` in every `return expr;`.

The declared return type is the receiving type of every path, as in the branch result rules of the [expressions overview](../003-expressions/README.md#branch-result-types): each path must be compatible with it, and paths of different types are fine when the declared type is a sum that takes each of them.

```rok
fn maybe_int(cond: bool) -> i32 | str {
    if cond { return 42i32; }
    "no"                          // each path fits the declared i32 | str
}
```

## 4.3.4. Omitting the return type

As shown in [function declarations](401-function-declaration.md), the return type may be omitted, in which case it is inferred from the body. On a `pub` or `pub(pkg)` function a missing `->` means `()` instead ([4.1.3](401-function-declaration.md#413-omitting-the-return-type)).

```rok
fn double(x: i32) {
    x * 2                          // inferred: i32
}
```

The inferred type is the type of all value paths (`return expr` and the final expression), which must agree: no sum is invented for paths of different types, so they are a compile error (E0041) whose help asks for the return type (`-> i32 | str`). It is also a compile error if a value path is mixed with a path that returns no value, or if the return type depends on itself: a recursive function needs `-> T`. If no path reaches the end of the body or a `return`, the inferred type is the diverging type (4.3.7).

## 4.3.5. Multiple returns: tuples

To return multiple values, use a tuple.

```rok
fn divmod(n: i32, d: i32) -> (i32, i32) {
    let q = n / d;
    let r = n % d;
    (q, r)
}

let (quotient, remainder) = divmod(10i32, 3i32);
```

The tuple return type follows tuple types in [type expressions](../002-types/201-type-expressions.md), and the receiver uses tuple destructuring from [destructuring assignment](../003-expressions/315-destructuring.md).

No separate multiple-return keyword or syntax is introduced; tuples suffice.

## 4.3.6. Calls that discard the result

When a function result is not used, write:

```rok
compute();                        // result discarded (expression + ;)
_ = compute();                    // intentional discard (explicit)
```

Both forms discard the result, but `_ = ...` signals that the user intentionally ignores it. Where a tool or linter warns on ignored results, the `_ = ...` form acts as an acknowledgement that suppresses the warning.

## 4.3.7. Divergent functions

Some functions never complete normally: control always leaves through `return`, `throw`, an infinite loop and so on.

```rok
fn fail_always() throws {
    throw ValueError::new("fatal");
}

fn loop_forever() {
    loop { }
}
```

The return type of such a function can hold no value. It is written `!` (never), as in `fn fail() -> ! { ... }`.

A call to a divergent function is compatible with any type, like a divergent block in [block expressions](../003-expressions/309-block-expressions.md).

```rok
let x: i32 = if cond {
    42i32
} else {
    fail_always()                 // divergent: OK in an i32 position
};
```

## 4.3.8. Position of `return`

A `return` expression may appear anywhere in a function body: inside blocks, loops, `match` arms and so on.

```rok
fn find(xs: List<i32>, target: i32) -> i32 | none {
    for x in xs {
        if x == target { return x; }   // returns from the function itself inside the loop
    }
    none
}
```

A `return` nested inside a loop, `match`, etc. exits the enclosing function itself. To exit only the loop, use `break` from [loops](../003-expressions/313-loop-expressions.md).

## 4.3.9. When the return value is evaluated

The `expr` of `return expr;` is fully evaluated when the `return` expression is evaluated, and then the function exits.

```rok
fn f() -> i32 {
    return compute_a() + compute_b();   // both evaluated, then the sum is returned
}
```

Thus if an expression with side effects is inside `return`, the side effects happen just before returning.

## 4.3.10. `return` and try/catch

A `return` inside a `try` body likewise exits the enclosing function. However, if there is a `finally` body from [error control flow](../003-expressions/314-error-control-flow.md), it is always evaluated on the way out.

```rok
fn cleanup_aware() -> i32 {
    try {
        return 42i32;             // finally runs first
    } finally {
        cleanup();
    }
}
```

This function calls `cleanup()` and then returns `42i32`.

## 4.3.11. `return` in closures and nested functions

A `return` in a closure or nested function exits that closure or function itself, not the enclosing function.

```rok
fn outer() -> i32 {
    let f = |x: i32| -> i32 {
        if x < 0 { return 0; }   // exits only this closure
        x * 2
    };
    f(-5i32);                     // f returns 0 from this call
    100i32                         // outer itself returns 100
}
```

A bare `return;` in the [trailing closure](408-trailing-closure.md) of a call that is a statement reads like a return from the function, but it would end only that call of the closure, so it is a compile error. To end this call of the closure, write the labeled return `return@each` (4.3.12); to leave the function, write a `for` loop instead of the call. The help offers `return@each` with a fix.

```rok
fn main() {
    each(xs) |x| {
        if x == 2 { return; }     // ERROR: would end only this call
        println(f"{x}");
    }
    for x in xs {
        if x == 2 { return; }     // leaves main
        println(f"{x}");
    }
}
```

A `return` in a closure passed in parentheses, in a trailing closure whose call is not a statement (`let n = first(xs) |x| { .. }`) and in a closure nested inside a trailing closure stays allowed: it reads as the closure's own.

Details on defining nested functions and closures are in [Closures](405-closure.md).

## 4.3.12. Labeled return

`return@name` ends one call of the innermost closure when that closure is the trailing closure of a call to `name`. The label is the callee's last path segment or the method's name: `each(xs) |x| { .. }` and `m::each(xs) |x| { .. }` take `@each`, and `xs.visit() |x| { .. }` takes `@visit`. The `@` follows `return` with no space. A value follows as with `return`: `return@first true`.

```
labeled_return ::= "return" "@" identifier expression?
```

```rok
fn main() {
    each(xs) |x| {
        if x == 2 { return@each; }    // ends this call; 3 still prints
        println(f"{x}");
    }
    let n = first(xs) |x| { if x > 1 { return@first true; } false };
}
```

A label that does not name the innermost closure is a compile error: a closure cannot leave another one around it, and a nested closure, an `async` / `yield` block and a nested function each start their own scope for `return`.

```rok
each(xs) |x| {
    let g = || { return@each; };      // ERROR: the innermost closure is g
}
```
