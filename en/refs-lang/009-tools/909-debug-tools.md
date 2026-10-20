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

## 9.9.2. RHIR inspection

```
rokc rhir-text path/to/foo.rhir      # or libfoo.a — reads the sibling foo.rhir
```

This command prints the [RHIR](904-rhir-bytecode.md) of a prebuilt library in human-readable form. After the version, producer and meta, each interface file follows as `--- <path>` and its text (source with concrete function bodies cut down to `;`).

```
rhir 1
producer: rokc 0.1.0 (1f2a40-17f3a9c2e1b0c4d8)
package: geom (hash: 3f8a1c2b)
source: /home/me/geom
gc types: 4096..4160

--- lib.rok
pub struct Point { pub x: f64; pub y: f64; }
pub fn distance(p1: Point, p2: Point) -> f64;
pub fn id<T>(x: T) -> T { x }
```

This output is useful for verifying the correctness and compatibility of [RHIR](../000-meta/007-glossary.md#rhir-rok-hir-bytecode).

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

The header is `<severity>[<code>]: <message>` and `--> path:line:col` gives the primary location. The primary span is underlined with `^`, secondary spans with `-`; a secondary span in another file is introduced by `::: path:line:col`. Trailing `= note:` lines add context and `= help:` lines suggest a fix.

### JSON format (tool integration)

For tool integration, output can also be produced as JSON.

```
rokc build --message-format=json
```

Each diagnostic is one line of JSON object on stdout (the human format goes to stderr).

```json
{"type":"error","code":"E200","kind":"type","message":"...","spans":[{"file_name":"src/main.rok","line_start":5,"col_start":10,"byte_start":71,"byte_end":76,"is_primary":true,"label":"..."}],"notes":["..."]}
```

| Field      | Meaning                                                            |
|------------|--------------------------------------------------------------------|
| `type`     | `error` / `fatal` / `warning` / `note`                             |
| `code`     | Code of the error kind (table below)                               |
| `kind`     | Name of the error kind                                             |
| `message`  | Message                                                            |
| `spans`    | List of locations. The first element is the primary location (`is_primary: true`); the rest are secondary |
| `notes`    | List of note strings; a help entry starts with `help: `            |

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

`rokc build -g` or `--debug` (synonyms, unrelated to the profile name `debug`) emits DWARF debug information with the following properties. For Windows targets it emits CodeView in a PDB instead, carrying the same information.

- A function's `DW_AT_name` is its rok path (`geom::distance`); no linkage name (mangled symbol) is emitted, because debuggers do not understand `_Y` mangling.
- Line numbers refer to lines of the original .rok file.
- Local variables appear with their original names and types. Besides primitive types (integers, floating-point, `bool`, `char`), DWARF types are emitted for structs (field names), tuples, `str`, `any`, sum types and arrays. A sum type or enum is a DWARF variant part: an artificial discriminant member, and one `DW_TAG_variant` per variant (dataless variants included) with `DW_AT_discr_value`, each holding a struct named `<Sum>::<Variant>`. Both back ends write the same form, and compile units use `DW_LANG_Rust` so debuggers read variant parts (there is no DWARF language code for rok). A `GcBuf<T>` reference is described as a pointer to its object (header, then the `T` elements). Functions the compiler makes (the C `main`, static initializer guards) have subprograms too.
- A lexical block scope is emitted for each block, so shadowed names are visible only inside their block.
- Locals with stack slots are placed in a single `llvm.alloca` in both debug and release, so `-g` attaches debug info to the same slot regardless of optimization level.

### Using gdb / lldb

Standard gdb and lldb commands work on rok code as is.

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

### macOS / Windows debuggers

Each platform's native debugger handles rok binaries.

- On macOS, use lldb or the Xcode debugger.
- On Windows, use the Visual Studio debugger or WinDbg (PDB debug information).

## 9.9.7. Profiler integration

### perf (Linux)

CPU profiling can be done with `perf`.

```
perf record ./target/release/my_binary
perf report
```

rok functions are shown by their demangled names.

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
| `ROK_GC_LOG=1`            | Prints the kind (minor / full), pause time, mark count and sweep result of each collection to stderr |
| `ROK_GC_STATS=1`          | At exit, prints collection counts (minor / full), pause time and collector CPU time to stderr. The second line gives block count / resident size / evacuation count / bytes moved |
| `ROK_GC_CONCURRENT=auto\|always\|off` | Concurrent marking for full collections. `auto` applies when old objects are 32 MiB or more ([7.5](../007-runtime/705-concurrent-collection.md)) |
| `ROK_GC_EVAC=always`      | For testing. Evacuation on every full collection ([7.3.1](../007-runtime/703-gc-algorithm.md#731-immix-mark-region)) |
| `ROK_GC_STRESS=N`         | For testing. Collects every N allocations (every 8th of those is full)       |
| `ROK_HEAP_DUMP=on_oom`    | On OOM, prints the heap's object distribution and large objects to stderr. Format is runtime-defined |

## 9.9.9. REPL / Jupyter debugging

| Aid                  | Effect                                                               |
|----------------------|----------------------------------------------------------------------|
| `:type <expr>`       | Prints the type of the expression (without running it)               |
| `:reset`             | Resets the session to its initial state (the `use` of a library path is inserted again) |
| `:add <dep> ..`      | Adds dependencies to the current environment and attaches them to the session (via `rok`, [9.10.5](910-rok-frontend.md#9105-environments)) |
| `:help`, `:quit`     | Help, quit                                                           |
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

To see the full AST after all meta expansion and desugaring as rok source, use `--emit=ast` (`<name>.ast.rok`).

## 9.9.11. Benchmarks

Microbenchmarks are run with the following command.

```
rokc bench
```

This command ([9.1.2](901-rokc-cli.md#rokc-bench-benchmarks)) runs the functions marked `#[bench]` one at a time in an `-O2` build and prints the mean time per call and the standard deviation across samples as `ns/iter (+/- sd)`. `--iters` / `--samples` adjust the sampling.

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

`rokc test` exits with `1` if any test fails. Each test runs in a separate process, so a test's panic (`101`) does not become rokc's `101`.

CI can apply different handling per code (retry, notification, etc.).

## 9.9.13. LSP server

`rokc` provides an LSP (Language Server Protocol) mode. The same server is also provided as the standalone binary `rokls` (= `rok lsp`). It communicates over stdio and provides diagnostics, hover (including members), go-to-definition (returning the range of the defining name), references, rename, document symbols, member / path completion and formatting. References and rename are buffer-local.

```
rokc lsp
rokls
```

In this mode, LSP clients such as VS Code, Vim and Emacs receive completion, hover, go-to-definition, diagnostics and so on. Internally it uses the incremental results of `rokc check`.

The detailed LSP specification is outside this specification (the LSP standard + rokc implementation decisions) and is defined separately.

## 9.9.14. Package inspection

Information about a prebuilt library package can be viewed at once.

```
rokc inspect path/to/libfoo.a        # or foo.rhir — the sibling file is read too
```

This command shows information from the RHIR ([9.4](904-rhir-bytecode.md)) and the [archive](../000-meta/007-glossary.md#archive) as follows.

```
Package: foo
Hash: 3f8a1c2b
rokc version: 0.1.0
RHIR version: 1
GC type ids: 4096..4160

Exports:
  pub fn distance(p1: Point, p2: Point) -> f64
  pub struct Point
  ...

Generic items (RHIR):
  pub fn id<T>(x: T) -> T
  ...

Concrete .o files:
  foo.o (15 KB)
```

`rokc rhir-text <file>` prints the RHIR's version, producer, meta and all interface files (9.9.2).

This gives an at-a-glance view of what the package exposes externally.

## 9.9.15. Summary

The tools above form the core of rok's development and debugging experience. Integration with standard tools (gdb, perf, jupyter) combined with rok-specific tools (`rokc demangle`, `rokc rhir-text`, etc.) provides a rich diagnostic environment.

This concludes chapter 9 (execution and tools) and completes the overall picture of this specification.
