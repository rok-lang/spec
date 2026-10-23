# 3.3. Broadcast operators

The dot-prefixed operators defined in [punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md) (`.+`, `.*`, `.==`, etc.) are distinct tokens for element-wise operations (broadcast). They work through the same mechanism as ordinary operators - desugaring to trait method calls - but map to a separate set of traits.

## 3.3.1. Desugaring pattern

Dot-prefixed operators desugar as follows.

```
a .+ b      →   <typeof(a) as DotAdd<typeof(b)>>::dot_add(a, b)
a .* b      →   <typeof(a) as DotMul<typeof(b)>>::dot_mul(a, b)
a .== b     →   <typeof(a) as DotEq<typeof(b)>>::dot_eq(a, b)
```

This is the same form as the ordinary mapping in [operator dispatch](301-operator-dispatch.md); only the target trait differs. Dot-prefixed operators map to a separate trait family whose names start with **Dot**, as shown below.

| Dot-prefixed token | Undotted form     | Dot-prefixed trait |
|--------------------|-------------------|--------------------|
| `.+`         | `+` ↔ `Add`       | `DotAdd`        |
| `.-`         | `-` ↔ `Sub`       | `DotSub`        |
| `.*`         | `*` ↔ `Mul`       | `DotMul`        |
| `./`         | `/` ↔ `Div`       | `DotDiv`        |
| `.%`         | `%` ↔ `Rem`       | `DotRem`        |
| `.%%`        | `%%` ↔ `Modulo`   | `DotModulo`     |
| `.==`        | `==` ↔ `Eq`       | `DotEq`         |
| `.!=`        | `!=` ↔ `Ne`       | `DotNe`         |
| `.<`         | `<` ↔ (`Cmp::lt`) | `DotLt`         |
| `.>`         | `>`               | `DotGt`         |
| `.<=`        | `<=`              | `DotLe`         |
| `.>=`        | `>=`              | `DotGe`         |
| `.&&`        | `&&` (short-circ) | `DotLogicalAnd` |
| `.\|\|`        | `\|\|` (short-circ) | `DotLogicalOr`  |
| `.&`         | `&`               | `DotBitAnd`     |
| `.\|`         | `\|`               | `DotBitOr`      |
| `.^`         | `^`               | `DotBitXor`     |
| `.<<`        | `<<`              | `DotShl`        |
| `.>>`        | `>>`              | `DotShr`        |

The exact signatures of these traits and which types implement them are defined by the standard library. This document is responsible only for the fact that the language desugars dot-prefixed tokens to a separate trait family.

> [!IMPORTANT]
> **No short-circuiting for dot-prefixed logical operators**: unlike their undotted forms, `.&&` and `.||` do **not** short-circuit. Because a broadcast lowers to a fused loop, both operands are always evaluated; semantically it is an element-wise logical combination over a bool container. A function call with side effects placed in either operand always runs.

## 3.3.2. Precedence and associativity

Dot-prefixed operators have the same precedence and associativity as their undotted counterparts.

```
a .+ b .* c     →   a .+ (b .* c)              // .* binds tighter than .+ (same as * > +)
a .+ b .+ c     →   (a .+ b) .+ c              // left-associative
```

## 3.3.3. Function and method broadcast

Appending `.` to a function or method applies it to every element.

```rok
sin.(x)            // apply sin to each element of x
f64::sqrt.(arr)    // method form also allowed
g.(a, b)           // multiple arguments
```

This form returns a new collection mapping the result of the ordinary function or method call over every element. It desugars to a call to the standard library's `Iterator::map` (or an equivalent trait).

```
f.(x)        →   x.map(f)
g.(a, b)     →   a.zip(b).map(|(x, y)| g(x, y))
```

Function and method broadcast has no trait of its own; it is sugar over iterator transformation. This is where it differs from the binary dot-prefixed operators.

## 3.3.4. Automatic loop fusion

The compiler may fuse consecutive dot-prefixed expressions into a single kernel.

```
sin.(a) .+ cos.(b) .* 2.0f64
```

Where possible, this produces the result in a single pass without creating intermediate arrays.

Fusion is made possible by the MLIR stage of the [compilation pipeline](../000-meta/002-pipeline.md) recognizing an adjacent dot-prefixed chain as a single linalg operation. SIMD and GPU can be used through the same mechanism.

Whether fusion happens depends on compiler analysis and is not guaranteed by the specification. It is a practical promise that fusion happens in most cases.

## 3.3.5. Semantic difference from ordinary operators

> [!IMPORTANT]
> Dot-prefixed operators are separate traits because the same data can carry mathematically different meanings.

```
m * n           // matrix product (Mul impl, user-defined)
m .* n          // element-wise product (Hadamard product, DotMul)
```

To avoid the confusion caused by NumPy's `*` implicitly being element-wise, rok always keeps the two meanings visually separate.

This separation applies directly the principle "syntax belongs to the language, meaning to the library" from [the boundary between the language and the standard libraries](../000-meta/004-language-vs-standard-libraries.md): two different meanings map to two different syntaxes.

## 3.3.6. User-Defined trait implementations

To give a type dot-prefixed operations, implement the **Dot** trait family the same way as for ordinary operators.

```rok
impl DotAdd for MyVector {
    type Output = MyVector;
    fn dot_add(self, rhs: MyVector) -> MyVector { ... }
}
```

Dot-prefixed operations are supported through this same pattern, with no additional traits or macros.
