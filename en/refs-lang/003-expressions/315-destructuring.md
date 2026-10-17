# 3.15. Destructuring assignment

Writing an irrefutable pattern of [patterns](311-patterns.md) on the left-hand side of `let` unpacks the result of one expression into several variables.

## 3.15.1. Form

The grammar of destructuring assignment is:

```
destructure_let ::= "let" irrefutable_pattern (":" type_expr)? "=" expr ";"
```

```rok
let pattern = expr;
```

This form matches the expression's result against the pattern and declares all binding variables in the pattern at once.

## 3.15.2. Tuple destructuring

The most basic form, tuple destructuring, is used as follows.

```rok
let (a, b) = (1, 2);                    // a = 1, b = 2
let (q, r) = divmod(10, 3);             // receive two values at once
let (q, _) = divmod(10, 3);             // discard the second
```

The number of elements must match exactly; otherwise it is a compile error.

```rok
let (a, b) = (1, 2, 3);                 // ERROR: length mismatch
```

## 3.15.3. Rest patterns

To receive the remaining elements together, use a rest pattern with `..`.

```rok
let (first, ..rest) = (1, 2, 3, 4);     // rest = (2, 3, 4)
let (..init, last) = (1, 2, 3, 4);      // init = (1, 2, 3), last = 4
let (head, ..mid, tail) = (1, 2, 3, 4, 5);  // mid = (2, 3, 4)
let (..) = (1, 2, 3);                    // discard everything
```

`..` appears at most once in a pattern and is used to bundle the rest into one variable or to discard it.

## 3.15.4. Nested destructuring

Destructuring patterns can nest to any depth, so complex structures can be unpacked at once.

```rok
let ((a, b), c) = ((1, 2), 3);          // a = 1, b = 2, c = 3
let (Point { x, y }, scale) = (origin, 2.0f64);
```

The destructured shape need only match the structure of the value.

## 3.15.5. Struct destructuring

For struct values, the `struct` pattern of [patterns](311-patterns.md) can be used directly.

```rok
let Point { x, y } = origin;            // x = origin.x, y = origin.y
let Point { x, .. } = origin;           // discard other fields
let Point { x: px, y: py } = origin;    // rename
```

This extracts fields together, receives only some fields while discarding the rest, or renames them.

## 3.15.6. Type annotations

Types may be given for each part of the destructuring or for the whole pattern at once.

```rok
let (a: i32, b: str) = (1i32, "hi");    // type on each position
let (a, b): (i32, str) = (1i32, "hi");  // type on the whole pattern
```

An annotated type is a slot type, not a runtime type check. It must match the type of the value; otherwise it is a compile error.

```rok
let (a: i32, b: str) = (1, 2);          // ERROR: second type mismatch
```

## 3.15.7. Irrefutable patterns only

The pattern of `let pattern = expr;` must be irrefutable, meaning it must always match.

Allowed patterns:

- identifiers, wildcards.
- complete matches of tuples, structs, and tuple-structs (no missing fields or positions).
- nestings of the above.

Disallowed patterns:

- literal patterns (`1`, `"hello"`, etc.).
- patterns that match only a particular `enum` variant (`Circle(r)`, `Click { x, y }`, etc.).
- type-check patterns (`v: i32`, on a sum type or `any` slot).
- range patterns.

These forms can fail to match, so receiving them with `let` is unsafe. Use [`let-else`](316-let-else.md), the [`match` expression](312-match-expression.md), or `if let` ([`if` expression](310-if-expression.md)) instead.

## 3.15.8. Default values (none)

Default values in destructuring positions are not adopted.

```
let (a, b = 0i32) = ...;        // not adopted
```

Default values for function parameters are considered sufficient. To handle a possibly absent value in a destructuring position, use `match` or `let-else`.

## 3.15.9. Visibility of variables after destructuring

Every variable declared by destructuring is introduced into the scope where the `let` appears.

```rok
let (a, b) = (1, 2);
// a and b are both visible here
```

Variables declared by the same `let` become visible simultaneously; none becomes visible before another.

## 3.15.10. Number of evaluations of the matched expression

The right-hand expression of a destructuring is evaluated exactly once, and the result is matched against the pattern. An expression with side effects is therefore guaranteed to be called once.

## 3.15.11. Generalization of multiple assignment

Destructuring assignment can be seen as a generalization of multiple assignment.

```rok
let (a, b) = (b, a);                    // swap, in one expression
let (q, r) = divmod(num, den);          // two results
let (head, ..tail) = items;             // split
```

This form also expresses the C- or Python-style `a, b = b, a` pattern naturally.

## 3.15.12. Without `let`: destructuring reassignment (none)

Destructuring reassignment of existing variables is not adopted.

```
(a, b) = (1, 2);                // not adopted
```

`let` is always required, to keep the distinction between new declaration and reassignment clear (consistent with the decision in [assignment](304-assignment.md)).

To assign destructured results to existing variables, go through temporaries or assign them separately.
