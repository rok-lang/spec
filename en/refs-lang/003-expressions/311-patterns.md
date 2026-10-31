# 3.11. Patterns

A pattern checks a value by shape while binding parts of it to variables. Patterns appear in several positions: the arms of the [`match` expression](312-match-expression.md), `if let` of the [`if` expression](310-if-expression.md), the `let` left-hand side of [destructuring assignment](315-destructuring.md), and others.

```
pattern         ::= or_pattern
or_pattern      ::= unary_pattern ("|" unary_pattern)*
unary_pattern   ::= literal_pattern
                  | identifier_pattern
                  | "_"                                  // wildcard
                  | type_check_pattern
                  | tuple_pattern
                  | struct_pattern
                  | enum_variant_pattern
                  | range_pattern
                  | binding_pattern
                  | "(" pattern ")"                      // grouping
binding_pattern ::= identifier "@" unary_pattern
type_check_pattern  ::= identifier ":" type_expr
tuple_pattern   ::= "(" (pattern ("," pattern)* ","?)? ")"
struct_pattern  ::= path "{" (field_pattern ("," field_pattern)* ","?)? "}"
field_pattern   ::= identifier (":" pattern)?
enum_variant_pattern ::= path ("(" (pattern | "..") ("," (pattern | ".."))* ","? ")")?   // ".." at most once
range_pattern   ::= literal_pattern (".." | "..=") literal_pattern
```

## 3.11.1. Kinds of patterns

### Literal patterns

A literal pattern compares the value with a constant.

```rok
0
3.14f64
"hello"
true
none
'A'
```

Every literal form from [numeric literals](../001-lexical/106-numeric-literals.md) to [boolean and sentinel literals](../001-lexical/109-other-literals.md) may appear in pattern position; matching uses equality comparison (`==`). On a sum scrutinee, a literal or range pattern matches the member type it fits (`1` matches the `i32` member of `i32 | str`).

### Identifier patterns (bindings)

An identifier pattern introduces a new variable as it matches.

```rok
x
count
name
```

