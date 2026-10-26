# 1.9. `rok::bare::mlir` - direct access to MLIR dialects

The `rok::bare::mlir` module provides an escape hatch for calling dialect ops of the MLIR backend used by the Rok compiler directly. It is the place for external library authors to express, within Rok, kernels for acceleration domains such as tensors / automatic differentiation / SIMD / GPU / quantization.

Ordinary Rok code does not see this module. The traits and collections of [200-core/](../200-core/) are sufficient for Rok's everyday abstractions; MLIR exposure is confined to library authors.

## 1.9.1. Motivation - why MLIR is needed on the user surface

Ordinary Rok users need not know MLIR, since the standard library abstracts arithmetic, comparison, collections and concurrency sufficiently. The situation differs for library authors such as:

- Tensor libraries: composing kernels on the `linalg.generic` / `tensor.*` / `vector.*` / `gpu.*` dialects
- Automatic differentiation libraries: expressing source-to-source transformation results as MLIR op graphs
- SIMD libraries: making width / alignment / shuffle explicit with the `vector` dialect
- GPU backends: emitting device code with `gpu.launch_func` and `gpu.module`
- Quantization / sparse operations: expressing them in dedicated dialects and using standard passes

These libraries cannot be built from the Rok standard library alone because MLIR's multi-level abstraction is itself the source of performance. A single `linalg.generic` automatically unfolds into fusion of adjacent operations / elimination of temporary allocations / SIMD / GPU lowering; bypassing it leaves the performance gap with PyTorch / JAX / Mojo open.

## 1.9.2. Surface - `__mlir_op` calls

The basic form of an MLIR op call is:

```rok
use rok::bare::mlir::{__mlir_op, __mlir_attr, __mlir_type};

unsafe {
    let r: i32 = __mlir_op::<"arith.addi", i32>(a, b);
    let v: Tensor<f32, (3, 4)> = __mlir_op::<"linalg.generic", _>(...);
}
```

Calls are allowed only inside an `unsafe` block. The first generic argument is the dialect op name (`"dialect.op"`), followed by the result type `R`.

