# 3.13. Loop expressions

Repeated execution is expressed in three forms: `loop`, `while`, and `for`. All three are expressions; this document defines the syntax and value evaluation rules of each.

```
loop_expr   ::= label? "loop" block
              | label? "while" expr block
              | label? "for" pattern "in" expr block
              | label? "for" "await" pattern "in" expr block
              | label? "for" "ref" identifier "in" expr block
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

The rationale is that when `while` ends normally (`cond` is `false`), there is no natural choice of value to return. It is not a form for explicit `break value` like `loop { break v; }`: a `break` with a value whose target is a `while` (or a `for`) is a compile error (E0059).

To return a value, use `loop { ... break v; ... }`.

### `while let`: combining with pattern matching

```rok
while let v: T = compute() {     // compute(): T | none
    consume(v);
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

When the value implements `IntoIterator`, the loop steps its `Iterator`. Otherwise, when it implements `TryIterator` ([stdlib 2.7.13](../../refs-standard-libraries/200-core/207-iter.md#2713-tryiterator---fallible-iteration)), the loop steps it directly; its `next` throws, so the loop must be written `try for` (or `try! for` / `try? for`). Any other value is a compile error.

```
try for pat in xs { body }
```

is semantically equivalent to:

```
let it = try xs;                                     // the iterable is covered
loop {
    match try <typeof(it) as TryIterator>::next(ref it) {   // each step is covered
        pat: T => body,                              // the body is not covered
        none   => break,
    }
}
```

The `try` of `try for` covers the iteration only: the iterable expression and every `next`. It does not cover the body; a throwing call in the body is marked with its own `try`, as everywhere else ([3.14.2](314-error-control-flow.md#coverage)). An error from a step goes where the loop's `try` sends it: out of the function for `try for`, to a panic for `try! for`, and to the end of the loop for `try? for`, whose value is then `none`. A `catch` after the loop attaches to the whole `try for` expression ([4.14.10](../004-functions/414-throws.md#try-for)). The same rules apply to `try for await` over a `TryStream` ([stdlib 2.8.11](../../refs-standard-libraries/200-core/208-stream.md#2811-trystream---fallible-streams)).

A `TryIterator` held in a variable, or in a field of one, is stepped in place: `it` above is that place, not a copy, as a stream in `for await` is polled in place ([4.13.5](../004-functions/413-stream.md#4135-the-for-await-form)).

### Result value

The result value of `for` is always unit (for the same reason as `while`), and a `break` with a value whose target is a `for` is a compile error.

### Pattern position

Only irrefutable patterns of [patterns](311-patterns.md) may be used (names, wildcards, tuple destructuring, etc.).

```rok
for x in 0..10 { ... }                    // simple name
for (i, v) in xs.enumerate() { ... }      // tuple destructuring
for Point { x, y } in points { ... }      // struct destructuring
```

Refutable patterns (e.g. matching an `enum` variant) are rejected in the pattern position of `for`, because handling a failed match would be ambiguous. When such a form is needed, use `while let` or a combination of `for` and `if let`.

### `for ref`: elements as places

```rok
struct Point { x: f64; y: f64; }

let ps: List<Point> = [Point { x: 1.0, y: 2.0 }, Point { x: 3.0, y: 4.0 }];
for ref p in ps {
    p.x = 0.0;                            // writes the element of ps
}
println(f"{ps[1].x}");                    // 0
```

A plain `for` binds a copy of each element: a struct is a value ([2.3.8](../002-types/203-struct.md#238-value-semantics)), so a field write on the binding changes nothing outside the loop, and is a compile error when nothing reads it. `for ref x in xs` binds `x` to each element as a place instead. Reading `x` reads the element; `x = v`, `x.f = v`, `x.f += v` and a call of a `ref self` method (`x.m()`, [4.7](../004-functions/407-ref-keyword.md)) write it. The pattern is a single name.

`xs` is a `List`, a `SliceMut` or an array. An array is written where it is kept, so it must be a variable or a field, not a temporary. Any other iterable is a compile error: its items are not places. The loop is equivalent to:

```
let v = xs;                    // a List: a SliceMut view of it; an array: xs itself
for i in 0..v.len() {          // the length is read once
    body                       // with x standing for v[i]
}
```

A list is walked through a `SliceMut` view, so the view's rules apply ([stdlib 2.9.7a](../../refs-standard-libraries/200-core/209-collections.md#297a-slicemutt---writable-view)): resizing `xs` in the body (`push`, `remove`, `clear`, ...) is a compile error, `E0138`, where the body shows the resize and a later use of `x`, and otherwise (a resize inside a called function) a panic at the next use of `x`. Collect the elements to add or remove and change `xs` after the loop.

A closure or `async` block cannot capture `x`; copy the element first (`let v = x;`). `if x is T` does not narrow `x`, and `for await` cannot be `for ref`.

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