It always matches, whatever the value, and binds the value to the identifier. It must not collide with a reserved word of [keywords](../001-lexical/105-keywords.md). If a bare name resolves to an `enum` variant imported with `use`, it is not a binding but an `enum` pattern for that variant ([5.5.7](../005-modules/505-use.md#importing-enum-variants)). A bare name that is a variant of the scrutinee's `enum` type, but is not imported, is a compile error: as a binding it would match every value. Write the variant path `E::V`.

A capitalised name that resolves to a `const` or `static`, or that is within two edits of a variant of the scrutinee's `enum` type (`Gren` for `Green`), is still a new binding and matches every value. The compiler warns (`pattern_binds_const`, [6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control)). To compare with a constant, write a guard: `x if x == LIMIT`.

```rok
const LIMIT: i32 = 5;
match n {
    LIMIT => ..,              // warning: `LIMIT` here is a new variable that matches every value
    x if x == LIMIT => ..,    // compares with the const
    _ => ..,
}
```

### Wildcard pattern: `_`

```rok
_
```

The wildcard pattern also always matches, but discards the value instead of binding it. Use it to ignore a part explicitly.

A typed wildcard `_: T` matches only when the value is `T`, like a type-check pattern, and binds nothing.

```rok
match v {
    _: i32 => "int",          // matches if i32, no binding
    _ => "other",
}
```

### Type-Check patterns: ` name: Type`

```rok
n: i32
v: User
```

A type-check pattern matches only when the value of the dynamic slot or sum type on the left is of the given type. On a match, the value narrowed to that type is bound to the identifier.

This pattern expresses the narrowing mechanism of the [`any` slot](../002-types/202-any-slot.md) and [sum types](../002-types/206-sum-types.md) in pattern form.

The type after `:` is a full `type_expr`, so it includes `|`. `n: i32 | i64` is a single type-check pattern narrowing the value to the sum `i32 | i64`. An or-pattern of type-check patterns is written with parentheses: `(n: i32) | (m: i64)`. `v: A | B` on a value of plain type `A` always matches; `v` is the value as `A | B`.

```rok
match v {
    n: i32 => use_int(n),       // matches if v is i32
    s: str => use_str(s),       // matches if v is str
}
```

### Tuple patterns

```rok
(a, b)
(_, second)
(first, ..rest)
(..init, last)
(head, ..mid, tail)
(..)
```

`(p)` is a grouping pattern, the same as `p`, not a tuple. A one-element tuple is written with a trailing comma: `(p,)`. Tuple types (`(T,)`) and tuple expressions (`(e,)`) follow the same rule.

A tuple pattern matches each element of the tuple with a subpattern. `..` is the rest pattern, used to ignore part of the tuple or bind it to a single variable. `..` may appear at most once in a pattern, and the length must match the value (with `..` the length matches freely).

### `enum` patterns

```rok
Circle(r)
Rectangle(w, h)
Pending
Click { x, y }
Click { x, .. }
Rectangle(..)
Rectangle(w, ..)
```

An `enum` pattern starts with a variant name. A tuple variant uses positional subpatterns (`Circle(r)`), a named variant uses field names and subpatterns (`Click { x, y }`), and a unit variant uses just the name (`Pending`). A tuple variant pattern may use the rest `..` once (`V(..)`, `V(a, ..)`, `V(.., z)`).

> [!NOTE]
> What other languages express as `Option<T>` is expressed in Rok as `T | none` of [sum types](../002-types/206-sum-types.md). There are therefore no `enum` variant patterns like `Some(v)` or `None`. Matching on `T | none` uses a type-check pattern (`v: T`) and a literal pattern (`none`).

### `struct` patterns

```rok
Point { x, y }
Point { x, .. }
Point { x: px, y: py }
Point { x, y: 0.0 }      // y matches a literal, x is bound
```

A `struct` pattern consists of the `struct` name and field subpatterns inside `{ ... }`. When the field name and the binding variable name are the same, it can be abbreviated: `Point { x }` is the same as `Point { x: x }`. `..` ignores the remaining fields.

### Type patterns (simple)

```rok
i32
f64
bool
```

If the scrutinee is a sum type or `any`, a bare type name (`f64 =>`, `i32 | str =>`) is a type pattern that only checks whether the value has that type and binds nothing. For any other scrutinee, the same name is an identifier pattern (binding).

### Range patterns

```rok
1..5
1..=5
'a'..='z'
```

The tokens of [range expressions](307-range-expressions.md) are used as is in pattern position, and the pattern matches if the value lies within the range. They can be used with integers and `char` (`'a'..='z'`).

### Or-Patterns

```rok
1 | 2 | 3
"yes" | "y" | "Y"
Pending | Circle(0.0f64)
```

An or-pattern combines several subpatterns with `|`; the whole matches if any subpattern matches. The variables bound by the subpatterns must all have the same names and types.

```rok
match v {
    Ok(x) | OkAlt(x) => consume(x), // both variants bind the same name x with the same type
}
```

### Binding patterns: `name @ pat`

```rok
n @ 1..=9
p @ Point { x: 0, .. }
```

`name @ pat` binds the whole value to `name` if the subpattern `pat` matches. The bindings inside `pat` are also created. An or-pattern is wrapped in parentheses, as in `n @ (1 | 2)`. Binding patterns may be used not only in `match` but also in the patterns of `let` and `for`.

```rok
match age {
    n @ 0..=17 => f"minor {n}",
    n          => f"adult {n}",     // ranges are open sets, so a catch-all is needed
}
```

## 3.11.2. Guards

```rok
pattern if condition
```

A guard is a `bool` condition checked after the pattern matches; if the condition is `false`, the arm counts as a failed match.

```rok
match (x, y) {
    (a, b) if a == b => "equal",
    (a, b) if a > b  => "first bigger",
    _                => "second bigger",
}
```

A guard is a `bool` expression with the same meaning as the condition of the [`if` expression](310-if-expression.md).

## 3.11.3. Exhaustiveness checking

Exhaustiveness checking verifies that a set of patterns covers every possible input; the exact rules are defined in the [`match` expression](312-match-expression.md).

In short, a match on a closed sum type or an `enum` must cover every variant, or cover the rest with a wildcard, identifier pattern, or type pattern.

## 3.11.4. Consistency of variable bindings in or-patterns

```rok
(v: i32) | (_: i32) => v    // ERROR: one side binds, the other ignores
(n: i32) | (n: i64) => ...  // ERROR: n has different types. The diagnostic suggests n: i32 | i64
n: i32 | i64 => ...         // OK: n: i32 | i64
A(x) | B(y) => ...          // ERROR: x / y are asymmetric
```

Every subpattern of an or-pattern must bind variables of the same names with the same types. A name has a single type across all alternatives.

## 3.11.5. Summary of pattern positions

| Position                    | Form                                          |
|-----------------------------|-----------------------------------------------|
| `match` arm                 | `match v { pattern => expr, ... }`            |
| `if let`                    | `if let pattern = expr { ... } else { ... }`  |
| `let-else`                  | `let pattern = expr else { ... };`            |
| `let` destructuring         | `let pattern = expr;`                         |
| `for` loop                  | `for pattern in iter { ... }`                 |

Which patterns each position allows is defined in that position's specification. Some positions allow refutable patterns; others allow only irrefutable patterns.

## 3.11.6. Refutable and irrefutable

A refutable pattern is one whose match can fail: literals, `enum` variants, type checks, ranges, and so on. An irrefutable pattern always matches: identifiers, wildcards, and tuples or `struct`s whose destructured shape is guaranteed by the type.

`let` destructuring allows only irrefutable patterns, whereas `match`, `if let`, and `let-else` allow both.
