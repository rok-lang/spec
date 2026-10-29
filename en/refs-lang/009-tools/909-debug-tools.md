# 9.9. Debug / diagnostic tools

This section defines the debug / diagnostic tools provided by `rokc` and their integration with external tools.

## 9.9.1. Intermediate output: `--emit`

The per-step results of the [AOT build pipeline](902-aot-pipeline.md) can be dumped.

```
rokc build --emit=ast           # full AST after meta expansion and desugaring, as rok source
rokc build --emit=hir          # rok-hir text output
rokc build --emit=mir           # rok-mir text output
rokc build --emit=mlir          # standard MLIR text
rokc build --emit=llvmir        # LLVM IR
rokc build --emit=asm           # assembly
rokc build --emit=obj           # object file
rokc build --emit=expanded      # meta expansion result (9.9.10)
```

`mlir` and `llvmir` come from the MLIR → LLVM backend at any optimization level; `asm` and `obj` come from the backend the level selects ([9.2.10](902-aot-pipeline.md#9210-backend-selection)).

The output is written to the following locations.

```
target/debug/build/foo.ast.rok
target/debug/build/foo.hir
target/debug/build/foo.mir
target/debug/build/foo.mlir
target/debug/build/foo.ll
target/debug/build/foo.s
target/debug/build/foo.o
target/debug/build/foo.expanded.rok
```

To select several steps at once, separate them with commas, as in `--emit=mlir,llvmir,asm`. With a single kind, `-o <path>` changes the location. With `--emit`, no executable or archive is produced.

## 9.9.2. ROKHIR inspection

```
rokc rokhir-text path/to/foo.rokhir      # or libfoo.a; reads the sibling foo.rokhir
```

This command prints the [ROKHIR](904-rokhir-bytecode.md) of a prebuilt library in human-readable form. After the version, producer and meta, each interface file follows as `--- <path>` and its text (source with concrete function bodies cut down to `;`).

```
rokhir 6
producer: rokc 0.1.0 (1f2a40-17f3a9c2e1b0c4d8)
package: geom (hash: 3f8a1c2b)
source: /home/me/geom
gc types: 4096..4160
features: tls

--- lib.rok
pub struct Point { pub x: f64; pub y: f64; }
pub fn distance(p1: Point, p2: Point) -> f64;
pub fn id<T>(x: T) -> T { x }
```

This output is useful for verifying the correctness and compatibility of [ROKHIR](../000-meta/007-glossary.md#rokhir-rok-hir-bytecode).

## 9.9.3. Symbol demangle

```
rokc demangle <symbol>
rokc demangle _Y3f8a1c2b_geom_distance_a7c91d8e44b2f190
```

This command prints a mangled symbol of [Symbol mangling](905-symbol-mangling.md) in its original form. Given several symbols, the blocks are separated by blank lines. Mangling does not keep the root `rok::`, so if the first path segment is `core` / `std` / `bare`, `rok::` is prepended to show it as a standard library path.

```
package:  3f8a1c2b
path:     geom
name:     distance
generics: a7c91d8e44b2f190 (type tuple hash)
```

## 9.9.4. Diagnostics / error messages

### Standard format

Diagnostic messages are printed in the following standard format.

```
error[E100]: cannot find value `count` in this scope
  --> src/main.rok:5:13
   |
 2 |     let counter = 0;
   |         ------- similar name defined here
...
 5 |     let x = count + 1;
   |             ^^^^^ not found in this scope
   |
   = help: did you mean `counter`?
```

This format uses the file + line + column diagnostic units of [Source files](../001-lexical/101-source-files.md).

The header is `<severity>[<code>]: <message>` and `--> path:line:col` gives the primary location. The primary span is underlined with `^`, secondary spans with `-`; a secondary span in another file is introduced by `::: path:line:col`. Trailing `= note:` lines add context and `= help:` lines suggest a fix. For a name not found that an item elsewhere has, the help gives the `use` lines that bring it in (up to three: the package's own items first, then the standard library's, then the dependencies'), or the missing `pub` when the package's only such item is hidden (``add `use geo::Point;` ``; `` `geo::secret` has that name, but it is private to module `geo`: add `pub` to it ``).

The primary span is the code at fault: for a value of the wrong type, the value, not the statement that holds it. Messages name library items as they are written (`List::sort`, `Send`, `print`, not `rok::core::marker::Send`). The human format leaves out citations of this specification (`(spec 3.12.5)`); the JSON format keeps them in the text and lists them in `spec`.

### JSON format (tool integration)

For tool integration, output can also be produced as JSON.

```
rokc build --message-format=json
```

Each diagnostic is one line of JSON object on stdout (the human format goes to stderr).

```json
{"type":"error","code":"E200","kind":"type","message":"...","spans":[{"file_name":"src/main.rok","line_start":5,"col_start":10,"byte_start":71,"byte_end":76,"is_primary":true,"label":"..."}],"notes":["..."],"spec":["..."]}
```

| Field      | Meaning                                                            |
|------------|--------------------------------------------------------------------|
| `type`     | `error` / `fatal` / `warning` / `note`                             |
| `code`     | Code of the error kind (table below)                               |
| `kind`     | Name of the error kind                                             |
| `message`  | Message                                                            |
| `spans`    | List of locations. The first element is the primary location (`is_primary: true`); the rest are secondary |
| `notes`    | List of note strings; a help entry starts with `help: `            |
| `spec`     | Sections the message and notes cite (`spec 3.12.5`), which the human format leaves out |

A span has `file_name`, `line_start`, `col_start`, `byte_start`, `byte_end`, `is_primary` and `label`. A diagnostic that knows only the file and no position has just `{"file_name": .., "is_primary": true}`.

This format lets IDEs and CI systems read diagnostics mechanically.

### Diagnostic categories

Diagnostic codes are classified by area as follows.

| Code prefix     | Area                   |
|-----------------|------------------------|
| `E001` ~ `E099` | Lexical / syntax       |
| `E100` ~ `E199` | Name resolution / visibility |
| `E200` ~ `E299` | Type system            |
| `E300` ~ `E399` | Meta / magic           |
| `E400` ~ `E499` | Build / link           |
| `E500` ~        | Additional categories  |

There is one code per error kind. Diagnostics of the same kind have the same code.

| Code   | `kind`        | Kind                         |
|--------|---------------|------------------------------|
| `E001` | `lexical`     | Lexical                      |
| `E002` | `parse`       | Syntax                       |
| `E100` | `resolve`     | Name resolution              |
| `E101` | `visibility`  | Visibility                   |
| `E102` | `module`      | Module / file                |
| `E200` | `type`        | Type                         |
| `E201` | `pattern`     | Pattern / exhaustiveness     |
| `E202` | `handoff`     | Use after hand-off ([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers)) |
| `E300` | `meta`        | Meta block / attribute / keyword magic |
| `E400` | `lowering`    | HIR / MIR lowering           |
| `E401` | `codegen`     | Code generation              |
| `E402` | `linkage`     | External tool / link         |
| `E500` | `unsupported` | Syntax the compiler does not handle |
| `E501` | `internal`    | Compiler internal            |
| `E502` | `other`       | Other                        |
| `E503` | `lint`        | lint ([6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control), `rokc lint`) |

This specification guarantees the per-area code prefixes. When individual codes are introduced, they are subdivided within the same area.

## 9.9.5. Logging / progress display

### Standard output

Default build output looks like this.

```
   Compiling foo (path/to/foo)
    Building target/debug/libfoo.a
    Finished in 1.23s
```

Progress lines go to stderr; the list of verbs is in [9.1.8](901-rokc-cli.md#918-output-format).

### `--verbose` / `-v`

Verbose mode also shows per-stage timing.

```
   Compiling foo (path/to/foo)
   Stage: front end (134ms)
    Building target/debug/foo
   Stage: codegen + link (812ms)
    Finished in 0.95s
```

Showing the time of each stage is useful for diagnosing build bottlenecks. The stages are `front end` and `codegen + link` (`emit <ext>` with `--emit`).

### `ROK_TIME_PASSES`

`ROK_TIME_PASSES=1` prints the wall time of each compiler pass on stderr.

### `-vv` (very verbose)

In addition to `-v`, prints a `Reading <file>` line for each file processed and a `Specialized <path>(sig)` line for each (item, type) pair [specialized](../000-meta/007-glossary.md#specialize).

## 9.9.6. Debugger integration: gdb / lldb

### DWARF debug information

A `debug` profile build, and a `release` build with `-g` ([9.2.15](902-aot-pipeline.md#9215-output)), emits DWARF debug information with the following properties. For Windows targets it emits CodeView in a PDB instead, carrying the same information.

- A function's `DW_AT_name` is its Rok path (`geom::distance`); no linkage name (mangled symbol) is emitted, because debuggers do not understand `_Y` mangling.
- Line numbers refer to lines of the original .rok file.
- Local variables appear with their original names and types. Besides primitive types (integers, floating-point, `bool`, `char`), DWARF types are emitted for structs (field names), tuples, `str`, `any`, sum types and arrays. A sum type or enum is a DWARF variant part: an artificial discriminant member, and one `DW_TAG_variant` per variant (dataless variants included) with `DW_AT_discr_value`, each holding a struct named `<Sum>::<Variant>`. Both back ends write the same form, and compile units use `DW_LANG_Rust` so debuggers read variant parts (there is no DWARF language code for Rok). A `GcBuf<T>` reference is described as a pointer to its object (header, then the `T` elements). An `any` value is a variant part over the types stored into it, with the stored tag as the discriminant. The discriminant is an artificial `i32` (the stored tag; a `throws` carrier's `bool` tag is a byte). A function value (a `fn` type or a closure) is a structure named for its type (`fn(i32) -> i32`) with members `code`, a pointer to a `DW_TAG_subroutine_type` (the environment, then the parameters), and `env`, its environment (a pointer to the environment object when the closure's type is known), so a debugger shows the target function. A variable captured by reference shows as its value. Functions the compiler makes (the C `main`, static initializer guards) have subprograms too. Compiler-generated locals (names beginning `#` or `__`) are omitted.
- A lexical block scope is emitted for each block, so shadowed names are visible only inside their block.
- Locals with stack slots are placed in a single `llvm.alloca` in both debug and release, so `-g` attaches debug info to the same slot regardless of optimization level.

### Using gdb / lldb

Standard gdb and lldb commands work on Rok code as is.

```
gdb ./target/debug/my_binary
(gdb) break main
(gdb) run
(gdb) print x
$1 = (i32) 42          # primitive-type variables only
(gdb) bt
#0  main at src/main.rok:5
...
```

### Formatters

gdb shows only the active variant of a sum, an enum or an `any` from the DWARF alone. lldb has no plugin for the unit's language and lists every variant; the toolchain ships lldb formatters, `lib/rok/lldb/rok_lldb.py`, that show the active variant only (its name as the summary, its fields as the children; a boxed `any` payload through its box). `rokc --print lldbinit` ([9.1.4](901-rokc-cli.md#rokc---print-lldbinit-lldb-formatters)) prints the lldb command that loads them:

```
lldb -O "$(rokc --print lldbinit)" ./target/debug/my_binary
rokc --print lldbinit >> ~/.lldbinit
```

### macOS / Windows debuggers

Each platform's native debugger handles Rok binaries.

- On macOS, use lldb or the Xcode debugger.
- On Windows, use the Visual Studio debugger or WinDbg (PDB debug information).

## 9.9.7. Profiler integration

### perf (Linux)

CPU profiling can be done with `perf`.

```
perf record ./target/release/my_binary
perf report
```

Rok functions are shown by their demangled names.

### Profile-guided optimization

Profile-guided optimization ([9.1.2](901-rokc-cli.md#rokc-build-aot-build)) takes three steps:

```
rokc build --release -C profile-generate=prof    # 1. instrumented build
./target/release/my_binary <representative input> # 2. run; writes prof/default_*.profraw
rokc profdata merge -o app.profdata prof          # 3. merge, then
rokc build --release -C profile-use=app.profdata  #    rebuild with the profile
```

An optional context-sensitive stage builds with `-C profile-use=app.profdata -C cs-profile-generate=cs`, runs again, and merges `cs` with `app.profdata` into the final profile. `wasm32-wasi` is supported; `wasm32-unknown-unknown` is not.

### Tracy / Chrome trace (optional)

Through the standard library's tracing surface, traces compatible with `tracy` or Chrome's `chrome://tracing` can be produced. The choice of a specific library is outside the scope of this specification and is covered by a separate library spec.

## 9.9.8. Memory / GC tools

The runtime reads the following environment variables.

| Variable                  | Effect                                                                       |
|---------------------------|------------------------------------------------------------------------------|
| `ROK_GC_LOG=1`            | Prints the kind (minor / full), pause time, mark count and sweep result of each collection to stderr. A concurrent full collection logs its start pause (a minor collection) and its finish pause separately; the sweep result is the live bytes before -> after |
| `ROK_GC_STATS=1`          | At exit, prints collection counts (minor / full), pause time and collector CPU time to stderr. The second line gives block count / resident size / evacuation count / bytes moved |
| `ROK_GC_CONCURRENT=auto\|always\|off` | Concurrent marking for full collections. `auto` applies when old objects are 32 MiB or more ([7.5](../007-runtime/705-concurrent-collection.md)) |
| `ROK_GC_EVAC=always`      | For testing. Evacuation on every full collection ([7.3.1](../007-runtime/703-gc-algorithm.md#731-immix-mark-region)) |
| `ROK_GC_STRESS=N`         | For testing. Collects every N allocations (every 8th of those is full)       |
| `ROK_HEAP_DUMP=on_oom`    | On OOM, prints to stderr the totals, the top 32 types by bytes and the 32 largest large objects. Best effort: it allocates nothing and takes no lock. Format is runtime-defined |
| `ROK_WORKERS=<n>`         | Number of executor workers N (default: the online CPUs) ([7.6.8](../007-runtime/706-scheduler.md#768-thread-per-core-option)) |
| `ROK_THREAD_PER_CORE=1`   | Thread-per-core: tasks pinned to their worker, no stealing (7.6.8) |
| `ROK_DYNAMIC_WORKERS=1`   | Idle workers park and are taken back when every active worker is busy ([7.7.8](../007-runtime/707-io-and-timer.md#778-dynamic-adjustment-of-worker-count)) |

An unset or empty variable has its default. A value outside those listed prints one warning line on stderr naming the variable, the value and the accepted values, and the variable has its default.

## 9.9.9. REPL / Jupyter debugging

| Aid                  | Effect                                                               |
|----------------------|----------------------------------------------------------------------|
| `<expr>?`            | Evaluates the expression and prints its `Debug` form ([9.7.7](907-jit-cells.md#977-visualization--output)) |
| `it`                 | The last value an expression cell showed; a user's `it` wins ([9.7.7](907-jit-cells.md#977-visualization--output)) |
| `:type <expr>`, `:t` | Prints the static type of the expression (without running it)        |
| `:doc <name>`        | Prints the name's signature and doc comment, the text of a Jupyter `inspect_request` ([9.8.6](908-jupyter-integration.md#986-inspect_request-help)); a method as `:doc xs.len` |
| `:time <expr>`       | Evaluates the expression as a cell and prints the time of each stage (as `ROK_REPL_TIME=1`) and the total on stderr |
| `:vars`              | Lists the bindings as `name: type`, then the items' signatures       |
| `:load <file>`       | Loads a program into the running session ([9.7.10](907-jit-cells.md#9710-loading-a-program-repl-path)); its items replace those of the same name. A script written by `:save` runs its cells again one by one |
| `:reload`            | Loads the last loaded program again                                  |
| `:edit`              | Opens `$VISUAL` or `$EDITOR` (else `vi`; `notepad` on Windows) on a scratch file, then runs what it holds as cells; after a failed cell the text opens again on the next `:edit` |
| `:save <file>`       | Writes the session as a script: the items, then the cells that ran. A name declared again with the same type is assigned; with another type, it opens a block that lasts to the end of the file ([5.1.4](../005-modules/501-variable-declaration.md#514-no-redeclaration-in-the-same-scope)). `it` is declared only where a later cell reads it. The first line is `// rok repl session (:save); :load replays it cell by cell` and a `// :cell` line comes before each cell: `:load` of the script runs the cells again one by one, so a name declared again keeps its last binding; `rokc run` runs it as a program |
| `:opt [<level>]`     | Prints, or sets for later cells, the optimization level (`0`, `1`, `2`, `3`, `s`, `z`, as `-O`) |
| `:reset`             | Resets the session to its initial state (the `use` of a library path is inserted again) |
| `:add <dep> ..`      | Adds dependencies (`rok install` into the current environment, or `rok add` in a package's session) and attaches them to the session ([9.10.5](910-rok-frontend.md#9105-environments)) |
| `:help`, `:h`        | Lists every command and alias                                        |
| `:quit`, `:q`, `:exit` | Quit (end of input also quits)                                     |
| `exit`, `quit`, `exit()`, `quit()` | Alone in a cell, quit; unless the session defines the name (a binding, an item or a `use`), which wins |
| `ROK_REPL_TIME=1`    | Per cell, per-stage timing and the number of newly compiled functions / functions reused from earlier cells ([9.6.4](906-jit-engine.md#964-reuse-of-library-code)) |

To see a cell's MLIR / LLVM IR, put the same code in `main.rok` and use `rokc build --emit=mlir,llvmir` (9.9.1).

## 9.9.10. Meta block debugging

You can check what code the derives / attributes of [user-defined meta blocks](../006-meta-system/608-meta-blocks.md) generate.

```
rokc build --emit=expanded      # → target/<profile>/build/<name>.expanded.rok
```

This option prints the result of [meta expansion](../006-meta-system/609-template-tokens.md) of the user package as source text. Each expansion's generated source follows a comment like the one below; when generated code is expanded again, that result follows the outer expansion.

```
// expanded from this #[derive(MyHash)] (src/geom.rok:12:1)
impl MyHash for Point { ... }
```

To see the full AST after all meta expansion and desugaring as Rok source, use `--emit=ast` (`<name>.ast.rok`).

## 9.9.11. Benchmarks

Microbenchmarks are run with the following command.

```
rok bench
```

`rokc bench` builds the benchmark harness and `rok bench` ([9.10](910-rok-frontend.md)) runs it ([9.1.2](901-rokc-cli.md#rokc-bench-benchmarks)), measuring the functions marked `#[bench]` one at a time in an `-O2` build. Each takes a `Bench` handle and times a closure through `b.iter`, so setup outside the closure is not measured. The measurement is:

1. **Clock.** A monotonic nanosecond clock. Its fixed overhead is amortized by timing `iters` calls per sample, never a single call.
2. **Calibration.** Unless `--iters` is given, `iters` is raised until one sample takes about 100 ms, so clock quantization and overhead are negligible.
3. **Warm-up.** Calls are run for about one sample's worth of time and discarded, to settle caches, branch predictors and paging.
4. **Sampling.** `--samples` samples (default 100) are taken; each records the total time of `iters` calls, and the per-call time is that total divided by `iters`.
5. **Statistics.** Mean, median, standard deviation, median absolute deviation (MAD), min and max are computed. Samples outside the Tukey fences (1.5 IQR mild, 3 IQR severe) are reported as outliers but kept out of the summary figure. The reported figure is the **median (+/- MAD)**, which is robust to an occasional slow sample.
6. **GC awareness.** The runtime's collection counter is read around each sample; a sample during which a collection ran is classed as an outlier and counted separately, so GC pauses do not distort the median.

`black_box(x)` is an identity the optimizer cannot see through; apply it to the inputs and result of the timed closure so the work is neither hoisted out nor folded away. `b.bytes(n)` / `b.items(n)` turn the per-call time into a throughput (`MiB/s`, `Melem/s`).

Terminal output is aligned and in human units; non-terminal output is a stable one-line-per-bench machine form for regression tools. `--iters` / `--samples` adjust the sampling.

## 9.9.12. CI / test integration

### CI mode

In CI environments, machine-readable output is recommended.

```
rokc build --message-format=json --no-color > build.log
```

This output is suitable for CI systems to parse.

### Exit codes

The exit codes of the [`rokc` CLI interface](901-rokc-cli.md) are defined as follows.

| Code  | Meaning          |
|-------|------------------|
| `0`   | Success          |
| `1`   | User code error  |
| `2`   | CLI error        |
| `101` | rokc internal panic |

`rokc test` builds the harness; it does not run the tests. `rok test` runs them and exits with `1` if any fails. Each test runs in a separate process, so a test's panic (`101`) does not become the runner's `101`.

CI can apply different handling per code (retry, notification, etc.).

## 9.9.13. LSP server

`rokc` provides an LSP (Language Server Protocol) mode. The same server is also provided as the standalone binary `rokls` (= `rok lsp`). It communicates over stdio and provides diagnostics, hover (including members), go-to-definition (returning the range of the defining name), references, rename, document symbols, member / path completion, formatting and semantic tokens (macro calls, user-defined keywords, literal suffixes, operators and attributes, which the static grammar cannot recognize). References and rename are buffer-local.

```
rokc lsp
rokls
```

In this mode, LSP clients such as VS Code, Vim and Emacs receive completion, hover, go-to-definition, diagnostics and so on. Internally it uses the incremental results of `rokc check`.

- **Completion.** After `x.`, the fields and methods of `x`'s type. Of a type from the standard library or another package only the `pub` ones are offered, without `#[doc(hidden)]` members and without the methods of operator traits (those written as `a + b`, `xs[i]` and so on). After `a::`, also in `use a::b::`, the modules and items under `a`. Elsewhere, the locals in scope at the cursor, the buffer's items, the names in scope without a path (the core library's and those the `use` items import) and keywords. The REPL and the Jupyter kernel complete with the same engine ([9.7.8](907-jit-cells.md#978-input-unit-repl-recognition)).
- **Hover and definition.** Hover on a function, method or field, the standard library's included, shows its signature and then its doc comment. Go-to-definition on `x.name` looks `name` up in the type of `x`; on a standard library item it goes to the item in the library's source.
- **Document symbols.** A symbol's range covers the whole item, from its modifiers (`pub`, `const`, ..) to its closing `}` or `;`; its selection range is the name.
- **Diagnostic ranges.** A diagnostic whose span is empty is reported over the token at that position, or at the end of a line over the token before it.
- **Started by hand.** On a terminal the server first prints ``rokls speaks the language server protocol on stdin; start it from an editor (see `rok help lsp`)`` to standard error. When the input ends before the client sends `initialize`, it prints ``rokls: input ended before the editor sent `initialize` `` and exits with 1. After `initialize` the exit code is 0 after `shutdown` and 1 otherwise, as the LSP standard says.

The detailed LSP specification is outside this specification (the LSP standard + rokc implementation decisions) and is defined separately.

## 9.9.14. Package inspection

Information about a prebuilt library package can be viewed at once.

```
rokc inspect path/to/libfoo.a        # or foo.rokhir; the sibling file is read too
```

This command shows information from the ROKHIR ([9.4](904-rokhir-bytecode.md)) and the [archive](../000-meta/007-glossary.md#archive) as follows.

```
Package: foo
Hash: 3f8a1c2b
rokc version: 0.1.0
ROKHIR version: 1
GC type ids: 4096..4160

Exports:
  pub fn distance(p1: Point, p2: Point) -> f64
  pub struct Point
  ...

Generic items (ROKHIR):
  pub fn id<T>(x: T) -> T
  ...

Concrete .o files:
  foo.o (15 KB)
```

`rokc rokhir-text <file>` prints the ROKHIR's version, producer, meta and all interface files (9.9.2).

This gives an at-a-glance view of what the package exposes externally.

## 9.9.15. Summary

The tools above form the core of Rok's development and debugging experience. Integration with standard tools (gdb, perf, jupyter) combined with rok-specific tools (`rokc demangle`, `rokc rokhir-text`, etc.) provides a rich diagnostic environment.

This concludes chapter 9 (execution and tools) and completes the overall picture of this specification.
