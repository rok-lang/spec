# 4.5. Closures

A closure is a function value without a name that appears in expression position. It can capture variables from the outer environment, which makes it convenient for small functions that depend on surrounding state.

## 4.5.1. Form

```
closure_expr   ::= capture_list? closure_params return_type? closure_body
closure_params ::= "||"
                 | "|" closure_param_list "|"
closure_param_list ::= closure_param ("," closure_param)* ","?
closure_param ::= identifier (":" type_expr)?
closure_body   ::= expr | block
capture_list   ::= "[" capture_item ("," capture_item)* "]"
```

The zero-parameter form `||` is a single token (see [punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md)) and is not split into two `|`.

```rok
|params| body
```

Parameters are enclosed by left and right `|` tokens ([punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md)). The body is an expression or a block.

### Single-Expression body

```rok
|x| x + 1
|x: i32| x * 2
|a, b| a + b
```

A single expression follows `|...|` directly, and its value is the closure's return value.

### Block body

```rok
|x: i32| -> i32 {
    let y = x * 2;
    y + 1
}
```

With a block body, the return type may be given with `->` (if omitted, it is inferred).

### Zero-Parameter form

```rok
|| body
|| { body }
```

With no parameters, use `||` (two consecutive pipes). For tokenization, `||` is defined as one token in the fixed token table of [punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md).

## 4.5.2. Parameter types

Parameter types may be written or omitted.

```rok
|x: i32| x * 2                    // explicit
|x| x * 2                          // inferred
```

Inference is determined by the context in which the closure is used, for example the function type of the argument position it is passed to or the type of a variable declaration. Without context, subsequent calls determine the parameter type (`x: i32` in `let f = |x| x * 2; f(4);`, [2.10.2](../002-types/210-generics.md#type-argument-inference)).

```rok
let f: fn(i32) -> i32 = |x| x * 2;   // x: i32 inferred from context
list.map(|x| x * 2);                  // inferred from map's signature
```

## 4.5.3. Inferred or explicit return type

```rok
|x| x * 2                          // inferred
|x: i32| -> i32 { x * 2 }          // explicit — only with a block body
```

An explicit return type (`-> type`) is used only with a block body. A single-expression body always relies on inference.

## 4.5.4. Captures

A closure can capture variables from outer scopes.

```rok
let n = 10;
let f = |x| x + n;                // captures n
f(5);                              // 15
```

Capture semantics (value capture by default, with capture lists like `[ref x]`) are defined in detail in [Capture Semantics](406-capture-semantics.md).

## 4.5.5. Closures vs functions

| Aspect                   | `fn` named function | `\| ... \|` closure     |
|--------------------------|---------------------|-------------------------|
| Name                     | yes                 | no                      |
| Definition position      | module or `impl`    | any expression position |
| Captures outer variables | no                  | yes                     |
| Call                     | `name(...)`         | `value(...)`            |
| Type                     | `fn(...) -> ...`    | same                    |

Both are expressed with the same function type and are interface-compatible, so either fits the same position.

## 4.5.6. Using closure values

```rok
// assigned to a variable
let f = |x| x * 2;

// function argument
list.map(|x| x * 2);

// function return
fn make_adder(n: i32) -> fn(i32) -> i32 {
    |x| x + n
}

// collection
let handlers: List<fn(i32) -> i32> = [|x| x + 1, |x| x - 1, |x| x * 2];
```

## 4.5.7. Calls

```rok
let f = |x| x * 2;
let r = f(5);                     // same form as an ordinary function call
```

A closure is called as `f(args)`. On a call, the captured environment and the arguments are used together to evaluate the body.

## 4.5.8. Immediate invocation: IIFE

A closure may be called immediately after it is defined.

```rok
let r = (|x: i32| -> i32 { x * 2 })(5);     // 10
```

This is uncommon, but it is one way to express a local scope with immediate evaluation. In most cases an ordinary block from [block expressions](../003-expressions/309-block-expressions.md) is more natural.

## 4.5.9. Readability: single expression vs block

```rok
list.map(|x| x * 2)               // single expression — short and clear
list.map(|x| { let y = x * 2; y + 1 })   // block — multi-step processing
```

Choose the form that suits the body's length. The specification fully supports both forms.

## 4.5.10. Trailing closures

When the last argument of a function call is a closure, it may be moved outside the parentheses for readability. Details are defined in [Trailing Closures](408-trailing-closure.md).

## 4.5.11. Special variants: `async` and `yield` modifiers

Adding the `async` or `yield` modifier to a closure creates an asynchronous or generator closure.

```rok
async || { ... }                  // closure returning a Future
yield || { ... }                  // closure returning an Iterator
async yield || { ... }            // closure returning a Stream
```

The detailed rules are defined in [Generators](411-generators.md), [Async](412-async.md) and [Streams](413-stream.md).

## 4.5.12. Zero-Argument closure shorthand: `{ body }`

In certain contexts, `{ body }` is recognized as a zero-parameter closure shorthand. This shorthand is often used together with trailing closures.

```rok
thread::spawn { heavy_work() }    // == thread::spawn(|| { heavy_work() })
```

The exact rules of this shorthand are defined in [Trailing Closures](408-trailing-closure.md).
