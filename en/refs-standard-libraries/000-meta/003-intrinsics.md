# 0.3. Intrinsic mechanism - the compiler ↔ standard library boundary

The body of a standard library trait method, such as `i32::add`, must ultimately be lowered to LLVM's `add` instruction. This chapter defines the path by which standard library method bodies are connected to the compiler backend (PrimOp / MIR / MLIR).

## 0.3.1. The problem - bootstrap cycle

The core of the problem is a bootstrap cycle between the standard library and the compiler. The following flow shows where it gets stuck.

```
user code:         let r = a + b;        (a, b: i32)
↓
compiler:          dispatch Add::add(a, b) call     (refs-lang 3.1)
↓
standard library:  impl Add for i32 { fn add(self, rhs) { ... } }
              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
              what goes in the "..." here?
              writing "self + rhs" dispatches Add::add again → infinite recursion
              raw `arith.addi` cannot be written either (not part of the user language surface)
↓
LLVM:        add i32 %a, %b
```

A path is therefore needed by which the standard library body leads straight to the compiler backend's `add` instruction without calling itself.

## 0.3.2. The solution - the `#[intrinsic("rok.OP.TYPE")]` attribute

The solution is to attach an attribute recognized by the compiler to the standard library method body. When the compiler encounters such a call, it converts it directly to PrimOp / MIR instead of lowering the body.

```rok
impl Add for i32 {
    type Output = i32;

    #[intrinsic("rok.add.i32")]
    fn add(self, rhs: i32) -> i32 {
        // an empty body is OK - the compiler uses the intrinsic path
        // even if a body exists, the compiler takes precedence
    }
}
```

The compiler proceeds as follows. When dispatching an `Add::add(a, b)` call, it finds `i32::add` in the method_table and then checks whether `#[intrinsic("rok.add.i32")]` is attached to that method. If so, it emits `PrimOp::Add { ty: i32 }` directly instead of lowering the body; this then passes through the MIR / MLIR / LLVM lowering stages and ultimately reaches the `add i32` instruction.

This mechanism absorbs the compiler's interception into trait dispatch.

## 0.3.3. Intrinsic naming convention

Intrinsic names have the form `rok.<OP>.<TYPE>`.

```
rok.add.i32        i32 + i32
rok.add.i64
rok.add.f32
rok.add.f64
rok.sub.i32
rok.mul.i32
rok.div.i32
rok.rem.i32
rok.and.i32        bitwise AND
rok.or.i32
rok.xor.i32
rok.shl.i32
rok.shr.i32
rok.eq.i32
rok.lt.i32
...
rok.neg.i32        unary
rok.not.bool       logical not (bool)
rok.not.i32        bitwise not (integer)
rok.cast.i32.i64   type cast
```

Each OP corresponds 1:1 to the compiler's `PrimOp` enum. The names themselves are influenced by LLVM instructions and Rust's intrinsic naming conventions.

## 0.3.4. Body fallback policy

A method with an intrinsic attached may still have a body. The following example illustrates this.

```rok
impl Add for BigInt {
    type Output = BigInt;

    // BigInt is not fixed-width, so no intrinsic - the body is authoritative
    fn add(self, rhs: BigInt) -> BigInt {
        // actual BigInt addition implementation
        ...
    }
}

impl Add for i32 {
    type Output = i32;

    #[intrinsic("rok.add.i32")]
    fn add(self, rhs: i32) -> i32 {
        // body - runs when taken out as an fn value or in environments that do not know the intrinsic
        self + rhs                    // primitive operators lower directly to machine operations
    }
}
```

The rules are summarized as follows. On the normal compilation path with intrinsics enabled, the body is ignored; in experimental backends or interpreted environments that do not recognize the intrinsic, the body is actually used. If the body is empty, it is treated as a stub `panic("intrinsic not available")`.

When the method is taken out as an `fn` value (`let f = i32::add;`), calling that value runs the body. The intrinsic applies only at direct call sites, so the body of an intrinsic method must also be a correct implementation with the same meaning.

## 0.3.5. Intrinsics in user code

`#[intrinsic(...)]` is a mechanism that operates only at the boundary where the standard library is coupled to the compiler, so user code cannot attach this attribute directly: on a function, a generic function or a method alike.

```rok
// user code - rejected.
#[intrinsic("rok.add.i32")]
fn my_add(a: i32, b: i32) -> i32 { ... }
// compile error: "the intrinsic attribute is restricted to the standard library"
```

This restriction prevents opening an escape hatch through which the compiler backend could be called arbitrarily from outside the standard library traits.

## 0.3.6. Comparison with other mechanisms

The intrinsic is only one of several magics that connect the compiler and the standard library; the following table summarizes the role of each.

| Mechanism             | Purpose                                                          | Spec                                                                   |
|-----------------------|------------------------------------------------------------------|------------------------------------------------------------------------|
| `#[intrinsic("...")]` | Standard library method ↔ compiler backend PrimOp                | This document                                                          |
| `#[lang(...)]`        | Standard library item ↔ compiler slot (none, iter, index, etc.)  | [refs-lang 6.4](../../refs-lang/006-meta-system/604-lang-magic.md)     |
| `#[infix(...)]`       | Token ↔ standard library trait method                            | [refs-lang 6.7](../../refs-lang/006-meta-system/607-operator-magic.md) |
| `#[literal(...)]`     | Literal form ↔ standard library handler                          | [refs-lang 6.5](../../refs-lang/006-meta-system/605-literal-magic.md)  |

The four magics are mutually orthogonal. `#[intrinsic]` handles coupling with the compiler backend, while all the others handle coupling with the language syntax.
