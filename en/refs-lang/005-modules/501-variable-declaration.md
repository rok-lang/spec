# 5.1. Variable declarations

A new variable is declared with the `let` keyword ([keywords](../001-lexical/105-keywords.md)). This document defines its forms and the rule that distinguishes reassignment from a new declaration.

## 5.1.1. New declarations: `let`

```
let_stmt    ::= "let" let_lhs (":" type_expr)? "=" expr ";"
let_lhs     ::= identifier
              | irrefutable_pattern    // tuple / struct destructure (see 5.1.11)
```

The most common form is a single identifier (`let x = ...`). An irrefutable pattern (5.1.11) on the left-hand side makes it a [destructuring assignment](../003-expressions/315-destructuring.md). The `let-else` form ([let-else](../003-expressions/316-let-else.md)) is a separate syntax.

```rok
let x = 42i32;
let y: i32 = 10i32;
let z;                            // ERROR: no initializer
let z: i32;                        // ERROR: no initializer
```

The basic form is `let`, a name, an optional `: type`, then `=` and an initializer. The initializer is mandatory; uninitialized variables do not exist. With a type annotation the variable is a static slot of that type; without one it is a static slot of the type inferred from the initializer (5.1.7). An [`any` slot](../002-types/202-any-slot.md) arises only when `: any` is written explicitly.

