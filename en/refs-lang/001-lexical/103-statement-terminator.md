# 1.3. Statement terminator

> [!IMPORTANT]
> Rok statements end with a semicolon (`;`). A line break alone cannot end a statement.

```
stmt_terminator ::= ";"
```

## 1.3.1. Rules

The statement terminator follows a few simple rules. A statement ends with exactly one `;`, and several statements may share one line. A statement may also span several lines, because a line break is merely whitespace between tokens.

Schematically:

```
<stmt> ;
<stmt> ; <stmt> ;             // several on one line
<part of a stmt
 spanning lines> ;            // multiple lines. ends at the final ;
```

## 1.3.2. Why there is no automatic insertion at line breaks

Rok does not adopt automatic rules such as JavaScript's ASI (Automatic Semicolon Insertion), Go's automatic end-of-line semicolons, or Swift's line-based termination.

The reasons are as follows. First, automatic rules create ambiguity in rare cases; JavaScript's ASI pitfalls are a well-known example. Second, an explicit terminator lets tools such as formatters, LSPs and editors' jump-to-next-statement determine statement boundaries deterministically. Finally, writing several statements on one line is natural in places such as quick REPL input.

## 1.3.3. Empty statements

An empty statement consisting only of `;` is allowed and has no effect.

```rok
;;;
```

The code above is recognized as three empty statements. This form is accepted to give tool-generated code more freedom.

## 1.3.4. Block-like expression statements

A block-like expression ends in `}`: `if`, `match`, `loop`, `while`, `for`, or a block (`{ }`, `unsafe { }`, labeled block). When one appears in statement position, the trailing `;` may be omitted (the same rule as Rust). The value of that expression must then be unit; to discard a non-unit value, add `;`.

A call with a block-body [trailing closure](../004-functions/408-trailing-closure.md) (`retry(3) { .. }`, `xs.each |x| { .. }`) also ends a statement without `;` ([4.8.8](../004-functions/408-trailing-closure.md)) when nothing continues it after the closure's `}`. In that case the call's value is discarded as if `;` had been written. Unlike a keyword-led statement, such a call is parsed as an ordinary expression, so a binary operator after the `}` continues it (see below).

```rok
if ready { start(); }        // no ;
for x in xs { print(x); }
let n = if a { 1 } else { 2 };   // a let statement still ends with ;
```

If a binary operator follows the `}` of a block-like expression statement that starts with a keyword (`if`, `match`, etc.), it begins a new statement. Only member access such as `.method()` and postfix `?` continue the expression. A binary operator after a trailing closure call, even on the next line, is parsed as continuing that call: `run { .. }` followed by the line `-x;` is `run { .. } - x;`, which usually fails to type-check (``operator `-` cannot be applied to `()` and `i32` ``). Write `;` after the closure's `}` to end the statement there. When the call starts a statement and the operator starts the next line, the compiler warns (lint `trailing_closure_operator`): Rok 0.2.0 ends the statement at that `}`.

```rok
if c { a(); } else { b(); }
-x;                          // new statement (not the if statement minus x)
match v { ... }.len();       // . continues
run { a(); };                // ; ends the call statement here
-x;                          // new statement
```

A non-block-like expression appearing as a statement in the middle of a block requires `;`. Only the last expression of a block (the block's value) may appear without `;`.

The last statement before `}` may omit its `;`; omitting it is the same as writing it. This covers `let` (of any pattern), assignment, compound assignment, `return`, `break`, `continue` and `throw`. The block's value is what it is with the `;`: unit, or divergent for `return` / `break` / `continue` / `throw`. A last expression without `;` is still the block's value, and a statement in the middle of a block still needs `;`.

```rok
while i < n { i += 1 }           // same as { i += 1; }
if found { idx = k } else { idx = -1 }
let f = [ref total] |v: i32| { total += v };
let u = { let y = 1 };           // u is unit, as with { let y = 1; }
let s = { let y = 1; y + 1 };    // s is 2: the last expression is the value
```

The grammar is defined by `stmt` in [Block expressions](../003-expressions/309-block-expressions.md#391-form).

## 1.3.5. Adjacency with other tokens

The `;` token may be attached to the tokens before and after it without whitespace. For example, `<stmt>;<stmt>;` written without gaps still tokenizes correctly as two statements. Spacing for readability is the recommended style, but the specification itself does not enforce style.