- `R` is a Rok type. `_` means the expected type; `()` means no result.
- The remaining arguments are operands. Attributes are given as `attributes = { "k": __mlir_attr::<"text"> }`.
- The compiler emits the op in MLIR generic form (`"dialect.op"(%a, %b) {k = text} : (..) -> R`). The op name, operands and attributes are verified by the MLIR embedded in `rokc` ([0.2.4](../../refs-lang/000-meta/002-pipeline.md#024-backends)). A module using `__mlir_op` is compiled by the MLIR → LLVM backend at every optimization level ([refs-lang 9.2.10](../../refs-lang/009-tools/902-aot-pipeline.md#9210-backend-selection)).

### Attributes (`attributes`)

Many MLIR ops take compile-time attributes (predicate, axis index, alignment, etc.) besides SSA operands. They are passed with the `attributes` keyword argument.

```rok
unsafe {
    // predicate of arith.cmpi ("eq" / "ne" / "slt" / ...)
    let eq: bool = __mlir_op::<"arith.cmpi", bool>(
        a, b,
        attributes = { "predicate": __mlir_attr::<"#arith<cmpi_predicate eq>"> },
    );

    // alignment of memref.load
    let v: i32 = __mlir_op::<"memref.load", i32>(
        buf, idx,
        attributes = { "alignment": __mlir_attr::<"4 : i64"> },
    );
}
```

`__mlir_attr::<"...">` is an attribute literal in MLIR's textual representation; the text is passed to MLIR verbatim. Writing it as an empty call, `__mlir_attr::<"...">()`, is equivalent.

### MLIR type representation

MLIR types that do not map directly to Rok types (`index`, `!llvm.ptr`, `memref<?xi32>`, etc.) are written as `__mlir_type<"...">` in type position (`__mlir_type::<"...">` also works).

```rok
unsafe {
    let p = __mlir_op::<"llvm.alloca", __mlir_type<"!llvm.ptr">>(
        size,
        attributes = { "alignment": __mlir_attr::<"8 : i64"> },
    );
}
```

`__mlir_type<"..">` can be used as the result type `R` of `__mlir_op`, as an operand, as a function parameter or return, and as the type of a `let`. Its values cannot be stored inside aggregates (structs, arrays, GC objects).

### Ops with regions / blocks

Ops with region bodies, such as `scf.if`, `scf.for` and `linalg.generic`, receive their regions as closures in `regions = [..]`.

```rok
unsafe {
    // scf.if - two regions, then / else
    let r: i32 = __mlir_op::<"scf.if", i32>(
        cond,
        regions = [
            || { __mlir_op::<"scf.yield", ()>(a); },
            || { __mlir_op::<"scf.yield", ()>(b); },
        ],
    );

    // scf.for - lb / ub / step are index values
    let lb = __mlir_op::<"arith.index_cast", __mlir_type<"index">>(0i64);
    let ub = __mlir_op::<"arith.index_cast", __mlir_type<"index">>(n);
    let step = __mlir_op::<"arith.index_cast", __mlir_type<"index">>(1i64);
    let sum: i32 = __mlir_op::<"scf.for", i32>(
        lb, ub, step, init,
        regions = [
            |i: __mlir_type<"index">, acc: i32| {
                let next = __mlir_op::<"arith.addi", i32>(acc, x);
                __mlir_op::<"scf.yield", ()>(next);
            },
        ],
    );
}
```

Rules:
- One closure is one region. The number of closures must equal the number of regions the op requires.
- The closure's parameters are the arguments of the entry block and must have their types written.
- The result of a region is the operands of its terminator (`scf.yield` etc.). The closure's last value and `-> R` have no meaning.
- The body is straight-line code listing ops in order. Control flow is written with `scf` ops; `if` / `match` / loops / `return` / `break` / `continue` / panic are errors.
- Regions are emitted in generic form and may use outer values directly. Ops whose regions cannot see outer values (`gpu.func` etc.) are therefore not supported.
- Current limitations: one result per op, one block per region body.

### Variadic operand

Ops taking variadic arguments, such as `vector.shuffle`, receive them through Rok's spread syntax.

```rok
unsafe {
    let masked: Vector<i32, 4> = __mlir_op::<"vector.shuffle", _>(
        v1, v2,
        attributes = { "mask": __mlir_attr::<"[0, 2, 4, 6]"> },
    );
}
```

## 1.9.3. Available dialects

All dialects the compiler understands, per [refs-lang 8.7](../../refs-lang/008-abi/807-mlir-dialects.md), are accessible.

| Dialect    | Main uses                           |
|------------|-------------------------------------|
| `arith`    | Arithmetic / comparison (already used by the stdlib) |
| `math`     | sqrt / sin / cos / exp / log        |
| `vector`   | SIMD width / alignment / shuffle    |
| `linalg`   | element-wise / matmul / reduction   |
| `tensor`   | shape / extract / insert / reshape  |
| `memref`   | Buffer alloc / load / store         |
| `scf`      | Structured control flow (lowered by `--convert-scf-to-cf` by default) |
| `cf`       | branch / switch                     |
| `gpu`      | GPU launch / kernel / shared memory |
| `affine`   | Loops amenable to affine analysis   |
| `async`    | async fork / await (separate from the runtime) |
| `llvm`     | Low-level intrinsics, raw pointer interop |

Besides the 8 dialects used by the base stdlib (`arith` / `scf` / `cf` / `memref` / `index` / `math` / `func` / `llvm`), `linalg` / `tensor` / `vector` / `gpu` / `affine` / `async` are open to library authors.

Only these dialects, plus `bufferization` / `ub` / `dlti` for lowering, are registered. Ops of other upstream dialects are unknown to `rokc`, the same as an unloaded plugin's ([1.10.7](110-mlir-plugin.md#1107-build-isolation)).

## 1.9.4. Safety responsibilities

Users of `rok::bare::mlir` bear the following responsibilities.

- **Type agreement**: the operand / result types of the MLIR op and the Rok types must lower exactly.
- **Dialect availability**: some dialects (notably `gpu`) can be lowered only in builds with the backend enabled.
- **Stability**: MLIR dialect op names / signatures may change between LLVM versions; libraries using this module directly are advised to pin the LLVM version.
- **GC root**: when exposing GC object data through `memref` / `llvm.ptr`, it must be pinned with [1.6 `Pinned<T>`](106-gc-pin.md).

Libraries for which these responsibilities are too burdensome should be written with the stdlib's safe abstractions alone. `rok::bare::mlir` is a tool reserved for areas where acceleration is essential.

## 1.9.5. Relationship with ordinary user code

```rok
// ordinary user - does not see MLIR
let xs: List<f32> = [1.0, 2.0, 3.0];
let sum = xs.iter().sum();

// inside a library - uses rok::bare::mlir
mod tensor::ops {
    pub fn matmul<...>(a: Tensor<f32, (M, K)>, b: Tensor<f32, (K, N)>) -> Tensor<f32, (M, N)> {
        unsafe {
            __mlir_op::<"linalg.matmul", Tensor<f32, (M, N)>>(a, b)
        }
    }
}
```

The recommended pattern is for a library to put a safe surface over `rok::bare::mlir` and expose that to ordinary users. Users call only `tensor::matmul(a, b)`, and the internal MLIR calls are encapsulated as the library's responsibility.

This is isomorphic to the pattern of encapsulating [1.7 FFI](107-ffi.md)'s `extern "C"` inside a library, and is the consistent usage model of the `unsafe` tier.

## 1.9.6. Comparison with other languages

| Language | MLIR / IR exposure on the user surface | Position                                       |
|---------|------------------------------------|------------------------------------------------------|
| Mojo    | `__mlir_op` directly (default)     | Accessible to ordinary users too                     |
| Swift   | `Builtin.X` (inside the stdlib)    | Stdlib authors only                                  |
| Julia   | `llvmcall` (available to users)    | Discouraged, external libraries only                 |
| Rust    | LLVM intrinsics (not available to users) | nightly + intrinsics crate                     |
| **Rok** | **`rok::bare::mlir` (explicit unsafe)** | **Library authors only - ordinary users do not see it** |

Rok's model is close to Swift and Julia. Unlike Mojo, it does not expose MLIR to ordinary users, and it enforces an explicit `unsafe` entry so that the code shows which territory the user is stepping into.

## 1.9.7. Permission / escape hatch policy

`rok::bare::mlir` operates under the same policy as the user-code restrictions of the [0.3 intrinsic mechanism](../000-meta/003-intrinsics.md).

- User code may call `__mlir_op`, but only inside an `unsafe { ... }` block
- Attaching the `#[intrinsic("...")]` attribute remains restricted to the standard library
- Relationship between the two mechanisms: an intrinsic is *the compiler intercepting a stdlib method body*; `__mlir_op` is *the user expressing an MLIR op directly*
