# 3.12. `match` expression

`match` is an expression that compares a value against several patterns and evaluates the first arm that matches.

## 3.12.1. Form

```
match_expr  ::= "match" expr "{" match_arm ("," match_arm)* ","? "}"
match_arm   ::= pattern ("if" expr)? "=>" expr
```

```rok
match expr {
    pattern1 => expr1,
    pattern2 => expr2,
    ...
    patternN => exprN,
}
```

A `match` expression consists of the `match` keyword, a scrutinee, and a `{ ... }` body. A struct literal cannot appear at the top level of the scrutinee ([3.10.3](310-if-expression.md#struct-literals-in-condition-position)). The body contains zero or more arms separated by commas; each arm consists of a pattern, the `=>` token ([punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md)), and an expression. A trailing comma is allowed.

## 3.12.2. Arm bodies

```rok
match v {
    1 => "one",
    2 => "two",
    _ => "other",
}
```

A single expression follows `=>`. If that expression is a [block expression](309-block-expressions.md), it can contain several statements.

```rok
match v {                        // v: i32 | none
    n: i32 => {
        let x = n * 2;
        x + 1
    },
    none => 0,
}
```

The value of the last expression of a block arm is the result of that arm.

## 3.12.3. Matching order

Arms are checked top to bottom, and the first matching arm is evaluated. The value of that arm's expression is the result of the whole `match` expression.

## 3.12.4. Branch result type

Per the branch result rules of the [expressions overview](README.md), if all arm result expressions have the same type, that type is the result type of the `match` expression. If arm types differ, the receiving side must have an explicit type; sum types are not inferred automatically. The type of the receiving slot follows the rules of [variable declaration](../005-modules/501-variable-declaration.md).

```rok
// no type on the slot → inferred from the expression's result type
let x = match v {
    1 => "one",
    _ => "other",
};                      // expression result str, slot x: str

let y = match v {
    1 => 1i32,
    _ => "other",
};                      // ERROR: arm type mismatch - type annotation required

// sum type on the slot → member check
let w: i32 | str = match v {
    1 => 1i32,
    _ => "other",
};                      // OK

// single type on the slot → the expression's result must be that type
let z: i32 = match v {
    1 => 1i32,
    _ => "other",
};                      // ERROR: str arm does not fit i32
```

## 3.12.5. Exhaustiveness checking

The set of arm patterns must cover every value the scrutinee can have; otherwise it is a compile error.

```rok
let x: bool = ...;
match x {
    true => 1,
    // ERROR: false is missing
}
```

The rules are as follows. Covering every member of a closed sum type (`A | B`) is exhaustive. For an `enum`, covering every variant is exhaustive. `bool` is exhaustive when both `true` and `false` are covered. Literal and range patterns of integers, `char`, strings, and floating-point numbers are treated as open sets. Even if ranges cover every value of the type (`0..=255` for all of `u8`), this is not considered exhaustive, so a catch-all arm such as a wildcard (`_`) or identifier pattern (`x`) is required. The non-exhaustiveness error gives one missing value as an example (e.g. `false`).

A match on the dynamic slot of an [`any` slot](../002-types/202-any-slot.md) always requires a catch-all arm, because the compiler cannot know which type will arrive.

## 3.12.6. Unreachable arms

An arm that can never be reached in matching order is a compile error (not a warning). Literal / range patterns of integers and `char` are compared as intervals, so a range already covered by the intervals of earlier arms is also unreachable (`3..=5` after `0..=9`). Ranges do not establish exhaustiveness (3.12.5).

```rok
match v {
    _ => 0,
    1 => 1,         // ERROR: the _ above always matches
}
```

## 3.12.7. Or-Patterns

An or-pattern (`|`) of [patterns](311-patterns.md) groups several patterns sharing the same body.

```rok
match v {
    1 | 2 | 3 => "small",
    4 | 5 | 6 => "medium",
    _         => "large",
}
```

## 3.12.8. Guards

A guard (`if condition`) of [patterns](311-patterns.md) can be attached to an arm.

```rok
match (x, y) {
    (a, b) if a == b => "equal",
    (a, b) if a > b  => "first bigger",
    _                => "second bigger",
}
```

An arm with a guard is not counted as a catch-all in exhaustiveness checking, because the guard may be `false`. Guarded arms alone therefore do not guarantee exhaustiveness.

```rok
let x: bool = ...;
match x {
    a if a == true  => 1,
    a if a == false => 2,
    // ERROR: guarded arms alone do not guarantee exhaustiveness; _ required
}
```

## 3.12.9. Narrowing from match results

Inside an arm matched by a type pattern or an `enum` pattern, the compiler narrows the type of the scrutinee.

```rok
let v: i32 | str = ...;
match v {
    n: i32 => use_int(n),       // in this arm n: i32, static dispatch possible
    s: str => use_str(s),
}
```

This narrowing works by the same mechanism as narrowing of the [`any` slot](../002-types/202-any-slot.md) and [sum types](../002-types/206-sum-types.md).

## 3.12.10. Evaluation order

`match` is evaluated in this order:

1. Evaluate the scrutinee (once).
2. Check the arms top to bottom.
3. Evaluate the expression of the first matching arm.

Any side effects of the scrutinee occur exactly once. The guard of a guarded arm is evaluated only when that arm is reached.

## 3.12.11. Result of the match expression itself

The result of the `match` expression itself follows the branch result rules above (if all arms have the same type, that type; otherwise the receiving side must have an explicit type).

If the scrutinee is divergent (e.g. `match return 0 { ... }`), the `match` is also divergent.

## 3.12.12. Divergent arms

If an arm's expression is a divergent form of [block expressions](309-block-expressions.md) (`return`, `break`, `throw`, etc.), that arm is treated as compatible with any type in the branch result type check.

```rok
let x: i32 = match v {           // v: i32 | none
    n: i32  => n,
    none    => return 0,         // divergent, OK in an i32 position
};
```
