# 9.6. JIT engine

This section defines the JIT execution engine used by the REPL and Jupyter. It is the implementation of the JIT engine defined in [Execution engines and user surfaces](../000-meta/001-execution-modes.md). The rules of the cell session are defined by [9.7.0](907-jit-cells.md#970-rules-of-the-cell-session).

## 9.6.1. Structure: rokc and the JIT host

The JIT lives not inside `rokc` but in a separate **host process** that lives for the session.

```
rokc (front end)                         JIT host (one per session)
  cell → parse → HIR → MIR → MLIR text       all of librokr.a + libLLVM (ORC LLJIT)
     → mlir-opt | mlir-translate                runtime initialized once
     → LLVM IR text  ─────────────────────►     parse IR → add to JITDylib
                                                → look up cell entry function → run once
```

- `rokc` builds each cell only as far as LLVM IR text, using the same pipeline as [AOT](902-aot-pipeline.md). No clang, object files or shared libraries are used.
- The host links libLLVM's ORC (`LLJIT`) through the LLVM-C API. It parses the IR, JIT-compiles it, finds the cell's entry function and runs it once.
- `rokc` itself still does not link MLIR / LLVM libraries ([0.2.4](../000-meta/002-pipeline.md#024-emitting-mlir-text-directly)). Only the host links libLLVM.
- The host exports every symbol of the runtime (`librokr.a`), and cell code finds runtime and libc symbols in the host process.

## 9.6.2. libLLVM requirement

The REPL and the Jupyter kernel require libLLVM. When the host is built, libLLVM is located in the LLVM installation holding the `mlir-translate` on `PATH` (`<prefix>/lib/libLLVM` and `<prefix>/include/llvm-c`), so the LLVM that produces the IR and the LLVM that reads it are the same. If it is missing, it is an error stating what is missing; there is no fallback to another execution mode.

## 9.6.3. Symbols and redefinition

A session uses one JITDylib. The definitions of all cells accumulate there.

- When a cell redefines an already defined symbol, the host renames that definition to `<name>.r<cell>`. References in later cells link to the newest name.
- A redefinition takes effect from the next cell ([9.7.0](907-jit-cells.md#970-rules-of-the-cell-session) rule 3).
- Functions depending on a redefined symbol are recompiled by `rokc`. So even code compiled before the redefinition uses the new definition when a later cell reaches it again (a `g` calling a redefined `f`).
- Cell code and old definitions are not freed until the session ends.

## 9.6.4. Reuse of library code

Code of the standard library and dependency packages is compiled only once per session.

- `rokc` remembers the functions the host has already compiled and their IR text. If a cell's function is textually identical to an already compiled one, only a declaration is passed instead of the definition.
- Names across cells are fixed so that identical text means identical meaning. String literal globals are named by a hash of their content, and GC type ids are fixed for the session keyed by [ScanTable](../008-abi/802-scan-table.md) content.
- Functions that directly or indirectly call something this cell newly defines (a changed function, a `static`, a binding slot) are recompiled.

## 9.6.5. Execution flow (one REPL line / one Jupyter cell)

```
1. rokc: input → parse → HIR (on error the cell is discarded; session state unchanged)
2. rokc: cell → MLIR → mlir-opt | mlir-translate → LLVM IR text
         (already compiled functions as declarations, redefinitions under new names)
3. host: JIT-compile the IR in a process forked from the session snapshot and run the cell once
4. success: that process becomes the new snapshot. If there is a value, print `type: value`
   failure (panic, exit, signal, JIT failure): the previous snapshot takes over (9.7.0 rule 5)
```

The latency of one cell is about 0.1 seconds, most of it spent in the front end (informative).

## 9.6.6. `--replay`

`--replay` mode uses the same host. For each input, the entire session so far is compiled as one program, JIT-compiled in a throwaway child process forked by the host, and run from the start. No executable is produced. The side effects of earlier cells are repeated every time, and there is no library reuse (9.6.4).

## 9.6.7. Differences between JIT and AOT

| Aspect      | JIT (REPL / Jupyter)                         | AOT                               |
|-------------|----------------------------------------------|-----------------------------------|
| Pipeline    | Same (up to LLVM IR text)                    | Same                              |
| Final step  | The host's ORC JIT-compiles the IR           | mlir-translate, then clang to object / link |
| Unit        | Cell (libraries once per session)            | Package                           |
| Result      | Host memory                                  | Executable / `.a`                 |

The two engines share the same monomorphization and the same MLIR lowering, so their semantics are the same.

## 9.6.8. Next: cell execution / state / errors

[JIT cell execution flow and state management](907-jit-cells.md) defines per-cell behavior (top-level variables, redefinition, errors).
