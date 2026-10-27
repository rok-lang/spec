# 3.10. `if` expression

`if` and `else` form an expression that branches on a boolean.

## 3.10.1. Form

```
if_expr     ::= "if" expr block ("else" (if_expr | block))?
              | "if" "let" pattern "=" expr block ("else" (if_expr | block))?
```

```rok
if cond { ... }
if cond { ... } else { ... }
if cond1 { ... } else if cond2 { ... } else { ... }
```

Here `cond` is an expression of type `bool`, and both the body and the `else` body are [block expressions](309-block-expressions.md). `else if` is shorthand for `else { if ... }`; it is not a separate token but a natural combination.

## 3.10.2. Value as an expression

For `if` to be used as a value, it must have an `else` clause.

```rok
let a = if cond { 1i32 } else { 2i32 };          // OK, i32
let b = if cond { 1i32 };                          // OK, but b has type unit (no else)
```

An `if` without `else` has only one branch, so it returns no value and its result is unit. An `if` with `else` combines the values of both branches into its result.

### Branch result type

Per the branch result rules of the [expressions overview](README.md), if all branches have the same type, that type is the result type of the `if` expression. If branch types differ, the receiving side must have an explicit type and each branch must be compatible with it. Sum types are not inferred automatically. The type of the receiving slot follows the rules of [variable declaration](../005-modules/501-variable-declaration.md).

```rok
// no type on the slot → inferred from the expression's result type
let x = if cond { 1i32 } else { 2i32 };          // expression result i32, slot x: i32
let y = if cond { 1i32 } else { 2.0f64 };        // ERROR: branch type mismatch - type annotation required

// sum type on the slot → member check
let w: i32 | f64 = if cond { 1i32 } else { 2.0f64 };   // OK

// single type on the slot → the expression's result must be that type
let z: i32 = if cond { 1i32 } else { 2.0f64 };   // ERROR: f64 branch does not fit i32
```

Every branch of an `else if` chain - every `else if` and the final `else` - is subject to this check.

## 3.10.3. Position of the condition expression

```rok
if cond { ... }
```

Any `bool` expression may appear in the condition position `cond`: calls, method calls, comparisons, logical combinations, and so on.

```rok
if user.is_admin() && !user.is_locked { ... }
if items.len() > 0 { ... }
```

A non-`bool` type is a compile error. There is no implicit truthy conversion treating 0 or an empty string as `false`. Where a boolean is needed, write an explicit conversion or comparison.

```
if x { ... }              // ERROR if x: i32 (i32 is not bool)
if x != 0i32 { ... }      // OK
```

### Struct literals in condition position

In the following positions, a struct literal cannot appear at the top level of the expression, because its `{` would be read as the start of the body.

- the condition of `if` / `while` / `if let` / `while let`
- the scrutinee of `match` ([3.12](312-match-expression.md))
- the iteration target of `for` ([3.13](313-loop-expressions.md))
- the resource expression of `using` ([3.17](317-using-expression.md))

Inside delimiters (`( )`, `[ ]`, call / method arguments, arrays, tuples, indices, dict literals, blocks `{ }`) it is allowed. To use one at the top level, wrap it in parentheses.

```rok
if S { a: 1 }.ok() { .. }       // ERROR - the `{` after `S` is the body
if (S { a: 1 }).ok() { .. }     // OK
if check(S { a: 1 }) { .. }     // OK - inside an argument
```

## 3.10.4. Divergent branches

Divergent blocks of [block expressions](309-block-expressions.md) are compatible with any type in the branch result type check.

```rok
let x: i32 = if cond {
    42i32
} else {
    return -1i32;          // divergent, OK even in an i32 position
};
```

This rule lets early exits be expressed naturally.

## 3.10.5. `if let`: combining with pattern matching

The condition position of `if` can also combine with pattern matching.

```rok
if let v: User = compute() {       // compute(): User | none
    consume(v);
} else {
    fallback();
}
```

```
if_let ::= "if" "let" pattern (":" type_expr)? "=" expr block ("else" (block | if_expr))?
```

- Without a type, the expression's value is matched against `pattern`.
- With `: T`, the value is first narrowed to `T`, then destructured by `pattern`. In this case `pattern` must be irrefutable for `T` (`if let (a, b): (i32, i32) = v { .. }`).
- On a match, the body runs and the pattern's bindings are visible inside it. Otherwise, `else` runs.

The pattern syntax of `if let` is defined in [patterns](311-patterns.md).

## 3.10.6. Chained `if let` (none)

Chaining several `if let`s on one line, in the style of Rust RFC 2497, is not adopted.

```
if let A(a) = x && let B(b) = y { ... }     // not adopted
```

As alternatives, use a tuple pattern of [patterns](311-patterns.md) or nested `if let`.

## 3.10.7. Restriction on the form of the `else` body

Only a block (`{ ... }`) or another `if` expression may follow `else`. A plain expression directly after it is not allowed.

```
if a { 1 } else 2          // ERROR: block required after else
if a { 1 } else { 2 }      // OK
if a { 1 } else if b { 2 } else { 3 }    // OK, if after else
```

This restriction prevents an ambiguous dangling `else`.

## 3.10.8. Evaluation order

`if` is evaluated as follows.

1. Evaluate `cond`.
2. If `cond` is `true`, evaluate the `then` body; if `false`, evaluate the `else` body.

Evaluation short-circuits. As soon as `cond` is determined, only one side is evaluated, and the other side's side effects do not occur. An `else if` chain is checked top to bottom, and the first branch whose condition is `true` runs.