This rules out reading a slot that has not been filled (the kind of problem Rust's borrow checker blocks). When the initial value cannot be determined up front, use the [sum type](../002-types/206-sum-types.md) `T | none` to make absence explicit.

```rok
let cache: Data | none = none;     // explicitly "not yet"
cache = compute();                  // filled later
```

## 5.1.2. Reassignment: without `let`

To assign a new value to an already declared variable, omit `let`.

```rok
let counter: i32 = 0i32;
counter = counter + 1i32;         // OK: reassignment
counter = counter + 1i32;         // OK: reassignment again
```

Assigning without `let` to a name that has never been declared is a compile error.

```rok
let counter: i32 = 0i32;
coutner = counter + 1i32;         // ERROR (E0057): `coutner` is not declared
                                   //        help: did you mean `counter`?
```

This blocks the pitfall common in Julia and Python where a typo silently creates a new variable.

## 5.1.3. Deciding between new declaration and reassignment

| Form              | Meaning                                                   |
|-------------------|-----------------------------------------------------------|
| `let name = ...;` | New declaration. A new binding in that scope.             |
| `name = ...;`     | Reassignment. Writes again to the existing binding's slot. |

Explicitness takes priority: a typo is a compile error.

## 5.1.4. Redeclaration in the same scope

A `let` may declare a name that is already bound in the same scope. This makes a new binding with its own slot; from the end of the statement on, the name refers to the new binding (shadowing).

```rok
let line = "  42  ";
let line = line.trim();                // the initializer reads the old `line`
let n: i32 = try! line.parse();
let n = n * 2;                         // a new slot, of the same type here
let x = "abc";
let x = x.len();                       // the type may differ: str, then usize
let (a, b) = (1, 2);
let (a, b) = (b, a);                   // patterns follow the same rule
```

- The initializer is checked before the new binding exists, so the name in it refers to the old binding. The new binding starts after the statement.
- The new binding has its own type, written or inferred (5.1.7); it need not match the old one. Each slot keeps the type it was declared with: the old slot is not changed, and a reassignment without `let` (5.1.2) writes the newest binding.
- The old binding is not reassigned, dropped or closed. It lives to the end of its scope as before (5.1.10), out of reach by name. Closures, views and guards made from it keep referring to it.
- A value that must be closed ([stdlib 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop---cleanup-on-release)) is not closed by a redeclaration: close or move it before redeclaring its name. Moving it into the new value is the intended use. Leaving it open is the usual compile error (E0135), whose help says to close or move it before the redeclaration.

```rok
let f = try fs::open(path);
let f = io::BufReader::new(f);         // OK: the reader owns the file now
try for l in f.lines() { println(l); }
try f.close();

let g = try fs::open(path);
let g = 1;                             // ERROR (E0135): the first `g` is never closed
                                        //        help: close or move it before redeclaring it
```

A view of a stream made from the old binding ends where the old value moves, also when it moves into the new binding (E0140, [stdlib 2.23.1](../../refs-standard-libraries/200-core/223-io.md#2231-read)).

To change the value of a binding rather than make a new one, assign without `let` (5.1.2). The REPL and Jupyter follow the same rule at the top level of a cell ([9.7.3](../009-tools/907-jit-cells.md#redeclaration-of-the-same-name)).

## 5.1.5. Scope: per block

The range in which a variable is visible is determined per block of a [block expression](../003-expressions/309-block-expressions.md).

```rok
let y = 1;
if cond {
    y = 2;                         // reassigns the outer y (same slot)
    let z = 2;                     // declares block-local z
}
// z is not accessible here
```

A `let` inside a block is a new local binding belonging to that block, so the variable is not visible outside the block. An inner `let` of the same name hides an outer variable only within that block.

## 5.1.6. Nested scopes: separate bindings

When a variable of the same name is declared anew in an inner block, it is a separate binding.

```rok
let a = 1;
{
    let a: i32 = 42i32;            // block-local a, a different slot from the outer a
    a = 10i32;                     // reassigns the inner a (outer a unaffected)
}
// here a = 1 (outer a unchanged)
```

An inner block's `let` creates an entirely new binding. Even if the same name exists outside, the two are independent slots.

The design keeps both the meaning that an inner variable disappears when the block ends and the intent to protect outer variables.

### Difference from redeclaration in the same scope

Both make a new binding that hides the old name, and neither changes the old slot. They differ in where the old name comes back: a binding in an inner block ends with the block, and after it the outer name is visible again. A redeclaration in the same scope (5.1.4) lasts to the end of that scope, so the old binding is not reachable by name again.

```rok
let a = 1;
{
    let a = "inner";               // hides the outer a inside the block
}
// a = 1 again
let a = a + 1;                     // same scope: a = 2 from here to the end
```

## 5.1.7. Type annotations and type inference

```rok
let a = 42i32;                     // a: i32 (inferred)
let b: i32 = 42i32;                // b: i32 (annotated)
let c: any = 42i32;                // c: any (annotated)
let d = if cond { 1i32 } else { 2i32 };   // d: i32 (inferred)
```

With a type annotation the variable is a static slot of that type; without one it is a static slot of the type inferred from the initializer expression (local type inference). Later reassignments are checked against that type. An [`any` slot](../002-types/202-any-slot.md) arises only when `: any` is written explicitly.

```rok
let n = 1i32;
n = 2i32;                          // OK
n = "hi";                          // ERROR: n is i32
```

If the initializer does not fully determine the type, inference continues from later uses within the variable's scope (uses that pass the value to a position whose type is determined) ([2.10.2](../002-types/210-generics.md#type-argument-inference)). If the type still cannot be determined, it is a compile error, and the type must be annotated.

```rok
let x = none;                      // ERROR: type cannot be determined
let x: Data | none = none;         // OK

let xs = [];                       // ERROR if there are no later uses: element type cannot be determined
let xs: List<i32> = [];            // OK

let y = return;                    // ERROR: initializer only diverges
```

No type is inferred from an `if` / `match` expression whose branches have different types. If a sum type is needed, annotate the slot.

```rok
let e = if cond { 1i32 } else { "hi" };          // ERROR: branch type mismatch
let e: i32 | str = if cond { 1i32 } else { "hi" };   // OK
```

For the resulting types of inference and the branch result rules, see the [expressions overview](../003-expressions/README.md).

## 5.1.8. Multiple declarations (none)

```
let a = 1, b = 2;                  // not adopted
```

The C-style form that declares several variables at once with commas is not provided. Write one per line, or use tuple destructuring from [destructuring assignment](../003-expressions/315-destructuring.md).

```rok
let (a, b) = (1, 2);               // destructuring assignment: OK
let a = 1; let b = 2;              // two statements: OK
```

## 5.1.9. Where `let` may appear

`let` is a statement (see the conversion rules in the [expressions overview](../003-expressions/README.md)).

`let` is not allowed at module level (module-level value slots must be declared with [`const`](502-const.md) or [`static`](503-static.md)). The exception is the entry file `main.rok`, whose top-level statements, `let` included, form the implicit `main` ([4.16.2](../004-functions/416-entry-point.md#4162-meaning-of-top-level-statements-implicit-main)); in any other module they are an error (E0036). It may be used in function bodies and blocks. The bodies of control structures such as `if` / `match` / `loop` are themselves blocks, so it may be used there too. `let` cannot appear directly in expression position, because `let` is a statement, not an expression.

## 5.1.10. Lifetime of variables declared with `let`

A variable lives from the point of declaration until the end of the scope that introduced it. A redeclaration of its name in the same scope (5.1.4) does not end it.

A variable captured by a closure through a `ref` capture ([capture semantics](../004-functions/406-capture-semantics.md)) is additionally kept alive while that closure object is alive. It is managed by the [GC](../000-meta/007-glossary.md#gc-garbage-collector), so the user never frees it manually.

## 5.1.11. The left-hand side of `let` and [destructuring assignment](../003-expressions/315-destructuring.md)

The left-hand side of `let` may be an irrefutable pattern from among the [patterns](../003-expressions/311-patterns.md). This is [destructuring assignment](../003-expressions/315-destructuring.md).

```rok
let (x, y) = compute();
let Point { x, y } = origin;
```

The semantics of destructuring are defined in [destructuring assignment](../003-expressions/315-destructuring.md). Each subpattern is bound to its variable, and all variables are introduced together into a single scope.
