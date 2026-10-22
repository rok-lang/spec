# 9.6. JIT engine

This section defines the JIT execution engine used by the REPL and Jupyter. It is the implementation of the JIT engine defined in [Execution engines and user surfaces](../000-meta/001-execution-modes.md). The rules of the cell session are defined by [9.7.0](907-jit-cells.md#970-rules-of-the-cell-session).

## 9.6.1. Structure: rokc and the JIT host

The session runs in a separate **host process** that contains the runtime and lives for the session. The host is prebuilt with `rokc` and linked by the embedded lld; it loads cell objects itself, with no libLLVM or ORC.

```
rokc (front end)                         JIT host (one per session)
  cell → parse → HIR → MIR                   all of librokr.a
     → native backend                        runtime initialized once
     → machine code + relocations ───────►   load into executable memory, resolve symbols
                                             → run the cell entry function once
```

- Cells follow the backend selection of [9.2.10](902-aot-pipeline.md#9210-backend-selection). At `-O0` a cell is a native object ([9.2.11](902-aot-pipeline.md#9211-native-backend)) mapped and linked by the host, with no LLVM. Above `-O0`, and for code the native backend does not take (`__mlir_op`, other targets), cells go through the embedded MLIR → LLVM pipeline into an object loaded into the same executable memory.
- The host places the code in executable memory, resolves its relocations against the runtime, libc and earlier cells, and runs the cell's entry function once.
- The host exports every symbol of the runtime (`librokr.a`), and cell code finds runtime and libc symbols in the host process.

## 9.6.2. Code loading

A cell is an object file: native at `-O0`, LLVM-compiled above `-O0`. Its code is position-independent with relocations. The host keeps one symbol table for the session; a cell's definitions are added to it after the cell is loaded. A symbol that cannot be resolved fails the cell with an error naming it ([9.7](907-jit-cells.md)).

## 9.6.3. Symbols and redefinition

A session uses one symbol table. The definitions of all cells accumulate there.

- When a cell redefines an already defined symbol, the host renames that definition to `<name>.r<cell>`. References in later cells link to the newest name.
- A redefinition takes effect from the next cell ([9.7.0](907-jit-cells.md#970-rules-of-the-cell-session) rule 3).
- Functions depending on a redefined symbol are recompiled by `rokc`. So even code compiled before the redefinition uses the new definition when a later cell reaches it again (a `g` calling a redefined `f`).
- Cell code and old definitions are not freed until the session ends.

## 9.6.4. Reuse of library code

Code of the standard library and dependency packages is compiled only once per session.

- `rokc` remembers the functions the host has already loaded and their MIR. If a cell's function is identical to an already loaded one, only a reference is passed instead of the code.
- Names across cells are fixed so that identical MIR means identical meaning. String literal globals are named by a hash of their content, and GC type ids are fixed for the session keyed by [ScanTable](../008-abi/802-scan-table.md) content.
- Functions that directly or indirectly call something this cell newly defines (a changed function, a `static`, a binding slot) are recompiled.
- A cell's object holds only what the cell adds: the functions it defines anew and, for statics of earlier cells its code reads, their initialization guard. It names only the symbols its code refers to.

## 9.6.5. Execution flow (one REPL line / one Jupyter cell)

```
1. rokc: input → parse → HIR (on error the cell is discarded; session state unchanged)
2. rokc: cell → MIR → native backend → machine code
         (already loaded functions as references, redefinitions under new names)
3. host: load the code in a process forked from the session snapshot and run the cell once
4. success: that process becomes the new snapshot. If there is a value, print `type: value`
   failure (panic, exit, signal, load failure): the previous snapshot takes over (9.7.0 rule 5)
```

Step 3 forks on Unix. Hosts without fork (Windows) save the session's memory and threads before the cell and put them back when it fails, with the same observable result as the forked snapshot.

The latency of one cell is about 0.1 seconds, most of it spent in the front end (informative).

## 9.6.6. Differences between JIT and AOT

| Aspect      | JIT (REPL / Jupyter)                         | AOT                               |
|-------------|----------------------------------------------|-----------------------------------|
| Pipeline    | Same up to rok-mir                           | Same                              |
| Backend     | native, loaded into the host                 | native (`-O0`) or MLIR → LLVM, then lld |
| Unit        | Cell (libraries once per session)            | Package                           |
| Result      | Host memory                                  | Executable / `.a`                 |

The two engines share the same monomorphization and rok-mir, and both backends follow the same ABI, so their semantics are the same.

## 9.6.7. Next: cell execution / state / errors

[JIT cell execution flow and state management](907-jit-cells.md) defines per-cell behavior (top-level variables, redefinition, errors).
