# 0.2. Compilation pipeline

This specification calls the rok compiler `rokc`. `rokc` is responsible for everything up to emitting source code as standard MLIR text. Optimization and binary generation after that are handled by the standard toolchain.

## 0.2.1. Stages

The overall flow is as follows.

```
  source code
        │
        ▼
   rok-hir         (High-level IR — pre type inference, user code structure preserved)
        │
        ▼
   rok-mir         (Mid-level IR — monomorphized / operators desugared, all types fixed)
        │
        ▼
   standard MLIR   (upstream MLIR dialect text — `func`, `arith`, `scf`, ...)
        │
        ▼
   LLVM IR / native (mlir-opt → mlir-translate → clang / lld)
```

> rok-hir is the high-level IR; it precedes type inference and preserves the structure of user code as is. rok-mir is the mid-level IR, in which monomorphization and operator desugaring are complete and all types are fixed.
> standard MLIR means upstream MLIR, expressed as upstream MLIR dialect text. Finally, LLVM IR or native code is produced.

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

This choice guarantees the following. Improvements in upstream MLIR flow into rok automatically. rok interoperates with external MLIR-based tools such as IREE, Enzyme and Triton. And there is no cost of registering and maintaining a custom dialect.

In exchange, the responsibility for lowering rok's high-level semantics to the standard dialects concentrates in the compiler, specifically in the `rok-mir → MLIR` stage. This burden is intentional: it is moved inward so as not to close off options at the end of the pipeline.

## 0.2.4. Emitting MLIR text directly

The compiler outputs MLIR in text form. That is, `rokc` does not call or link MLIR / LLVM library bindings (C++ or otherwise). Only the REPL / Jupyter JIT host process links libLLVM (ORC), and `rokc` passes LLVM IR text to that host ([9.6](../009-tools/906-jit-engine.md)).

This approach has several benefits. It reduces external library dependencies. It simplifies the cross-compilation matrix: results can be consumed anywhere MLIR tools are installed. And debug output such as `rokc --emit=mlir` flows directly into standard tools.
