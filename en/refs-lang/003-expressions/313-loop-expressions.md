# 3.13. Loop expressions

Repeated execution is expressed in three forms: `loop`, `while`, and `for`. All three are expressions; this document defines the syntax and value evaluation rules of each.

```
loop_expr   ::= label? "loop" block
              | label? "while" expr block
              | label? "for" pattern "in" expr block
              | label? "for" "await" pattern "in" expr block
label       ::= "'" identifier ":"
break_stmt  ::= "break" label_target? expr? ";"
continue_stmt ::= "continue" label_target? ";"
label_target ::= "'" identifier
```

> [!NOTE]
> `in` is a contextual keyword, a keyword only in the `for pat in` position ([1.5.4](../001-lexical/105-keywords.md#154-contextual-keywords)).

The meaning and uses of `for await` are defined in [streams](../004-functions/413-stream.md). It receives items one at a time from an asynchronous stream and runs the body for each.

The meaning of labels (`'name:`) is defined in [3.13.4 labels](#3134-labels-controlling-nested-loops). Labeled **blocks** (`'b: { ... }`) are defined in [block expressions](309-block-expressions.md).

## 3.13.1. `loop`: infinite repetition

```rok
loop {
    ...
}
```

The `loop` keyword is followed by a block body, which repeats indefinitely. It can be exited only by `break`, `return`, or `throw`.

### Returning a value with `break value`

```rok
let v = loop {
    let x = compute();
    if x.converged { break x; }
};
```

If an expression follows `break`, its value is the result of the `loop` expression. If the loop exits without a value, as in `break;`, the result of the `loop` is unit. All `break` values of one `loop` (or labeled block) must have one type, and if the receiving side has a type annotation, they must have that type. Mixing `break;` and `break v` is a compile error.

### Result type

A `loop` in which `break value` appears at least once has the type of that value as its result type. A `loop` with only `break;` has result type unit. A `loop` with no `break` at all is divergent and never completes.

## 3.13.2. `while`: condition-based repetition

```rok
while cond {
    ...
}
```

The `while` keyword is followed by a bool condition and a block body. At the start of each iteration, `cond` is evaluated: if `true`, the body runs; if `false`, the loop ends.

### Result value

The result value of `while` is always unit.

```rok
let x = while cond { ... };     // x: unit
```

The rationale is that when `while` ends normally (`cond` is `false`), there is no natural choice of value to return. It is not a form for explicit `break value` like `loop { break v; }`.

To return a value, use `loop { ... break v; ... }`.

### `while let`: combining with pattern matching

```rok
while let v: T = compute() {     // compute(): T | none
    use(v);
}
```

```
while_let ::= "while" "let" pattern (":" type_expr)? "=" expr block
```

On each iteration the expression is evaluated and matched by the same rules as `if let` ([3.10.5](310-if-expression.md#3105-if-let-combining-with-pattern-matching)). On a match the body runs; otherwise the loop ends. The pattern's bindings are visible in the body.

Every pattern form of [patterns](311-patterns.md) may be used; typically it matches `enum` or sum type variants.

## 3.13.3. `for`: iterator-based repetition

```rok
for pattern in iter_expr {
    ...
}
```

The `for` keyword is followed by a pattern, `in`, an iteration expression, and a block body. The iteration expression must be an iterator or an iterable value. A struct literal cannot appear at the top level of the iteration expression or of a `while` condition ([3.10.3](310-if-expression.md#struct-literals-in-condition-position)). On each iteration the next element is matched against the pattern and bound in the body.

### Desugaring of the iteration

```
for pat in xs { body }
```

is semantically equivalent to:

```
let it = <typeof(xs) as IntoIterator>::into_iter(xs);
loop {
    match <typeof(it) as Iterator>::next(ref it) {     // returns: T | none
        pat: T => body,
        none   => break,
    }
}
```

(The exact trait and method names are defined by the standard library.)

A `for` over an iterator whose `next` throws (`TryGenIter<T, E>`, etc.) is a throwing expression, so it must be covered by `try`, as in `try for pat in it { ... }`. The same applies to `for await` ([4.14.10](../004-functions/414-throws.md#try-for)).

### Result value

The result value of `for` is always unit (for the same reason as `while`).

### Pattern position

Only irrefutable patterns of [patterns](311-patterns.md) may be used (names, wildcards, tuple destructuring, etc.).

```rok
for x in 0..10 { ... }                    // simple name
for (i, v) in xs.enumerate() { ... }      // tuple destructuring
for Point { x, y } in points { ... }      // struct destructuring
```

Refutable patterns (e.g. matching an `enum` variant) are rejected in the pattern position of `for`, because handling a failed match would be ambiguous. When such a form is needed, use `while let` or a combination of `for` and `if let`.

## 3.13.4. Labels: controlling nested loops

```rok
'outer: loop {
    for x in xs {
        if cond { break 'outer; }
        if other { continue 'outer; }
    }
}
```

A loop can be prefixed with a label of the form `'name:`. `break 'name` and `continue 'name` control the loop with that label directly.

### Label token

A label consists of a single quote, an identifier, and a colon (`'outer:`). The single quote is the same token that starts a [character literal](../001-lexical/108-character-literals.md), but in a position combined with a colon it is recognized as a label. The exact tokenization rule is a supplementary definition of the lexical specification; this document specifies only the language-level form.

### `break label value`

A `loop` label can be combined with a value.

```rok
'outer: loop {
    for x in xs {
        if found(x) { break 'outer x; }
    }
}
```

`break 'outer x` exits the `loop` labeled outer, carrying the value x.

## 3.13.5. Default behavior of `break` and `continue`

Without a label, they refer to the innermost loop.

```rok
loop {
    for x in xs {
        if cond { break; }       // exits the for
    }
    // continues here
}
```

## 3.13.6. Evaluation order

Each loop is evaluated according to the meaning defined above. The expressions and statements of the body are evaluated in the order of [block expressions](309-block-expressions.md), and side effects follow that order.

Loop termination is checked as follows. `loop` performs no check and ends only by an explicit `break`. `while` evaluates `cond` at the start of each iteration. `for` calls `Iterator::next` at the start of each iteration.

## 3.13.7. Divergent loops

A loop that cannot terminate by `break`, `return`, `throw`, or the like (e.g. a `loop { ... }` with no `break` inside and no flow exiting outward) is classified as divergent. The type of such a loop expression is compatible with any type.
