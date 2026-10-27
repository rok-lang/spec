# 3. Expressions overview

This chapter defines the forms of expressions and statements, and the consequences of making every control structure an expression. The individual kinds of expressions and their semantics are covered in the documents that follow.

## Contents

- [3.1. Operator Dispatch](301-operator-dispatch.md)
- [3.2. Precedence and Associativity](302-precedence-and-associativity.md)
- [3.3. Broadcast Operators](303-broadcast-operators.md)
- [3.4. Assignment and Compound Assignment](304-assignment.md)
- [3.5. Member Access and Method Calls](305-member-access-and-method-call.md)
- [3.6. Indexing and Slicing](306-indexing-and-slicing.md)
- [3.7. Range Expressions](307-range-expressions.md)
- [3.8. Optional Chaining and Null Coalescing](308-optional-chaining-and-coalescing.md)
- [3.9. Block Expressions](309-block-expressions.md)
- [3.10. `if` Expression](310-if-expression.md)
- [3.11. Patterns](311-patterns.md)
- [3.12. `match` Expression](312-match-expression.md)
- [3.13. Loop Expressions](313-loop-expressions.md)
- [3.14. Error Control Flow](314-error-control-flow.md)
- [3.15. Destructuring Assignment](315-destructuring.md)
- [3.16. `let-else`](316-let-else.md)
- [3.17. `using` Expression](317-using-expression.md)

## Expressions and statements

An expression is a construct that produces a value: evaluating it yields a value of some type. A **statement** is a unit of code that produces no value and ends with the `;` [statement terminator](../001-lexical/103-statement-terminator.md).

Rok relates the two by the following rule.

```
<statement> = <expression> ';'        // a statement that discards the expression's result
```

A statement is an expression with a terminator attached. Almost every code construct is an expression that produces a value; discarding that value makes it a statement.

## Control structures are expressions

Every control structure (`if`, `match`, `loop`, `while`, `for`, `try`) is also an expression. The following example shows control structures producing values directly.

```rok
let a = if cond { x } else { y };

let b = match v {
    1 => "one",
    2 => "two",
    _ => "other",
};

let c = loop {
    let v = compute();
    if v.converged { break v; }
};

let d = { let a = 1; let b = 2; a + b };
```

This decision has two consequences. First, a control structure can appear in any expression position: a function argument, a variable initializer, part of another expression. Second, no separate ternary form is needed: there is no `cond ? x : y` token; `if cond { x } else { y }` fills that role.

The exact rules for the value each control structure returns are defined in the corresponding specification document.

## Branch result types

The result type of an expression and the type of the slot receiving its value are determined separately.

### Result type of an expression

If every branch of a control structure produces a value of the same type, that type is the result type of the expression. If branches produce values of different types, the receiving side must supply a type (the slot's explicit type, a parameter type, or a return type), and each branch must be compatible with it. If the receiving side has no type, it is a compile error. Automatic sum type inference and automatic promotion to a common supertype (for example implicitly promoting `i32` to `i64` or `f64`) are both disallowed.

```
if cond { 1i32 } else { 2i32 }          // result type of the expression: i32
if cond { 1i32 } else { 2.0f64 }        // the receiving side must specify i32 | f64 or similar
```

### Sum types are explicit; no promotion

The two cases are rejected for different reasons. Sum types are not inferred automatically so that a mistake in one branch cannot silently create a sum type that then spreads through all subsequent code. When a sum type is wanted, state it on the receiving side, as in `let d: i32 | str = if ...`. Automatic supertype promotion, by contrast, converts one type into another (`i32` → `f64`) and may lose precision or representation, so it is disallowed even when the receiving side has a type.

This distinction is consistent with the explicitness principle of [the boundary between the language and the standard libraries](../000-meta/004-language-vs-standard-libraries.md). Any needed conversion must be written explicitly with one of the three forms of [conversion](../002-types/214-cast-and-conversion.md): `as`, `.into()`, `.try_into()`.

### Receiving an expression result in a slot

When a variable slot receives the result of an expression, the slot's type follows the rules of [variable declaration](../005-modules/501-variable-declaration.md#517-type-annotations-and-type-inference). If a type is given, it becomes the receiving type; if omitted, the slot's type is inferred from the expression's result type.

```rok
// no type annotation → slot is inferred from the expression's result type
let a = if cond { 1i32 } else { 2i32 };          // a: i32
let b = if cond { 1i32 } else { 2.0f64 };        // ERROR: branch type mismatch; type annotation required

// any slot → accepts any combination of branches
let g: any = if cond { 1i32 } else { 2.0f64 };   // OK

// sum type slot → the expression's result must be within the slot's member set
let c: i32 | f64 = if cond { 1i32 } else { 2.0f64 };   // OK
let d: i32 | f64 = if cond { 1i32 } else { "hi" };     // ERROR: str is not in the slot

// single concrete type slot → the expression's result must be exactly that type
let e: i32 = if cond { 1i32 } else { 2i32 };           // OK
let f: i32 = if cond { 1i32 } else { 2.0f64 };         // ERROR: f64 branch does not fit i32
```

### Slot compatibility table

Compatibility between the expression's result type and the receiving slot is summarized below.

| Receiving slot                      | Expression result type                      | Allowed                    |
|-------------------------------------|---------------------------------------------|----------------------------|
| `any` (explicit `: any`)            | anything                                    | always                     |
| sum type `T1 \| T2 \| ...`            | result members are within the slot's set    | allowed                    |
| single concrete type `T`            | result is exactly `T`                       | allowed                    |
| single concrete type `T`            | result has members other than `T`           | rejected                   |
| type annotation omitted             | all branches have the same type             | allowed (inferred as it)   |
| type annotation omitted             | branch types differ                         | rejected                   |

### Gradual and static

On top of these compatibility rules, the user can choose among three modes. In gradual mode (`let x: any = ...`), the slot is `any` and freely accepts any combination of branches; dispatch is done by runtime tag checks, which suits fast exploration. In static sum type mode (`let x: T1 | T2 = ...`), the slot is an explicit sum type and branch results must fall within its member set; type narrowing and pattern-match exhaustiveness checking are available, which suits stability and performance. In static single-type mode (`let x: T = ...`, or `let x = ...` with the type omitted), the slot is a single type and every branch must be that type, enabling maximum performance and full static analysis.

The three modes can be mixed freely within the same code; the user chooses per variable.

## Distinguishing expressions and statements

Most lines of code take one of these two forms.

```rok
<declaration>;           // a statement: a definition of let / fn / struct / ...
<control-or-expr>;       // a statement that discards the expression's result
```

Expressions ending in `}`, such as `if` / `match` / loops / blocks, may omit `;` in statement position ([1.3.4](../001-lexical/103-statement-terminator.md#134-block-like-expression-statements)).

The last expression in a block is an exception.

```rok
{
    let a = 1;
    let b = 2;
    a + b           // no terminator: the value of this block
}
```

Only the last expression of a block may be left without a terminator, in which case it becomes the value of the whole block. The detailed rules are defined in [block expressions](309-block-expressions.md).

## Side effects

An expression may cause side effects during evaluation, such as I/O, variable reassignment, or external calls. Evaluation order is determined by operator precedence and function-call evaluation order, defined in later documents.

Where side effects create a dependency within a single expression, the dependency follows an explicit order: left to right, depth first. The compiler does not reorder it.
