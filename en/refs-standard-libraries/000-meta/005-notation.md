# 0.5. Notation

The notational conventions of this specification follow [refs-lang 0.8 Notation](../../refs-lang/000-meta/008-notation.md) as is. This chapter defines the additional conventions that apply only to the standard library specification.

## 0.5.1. Trait signature notation

Each trait spec begins with a signature block that gathers all methods, associated types and supertraits in one place.

```rok
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}
```

This form is identical to how the standard library is actually written, so users can use the same format in their own code.

## 0.5.2. Method signature tables

A trait defining several methods is also summarized in table form, as follows.

| Method | Signature                      | Meaning             |
|--------|--------------------------------|---------------------|
| `add`  | `fn(self, rhs: Rhs) -> Output` | Arithmetic addition |
| `sub`  | `fn(self, rhs: Rhs) -> Output` | Arithmetic subtraction |

## 0.5.3. Trait implementation matrix

At the end of each trait spec, a table states which standard library types implement the trait. The columns are only the type and the binding mechanism; implementation file paths are left outside the spec.

```markdown
## X.Y.Z. Trait implementation matrix

| Type  | binding                          | Notes                          |
|-------|----------------------------------|--------------------------------|
| `i32` | `#[intrinsic("rok.add.i32")]`    | Direct to compiler PrimOp      |
| `i64` | `#[intrinsic("rok.add.i64")]`    |                                |
| `f32` | `#[intrinsic("rok.add.f32")]`    |                                |
| `f64` | `#[intrinsic("rok.add.f64")]`    |                                |
| `int` | (no intrinsic, auto-promote)    | Defined by the standard library |
```

## 0.5.4. Semantics

The meaning of each method is written out in prose, covering not only form and examples but also limits and error cases.

```markdown
### `Add::add(self, rhs)`

Arithmetic addition. Returns the sum of the two values.

- `i32 + i32`: 2's complement wrap (debug=panic, release=wrap)
- `f64 + f64`: IEEE 754 addition
- `int + Int`: unbounded precision; promotes to BigInt when the result exceeds the i128 range

Overflow handling follows the fixed-width arithmetic rules of [2.1 Numeric types](../200-core/201-numbers.md).
Saturating / checked / wrapping variants are exposed as separate methods (`saturating_add`, etc.).
```

## 0.5.5. Examples

Each trait and method is accompanied by one or more working examples. Examples assume the implicit `use rok::core::**;` is in effect, so they must be code that runs as is without any imports.

```rok
let a: i32 = 5;
let b: i32 = 3;
let c = a + b;        //→ 8 (i32)

let x: f64 = 1.5;
let y: f64 = 2.5;
let z = x + y;        //→ 4.0 (f64)
```

## 0.5.6. Cross-reference

References to other documents use one of two forms depending on location. Within this specification, write `[2.7 iter](../200-core/207-iter.md)` or refer informally to "refs-standard-libraries 2.7"; references to refs-lang follow the form `[refs-lang 6.4](../../refs-lang/006-meta-system/604-lang-magic.md)`.

## 0.5.7. Separation of specification and implementation

This document defines only the **specification**. The actual implementations of the compiler and the standard libraries are maintained in separate repositories; in case of conflict with this specification, this specification takes precedence.
