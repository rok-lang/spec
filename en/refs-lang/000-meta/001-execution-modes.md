# 0.1. Execution engines and user surfaces

Rok's execution model has two axes. One is the execution engine, which determines how code becomes machine code; the other is the user surface, the environment in which the user enters code and receives results. There are two engines and three surfaces. Because several surfaces share the same engine, there are fewer engines than surfaces.

## 0.1.1. Execution engines

### JIT (Just-in-Time)

The JIT engine compiles and runs code fragments as soon as they are entered. One input unit, i.e. one line or one cell, can refer to the state accumulated up to that point: variables, functions, imported modules, and so on. When a new input overrides an existing definition, subsequent inputs see the new definition. Compilation results are cached in memory and on disk, so re-running the same input completes immediately.

### AOT (Ahead-of-Time)

The AOT engine compiles the entire source ahead of time into a single artifact: an executable or a static library `.a`. No compilation cost is incurred at startup, and the result can be linked as a library from other Rok code. Adding new code at run time, however, is not possible.

### The semantics are the same

> [!IMPORTANT]
> The two engines share the same semantics. Code that works under JIT produces the same result in an AOT build. The difference lies only in **when compilation happens**. This specification applies to both engines simultaneously; no semantics apply to only one of them.

This principle also applies to the entry point. Input to the REPL and Jupyter has the same meaning as appending top-level statements and items, in order, to the end of `main.rok` in an executable package. Only session operations (re-declaring the same name with `let` at cell top level to create a new binding, or re-running a cell) are specific to the interactive surfaces ([4.16.10](../004-functions/416-entry-point.md#41610-relationship-to-jit--repl)).

The abbreviations JIT and AOT are used as given in the [glossary](007-glossary.md).

### They share the same pipeline

JIT and AOT both pass through the same compilation pipeline up to rok-mir ([0.2. Compilation pipeline](002-pipeline.md)). They diverge only in code generation. The **JIT engine** compiles each input with Rok's native backend straight to machine code and runs it in a host process that lives for the session ([9.6](../009-tools/906-jit-engine.md)). The **AOT engine** generates code with the native backend at `-O0` and with the MLIR → LLVM pipeline at `-O1` and above, and links a native binary ([9.2](../009-tools/902-aot-pipeline.md)). Both backends follow the same runtime ABI.

In this structure a single compiler implementation serves both engines. Any behavioral difference must therefore be attributed not to the compiler but to the run-time environment: memory, threads, OS resources.

## 0.1.2. User surfaces

### REPL

On the REPL surface, a line or expression entered in the terminal is evaluated immediately and its result is printed. It is used mainly for exploration, demonstrating formulas and checking types. Variable and function definitions accumulate across inputs, and the value and type of the last expression are displayed automatically.

The engine is JIT, and the input unit is one line or one expression.

### Jupyter

The Jupyter surface runs as the Rok kernel for Jupyter / JupyterLab. Evaluation is per cell, and results remain in the notebook as visualizations, tables and text. This supports the standard data analysis workflow; re-running a cell replaces the previous definitions.

The engine is JIT, and the input unit is one cell. A Jupyter frontend (JupyterLab, Notebook, VS Code, etc.) runs the Rok kernel as a separate process and communicates with it over the ZeroMQ-based Jupyter wire protocol ([9.8](../009-tools/908-jupyter-integration.md)). The kernel uses the OS file system, network and external processes directly.

### AOT artifacts

On the AOT artifact surface, an executable built from source is run directly, or a produced static library is linked into another build.

The engine is AOT.

## 0.1.3. Mapping between engines and surfaces

```
              ┌── REPL (terminal)
   JIT  ──────┤
              └── Jupyter (kernel, ZMQ)

   AOT  ────── direct artifact execution
```

Three surfaces share two engines.

## 0.1.4. Semantic baseline

Where the specification is ambiguous, AOT semantics take precedence. JIT reproduces AOT semantics cell by cell.
