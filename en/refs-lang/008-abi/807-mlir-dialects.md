# 8.7. MLIR dialect usage

Applies the standard MLIR dialects set out in the [compilation pipeline](../000-meta/002-pipeline.md) to the lowering specification of this [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) block. This is the lowering of the MLIR → LLVM backend (`-O1` and above); the native backend (`-O0`, REPL) generates machine code from rok-mir directly and follows the same ABI ([0.2.4](../000-meta/002-pipeline.md#024-backends)). Ordinary code is lowered to five upstream dialects; acceleration-domain libraries are covered in 8.7.5.

## 8.7.1. Dialects used by ordinary code (recap)

The table from the [compilation pipeline](../000-meta/002-pipeline.md), repeated:

| Dialect    | Purpose                                  | Use in this block                                                       |
|------------|------------------------------------------|-------------------------------------------------------------------------|
| `func`     | Function definition / call               | Function lowering, calls to Rok functions and most runtime functions    |
| `arith`    | Integer / floating-point arithmetic / comparison | Operator lowering, `min` / `max` of floats                      |
| `math`     | Exact float functions                    | `sqrt`, rounding, `abs`, `copysign`, `mul_add`                          |
| `cf`       | Branches (`cf.br`, `cf.cond_br`)         | All control flow: `if`, loops, `match`, `try`                           |
| `llvm`     | Low-level ops (intrinsic, `struct`, `alloca`, load / store) | Stack slots, objects / arrays, [GC](../000-meta/007-glossary.md#gc-garbage-collector), C calls |

There is no custom Rok dialect. The semantics of ordinary Rok code are expressed with the five standard dialects above; acceleration libraries are covered in 8.7.5. `scf` and `index` are not emitted: rok-mir is already a control-flow graph of basic blocks, and the conversion from `scf` to `cf` would produce the same branches (8.7.4).

## 8.7.2. Translation flow

The full translation flow is:

```
rok source
   ↓ (parse, name resolution, type check)
rok-hir
   ↓ (specialize / monomorphize, desugar)
rok-mir
   ↓ (lowering: 808 ~ 810 of this block)
standard MLIR  (5 dialects for ordinary code; libraries may add more)
   ↓ (standard passes, conversion to the LLVM dialect)
LLVM IR
   ↓ (LLVM optimization and code generation)
object
   ↓ (lld)
native binary
```

All steps run inside `rokc`. This ABI block defines the lowering rules for the step from rok-mir to standard MLIR.

## 8.7.3. Core ops per dialect

`rokc` emits `func`, `arith`, `math`, `cf` and `llvm`. The ops below are the core of each dialect.

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

### `cf`

```
cf.br ^next(args)
cf.cond_br %cond, ^true_block(args), ^false_block(args)
```

Every branch of rok-mir is a `cf.br` (goto) or a `cf.cond_br` (two-way `if`): [`if` expressions](../003-expressions/310-if-expression.md), [loops](../003-expressions/313-loop-expressions.md), the decision tree of a [`match`](../003-expressions/312-match-expression.md) and error propagation. A `match` on a tag or an integer is a chain of `cf.cond_br` comparing the same value against each constant (8.9.3); LLVM's optimizer turns such a chain into one `switch` (a jump table or a binary search) at `-O1` and above.

### `math`

```
math.sqrt %x : f64
math.floor %x : f64
math.fma %a, %b, %c : f64
```

The float methods whose result is exact or correctly rounded are lowered inline, so the optimizer sees them as LLVM intrinsics (one instruction on most targets):

| Method                                  | Op                                         |
|-----------------------------------------|--------------------------------------------|
| `sqrt`                                  | `math.sqrt`                                |
| `floor` / `ceil` / `trunc`              | `math.floor` / `math.ceil` / `math.trunc`  |
| `round` (ties away from zero)           | `math.round`                               |
| `abs` / `copysign`                      | `math.absf` / `math.copysign`              |
| `mul_add` (one rounding)                | `math.fma`                                 |
| `min` / `max` (a NaN operand is ignored)| `arith.minnumf` / `arith.maxnumf`          |

This holds for `f64` and `f32`. The other math functions (`sin`, `exp`, `ln`, `pow`, `powi`, `cbrt`, `hypot`, ..) are calls to the runtime's `rok_f64_*` / `rok_f32_*` functions, which call libm ([8.5](805-extern-symbols.md)): their results come from the target's libm, never from a compile-time evaluation on the host.

### `llvm`

The LLVM dialect is a last resort, used only for semantics no other dialect can express.

```
llvm.alloca / llvm.load / llvm.store
llvm.struct<(...)>
llvm.intr.expect / llvm.intr.assume
```

Stack slots, object access, C calls (`llvm.call`, with `byval` / `sret` attributes, 8.6) and low-level intrinsics (`llvm.intr.ctpop`, `llvm.intr.ctlz`, saturating casts through `llvm.call_intrinsic`) belong to the `llvm` dialect. Coroutines are compiler-built state machines and do not use `llvm.coro.*` ([7.8](../007-runtime/708-coroutine.md)). GC roots are handled with ordinary load / store on the shadow stack frame, without intrinsics ([8.9.12](809-lowering-control.md#8912-shadow-stack-frame)). Error propagation is expressed with return values and ordinary branches ([8.9.6](809-lowering-control.md#896-try--catch--finally-value-based-branching)), so it uses no special features of the `llvm` dialect.

## 8.7.4. Dialect priority

An op of `arith`, `math` or `func` is used where one expresses the semantics, `llvm` where none does. Control flow stays in `cf`: rok-mir has no structured regions to rebuild, and `scf` would be converted back to the same `cf` branches before LLVM IR, so it gains nothing. The MLIR pipeline runs `canonicalize` and `cse` (at `-O1` and above), then converts `math`, `func`, `arith` and `cf` to the `llvm` dialect.

## 8.7.5. Acceleration-domain libraries

Rok source has no direct access to MLIR: only the compiler emits MLIR, and only the dialects of 8.7.1. Libraries in acceleration domains such as tensors / automatic differentiation / SIMD / GPU (`affine`, `linalg`, `tensor`, `vector`, `gpu`, `async`) reach the compiler's backend through the backend interface, planned for Rok 0.2.2.

## 8.7.6. `extern` functions (runtime functions)

The `rok_*` functions listed in [Extern "C" symbols](805-extern-symbols.md) are declared as `func.func private` and called with `func.call`. The three that write only references and headers (`rok_runtime_yield`, `rok_gc_alloc_slow`, `rok_gc_wb_slow`) are declared `llvm.func` and called with `llvm.call`, which carries the TBAA tag that keeps other heap loads valid across the call.

```
func.func private @rok_str_cmp(!llvm.struct<(ptr, i64)>, !llvm.struct<(ptr, i64)>) -> i32
llvm.func @rok_gc_alloc_slow(i64, i64, i32) -> !llvm.ptr
llvm.func @rok_runtime_yield()
```

Some runtime functions are not called at all: `rok_str_len` and `rok_str_byte` read the `str` pair, `rok_str_eq` compares strings of up to 16 bytes inline (a call above), and the exact float functions are `math` ops (8.7.3).

## 8.7.7. TLS access

Access to the TLS slots from [Mutator TLS slots](803-tls-slots.md) uses the `llvm` dialect's thread_local global `rok_tls`, at each slot's offset.

```
llvm.mlir.global external thread_local @rok_tls() : !llvm.array<N x i8>

%g = llvm.mlir.addressof @rok_tls : !llvm.ptr
%t = "llvm.intr.threadlocal.address"(%g) : (!llvm.ptr) -> !llvm.ptr
%flag_ptr = llvm.getelementptr %t[88] : (!llvm.ptr) -> !llvm.ptr, i8   ; yield_flag at 10p+8, p = 8
%flag = llvm.load %flag_ptr : !llvm.ptr -> i8
```

The thread's base address is computed once, in the entry block, and reused for every slot. The LLVM backend lowers it to target-specific TLS instructions.

## 8.7.8. Responsibility of each stage

```
rok-mir → standard MLIR     rokc lowering (this block)
MLIR → LLVM IR              MLIR passes and translation, in-process
LLVM IR → object            LLVM, in-process, at the requested -O level
objects → binary            lld, in-process (ROK_CC overrides, 9.2.14)
```

## 8.7.9. Verifying the translation

End-to-end tests (`compiler/rokc/tests/integration.rs`, over the programs in `compiler/rokc/tests/cases/`) build and run .rok files, at `-O0` and, with `ROK_TEST_OPT=-O2`, through this lowering.

This verification ensures that the lowering rules defined in this block are correct.
