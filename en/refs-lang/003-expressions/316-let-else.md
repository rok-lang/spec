# 3.16. `let-else`

`let-else` places a refutable pattern on the left-hand side of `let` and writes, together with it, the divergent branch taken when the match fails.

## 3.16.1. Form

The grammar is:

```
let_else_stmt   ::= "let" pattern (":" type_expr)? "=" expr "else" block ";"
```

```rok
let pattern = expr else { diverge };
```

`let` is followed by a pattern, `=` and an expression, then `else` and a block. Any refutable pattern is accepted. The `else` body must be a divergent block of [block expressions](309-block-expressions.md), so it must exit via `return`, `break`, `continue`, `throw`, or the like.

## 3.16.2. Behavior

```rok
let value: T = compute() else {       // compute(): T | none
    return default;
};
// here value is a value of type T (if none, control left via the else above)
```

Evaluation proceeds as follows. The expression's result is matched against the pattern; on success, the pattern's bindings are introduced into the current scope and evaluation continues with the next line. On failure, the `else` body is evaluated, and because `else` is divergent, control leaves at that point.

## 3.16.3. Divergence required

The `else` body must be divergent. A body that completes normally is a compile error.

```rok
let v: T = compute() else {
    print("no value");          // ERROR: not divergent
};
```

This requirement prevents evaluation from continuing without the binding variables after a failed match. Only if the `else` body exits can the existence of the bindings be guaranteed on the next line.

## 3.16.4. Unwrapping sum types and `enum`s

The most common use is accepting one case of a `T | none` or an `enum` variant and exiting early in all other cases.

```rok
// unwrap one member of a sum type
let n: i32 = x else {                    // x: i32 | f32
    panic("expected i32");
};
// here n: i32

// unwrap an enum variant
let Ok(data) = parse(input) else {
    return Err(ParseFailed);
};
// data is the contents of the Ok variant
```

This pattern keeps the vertical flow more concise than `match`, which is useful for explicitly separating the main flow from exceptional cases.

> [!NOTE]
> In the form `let identifier : Type = expr else { ... };`, the `: Type` is interpreted as a **refutable type-check pattern** (the `type_check_pattern` of [patterns](311-patterns.md)). Unlike a slot type annotation in an ordinary `let`, it can fail to match, which is what gives the `else` branch its meaning. That is, `let n: i32 = x else { ... };` is equivalent to `match x { v: i32 => v, _ => { ... } }`, and `: i32` is not the slot type of `n` but the check `x` must pass.

## 3.16.5. Visibility

On a successful match, the pattern's binding variables are introduced into the scope where the `let-else` appears — not inside the `else` body.

```rok
{
    let v: T = compute() else { return; };
    // v is visible here
    use(v);
}
// v is not visible here (scope ended)
```

The binding variables are not visible in the `else` body, because the match has just failed and no value exists.

## 3.16.6. Guards (none)

```
let v: T = expr if v > 0 else { ... };       // not adopted
```

A guard (`if condition`) in a `let-else` is not adopted. If a guard is needed, check it separately afterward.

```rok
let v: T = compute() else { return; };
if v <= 0 { return; }
```

## 3.16.7. Exit keywords of `else`

Exit keywords commonly found in the `else` body and their effects:

| Keyword                         | Effect                                     |
|---------------------------------|--------------------------------------------|
| `return val;`                   | return from the function                   |
| `break;` and `break value;`     | exit the enclosing loop                    |
| `continue;`                     | go to the next iteration of the loop       |
| `throw err;`                    | raise an error                             |
| `panic("msg")`                  | library panic function, unrecoverable exit |
| `loop { ... }` (no `break`)     | divergent loop                             |

Any of these, and any form the compiler classifies as divergent, is allowed.

## 3.16.8. Comparison with `match`

```rok
// let-else
let v: T = compute() else { return; };       // compute(): T | none
use(v);

// match equivalent
let v = match compute() {
    v: T => v,
    none => return,
};
use(v);
```

The two express the same meaning. Choose `let-else` when its flat main flow is more readable.

## 3.16.9. Evaluation order

`let-else` is evaluated in this order:

1. Evaluate the right-hand expression.
2. Match the result against the pattern.
3. On success, bind and continue with the next line.
4. On failure, evaluate the `else` body; being divergent, control leaves.

If the right-hand expression has side effects, it is evaluated only once.
