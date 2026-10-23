# 4.10. Nested functions

A function may be defined inside the body of another function.

## 4.10.1. Form

```rok
fn outer(x: i32) -> i32 {
    fn inner(y: i32) -> i32 {
        y + 1
    }

    inner(x) + inner(x * 2)
}
```

Place the `fn` keyword inside the outer function's body. The rules for definition and use are the same as for module-level functions in [function declarations](401-function-declaration.md).

## 4.10.2. Scope

A nested function is visible only within the block scope in which it is defined.

```rok
fn outer() {
    fn helper() { ... }
    helper();                     // OK
}

helper();                          // ERROR - not visible outside outer
```

Once outside that block, the nested function's name is no longer visible.

## 4.10.3. Capturing outer variables (none)

Nested functions do not capture variables of the outer function.

```rok
fn outer(n: i32) {
    fn inner() {
        let x = n;                // ERROR - n is not visible
    }
    inner();
}
```

This makes clear that a nested function is a clean function unit. When a stateful function depending on outer variables is needed, use a [closure](405-closure.md).

```rok
fn outer(n: i32) {
    let inner = |x| x + n;        // OK - closures can capture
    inner(5);
}
```

This yields the following separation:

- A nested function cannot see outer variables; apart from its limited scope, it is the same as a module-level function.
- A closure can capture (see [Capture Semantics](406-capture-semantics.md)).

A nested function sees the items and `use` declarations of its block and the blocks around it, `const`s included ([3.9.1](../003-expressions/309-block-expressions.md#391-form)); only their locals, parameters and generic parameters are out of reach.

## 4.10.4. Behavior on call

Calling a nested function follows the same rules as calling a module-level function. It is called directly through the function address, so there is no capture and no overhead.

## 4.10.5. Definition order

Within the same block, a nested function may be called before or after its definition. Both forms are allowed:

```rok
fn outer() {
    helper();                     // call before definition
    fn helper() { ... }
}

fn outer() {
    fn helper() { ... }
    helper();                     // call after definition
}
```

Hoisting makes it visible throughout the block, so it may be called from any position.

## 4.10.6. Choosing between nested functions and closures

| Property                         | Nested function        | Closure                  |
|----------------------------------|------------------------|--------------------------|
| Definition form                  | `fn name(...) { ... }` | `\| ... \| body`         |
| Captures outer variables         | no                     | yes                      |
| Name                             | yes                    | no                       |
| Passed as argument to functions  | yes (by name)          | yes                      |
| Stateless helper                 | suitable               | possible but excessive   |
| Stateful callback                | unsuitable             | suitable                 |

Choose a nested function when no outer variables are needed and a name aids readability; use a closure when outer variables are needed or anonymity suffices.

## 4.10.7. Visibility of nested functions

A nested function is visible only inside the outer function's body, so visibility keywords such as `pub` have no meaning.

```rok
fn outer() {
    pub fn helper() { ... }       // ERROR - pub is meaningless on a nested function
}
```

Visibility applies only to module-level items. The same holds for every item declared in a block ([3.9.1](../003-expressions/309-block-expressions.md#391-form)).

## 4.10.8. Nested function and outer variable with the same name

```rok
fn outer() {
    let helper = 42;
    fn helper() { ... }            // ERROR - variable and function with the same name
}
```

A variable and a function with the same name in the same block conflict, which is a compile error.

## 4.10.9. Multi-Level nesting

Functions may be nested to any depth.

```rok
fn outer() {
    fn middle() {
        fn inner() { ... }
        inner();
    }
    middle();
}
```

A nested function is visible in its whole block (hoisted). It sees the nested functions, other items and `use` declarations of its own block and of every enclosing block, but no locals of any of them. A `use` in a block hides an outer nested function of the same name.

## 4.10.10. Where nested functions are useful

```rok
fn parse(input: str) -> AST {
    fn parse_expr(s: str) -> Expr { ... }
    fn parse_stmt(s: str) -> Stmt { ... }

    let result = AST::new();
    while !input.is_empty() {
        result.add(parse_stmt(input));
    }
    result
}
```

They are natural as internal helpers of a large function. For auxiliary logic that need not be exposed to other modules and is tied to the outer function's context, nested functions fit well.
