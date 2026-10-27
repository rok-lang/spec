# 9.7. JIT cell execution flow and state management

This section defines in detail how the [JIT engine](906-jit-engine.md) processes user input (cells).

## 9.7.0. Rules of the cell session

The cell session obeys the following rules. The implementation is the [9.6 JIT engine](906-jit-engine.md).

1. **Run once.** The code of each cell runs exactly once. Side effects of earlier cells (I/O, time, random numbers) are not repeated.
2. **State persists.** Top-level `let` bindings of a cell, the state of `static`s, and `any` values (including `any Tr`) carry over to later cells. Bindings may be assigned, and a new `let` of the same name shadows with a new binding (9.7.3).
3. **Items persist and may be redefined.** Items such as `fn` / `struct` / `impl` persist. If the same item is defined again, the new definition is used from the next cell. Even code compiled before the redefinition uses the new definition when a later cell reaches it again (a `g` calling a redefined `f` calls the new `f`). Redefining a `static` restarts its state. A binding whose type was redefined can no longer be used; an attempt to use it is reported with a note.
4. **Identity of `any`.** The type tag of `any` is the same across all cells of a session. An `any` value stored in an earlier cell can be extracted in a later cell with `is` / patterns.
5. **Failed cells.** A cell that panics, terminates the process or dies leaves no partial effect on the bindings. The session continues from the state before that cell. External effects that already happened (I/O etc.) are not undone. Threads from earlier cells: after a failed cell the session keeps only the threads that are back where they were (Windows) or none of them (fork hosts). The runtime restarts its own executor threads; joining a user thread that is gone reports it lost (`JoinError` code `JOIN_LOST`, [stdlib 3.4](../../refs-standard-libraries/300-std/304-thread.md)). The session is saved at a quiet moment: no task is mid-poll and no blocking job is mid-run (the host waits for them before the cell; interrupting the wait cancels the cell), so tasks and jobs are never lost. Locks that a thread that is gone held while blocked are released.
6. **Display.** An expression cell prints its value with `Display`; a trailing `?` prints the `Debug` form. A value without the requested trait (`Display` for a bare expression, `Debug` for a trailing `?`) prints only its type name (9.7.7).

The following bindings and effects are not supported.

- Bindings of types that cannot be named at top level (closures, private types, etc.). They are rejected in the cell that creates them.
- `exit()` inside a cell. That cell is treated as a failed cell (rule 5).
- Threads spawned by a failed cell. They do not survive it (rule 5).

## 9.7.1. Kinds of cell

A REPL line or a Jupyter cell is processed differently depending on its content.

```
1. expression                → evaluate + store in hidden binding + print
2. statement (let / call)    → execute (return value is unit)
3. definition (fn / struct)  → add to the session's items
4. import (use / mod)        → update the session's visible names
5. meta / attribute          → handle as appropriate
```

All five follow the same cell processing flow; only the final output differs. An expression cell's value is kept in a hidden binding that user code cannot name. An expression cell prints its value (`Display`); a `let` cell prints nothing. A divergent expression cell ([4.3.7](../004-functions/403-return.md#437-divergent-functions); `panic(..)`, `exit(..)`, `throw`) runs as a statement.

