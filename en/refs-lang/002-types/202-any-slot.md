# 2.2. The `any` slot

`any` is the type name denoting a variable slot that can hold a value of any type (a contextual keyword, a keyword only in type position, [1.5.4](../001-lexical/105-keywords.md#154-contextual-keywords); declaring a type named `any` is an error). It is the core concept of rok's gradual typing.

> [!IMPORTANT]
> An `any` slot exists only where `any` is written. A variable whose type annotation is omitted becomes a static slot inferred from its initializer's type ([5.1.7](../005-modules/501-variable-declaration.md#517-type-annotations-and-type-inference)). rok's gradual typing is decided by a one-line rule: "static by default, dynamic only where `any` is written."

## 2.2.1. Contrast with static slots

First, how static slots and `any` slots differ:

```rok
let x: i32 = 42i32;     // x is i32 - cannot hold a value of another type
let y: any = 42i32;     // y is an any slot - any type allowed
let z = 42i32;          // z is i32 - static slot inferred from the initializer
```

In these three lines, `x` and `z` mean the same thing; `y` is the only `any` slot. When the type annotation is omitted, the compiler takes the initializer's type as the slot type. This lets REPL and notebook users type quickly without annotations while still getting static dispatch, and makes the places that incur dynamic dispatch cost visible in the declaration syntax.

## 2.2.2. Memory representation of the slot

A value in an `any` slot is represented as a tag paired with a payload.

```
+------+----------+
| tag  | payload  |
+------+----------+
```

The tag identifies which type the payload is; the payload is the value itself - small values (primitives etc.) are stored directly, large values as a reference to a heap object.

The exact bit width, alignment and encoding of tag and payload are defined in [8. Runtime ABI](../008-abi/README.md); this document guarantees only the shape (two parts: tag + payload).

## 2.2.3. Dynamic dispatch

When an operation is applied to a value in an `any` slot, the runtime inspects the tag and calls the appropriate trait method.

```rok
let v: any = 42i32;
let w = v + 1i32;           // at runtime, looks at v's tag and calls i32::add
```

This dispatch costs noticeably more than a static call, so performance-critical code should use static slots.

### Operators on `any`

When an operator is applied to an `any` operand, the rules are:

- The candidates are the primitive types that have that operator, and both operands must be of the same type.
- If one operand has a static type, that type fixes the candidate to one. If the static operand is not a primitive type, it is a compile error.
- If the operand types do not match at runtime, the program panics (exit code 101).
- If every candidate's result is `bool`, the result is `bool`; otherwise it is a boxed `any`.

```rok
let v: any = 42i32;
let w = v + 1i32;           // w: any - i32 + i32
let b = v == 1i32;          // b: bool
let p = v + Point {};       // ERROR - Point is not a primitive type
```

## 2.2.4. Reassignment: changing type is OK

An `any` slot can be freely reassigned with a value of a different type.

```rok
let s: any = 42i32;
s = "hello";        // OK - the slot is any
s = 3.14f64;        // OK
```

A static slot, by contrast, allows reassignment only with the same type.

```rok
let s: i32 = 42i32;
s = 10i32;          // OK
s = "hello";        // ERROR
```

## 2.2.5. Checking the slot's current type

The current type of the value in an `any` slot can be checked with the postfix operator `is`. The result of `e is T` is `bool`, and its precedence is the same as `as` ([3.2.3](../003-expressions/302-precedence-and-associativity.md#323-precedence-and-associativity-table)).

```rok
if v is i32 { ... }
```

The check is a single runtime tag comparison. On `any`, `is (A | B)` and the pattern `x: A | B` match both when the stored value is an `A` or a `B` and when a whole sum `A | B` value is stored. The binding is re-tagged as the sum `A | B`. If the operand's static type is neither `any` nor a sum type, the type check `e is T` is a compile-time constant. With a sum target it tests membership: on a value of a plain type `A`, `e is (A | B)` is `true`.

For a value whose static type is an `enum`, `e is Enum::Variant` checks the variant. It is a runtime variant comparison and is `true` for that variant regardless of the payload. It does not narrow; use `match` or `if let` when the payload is needed.

```rok
if shape is Shape::Circle { count += 1; }        // checks only whether it is Circle(_)
```

## 2.2.6. Narrowing

Inside a branch where an `is` check or pattern match succeeded, the compiler treats the slot type as narrowed to the checked type, so static dispatch becomes possible within that branch.

```rok
let v: any = ...;
if v is i32 {
    let w = v + 1i32;       // inside this branch v is i32 - static dispatch
}
```

The narrowing rules are:

- The narrowing condition is `x is T`, where `x` must be a local variable. It may be the whole condition or one term of an `a && b && ..` chain.
- `x` is `T` in the following `&&` terms, the `then` block of `if`, and the body of `while`.
- No narrowing happens under `||` or `!`, in the `else` block, or on field paths (`p.x is T`).
- A variant check `x is Enum::Variant` (2.2.5) does not narrow.
- When the operand's static type is a plain member `A` of the checked sum, `x is (A | B)` does not narrow; `x` keeps type `A`.
- When `x` is captured by `[ref x]` ([4.6.2](../004-functions/406-capture-semantics.md#462-reference-capture-the-ref-x-capture-list)), each read of `x` in the narrowed scope checks the shared cell again; if a closure stored a value that is not `T`, the read panics.
- Within the narrowed scope, `x` has type `T`. Assigning to `x` (or its fields / elements) is converted back and written to the original slot. Assigning a value of the declared type that is not `T` ends the narrowing for the rest of that scope: `x` has its declared type again, and normal flow typing continues. Assigning a value outside the declared type is still a type error.

```rok
let v: any = 1i32;
if v is i32 && v > 0i32 {      // v: i32 in the second term
    v = v + 1i32;              // written to the original any slot
    v = "x";                   // ends the narrowing: v is any again
}
if v is i32 || flag { ... }    // no narrowing
```

For narrowing with patterns, see the type-check patterns in [Patterns](../003-expressions/311-patterns.md).

## 2.2.7. Boundary of gradual typing

An `any` slot exists only where `any` is written. The meaning of an omitted type annotation is defined per position as follows.

### Variables without a type annotation

```rok
let n = 42i32;          // n: i32
let s = "hi";           // s: the string type chosen by the literal handler
let n2: any = 42i32;    // dynamic slot only when written explicitly
```

`let` follows local type inference. The initializer's type becomes the slot's type, and later reassignments are checked against it. The rule for when the type cannot be determined from the initializer is defined in [5.1.7](../005-modules/501-variable-declaration.md#517-type-annotations-and-type-inference).

### Function parameters without a type annotation

```rok
fn compute(x, y) { ... }
```

This is a generic function. Like variables, it follows the rule "omitted type = decided statically by the compiler". It is monomorphized according to the types at the call site: if a caller passes two `i32`, that call is compiled as a function taking two `i32`; if another caller passes two `f64`, another function is compiled.

To make a parameter a dynamic slot, `any` must be written explicitly.

```rok
fn compute(x: any, y: any) { ... }    // dynamic slots
```

### Struct fields

Struct fields must have their type written.

```rok
struct Point { x: f64; y: f64; }      // OK
struct Mixed { name; value; }          // ERROR - missing field types
```

To make a field dynamic, likewise use an explicit `any`.

```rok
struct Cell { tag: i32; payload: any; }   // OK
```

This requirement lets the compiler determine the struct's memory layout.

## 2.2.8. The slot kind is fixed by the declaration

Whether a variable's slot is dynamic or static is visible from the declaration syntax alone. Writing `: any` makes a dynamic slot; anything else (explicit or inferred type) makes a static slot. Redeclaring the same name in the same scope is not allowed, so once fixed, a slot's kind never changes within its scope.

```rok
let x = 1;              // static slot (inferred from initializer)
let x: any = 5i32;      // ERROR - x already exists in the same scope
```

In a new scope it acts as a separate binding; details are defined in [Variable declarations](../005-modules/501-variable-declaration.md).

## 2.2.9. Compiler / tooling friendliness

Code with `any` slots has weaker type checking. The compiler cannot know the slot's actual type in advance, so tools (LSP, static analysis) can only do their best within that limit. The choice between gradual and static is an explicit user decision, visible as whether `any` is written, and the specification fully supports both modes.

## 2.2.10. Constrained `any`: `any Trait`

Writing a trait after `any` makes a dynamic slot that holds only values of types implementing that trait.

```rok
let e: any Error = ParseError { input: s, pos: 0, reason: "empty" };
let shapes: List<any Shape> = [Circle { r: 1.0 }, Square { side: 2.0 }];
let item: any (Display + Hash) = 42i32;
```

There are three forms: bare `any`, `any Trait` with one trait, and `any (A + B)` with several traits in parentheses. Bare `any` is the same as `any ()` with no bounds (grammar: `any_type` in [Type expressions](201-type-expressions.md)). `any` binds only the single trait that follows it, so `any Error | none` is `(any Error) | none`.

Writing a bare trait name in type position is still forbidden ([Trait-as-type ban](212-trait-as-type-banned.md)). `any Trait` is allowed because the dynamic dispatch cost is made visible by the `any` keyword.

### Check on store

When a value is stored into an `any Tr` slot, the compiler checks at compile time that the value's static type implements `Tr`. `any (A + B)` requires implementing both `A` and `B`.

```rok
let a: any Display = 42i32;          // OK - i32: Display
let b: any Display = Opaque {};      // ERROR - Opaque does not implement Display

let v: any = 42i32;
let c: any Display = v;              // ERROR - no implicit any → any Display conversion
if v is i32 {
    let d: any Display = v;          // OK - after narrowing, the static type is i32
}
```

Conversions between `any` types are allowed only in the widening direction.

- `any A` → `any B` is allowed when every bound of `B` is covered by some bound of `A`. Covering means the same trait, or reachable by following supertraits (over several steps). `Tr<args>` covers `Tr`.
- `any X` → bare `any` is always allowed.
- Any other direction (narrowing conversions such as `any` → `any Tr`) is an error. This avoids implicit conversions that hide a runtime check; extract the concrete type with an `is` check or a type pattern first, then store it.

```rok
let e: any Error = ...;
let d: any Display = e;              // OK - Display is a supertrait of Error
let x: any = e;                      // OK
let e2: any Error = x;               // ERROR - narrowing conversion
```

### Use

On an `any Tr` value, only methods of `Tr` and its supertraits can be called, and calls are dispatched at runtime by tag. However, a method cannot be called if its signature, excluding `self`, mentions any of:

- `Self`
- a trait type parameter not fixed by an argument of the bound
- an associated type not fixed by an `X = ..` binding of the bound

A bound may omit arguments and bindings; omitted ones are treated as unconstrained. A call on `any Tr<A, X = B>` is checked against the trait declaration's signature with `A` and `B` substituted. So it can be called even when `Tr` is implemented only by generic impls. For example, `next` cannot be called on `any Iterator` (`Item` not fixed), but can on `any Iterator<Item = i32>`. The cost model is the same as `any` in 2.2.3. To use other methods or fields, narrow to the concrete type with an `is` check or a type pattern as in 2.2.5-2.2.6.

```rok
fn describe(e: any Error) -> str {
    let msg = f"{e}";                       // Display - supertrait of Error
    match e {
        p: ParseError => f"{msg} at {p.pos}",   // narrowed to the concrete type
        _ => msg,
    }
}
```

### Representation

An `any Tr` value has the same tag + payload representation as `any` (2.2.2). Dispatch tables for trait methods are built at link time from all types stored into `any` slots in the program (closed world, assuming single-package compilation). Types that implement the trait through generic or blanket impls are included. So no value carries its own vtable pointer.
