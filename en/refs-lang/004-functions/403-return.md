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

If these differ, they are joined into a sum type by the branch result rules of the [expressions overview](../003-expressions/README.md), and that sum type must be compatible with the declared return type.

```rok
fn maybe_int(cond: bool) -> i32 | str {
    if cond { return 42i32; }
    "no"                          // sum of both paths = i32 | str, matches the declaration
}
```

## 4.3.4. Omitting the return type

As shown in [function declarations](401-function-declaration.md), the return type may be omitted, in which case it is inferred from the body.

```rok
fn double(x: i32) {
    x * 2                          // inferred: i32
}
```

The inferred type is the type of all value paths (`return expr` and the final expression) when they agree, or their sum when they differ. It is a compile error if a value path is mixed with a path that returns no value, or if the return type depends on itself: a recursive function needs `-> T`. If no path reaches the end of the body or a `return`, the inferred type is the diverging type (4.3.7).

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
    throw ValueError { msg: "fatal" };
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
    fail_always()                 // divergent - OK in an i32 position
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

Details on defining nested functions and closures are in [Closures](405-closure.md).