> [!IMPORTANT]
> In terms of language semantics, a sequence of cells is equivalent to appending top-level statements and items to the end of an implicit `main.rok` ([4.16.10](../004-functions/416-entry-point.md#41610-relationship-to-jit--repl)). A cell's `let` is a local variable of the implicit `main`, so it is visible to statements of later cells but not inside the bodies of `fn`s declared in cells. The global storage of 9.7.3 below is an implementation technique for keeping local variables alive across cell boundaries and does not change this visibility rule. Same-name re-`let` at cell top level (9.7.3), cell re-execution (9.7.4) and automatic display of the last expression value (9.7.7) are session operations that exist only on the interactive surfaces.

## 9.7.2. Execution flow: in detail

```
1. parse → AST                      (error: display, discard cell)
2. lower to HIR, type check         (error: display, discard cell)
3. MIR → native machine code        (reuse / redefinition handling, 9.6.3 / 9.6.4)
4. host: load in a process forked from the snapshot, run the cell entry function
5. success: new snapshot, apply bindings and items to the session, print result
   failure: continue from the previous snapshot, display error
```

Each step is transactional; on failure it rolls back to the previous state.

Every cell runs with `/dev/null` (`NUL` on Windows) as its standard input, so a cell that reads stdin sees end of input. The REPL host sets `ROK_GC_CONCURRENT=off` before the runtime starts, whatever the user's environment says: a concurrent marking thread would not be part of a snapshot, so full collections in a session mark without it ([7.5](../007-runtime/705-concurrent-collection.md)).

## 9.7.3. Top-level variable storage

Variables must be shared between REPL and Jupyter cells, so top-level variables must stay alive for as long as the session lasts.

```rok
[1] let x = 42i32;
[2] let y = x + 1;       // uses the earlier x
```

### Storage form

```
let x = 42                  → global of the inferred type (i32)
let x: i64 = 42             → global of the written type (i64)
heap payload                → register GC root
                             (compiler-emitted shadow stack frame,
                              linked into rokr shadow stack chain)
```

Each top-level variable is stored in the host as an exported global of its type, written or inferred. When a later cell references the same variable it is found through that global's address.

### GC root registration

Top-level variables pointing to heap objects must be traced by the GC as roots, so they are linked into the [shadow stack](../000-meta/007-glossary.md#shadow-stack) chain.

See the shadow stack part of [Barriers and root tracking](../007-runtime/704-barriers-and-roots.md) for the detailed mechanism.

### Redeclaration of the same name

```rok
[1] let x = 1i32;            // x: i32
[2] let x = "hi";            // redeclaration at cell top level - new binding (x: str)
```

In [Variable declaration](../005-modules/501-variable-declaration.md#514-no-redeclaration-in-the-same-scope), redeclaration within the same scope is an ERROR. However, to support editing and re-running cells, the same name may be declared again with `let` at the top level of REPL and Jupyter cells. A redeclaration creates a new binding, whose type may differ from the previous binding. Later cells see the new binding. This exception applies only at cell top level, not to blocks inside a cell or to ordinary source files.

The existing global is then overwritten: its memory is left in place and only the name slot is redirected to the new definition.

This supports the common REPL pattern of redefining a variable.

## 9.7.4. Undo / cell re-execution

### Re-executing a cell in Jupyter

When a cell is edited and run again in Jupyter, it runs as a new cell. Items and bindings it redefines are used in their new form from later cells on (9.7.0 rules 2, 3). The effects of previously executed cells are not erased.

### Meaning of undo

The REPL has no explicit undo; work proceeds by overwriting the last cell or adding new cells.

In Jupyter as well, deleting or re-running a cell does not undo the effects of earlier cells. To return to the initial state, restart the session (`:reset` in the REPL).

## 9.7.5. Error modes

### Cell compile errors

```
parse error / type error / semantic error.
  - the cell is not added.
  - the session's bindings and items are unchanged.
  - reuse information for already compiled functions is kept.
  - the error is shown to the user.
```

This step is fully transactional, so the state just before the error is preserved. If rokc itself panics (internal compiler error) while compiling a cell, `rokc lsp`, `rokc repl` and `rokc jupyter` survive it: the language server reports it as an error diagnostic of the analyzed buffer, the REPL / kernel fails that cell; both keep serving.

### Cell runtime errors

```
panic / uncaught throw / process exit / segfault during execution.
  - the cell is not added - bindings and items stay as they were before the cell.
  - the error is shown to the user.
  - the next cell can proceed.
```

Bindings and items revert to the state before the cell (9.7.0 rule 5), but external effects that already happened (output, file writes, etc.) are not undone. A failed cell is not kept in the session, so the corrected cell can simply be entered again. Hosts without fork (Windows) save the session's memory and threads before the cell and put them back when it fails, with the same observable result as the forked snapshot.

### JIT failure

If the host cannot load or link a cell's code (an unknown `extern` symbol, etc.), it is handled like a cell runtime error. The forked process that ran the cell ends and the previous snapshot takes over.

### Locations

Diagnostics and panic locations in a cell name `<repl>:line:col`, counted over the cell's own text.

## 9.7.6. Cache

Within a session the host reuses functions it has already compiled (9.6.4). There is no disk cache across sessions.

## 9.7.7. Visualization / output

### REPL output

```
> 1 + 2
3
> "hi"
hi
```

The last expression's value is printed with `Display`. A trailing `?` (as in `xs?`) prints the `Debug` form instead, and `:type xs` prints the type alone. A value without the requested trait (`Display` for a bare expression, `Debug` for `?`) prints only its type name. A `let` cell prints nothing.

```
> let xs = [1, 2, 3];
> xs
List<i32>
> xs?
[1, 2, 3]
```

### Jupyter output

The result of each cell is sent as an `execute_result` with a single `text/plain` entry, the same text the REPL prints. No other MIME types are produced.

Jupyter integration details are defined in [Jupyter integration](908-jupyter-integration.md).

## 9.7.8. Input unit: REPL recognition

The REPL processes input in units of complete expressions or statements.

```
> 1 + 2
3
> let x =
... 42i32;          (continued input)
> x
42
```

If an expression or block is not closed, the `...` prompt is shown and input continues. When standard input ends inside an incomplete cell, the cell is not run, an error is reported, and `rokc repl` exits with 1. A cell that fails does not change the exit code.

## 9.7.9. Input unit: Jupyter cell

A Jupyter cell is a multi-line input processed all at once.

```
[Cell 1]:
let x = 42i32;
let y = x * 2i32;
y                       # last expression = cell result
```

The last expression of the cell is the cell's result; everything else is treated as side effects (definitions or calls).

## 9.7.10. Loading a program (`repl <path>`)

`rokc repl <path>` with a program path loads the program into a fresh session, then continues with the REPL on that session. With a library package path, the package is added as a dependency and `use <name>::*;` runs as the first cell (again after `:reset`); nothing else is loaded.

- **Checked first.** The whole program is checked before anything runs. Errors are reported at the file's own locations; nothing runs and the session starts empty.
- **Items.** Items and imports join the session.
- **Statements.** Top-level statements run as cells. Each cell ends at a top-level `let` (or at the last statement).
- **Failure.** On a panic, `exit()` or an escaping error, bindings from finished cells keep their values as of the end of their cell; assignments in the failed stretch are lost. The rest of the program does not run, and the session continues.
- **Unusable bindings.** A binding later cells cannot use (private or unnameable type) is dropped if no later statement names it; otherwise the statements through its last use run as one cell.
- **`fn main`.** A program with `fn main` runs `main` as the entry; `main` then leaves the session, so its locals are not visible.
- **Locations.** Diagnostics and panics use the file's own locations.
- **Input.** The program's stdin is `/dev/null`.

## 9.7.11. Next

[Jupyter integration](908-jupyter-integration.md) defines the Jupyter kernel integration over local ZMQ.
