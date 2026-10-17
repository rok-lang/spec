# 3.7. Range expressions

A range expression uses the `..` or `..=` token ([punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md)) to produce a range value with a start and an end.

```
range_expr      ::= expr ".." expr        // half-open
                  | expr "..=" expr       // inclusive
                  | ".." expr             // from start
                  | "..=" expr            // from start, inclusive
                  | expr ".."             // unbounded end
                  | ".."                  // full
```

## 3.7.1. Six forms

A range expression can be written in six forms.

```rok
0..10           // [0, 1, ..., 9]      exclusive end
0..=10          // [0, 1, ..., 10]     inclusive end
..10            // same as 0..10       start omitted (prefix)
..=10           // same as 0..=10      start omitted, inclusive end
10..            // [10, 11, ...]       end omitted, infinite
..              // full                both omitted, mainly for slicing
```

`..` produces a half-open interval excluding the end; `..=` includes the end. An omitted start means 0 or a default depending on the data; an omitted end means infinity or the end of the data.

## 3.7.2. Desugaring

Each form desugars to a standard library type as follows.

```
a .. b          →   Range::new(a, b)
a ..= b         →   RangeInclusive::new(a, b)
.. b            →   RangeTo::new(b)
..= b           →   RangeToInclusive::new(b)
a ..            →   RangeFrom::new(a)
..              →   RangeFull
```

The exact standard library traits and types used by this conversion are defined in the library specification. This document is responsible only for the fact that the language recognizes the six forms as distinct expressions and desugars them to instances of standard library types.

## 3.7.3. Precedence

In the levels of [precedence and associativity](302-precedence-and-associativity.md), the range operators (`..`, `..=`) are prec=0 non-associative: lower than every binary operator including null coalescing (`??`, prec=1), and higher only than assignment.

```
0..n + 1            →   0 .. (n + 1)        // + binds tighter than ..
a ?? 0..3           →   (a ?? 0) .. 3       // ?? binds tighter than ..
a..b..c             // ERROR: range operators are non-associative
```

Because they bind most weakly among the binary operators, arithmetic expressions in the start and end read naturally.

### `..x` in call arguments

In call-argument position, `..x` is not a range but a spread ([4.2.6](../004-functions/402-parameters.md#426-spread-args)). So `f(..5)` is a call spreading `5`; to pass a `RangeTo`, wrap it in parentheses (`f((..5))`). `..=x` is not a spread, so it is a range even in argument position (`f(..=5)` passes a `RangeToInclusive`).

## 3.7.4. Uses

### Iteration

Range values are mainly used in loops.

```rok
for i in 0..10 { ... }
```

Here the `for` loop of [loop expressions](313-loop-expressions.md) takes the range value and iterates over it. `Range` and `RangeInclusive` iterate automatically when they implement the `Iterator` trait ([`trait`](../002-types/208-trait.md)).

### Distinct from slicing

Slicing and range expressions are different notations.

```
arr[1:3]            // a slice of [indexing and slicing](306-indexing-and-slicing.md), uses a colon (`:`)
0..3                // a range value, uses two dots (`..`)
```

The two are different tokens with different meanings. Slicing is an internal form of an index expression; a range is a value (an instance of a Range type).

### Pattern matching

Range patterns ([patterns](311-patterns.md)) use the same tokens. Their meaning is defined there.

```
match x {
    1..=5 => ...,
    _ => ...,
}
```

## 3.7.5. Step is a method

A step is not part of the notation. It is provided instead by the standard library method `.step_by(n)`.

```rok
(0..100).step_by(2)         // [0, 2, 4, ..., 98]
```

This is consistent with rok's general direction: keep language syntax simple and provide extras as methods. The `[::2]` of slicing is a separate notation, introduced because a step is natural there.

## 3.7.6. Reverse direction

To iterate in reverse, use the `.rev()` method.

```rok
(0..10).rev()
```

The range notation itself has no negative step.

## 3.7.7. Empty ranges

If the start is greater than the end, the range is empty.

```rok
5..3            // empty range, valid expression, iterates 0 times
5..=3           // empty range
```

This is not a compile error but a valid value that iterates zero times.

## 3.7.8. Using infinite ranges

A `for` loop can run over an open range such as `0..`, ending with `break` inside the loop. If the value exceeds the type's maximum, it is treated like arithmetic overflow (panic if checks are enabled).

```rok
for i in 0.. {
    if condition(i) { break; }
}
```

Infinite iteration itself is equivalent to [`loop { ... }`](313-loop-expressions.md); the only difference is convenience.
