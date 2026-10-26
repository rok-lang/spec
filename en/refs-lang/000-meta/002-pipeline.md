# 0.2. Compilation pipeline

This specification calls the Rok compiler `rokc`. `rokc` embeds LLVM, MLIR and lld and runs the whole pipeline, from source to a linked binary, in-process.

## 0.2.1. Stages

The overall flow is as follows.

```
  source code
        │
        ▼
   rok-hir         (High-level IR - pre type inference, user code structure preserved)
        │
        ▼
   rok-mir         (Mid-level IR - monomorphized / operators desugared, all types fixed)
        │
        ▼
   standard MLIR   (upstream MLIR dialects - `func`, `arith`, `scf`, ...)    -O1 and above
        │
        ▼
   LLVM IR → native object → lld
```

At `-O0` (builds and REPL), rok-mir goes straight to machine code through Rok's native backend (0.2.4).

> rok-hir is the high-level IR; it precedes type inference and preserves the structure of user code as is. rok-mir is the mid-level IR, in which monomorphization and operator desugaring are complete and all types are fixed.
> standard MLIR means the upstream MLIR dialects. Finally, LLVM IR and native code are produced.

## 0.2.2. One-way visibility

Each stage looks only at the stage immediately before it. The `rok-hir` generator refers only to the source. The `rok-mir` generator refers only to `rok-hir`. The MLIR emitter refers only to `rok-mir`.

Results of earlier stages are thus never consulted again by later stages. The specification documents follow the same rule: a lower-numbered file does not refer to a higher-numbered file.

## 0.2.3. Standard MLIR dialects only

No rok-specific MLIR dialect is defined. All dialects used are official upstream MLIR dialects.

| Dialect    | Purpose                                         |
|------------|-------------------------------------------------|
| `func`     | Function definition / call                      |
| `arith`    | Integer / floating-point arithmetic / comparison |
| `scf`      | Structured control flow (`if` / `for` / `while`) |
| `cf`       | Unstructured branches (switch, cond_br)         |
| `index`    | Index type operations                           |
| `math`     | Math functions                                  |
| `llvm`     | Low-level ops (intrinsic, `struct`)             |

This choice guarantees the following. Improvements in upstream MLIR flow into Rok automatically. Rok interoperates with external MLIR-based tools such as IREE, Enzyme and Triton. And there is no cost of registering and maintaining a custom dialect.

In exchange, the responsibility for lowering Rok's high-level semantics to the standard dialects concentrates in the compiler, specifically in the `rok-mir → MLIR` stage. This burden is intentional: it is moved inward so as not to close off options at the end of the pipeline.

## 0.2.4. Backends

`rokc` has two code generators. Both consume rok-mir and follow the same runtime ABI ([8](../008-abi/README.md)), so their objects link together.

| Backend           | Used for                     | Path                                         |
|-------------------|------------------------------|----------------------------------------------|
| native            | `-O0` builds and REPL cells  | rok-mir → machine code                       |
| MLIR → LLVM       | `-O1` and above              | rok-mir → MLIR → LLVM IR → optimized object  |

LLVM, MLIR and lld are part of `rokc`; no external LLVM tools are run and users do not install LLVM. Targets on macOS need the Xcode Command Line Tools for the SDK. Cross-compilation is built in: `--target <triple>` works from any host, and the toolchain ships the C library (musl, glibc link stubs) for its supported Linux targets, so a Linux target needs no sysroot. There are no C headers: Rok compiles no C. Windows targets link against import libraries built from module definitions the toolchain carries, over the UCRT, with no MSVC tools or Windows SDK. Apple targets need the macOS SDK (Xcode Command Line Tools), which is not redistributable ([9.2.14](../009-tools/902-aot-pipeline.md#9214-linking-for-executables), [0.6](006-platform-targets.md)). `--emit=mlir` / `--emit=llvmir` dump the intermediate forms ([9.9.1](../009-tools/909-debug-tools.md#991-intermediate-output---emit)).
