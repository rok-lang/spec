# 8.7. MLIR dialect usage

Applies the standard MLIR dialects set out in the [compilation pipeline](../000-meta/002-pipeline.md) to the lowering specification of this [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) block. This is the lowering of the MLIR → LLVM backend (`-O1` and above); the native backend (`-O0`, REPL) generates machine code from rok-mir directly and follows the same ABI ([0.2.4](../000-meta/002-pipeline.md#024-backends)). Ordinary code is lowered to seven core dialects; libraries in acceleration domains may use additional dialects such as `linalg` / `tensor` / `vector` / `gpu` / `affine` / `async` directly through [`rok::bare::mlir`](../../refs-standard-libraries/100-bare/109-mlir.md).

## 8.7.1. Dialects used by ordinary code (recap)

The table from the [compilation pipeline](../000-meta/002-pipeline.md), repeated:

| Dialect    | Purpose                                  | Use in this block                                                       |
|------------|------------------------------------------|-------------------------------------------------------------------------|
| `func`     | Function definition / call               | Function lowering                                                       |
| `arith`    | Integer / floating-point arithmetic / comparison | Operator lowering                                               |
| `scf`      | Structured control flow (`if` / `for` / `while`) | `if` / loop lowering                                            |
| `cf`       | Unstructured branches (switch, cond_br)  | `match` decision tree                                                   |
| `index`    | Index type operations                    | Indexing lowering                                                       |
| `math`     | Math functions                           | sqrt / sin / cos, etc.                                                  |
| `llvm`     | Low-level ops (intrinsic, `struct`, `alloca`, load / store) | Stack slots, objects / arrays, [GC](../000-meta/007-glossary.md#gc-garbage-collector) |

There is no custom Rok dialect. The semantics of ordinary Rok code are expressed with the seven standard dialects above; additional dialects used by acceleration libraries are covered in 8.7.5.

## 8.7.2. Translation flow

The full translation flow is:

```
rok source
   ↓ (parse, name resolution, type check)
rok-hir
   ↓ (specialize / monomorphize, desugar)
rok-mir
   ↓ (lowering - 808 ~ 810 of this block)
standard MLIR  (7 dialects for ordinary code; libraries may add more)
   ↓ (standard passes, conversion to the LLVM dialect)
LLVM IR
   ↓ (LLVM optimization and code generation)
object
   ↓ (lld)
native binary
```

All steps run inside `rokc`. This ABI block defines the lowering rules for the step from rok-mir to standard MLIR.

## 8.7.3. Core ops per dialect

Which dialects `rokc` emits is implementation-defined. `rokc` emits `func`, `arith`, `cf` and `llvm` (and `math` where it lowers math intrinsics); `scf` and `index` may be used. The ops below are the core of each dialect.

### `func`

```
func.func @name(%arg: i32) -> (i32) { ... }
func.call @name(%arg) : (i32) -> (i32)
func.return %val : i32
```

Standard ops for function definition, call and return.

### `arith`

```
arith.addi %a, %b : i32
arith.subi %a, %b : i32
arith.muli %a, %b : i32
arith.cmpi "eq", %a, %b : i32
arith.addf %a, %b : f64
arith.cmpf "olt", %a, %b : f64
```

Arithmetic and comparison operations from [operator dispatch](../003-expressions/301-operator-dispatch.md) are lowered to the `arith` dialect.

User-defined trait implementations, however, are lowered as function calls after generic monomorphization.

### `scf`

```
scf.if %cond -> (i32) {
  scf.yield %a : i32
} else {
  scf.yield %b : i32
}

scf.while ... do { ... }
scf.for %i = %lb to %ub step %step { ... }
```

Normal control flow of [`if` expressions](../003-expressions/310-if-expression.md) and [loops](../003-expressions/313-loop-expressions.md) is lowered to `scf`.

### `cf`

```
cf.cond_br %cond, ^true_block(args), ^false_block(args)
cf.switch %v : i32, [
  default: ^default,
  0: ^case0,
  1: ^case1
]
```

The decision tree of a complex [`match`](../003-expressions/312-match-expression.md) is lowered using cond_br and switch from the `cf` dialect.

### `index`

```
%idx = index.constant 5
%new_idx = index.add %idx, %step
```

Places that need the `index` type (bounds of `scf.for`, etc.) are lowered to the `index` dialect.

### `math`

```
math.sqrt %x : f64
math.sin %x : f64
math.cos %x : f64
```

Standard library math functions are expressed with the `math` dialect where possible. Functions `math` does not support directly are handled as ordinary function calls.

### `llvm`

The LLVM dialect is a last resort, used only for semantics no other dialect can express.

```
llvm.alloca / llvm.load / llvm.store
llvm.struct<(...)>
llvm.intr.expect / llvm.intr.assume
```

Stack slots, object access and low-level intrinsics belong to the `llvm` dialect. Coroutines are compiler-built state machines and do not use `llvm.coro.*` ([7.8](../007-runtime/708-coroutine.md)). GC roots are handled with ordinary load / store on the shadow stack frame, without intrinsics ([8.9.12](809-lowering-control.md#8912-shadow-stack-frame)). Error propagation is expressed with return values and ordinary branches ([8.9.6](809-lowering-control.md#896-try--catch--finally-value-based-branching)), so it uses no special features of the `llvm` dialect.

## 8.7.4. Dialect priority

Lowering may use `scf`, `index` and `math` where they express the semantics, and `llvm` where nothing else does; the choice is implementation-defined (8.7.3).

## 8.7.5. Library / acceleration-domain dialects

The following dialects are not used in lowering Rok's *ordinary code*, but library authors can use them directly through [`rok::bare::mlir`](../../refs-standard-libraries/100-bare/109-mlir.md).

| Dialect    | Domain                                        |
|------------|-----------------------------------------------|
| `affine`   | Loops amenable to affine analysis (numeric libraries) |
| `linalg`   | element-wise / matmul / reduction (tensor lib) |
| `tensor`   | shape / extract / insert / reshape (tensor lib) |
| `vector`   | SIMD width / alignment / shuffle (SIMD lib)   |
| `gpu`      | GPU launch / kernel / shared memory (GPU lib) |
| `async`    | async fork / await (dedicated runtime library) |

Lowering ordinary Rok code needs only the seven dialects of 8.7.1, but libraries in acceleration domains such as tensors / automatic differentiation / SIMD / GPU call the dialects above directly to build a performance surface comparable to PyTorch / JAX / Mojo.

The mechanism by which library authors call dialects, and their safety responsibilities, are defined in [`rok::bare::mlir`](../../refs-standard-libraries/100-bare/109-mlir.md).

## 8.7.6. `extern` functions (runtime functions)

The `rok_*` functions listed in [Extern "C" symbols](805-extern-symbols.md) are declared as external `llvm.func` declarations (the declaration form is implementation-defined).

```
llvm.func @rok_gc_alloc_slow(i64, i64, i32) -> !llvm.ptr
llvm.func @rok_runtime_yield()
```

Calls use `llvm.call`.

## 8.7.7. TLS access

Access to the TLS slots from [Mutator TLS slots](803-tls-slots.md) uses the `llvm` dialect's thread_local global `rok_tls`, at each slot's offset.

```
llvm.mlir.global external thread_local @rok_tls() : !llvm.array<N x i8>

%t = llvm.mlir.addressof @rok_tls : !llvm.ptr
%flag_ptr = llvm.getelementptr %t[88] : (!llvm.ptr) -> !llvm.ptr, i8   ; yield_flag at 10p+8, p = 8
%flag = llvm.load %flag_ptr : !llvm.ptr -> i8
```

The LLVM backend lowers this to target-specific TLS instructions automatically.

## 8.7.8. Responsibility of each stage

```
rok-mir → standard MLIR     rokc lowering (this block)
MLIR → LLVM IR              MLIR passes and translation, in-process
LLVM IR → object            LLVM, in-process, at the requested -O level
objects → binary            lld, in-process (ROK_CC overrides, 9.2.14)
```

## 8.7.9. Verifying the translation

End-to-end tests under `tests/integration/` verify by building and running user-written .rok files.

This verification ensures that the lowering rules defined in this block are correct.
