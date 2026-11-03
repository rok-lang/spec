# 2.6. Broadcast - element-wise operators

The broadcast traits in `rok::core::ops` are the trait authority for Julia-style dot operators. Dot operators (`.+` / `.*` / `.==` / `f.(x)`) make element-wise meaning visually distinct in the syntax itself, which rules out confusion with a user-defined `*` (e.g. matrix multiply).

This chapter defines the 20 broadcast traits together with the desugaring rules for function broadcast.

The traits are declared in `rok::core::ops`, their one path ([0.1.6](../000-meta/001-tier-model.md#016-one-path-per-item)); there is no `broadcast` submodule. `Dot`, the `⋅` dot-product trait of [2.3.10](203-ops.md#2310-unicode-math-operator-traits), sits beside them. `Dot` is not one of the 20 broadcast traits: `a ⋅ b` reduces two operands to one value (`fn dot(self, rhs: Rhs) -> Output`) instead of working element-wise, and the standard library implements it for no type.

## 2.6.1. Motivation - why separate dot operators

In NumPy and MATLAB, the **silent ambiguity** of `*` meaning element-wise on arrays but multiplication on matrices has long been a source of bugs (the same concern motivated the introduction of `@` in PEP 465).

Rok's policy, by contrast, is clear.

```rok
m * n       // user-defined Mul: matrix multiplication, dot product, etc.
m .* n      // element-wise (Hadamard product), always broadcast
```

The dot prefix (`.`) acts as a visual signal stating element-wise intent, and the compiler enforces it. In any container (List, Vec, Matrix, etc.), `.+` consistently dispatches to `DotAdd::dot_add`.

## 2.6.2. 20 traits - operator mapping

The full set is arithmetic (7) + comparison (6) + logical (2) + bitwise (3) + shift (2) = **20** traits.

> [!NOTE]
> This chapter defines all 20 traits: 7 arithmetic, 6 comparison, 2 logical, 3 bitwise and 2 shift. The dot comparisons (`.==`/`.!=`/`.<`/`.>`/`.<=`/`.>=`) are given only as **brief signatures** in [2.5.8 Broadcast comparison](205-cmp.md#258-broadcast-comparison-traits);
> this chapter is the **full authority**.

### Arithmetic (7)

| Trait       | Operator | Precedence | Meaning                       |
|-------------|----------|------------|-------------------------------|
| `DotAdd`    | `.+`     | 9 (left)   | element-wise addition         |
| `DotSub`    | `.-`     | 9 (left)   | element-wise subtraction      |
| `DotMul`    | `.*`     | 10 (left)  | element-wise product (Hadamard) |
| `DotDiv`    | `./`     | 10 (left)  | element-wise division         |
| `DotRem`    | `.%`     | 10 (left)  | element-wise remainder (Rem)  |
| `DotModulo` | `.%%`    | 10 (left)  | element-wise floored modulo   |
| `DotPow`    | `.**`    | 13 (right) | element-wise power (`**`, 2.3.1) |

### Comparison (6)

| Trait   | Operator | Precedence | Meaning                             |
|---------|----------|------------|-------------------------------------|
| `DotEq` | `.==`    | 4 (none)   | element-wise == → `Container<bool>` |
| `DotNe` | `.!=`    | 4 (none)   | element-wise !=                     |
| `DotLt` | `.<`     | 4 (none)   | element-wise <                      |
| `DotGt` | `.>`     | 4 (none)   | element-wise >                      |
| `DotLe` | `.<=`    | 4 (none)   | element-wise <=                     |
| `DotGe` | `.>=`    | 4 (none)   | element-wise >=                     |

### Logical (2)

| Trait           | Operator | Precedence | Meaning                                  |
|-----------------|----------|------------|------------------------------------------|
| `DotLogicalAnd` | `.&&`    | 3 (left)   | element-wise logical AND (no short-circuit) |
| `DotLogicalOr`  | `.\|\|`    | 2 (left)   | element-wise logical OR (no short-circuit)  |

> [!IMPORTANT]
> **`.&&` / `.||` do not short-circuit**. Unlike ordinary `&&` / `||`, broadcast is lowered to a fused loop, so both operands are always evaluated. Semantically it is the element-wise logical combination of bool containers. A function call with side effects placed in either operand always runs.

### Bitwise (3)

| Trait       | Operator | Precedence | Meaning        |
|-------------|----------|------------|----------------|
| `DotBitAnd` | `.&`     | 7 (left)   | element-wise & |
| `DotBitOr`  | `.\|`     | 5 (left)   | element-wise \| |
| `DotBitXor` | `.^`     | 6 (left)   | element-wise ^ |

### Shift (2)

| Trait    | Operator | Precedence | Meaning         |
|----------|----------|------------|-----------------|
| `DotShl` | `.<<`    | 8 (left)   | element-wise << |
| `DotShr` | `.>>`    | 8 (left)   | element-wise >> |

Precedence and associativity mirror the ordinary traits ([2.3 ops](203-ops.md), [2.5 cmp](205-cmp.md)).

## 2.6.3. Trait signatures - uniform form

Every dot trait follows the same uniform form.

```rok
trait DotAdd<Rhs = Self> {
    type Output;
    #[infix(".+")]
    fn dot_add(self, rhs: Rhs) -> Output;
}

trait DotMul<Rhs = Self> {
    type Output;
    #[infix(".*")]
    fn dot_mul(self, rhs: Rhs) -> Output;
}

trait DotModulo<Rhs = Self> {
    type Output;
    #[infix(".%%")]
    fn dot_modulo(self, rhs: Rhs) -> Output;
}

trait DotEq<Rhs = Self> {
    type Output;
    #[infix(".==")]
    fn dot_eq(self, rhs: Rhs) -> Output;
}

trait DotLogicalAnd<Rhs = Self> {
    type Output;
    #[infix(".&&")]
    fn dot_logical_and(self, rhs: Rhs) -> Output;
}
```

Arithmetic dot traits are fully symmetric with `Add`/`Sub`/`Mul`/`Div`/`Rem`/`Modulo`: an `Output` associated type plus an infix attribute. The six comparison dot traits (`DotEq`, `DotNe`, `DotLt`, `DotGt`, `DotLe`, `DotGe`) have the same form, each with its own `Output`, usually `Container<bool>`. Logical dot traits (`DotLogicalAnd` / `DotLogicalOr`) typically take `Container<bool>` on both sides and return `Container<bool>`; with no short-circuit, both containers are evaluated to the end.

### Function broadcast - no trait

Function broadcast is written as follows.

```rok
f.(x)              // function broadcast
f.(a, b)           // multiple arguments
```

Function broadcast has no trait of its own; it desugars as follows.

```rok
f.(x)        →  x.into_iter().map(f).collect()
g.(a, b)     →  a.into_iter().zip(b.into_iter()).map(|(x, y)| g(x, y)).collect()
```

It dispatches directly to `map`/`zip` of [2.7 iter](207-iter.md); the broadcast module defines no separate trait. The difference is that the function itself is a callable, not a dispatch target.

## 2.6.4. Semantics - element-wise

### Arithmetic dot - two containers of the same shape

Arithmetic dot operations on two containers of the same shape behave as follows.

```rok
let a: List<i32> = [1, 2, 3];
let b: List<i32> = [10, 20, 30];

let c = a .+ b;          //→ [11, 22, 33]
let d = a .* b;          //→ [10, 40, 90]
let e = b .- a;          //→ [9, 18, 27]
let f = b ./ a;          //→ [10, 10, 10]
```

A length mismatch panics at runtime, naming the method (`List::dot_add: lengths 3 and 2 differ; make both sides the same length`); an operator expression fused into one loop (2.6.6) may hold several operators and says `broadcast:` instead. If the sizes are known at compile time, it is caught as a compile error.

```rok
let x: [i32; 3] = [1, 2, 3];
let y: [i32; 4] = [1, 2, 3, 4];
let z = x .+ y;          // ERROR (compile: fixed sizes differ)

let p: List<i32> = [1, 2, 3];
let q: List<i32> = [1, 2];
let r = p .+ q;          // panic at runtime: length mismatch
```

### Scalar broadcast

Combining a scalar with a container behaves as follows.

```rok
let a: List<i32> = [1, 2, 3];
let s = a .+ 10;         //→ [11, 12, 13]  (scalar broadcast)
let m = 2 .* a;          //→ [2, 4, 6]
let t = a ./ 2;          //→ [0, 1, 1]    (i32 truncate)
```

Here separate impls (`DotAdd<T> for List<T>`, and `DotAdd<List<T>> for T` for the scalar-left form) treat the scalar side. `Slice<T>` has the same pair; fixed arrays `[T; N]` have no scalar-left impls. A library container gets the scalar-left form the same way for its own type: the orphan rule admits `impl DotMul<Matrix> for f64` in the package that defines `Matrix`, since the local `Matrix` comes before any type parameter ([2.9.5](../../refs-lang/002-types/209-impl.md#295-orphan-rule)), so `2.0 .* m` and `m .* 2.0` both work.

### Comparison dot - returns `Container<bool>`

Comparison dot operators return a `Container<bool>`.

```rok
let a: List<i32> = [1, 5, 3];
let b: List<i32> = [2, 5, 1];

a .== b;                 //→ [false, true, false]
a .< b;                  //→ [true, false, false]
a .>= b;                 //→ [false, true, true]

a .< 4;                  //→ [true, false, true]   (scalar broadcast)
```

### Remainder dot - `.%` and `.%%`

Remainder dot operators apply the meaning of ordinary `%` / `%%` element-wise.

```rok
let a: List<i32> = [-7, 5, -3, 8];
let b: List<i32> = [3, 3, 3, 3];

a .% b;                  //→ [-1, 2, 0, 2]   (Rem: sign follows the left operand)
a .%% b;                 //→ [2, 2, 0, 2]    (Modulo: always in [0, 3))
```

### Logical dot - `.&&` and `.||`

Logical dot operators produce the element-wise logical combination of two `Container<bool>`s. Unlike the undotted forms there is no short-circuit, so both containers are always evaluated to the end.

```rok
let p: List<bool> = [true, false, true, false];
let q: List<bool> = [true, true, false, false];

p .&& q;                 //→ [true, false, false, false]
p .|| q;                 //→ [true, true, true, false]

// combining comparison results: the most common use
let xs: List<i32> = [1, 5, 3, 8];
(xs .>= 2) .&& (xs .< 6);  //→ [false, true, true, false]
```

### Bitwise / shift dot

Bitwise and shift dot operators work the same way.

```rok
let xs: List<u8> = [0b1100, 0b1010, 0b0011];
let ys: List<u8> = [0b1010, 0b1010, 0b1111];

xs .& ys;                //→ [0b1000, 0b1010, 0b0011]
xs .| ys;                //→ [0b1110, 0b1010, 0b1111]
xs .^ ys;                //→ [0b0110, 0b0000, 0b1100]

let ns: List<u32> = [1, 2, 4];
ns .<< 2;                //→ [4, 8, 16]
```

## 2.6.5. Function broadcast - `f.(x)`

Function broadcast is written by adding a dot at an ordinary function call.

```rok
let xs: List<f64> = [0.0, 0.5, 1.0];

let s = f64::sin.(xs);               //→ [0.0, 0.479..., 0.841...]
let r = f64::sqrt.(xs);              //→ [0.0, 0.707..., 1.0]
let g = (|x: f64| x * 2.0).(xs);     //→ [0.0, 1.0, 2.0]
```

Several arguments can be broadcast at once.

```rok
let a: List<f64> = [1.0, 2.0, 3.0];
let b: List<f64> = [10.0, 20.0, 30.0];

let r = f64::pow.(a, b);             //→ [1.0, 4_194_304.0, ...]
let m = f64::max.(a, b);             //→ [10.0, 20.0, 30.0]
```

Arguments that are a `List`, `Slice` or array are iterated together; any other argument is passed unchanged to every call (`f64::pow.(xs, 2.0)`). The iterated arguments must have the same length, checked at run time. There is **no limit** on the number of arguments, so any N-ary function applies.

## 2.6.6. Loop fusion

The compiler fuses consecutive dot operations into a single loop.

```rok
let result = f64::sin.(a) .+ f64::cos.(b) .* 2.0;
```

A naive lowering would need 3 temporary List allocations and 3 loops. Rok guarantees one loop with no intermediate collections over `List`, `Slice` and arrays, and over any `Broadcastable` type ([3.3.4](../../refs-lang/003-expressions/303-broadcast-operators.md#334-automatic-loop-fusion) gives the exact condition). In-place forms (`y .= ..`, `y .+= ..`, [3.3.7](../../refs-lang/003-expressions/303-broadcast-operators.md#337-in-place-broadcast)) write into the target with no allocation at all.

```rok
// conceptually equivalent to this single elementwise call
let result = elementwise(a, b, |x, y| x.sin() + y.cos() * 2.0);
```

### `Broadcastable` and `Materialize`

The fusion protocol is two traits of `rok::core::ops`.

```rok
pub trait Broadcastable<const N: usize> {
    type Elem;
    /// The elements as an array: a view sharing the storage.
    fn broadcast_view(self) -> Array<Elem, N>;
}

pub trait Materialize<U, const N: usize> {
    type Output;
    /// The result built around `out`, a fresh row-major array.
    fn materialize(self, out: Array<U, N>) -> Output;
}
```

An `Array` ([2.9.14](209-collections.md#2914-arrayt-const-n-usize---n-dimensional-array)) is a buffer, an offset, a shape and strides, so it describes any strided storage; a type of its own (a stone's `Tensor`, a dataframe column) implements `Broadcastable` by returning the `Array` it keeps, and `Materialize` to be the result when it leads. `Array` implements both: its view is itself, and an expression it leads gives an `Array<U, N>`.

```rok
struct Signal { data: Array<f64, 1>; }
impl Broadcastable<1> for Signal {
    type Elem = f64;
    fn broadcast_view(self) -> Array<f64, 1> { self.data }
}
impl Materialize<f64, 1> for Signal {
    type Output = Signal;
    fn materialize(self, out: Array<f64, 1>) -> Signal { Signal { data: out } }
}
let t = s .* 2.0 .+ k;      // one loop; a Signal
```

A fused expression views each operand (`broadcast_view`), stretches the views to one shape (2.6.8), allocates the result for the leading operand (the first of the highest rank) and hands it to that operand's `materialize`. The loop runs over the buffers directly: one run of positions when the result and every view are contiguous, otherwise a walk over rows of the last axis with each view's stride. A leading operand without `Materialize` is a compile error naming the trait.

## 2.6.7. Trait implementation matrix

Dot trait impls map onto container types as follows.

| Container        | DotAdd   | DotMul   | DotEq       | Notes                                   |
|------------------|----------|----------|-------------|-----------------------------------------|
| `List<T>`        | T:Add    | T:Mul    | T:PartialEq | element-wise, lengths must match        |
| `[T; N]` (fixed) | T:Add    | T:Mul    | T:PartialEq | fixed N, compile time                   |
| `Slice<T>`       | T:Add    | T:Mul    | T:PartialEq | view, element-wise                      |
| `i32`/`f64`/etc. | scalar-left | scalar-left | scalar-left | no scalar-scalar impl (ordinary `+`/`*`/`==`); `T` implements `DotAdd<List<T>>` / `DotAdd<Slice<T>>` for the scalar-left form (2.6.4) |
| `T \| none`       | (operator) | (operator) | (operator) | option propagation (`none .+ x = none`) |

The dot meaning of `T | none` is option propagation: if either side is `none`, the result is `none`. It is handled at the operator level, with no library impl: the whole dot tree's operands are evaluated once, in source order; then each `T | none` operand is tested, and the first `none` makes the tree's value `none` without running it. Otherwise the tree runs on the `T` values, fused or through the dot traits as 2.6.6 says, so no dot trait is called with `none`. The value's type is the tree's type `| none`.

The n-dimensional `Array<T, N>` of core ([2.9.14](209-collections.md#2914-arrayt-const-n-usize---n-dimensional-array)) implements every dot trait: array with array of the same rank, array with scalar, scalar with array, by the rule of 2.6.8. A library's numeric container (`Tensor<T>`, a dataframe column) implements the same traits directly, a scalar on the left included: the orphan rule admits `impl DotMul<Tensor> for f64` in the package that defines `Tensor` ([refs-lang 2.9.5](../../refs-lang/002-types/209-impl.md#295-orphan-rule)). A dot expression over such a type calls the traits one operator at a time unless the type is `Broadcastable` (2.6.6); `Array` is, so its dot expressions fuse and its trait impls serve bounds and calls by name.

## 2.6.8. NumPy-style broadcasting rules

One rule, defined in core so every array type agrees: `rok::core::array::broadcast_shape(a, b)` takes two shapes of the same rank and, axis by axis, keeps equal lengths and stretches a length 1 to the other side's length. Any other pair panics naming both shapes and the axis (`broadcast: shapes [2, 3] and [2, 2] differ on axis 1; each axis must match or have length 1`). A scalar applies to every element. In a dot expression operands of different rank line up at their last axes: the shorter one counts as having leading axes of length 1 (a `[3]` vector and a `[2, 3]` matrix give `[2, 3]`), and the result has the highest rank, which the types show. The dot trait impls themselves take one rank, so generic code and calls by name line ranks up explicitly (`v[none, :]` turns a `[3]` vector into a `[1, 3]` row). `Array::broadcast_to(shape)` gives the stretched view itself (stride 0 on a stretched axis, no copy).

1-D containers (`List<T>`, `[T; N]`, `Slice<T>`) have one axis and no length-1 stretch: their lengths must be equal (2.6.4).

## 2.6.9. Comparison with other languages

Compared with other major languages:

| Aspect             | NumPy                | Julia                      | MATLAB           | Rok                           |
|--------------------|----------------------|----------------------------|------------------|-------------------------------|
| element-wise       | `*` (silent)         | `.*` (explicit)            | `.*` (explicit)  | `.*` (explicit)               |
| matrix mul         | `@` (PEP 465)        | `*`                        | `*`              | `*` (user-defined Mul)        |
| element-wise add   | `+` (silent)         | `.+`                       | `.+`             | `.+`                          |
| function broadcast | `np.vectorize(f)(x)` | `f.(x)`                    | `arrayfun(f, x)` | `f.(x)`                       |
| loop fusion        | **none**             | `@.` macro (automatic)     | **none**         | automatic for dot chains (MLIR) |
| comparison         | `==` returns array   | `.==`                      | `==` (confusing) | `.==`                         |
| separate dot traits | **none**            | **none** (`*` overload only) | **none**       | **DotAdd/DotMul/...** (20)    |

Rok differs most from other languages in two points. First, every dot operator has its own trait. Julia layers dot syntax on top of an overloaded `+`, whereas in Rok `Add` and `DotAdd` are different traits, and `Matrix * Matrix` and `Matrix .* Matrix` follow entirely different dispatch. Second, the compiler enforces dot usage. Broadcasting through ordinary `*` is **impossible**, which guarantees a deterministic separation that does not depend on user code.
