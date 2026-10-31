# 4.11. Generators (`yield`)

The `yield` keyword ([keywords](../001-lexical/105-keywords.md)) is used in three forms that define generators, functions that produce values one at a time.

## 4.11.1. Three forms

Generators have this grammar:

```
generator_fn    ::= "yield" fn_decl
generator_block ::= "yield" block
generator_closure ::= "yield" closure_expr
yield_stmt      ::= "yield" expr? ";"
```

```rok
yield fn name(args) -> R { ... yield expr; ... }     // named generator function
yield { ... yield expr; ... }                         // inline block (value)
yield || { ... yield expr; ... }                      // closure modifier
```

In all three forms, the result is a value of the library type `GenIter<T>` (implementing `Iterator<Item = T>`), or `TryGenIter<T, E>` with `throws (E)`. This rule is defined in [4.1.11](401-function-declaration.md#result-types-of-coroutine-functions).

## 4.11.2. The `yield` statement: emitting values

The `yield` statement may be used with or without a value.

```rok
yield expr;
yield;
```

`yield expr;` emits the value of `expr` as the output of the current iteration step and suspends evaluation of the function or block. Evaluation resumes on the next iteration. `yield;` advances a step without a value (unit).

## 4.11.3. Named generator functions

A named generator function is written as follows.

```rok
yield fn count_up_to(n: i32) -> i32 {
    let i = 0;
    while i < n {
        yield i;
        i = i + 1;
    }
}
```

`yield fn` is a function declaration marked as a generator; the body emits values with `yield expr;`. The `-> R` position gives the type of the yielded values, and the call result is `GenIter<R>`. `-> R` must be written; the element type is not inferred from the body. It is called like an ordinary function, and the result is an iterator object.

```rok
let it = count_up_to(5i32);       // it: `GenIter<i32>`
for x in it { print(f"{x} "); }   // 0 1 2 3 4
```

## 4.11.4. Inline `yield` blocks: values

To create an iterator immediately in expression position, use an inline `yield` block.

```rok
let it = yield {
    yield 1i32;
    yield 2i32;
    yield 3i32;
};
```

This form appears in expression position and immediately produces a `GenIter<T>`. The body is a block from [block expressions](../003-expressions/309-block-expressions.md). It is the form for writing short generators anonymously inline.

## 4.11.5. `yield` closures

A `yield` closure produces a callable generator.

```rok
let make_iter = yield || {
    yield 1i32;
    yield 2i32;
    yield 3i32;
};

let it = make_iter();             // each call creates a new Iterator
```

`yield ||` creates a callable generator closure; each call creates a new iterator. Outer captures follow the rules of [Capture Semantics](406-capture-semantics.md).

## 4.11.6. Correspondence of the three forms

The three generator forms correspond one-to-one to the three forms (named, block, closure) of [function declarations](401-function-declaration.md), [closures](405-closure.md) and [block expressions](../003-expressions/309-block-expressions.md).

```
plain        fn f() { ... }       ↔  || { ... }       ↔  { ... } (block)
yield form   yield fn f() { ... } ↔  yield || { ... } ↔  yield { ... }
```

That is, `yield` attaches as a modifier to the same function, block and closure forms and turns them into generators.

## 4.11.7. Four positions of the `yield` token

The `yield` token can appear in four positions.

```rok
yield fn name(...) { ... }        // declaration modifier
yield { ... }                      // block marker
yield || { ... }                   // closure modifier
yield expr;                         // statement
```

These four positions are clearly distinguished by context; the parser decides the form from the next token.

| Token after `yield` | Meaning                                                                    |
|---------------------|----------------------------------------------------------------------------|
| `fn`                | declaration modifier                                                       |
| `{`                 | [block](../000-meta/007-glossary.md#block) marker (expression position) |
| `\|\|` or `\|`      | closure modifier                                                           |
| expression-start token | `yield` statement                                                       |

## 4.11.8. Interaction with the Iterator trait

The result of a generator function is a value implementing the standard library's `Iterator<Item = T>` trait, so it can be used directly in a `for` loop from [loops](../003-expressions/313-loop-expressions.md).

```rok
yield fn naturals() -> i32 {
    let i = 0;
    loop {
        yield i;
        i = i + 1;
    }
}

for n in naturals() {
    if n > 100 { break; }
    print(n);
}
```

The exact definition and methods (`map`, `filter`, `take`, `collect`, etc.) of the `Iterator` trait are defined in the standard library.

## 4.11.9. Infinite generators

`naturals()` above is an infinite generator that produces values endlessly. Such generators can be terminated with `break` in a `for` loop, the library's `take(n)` method and the like.

## 4.11.10. Generator termination

When the body completes normally, the iterator also terminates, and the next `next()` call returns `none`.

```rok
yield fn finite() -> i32 {
    yield 1i32;
    yield 2i32;
    yield 3i32;
    // function completes normally → iterator terminates
}
```

It can also be terminated explicitly with `return;`.

```rok
yield fn early_stop(stop: i32) -> i32 {
    let i = 0;
    loop {
        if i >= stop { return; }
        yield i;
        i = i + 1;
    }
}
```

A generator that a `for` loop leaves early is closed. This happens when the loop body leaves the loop by `break` (also a labelled `break` or `continue` of an outer loop), `return`, a thrown error, `try` propagation, a task's cancellation, or a `yield` of an enclosing generator. Closing resumes the generator at the `yield` it waits at, which then leaves the body as if by `return`: the `finally` blocks and the `using` drops around that `yield` run, innermost first. A `yield` reached while closing ends the generator the same way; its value is dropped. A generator closed before its first `next()` runs none of its body. After closing, `next()` returns `none`.

```rok
yield fn lines() -> str {
    using f = open_file() {          // dropped when the loop below breaks
        for l in f.lines() { yield l; }
    }
}

for l in lines() {
    if l == "" { break; }            // closes the generator: `f` is dropped
}
```

Only a generator the loop itself holds is closed: the `for` expression is the generator value (a call, a `yield` block). A generator in a variable, a field or an element is not closed, so iteration may continue after the loop:

```rok
let g = lines();
for header in g { if header == "" { break; } }   // g stays open
for body in g { print(body); }                    // goes on after the blank line
```

A generator that is never closed and never finished keeps its cleanups pending until the GC reclaims it; they do not run then.

## 4.11.11. Generator state

A generator keeps internal state between calls. The body's local variables, the iteration position and the coroutine frame are preserved until the next `next()` call.

This state works on the coroutine infrastructure of [7. Runtime Model](../007-runtime/README.md) and operates as a compiler-generated (stackless) state machine. The detailed implementation mechanism is defined in the runtime specification.

## 4.11.12. Parameters and return of `yield` functions

Since a `yield fn` is a generator, `-> T` denotes the type of the values yielded in the body, and the call result is `GenIter<T>`.

```rok
yield fn f() -> i32 { yield 1i32; }
// call result: `GenIter<i32>`
```

Parameters are used the same way as in ordinary functions.

## 4.11.13. Generators as methods

`yield fn` may also be used inside the `impl` block of a [method](409-method.md) to define generator methods.

```rok
impl<T> List<T> {
    yield fn iter(self) -> T {
        for x in self.elements { yield x; }
    }
}
```

Calls and behavior are the same as for ordinary instance methods.
