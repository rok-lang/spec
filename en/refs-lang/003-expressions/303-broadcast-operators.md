# 3.3. Broadcast operators

The dot-prefixed operators defined in [punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md) (`.+`, `.*`, `.==`, etc.) are distinct tokens for element-wise operations (broadcast). They work through the same mechanism as ordinary operators (desugaring to trait method calls) but map to a separate set of traits.

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
| `.==`        | `==` ↔ `PartialEq::eq` | `DotEq`    |
| `.!=`        | `!=` ↔ `PartialEq::ne` | `DotNe`    |
| `.<`         | `<` ↔ `PartialOrd::lt` | `DotLt`    |
| `.>`         | `>` ↔ `PartialOrd::gt` | `DotGt`    |
| `.<=`        | `<=` ↔ `PartialOrd::le` | `DotLe`   |
| `.>=`        | `>=` ↔ `PartialOrd::ge` | `DotGe`   |
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
f.(x)        →   x.iter().map(f)
g.(a, b)     →   a.iter().zip(b.iter()).map(|(x, y)| g(x, y))
```

Function and method broadcast has no trait of its own; it is sugar over iterator transformation. This is where it differs from the binary dot-prefixed operators.

## 3.3.4. Automatic loop fusion

An expression tree of dot-prefixed operators and function broadcasts is fused into one loop.

```
sin.(a) .+ cos.(b) .* 2.0f64
```

Fusion is guaranteed when the container operands are all core `List`, `Slice` or fixed arrays, or all `Broadcastable` ([stdlib 2.6.6](../../refs-standard-libraries/200-core/206-broadcast.md#266-loop-fusion)), and every other operand of a dot operator is a primitive or an element type: the result is built in one pass, with no intermediate collections. Each operand is evaluated once, in source order, before the loop.

- Over core collections the lengths are checked before the first element (fixed arrays at compile time). The result is a `List`, or an array when every container operand is an array.
- Over `Broadcastable` operands (`Array`, or a type of your own) each operand is viewed as an array, the views are stretched to one shape by the broadcast rule ([stdlib 2.6.8](../../refs-standard-libraries/200-core/206-broadcast.md#268-numpy-style-broadcasting-rules)) before the first element, and the leading operand (the first of the highest rank) builds the result through `Materialize`.
- A tree that mixes the two kinds is an error whose help converts the list (`Array::from_list(xs, [xs.len()])`).

Any other tree, one with an operand that is neither a container, a primitive nor an element type, is not fused: each operator calls its dot trait (3.3.1) and makes its own result.

The fused loop is an ordinary loop, vectorized like any other ([0.6.4](../000-meta/006-platform-targets.md#064-simd)).

## 3.3.5. Semantic difference from ordinary operators

> [!IMPORTANT]
> Dot-prefixed operators are separate traits because the same data can carry mathematically different meanings.

```
m * n           // matrix product (Mul impl, user-defined)
m .* n          // element-wise product (Hadamard product, DotMul)
```

To avoid the confusion caused by NumPy's `*` implicitly being element-wise, Rok always keeps the two meanings visually separate.

This separation applies directly the principle "syntax belongs to the language, meaning to the library" from [the boundary between the language and the standard libraries](../000-meta/004-language-vs-standard-libraries.md): two different meanings map to two different syntaxes.

## 3.3.6. User-Defined trait implementations

To give a type dot-prefixed operations, implement the **Dot** trait family the same way as for ordinary operators.

```rok
impl ops::DotAdd for MyVector {
    type Output = MyVector;
    fn dot_add(self, rhs: MyVector) -> MyVector { ... }
}
```

Dot-prefixed operations are supported through this same pattern, with no additional traits or macros. A type that implements `Broadcastable` instead gets fused expressions and in-place broadcast (3.3.4, 3.3.7); its dot trait impls then serve trait bounds and calls by name.

## 3.3.7. In-place broadcast

`target .= value` writes the elements of `value`, a dot expression, a container or a scalar, into `target` element by element; `target .op= value` for `op` in `+ - * / % **` is `target .= target .op value` with `target` evaluated once. They are statements, like `=`.

```rok
y .= a .* x .+ b;          // one loop, no array allocated
y .+= 1.0;                 // y .= y .+ 1.0
m[:, 0] .= 0.0;            // a view is a target: the column of m
xs .*= ys;                 // a List, lengths equal
```

- The target is evaluated first, then the operands in source order.
- A `List` or fixed array target takes operands of its length (checked before the first element, at compile time for fixed arrays); each element is stored through `IndexSet`. A `Slice` is a read-only view and is not a target.
- A `Broadcastable` target keeps its shape: the operands stretch to it (stdlib 2.6.8) and an operand of higher rank is a compile error; a result shape that differs panics (`broadcast: the result has shape [2, 3], which does not fit the destination of shape [1, 3]; the destination is not stretched`). The elements are written through the target's view, so a slice of an array (`a[1:]`) writes the array.
- An operand that shares the target's buffer at other positions (`v .= v[::-1]`) is read from a copy made before the loop; the same view (`v .+= v`) or a buffer apart from it is read in place. The result is as if every element were computed before any is stored.
- Other operators go through `.=`: `xs .= xs .%% 3`.
- Any other target is an error whose help writes the operation out (`x = x .+ ..;`).

## 3.3.8. Broadcast coalesce `.??`

`a .?? b` is `??` element-wise: each `none` element of `a` becomes the matching element of `b`, or `b` itself when it is a scalar. It fuses with the rest of the tree; it binds like `??` (level 1, right-associative). Its result's elements are the `??` of the elements (`List<f64 | none> .?? 0.0` is a `List<f64>`). Without a container operand it is an error whose help is `x ?? y`.

`none` and NaN stay apart: `.??` replaces `none` only, and a NaN element is a value that passes through (`[1.0, none, NaN] .?? 0.0` is `[1.0, 0.0, NaN]`). An `f64 | none` is not stored with NaN as its `none`: NaN is an ordinary result of float arithmetic, and conflating the two would lose a missing value or invent one. To treat NaN as missing, mask it: `a[f64::is_nan.(a)] = 0.0`.
