# 3.9. Block expressions

A group of statements and expressions enclosed in `{ ... }` is the most basic expression unit in rok. This document defines the lexical form of blocks and their value evaluation rules.

## 3.9.1. Form

The grammar of a block is:

```
block       ::= label? "{" stmt* tail_expr? "}"
label       ::= "'" identifier ":"
stmt        ::= expr ";"
              | block_like_expr ";"?
              | let_stmt
              | item_decl
block_like_expr ::= block | if_expr | match_expr | loop_expr
              | while_expr | for_expr | unsafe_block
tail_expr   ::= expr            // no trailing ";"
```

Schematically:

```rok
{
    statement1;
    statement2;
    ...
    last_expr     // no terminator, becomes the block's value
}
```

The body may contain zero or more statements and zero or one final expression. If the final expression is written without a terminator, its value is the block's value; if it is written with a terminator (`expr;`), the block has the unit value.

```rok
let a = {
    let x = 1;
    let y = 2;
    x + y               // block value = 3
};

let b = {
    let x = 1;
    x + 2;              // terminator present → result discarded
                        // block value = unit
};

let c = {
    let x = 1;
    return x;           // return exits the function itself (block value never reached)
};
```

## 3.9.2. Uses

A block may appear anywhere an expression may.

```rok
let v = { ... };                  // variable initializer
fn_call({ ... }, other);          // function argument
if { ... > 0 } { ... }            // condition (rare but allowed)
```

The bodies of other control structures (`if`, `match`, `loop`, function bodies) are also blocks.

## 3.9.3. Scope

A block creates a new scope. A variable declared with `let` inside a block is visible only within that block and is dropped when the block is exited. Outer variables remain visible inside the block unless shadowed by an inner `let` of the same name.

An inner `let` of the same name is a separate binding and does not affect the outer binding. The exact rules are defined in [variable declaration](../005-modules/501-variable-declaration.md).

## 3.9.4. Empty block

```rok
{}
```

An empty block has the unit value.

## 3.9.5. The last expression and terminators in a block

The following two inputs evaluate differently.

```rok
let a = { x };          // a is the value of x
let b = { x; };         // b is unit
```

An expression ending without a terminator becomes the block's value, whereas an expression ending with a terminator has its result discarded and the block's value is unit. This distinction is consistent with the rules of the [statement terminator](../001-lexical/103-statement-terminator.md).

- A block-like statement without `;` in the middle of a block must have type `()`.
- `;;` is an empty statement.
- A postfix operation (`.`, `?`, call, index) after a block-like expression continues the expression (`match v { .. }.len();`).

### Equivalent forms: last expression vs. explicit `return` / `break`

A final expression without a terminator is equivalent to the explicit exit statement appropriate to its context.

```rok
// function body — last expression == return value
fn f() -> i32 { 42 }
fn g() -> i32 { return 42; }      // same meaning

// labeled block — last expression == break 'label value
let x = 'b: { 42 };
let y = 'b: { break 'b 42; };     // same meaning

// loop — a loop without break is divergent. There is no last-expression position.
```

The last-expression position is the surface where a value arrives. In a function body it means the function's return; in a labeled block it means a break to that label. Whichever form is written, the compiler produces the same result.

## 3.9.6. Relation to function bodies

A function body is also a block, and the value of its last expression is the function's return value. The `return` keyword is therefore not mandatory.

```rok
fn add(x: i32, y: i32) -> i32 {
    x + y               // no terminator, returned automatically
}

fn other(x: i32) -> i32 {
    let r = x + 1;
    return r;           // explicit return also allowed, same meaning as above
}

fn early(x: i32) -> i32 {
    if x < 0 { return 0; }   // explicit return for early exit
    x * 2                    // last expression of normal flow, returned automatically
}
```

All three functions work. Which form to use is a matter of style; the specification fully supports both.

### Recommended use

Using the last expression (without `return`) suits the single return point of normal flow. An explicit `return` suits early exits, returns from the middle of a function, and marking for readability.

Function bodies, the `return` keyword, and return type rules are defined in detail in [4. Functions](../004-functions/README.md).

## 3.9.7. Type of a block

The type of a block is the type of its last expression (unit if there is no last expression).

```
{ 1i32 }                // i32
{ "hi" }                 // str
{ 1i32; }                // unit (expression ends with a terminator → result discarded)
{ }                      // unit (empty block)
```

This type determination combines with the branch result type rules of the [expressions overview](README.md). If several block expressions in the same position produce different types, the receiving side must have an explicit type.

## 3.9.8. Divergent blocks

Some blocks produce no value because of `return`, `break`, `continue`, `throw`, `loop { /* no break */ }`, and the like.

```rok
let x = {
    return 0;       // exits the function
};
```

Such blocks are classified as divergent (never complete). The type of a divergent block is compatible with any type and passes the branch result type check.

```rok
let x: i32 = if cond {
    42i32
} else {
    return -1i32;       // divergent, does not affect branch agreement
};
```

The exact rules for detecting divergence are defined in the function and control-flow specifications.

## 3.9.9. Evaluation order of a block

The statements and the last expression in a block are evaluated in the order written (top to bottom), and side effects follow that order.

If a `return`, `break`, exception, or the like occurs during evaluation, evaluation of the block stops at that point.

## 3.9.10. Labeled blocks: `'label: { ... }`

A labeled block may be used anywhere a block expression may appear — a `let` initializer, a function body, an `if` / `else` branch, a `match` arm body, the trailing expression of another block, and so on. There are no additional position-specific restrictions; the label only designates the exit point of that block.

Prefixing a block with a label of the form `'name:` allows `break 'name value;` inside the block to exit early while setting the block's value.

```rok
let result = 'b: {
    if early_check_fails() { break 'b -1; }
    if shortcut_taken()    { break 'b  0; }
    let x = compute();
    let y = refine(x);
    x + y                    // last expression of normal flow — the value of 'b
};
```

The label token has the same form (`'identifier:`) as in [3.13.4 labels](313-loop-expressions.md#3134-labels-controlling-nested-loops). `break 'name value` exits that labeled block with the given value. If the `'name` of `continue 'name` refers to a **block** label, it is a compile error, because `continue` applies only to loop labels. Also, `break` / `continue` without a label cannot target a block (they target only the innermost **loop**).

Instead of writing a small helper function to exit early, a labeled block cleanly expresses leaving the block early with a value.

## 3.9.11. `unsafe { ... }` blocks

Placing the `unsafe` keyword before a block makes it a block that enters an unsafe scope. As with an ordinary block, the value of the last expression is the block's value, and operations outside the guarantees of safe abstractions (raw pointer dereference, calling an `unsafe fn`, etc.) may be performed.

```rok
let n: usize = unsafe { strlen(cs.as_ptr()) };
```

The detailed meaning and responsibilities are defined in [4.15 unsafe](../004-functions/415-unsafe.md).
