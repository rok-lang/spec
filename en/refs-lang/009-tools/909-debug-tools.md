# 9.9. Debug / diagnostic tools

This section defines the debug / diagnostic tools provided by `rokc` and their integration with external tools.

## 9.9.1. Intermediate output: `--emit`

The per-step results of the [AOT build pipeline](902-aot-pipeline.md) can be dumped.

```
rokc build --emit=source        # full AST after meta expansion and desugaring, as rok source
rokc build --emit=hir          # rok-hir text output
rokc build --emit=mir           # rok-mir text output
rokc build --emit=mlir          # standard MLIR text
rokc build --emit=llvmir        # LLVM IR
rokc build --emit=asm           # assembly
rokc build --emit=obj           # object file
rokc build --emit=expansions    # meta expansion result (9.9.10)
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
rokc inspect --dump path/to/foo.rokhir   # or libfoo.a; reads the sibling foo.rokhir
```

This command prints the [ROKHIR](904-rokhir-bytecode.md) of a prebuilt library in human-readable form. After the version, producer and meta, each interface file follows as `--- <path>` and its text (source with concrete function bodies cut down to `;`).

```
rokhir 9
producer: rokc 0.2.0 (81a1b80-18e41085f35ccdd9-18be644-6ae7cf0f.6bda9c9)
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
error[E0027]: cannot find value `count` in this scope
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

The primary span is the code at fault: for a value of the wrong type, the value, not the statement that holds it. Messages name library items as they are written (`List::sort`, `Send`, `print`, not `rok::core::marker::Send`). Messages do not cite sections of this specification; what to write instead is in the message, the help lines and `rokc --explain <code>`.

### JSON format (tool integration)

For tool integration, output can also be produced as JSON.

```
rokc build --message-format=json
```

Each diagnostic is one line of JSON object on stdout; no human-format diagnostic is printed, and stderr holds only progress lines.

```json
{"type":"error","code":"E0040","kind":"type","message":"...","spans":[{"file_name":"src/main.rok","line_start":5,"col_start":10,"byte_start":71,"byte_end":76,"is_primary":true,"label":"...","line_end":5,"col_end":15}],"notes":["..."],"fixes":[],"help":["..."],"lint":null,"schema":2}
```

| Field      | Meaning                                                            |
|------------|--------------------------------------------------------------------|
| `type`     | `error` / `fatal` / `warning` / `note`                             |
| `code`     | The error code, `E0040` ([error codes](#error-codes))             |
| `kind`     | The area the code belongs to (`type`, `parse`, ..)                 |
| `message`  | Message                                                            |
| `spans`    | List of locations. The first element is the primary location (`is_primary: true`); the rest are secondary |
| `notes`    | List of note strings, without the help entries                     |
| `fixes`    | Edits that fix the code, each a span of one file with the `text` that replaces it and a `title` (below); often empty |
| `help`     | The help entries (the `= help:` lines of the human format), without the `help: ` prefix |
| `lint`     | The lint a warning or lint error belongs to (`unused_variables`), the name `#[allow(..)]` and `-A` take ([6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control)); `null` for the others |
| `schema`   | The version of this format, now `2`                                |

A span has `file_name`, `line_start`, `col_start`, `byte_start`, `byte_end`, `is_primary`, `label`, `line_end` and `col_end`. `line_end` / `col_end` are where the span ends (the position just past its last character), counted like `line_start` / `col_start` in code points; they are `null` when the source text is not at hand. A diagnostic that knows only the file and no position has just `{"file_name": .., "is_primary": true}`. A fix has `file_name`, `line_start`, `col_start`, `byte_start`, `byte_end`, `text`, `title` (``convert with `as i32` ``), `line_end` and `col_end`; the language server offers it as a quick fix ([9.9.13](#9913-lsp-server)).

Within one `schema`, fields are only added, never removed, renamed or given another meaning, so a reader should ignore fields it does not know. Schema `2` (0.2.0) changed two fields and removed one: `code` is the code of the error kind instead of its area (below), `notes` no longer repeats the help entries, which are only in `help`, and `spec`, the spec section of the rule, is gone. The test, bench and `rok deps` records ([9.9.12](#9912-ci--test-integration), [9.10.4](910-rok-frontend.md#9104-dependencies-rok-add--remove--update--deps)) follow the same rule with their own `schema`, still `1`.

This format lets IDEs and CI systems read diagnostics mechanically.

### Error codes

Every kind of error has its own code, `E` and four digits: `E0040` for mismatched types, `E0027` for a name that is not found. Codes are numbered in the order they were added, not by area; the `kind` field gives the area. A code is never reused: when a kind of error goes away its code stays reserved, and a new kind takes the next free number. A diagnostic of a lint ([6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control)) has the lint code `E0132` whatever part of the compiler raised it, and names the lint in its note and in `lint`.

`rokc --explain <code>` (also `rok --explain`) prints the code's page: its title, what it means and what to write instead ([9.1.5](901-rokc-cli.md#915-global-options)). `E0040`, `e0040` and `40` name the same code; a code that does not exist is a usage error (exit code 2). The error pages of the documentation are the same pages, one per code.

The titles are written exactly as `rokc --explain` prints them.

| Code    | `kind`        | Title                        |
|---------|---------------|------------------------------|
| `E0001` | `lexical` | unexpected character in source |
| `E0002` | `lexical` | bidirectional control character in source |
| `E0003` | `lexical` | malformed numeric literal |
| `E0004` | `lexical` | literal out of range for its type |
| `E0005` | `lexical` | unterminated literal or comment |
| `E0006` | `lexical` | invalid escape in a literal |
| `E0007` | `lexical` | a character literal holds one character |
| `E0008` | `lexical` | malformed f-string |
| `E0009` | `lexical` | name cannot be a raw identifier |
| `E0010` | `parse` | unexpected token |
| `E0011` | `parse` | nesting too deep |
| `E0012` | `parse` | syntax from another language |
| `E0013` | `parse` | struct literal in a block head |
| `E0014` | `parse` | modifiers out of order |
| `E0015` | `parse` | doc comment documents nothing |
| `E0016` | `parse` | visibility where it has no meaning |
| `E0017` | `parse` | item in the wrong place |
| `E0018` | `parse` | `return` with a value in top-level statements |
| `E0019` | `parse` | operators that need parentheses |
| `E0020` | `parse` | `return` in a trailing closure |
| `E0021` | `parse` | reserved word used as a name |
| `E0022` | `parse` | `let` without a value |
| `E0023` | `parse` | invalid function signature |
| `E0024` | `parse` | invalid generic parameters |
| `E0025` | `parse` | unsupported ABI |
| `E0026` | `resolve` | name defined more than once |
| `E0027` | `resolve` | cannot find value or function |
| `E0028` | `resolve` | cannot find type or trait |
| `E0029` | `resolve` | cannot find enum variant |
| `E0030` | `module` | unresolved import |
| `E0031` | `resolve` | ambiguous name |
| `E0032` | `module` | conflicting imports |
| `E0033` | `module` | invalid re-export |
| `E0034` | `visibility` | item is private |
| `E0035` | `module` | module file problem |
| `E0036` | `module` | invalid program entry point |
| `E0037` | `module` | invalid dependency |
| `E0038` | `resolve` | variable named like a nested function |
| `E0039` | `resolve` | nested item uses an outer local |
| `E0040` | `type` | mismatched types |
| `E0041` | `type` | branches of different types |
| `E0042` | `type` | type annotations needed |
| `E0043` | `type` | no such field |
| `E0044` | `type` | missing field in a literal |
| `E0045` | `type` | field given more than once |
| `E0046` | `type` | no such method or associated item |
| `E0047` | `type` | item used in the wrong form |
| `E0048` | `type` | wrong number of arguments |
| `E0049` | `type` | invalid named argument |
| `E0050` | `type` | `ref` parameter and argument mismatch |
| `E0051` | `type` | invalid spread argument |
| `E0052` | `type` | value is not callable |
| `E0053` | `type` | invalid cast |
| `E0054` | `type` | operator cannot be applied |
| `E0055` | `type` | literal fits several types |
| `E0056` | `type` | invalid indexing |
| `E0057` | `type` | invalid assignment |
| `E0058` | `type` | invalid closure capture |
| `E0059` | `type` | keyword outside its context |
| `E0060` | `type` | `let .. else` that does not diverge |
| `E0061` | `type` | operation needs `unsafe` |
| `E0062` | `type` | error not handled |
| `E0063` | `type` | thrown value is not an `Error` |
| `E0064` | `type` | optional operator on a value that is not optional |
| `E0065` | `type` | lock guard not released by `using` |
| `E0066` | `type` | `using` on a value without `Drop` |
| `E0067` | `type` | block statement leaves a value |
| `E0068` | `type` | value cannot be iterated |
| `E0069` | `type` | trait bound not satisfied |
| `E0070` | `type` | not a trait |
| `E0071` | `type` | trait used as a type |
| `E0072` | `type` | impl is missing a trait item |
| `E0073` | `type` | impl item is not in the trait |
| `E0074` | `type` | impl item does not match the trait |
| `E0075` | `type` | conflicting implementations |
| `E0076` | `type` | impl of a foreign trait for a foreign type |
| `E0077` | `type` | supertrait not implemented |
| `E0078` | `type` | impl not allowed |
| `E0079` | `type` | cannot evaluate at compile time |
| `E0080` | `type` | definition depends on itself |
| `E0081` | `type` | invalid const generic argument |
| `E0082` | `type` | wrong generic arguments |
| `E0083` | `type` | invalid enum value |
| `E0084` | `type` | invalid foreign-function use |
| `E0085` | `type` | value cannot cross threads |
| `E0086` | `type` | invalid union |
| `E0087` | `type` | ambiguous associated item |
| `E0088` | `type` | invalid format |
| `E0089` | `type` | invalid compiler built-in call |
| `E0090` | `type` | invalid coroutine use |
| `E0091` | `type` | invalid broadcast |
| `E0092` | `type` | `Drop::drop` called directly |
| `E0093` | `type` | generic instantiation too deep |
| `E0094` | `type` | invalid representation |
| `E0095` | `type` | type too large |
| `E0096` | `type` | `try` or `catch` that does nothing |
| `E0097` | `type` | ROKON value of the wrong shape |
| `E0098` | `pattern` | non-exhaustive match |
| `E0099` | `pattern` | unreachable match arm |
| `E0100` | `pattern` | pattern does not fit the value's type |
| `E0101` | `pattern` | wrong number of fields in a pattern |
| `E0102` | `pattern` | or-pattern alternatives bind differently |
| `E0103` | `pattern` | binding looks like a variant |
| `E0104` | `pattern` | invalid rest pattern |
| `E0105` | `handoff` | value used after it was handed over |
| `E0106` | `handoff` | value used after its lock was released |
| `E0107` | `handoff` | shared value handed over |
| `E0108` | `meta` | cannot derive |
| `E0109` | `meta` | malformed attribute |
| `E0110` | `meta` | attribute on the wrong item |
| `E0111` | `meta` | invalid `#[lang]` item |
| `E0112` | `meta` | invalid operator declaration |
| `E0113` | `meta` | invalid `#[keyword]` declaration |
| `E0114` | `meta` | invalid literal handler |
| `E0115` | `meta` | invalid `#[intrinsic]` |
| `E0116` | `meta` | malformed macro definition |
| `E0117` | `meta` | no macro rule matches |
| `E0118` | `meta` | invalid meta template |
| `E0119` | `meta` | invalid meta block |
| `E0120` | `meta` | recursive expansion |
| `E0121` | `meta` | invalid `cfg` predicate |
| `E0122` | `meta` | file include failed |
| `E0123` | `meta` | registered twice |
| `E0124` | `meta` | meta name cannot be resolved |
| `E0125` | `other` | cannot read or write a file |
| `E0126` | `linkage` | link failed |
| `E0127` | `codegen` | code generation failed |
| `E0128` | `linkage` | external tool failed |
| `E0129` | `module` | invalid WIT |
| `E0130` | `unsupported` | not supported by this backend |
| `E0131` | `internal` | internal compiler error |
| `E0132` | `lint` | lint |
| `E0133` | `other` | invalid compiler option |
| `E0134` | `handoff` | resource used after it was moved or closed |
| `E0135` | `handoff` | resource not closed |
| `E0136` | `resolve` | trait of the method not in scope |
| `E0137` | `handoff` | thread-local value kept past `with` |
| `E0138` | `handoff` | list view used after a resize |
| `E0139` | `type` | `pub` function signature not written out |
| `E0140` | `handoff` | stream view used past its stream |
| `E0141` | `type` | assignment to a value that is not a place |
| `E0142` | `resolve` | method generic parameter named like its impl's |
| `E0143` | `type` | rows of an array literal differ in length |
| `E0144` | `type` | multi-axis index on a type indexed along one axis |
| `E0145` | `type` | integer raised to a negative power |

The `kind` values are `lexical`, `parse`, `resolve`, `visibility`, `module`, `type`, `pattern`, `handoff` (ownership and lifetime of values: use after a hand-off, [stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers); a resource used after it was moved or closed, or not closed, [stdlib 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop---cleanup-on-release); a thread-local value kept past `with`; a stream view used past its stream, [stdlib 2.23.1](../../refs-standard-libraries/200-core/223-io.md#2231-read); a `SliceMut` used after its list was resized, [stdlib 2.9.7a](../../refs-standard-libraries/200-core/209-collections.md#297a-slicemutt---writable-view)), `meta`, `lowering` (reserved: no code has it now), `codegen`, `linkage`, `unsupported`, `internal`, `other` and `lint`.

#### Codes before 0.2.0

Up to 0.1.x a code named a whole area (`E200` for every type error). Each old code now splits into the codes below; a tool that matched an old code matches these, or matches on `kind`.

| Old code | New codes |
|----------|-----------|
| `E001` | `E0001`, `E0002`, `E0003`, `E0004`, `E0005`, `E0006`, `E0007`, `E0008`, `E0009` |
| `E002` | `E0008`, `E0010`, `E0011`, `E0012`, `E0013`, `E0014`, `E0015`, `E0016`, `E0017`, `E0018`, `E0019`, `E0020`, `E0021`, `E0022`, `E0023`, `E0024`, `E0025`, `E0026`, `E0100`, `E0104`, `E0109`, `E0110`, `E0114`, `E0119`, `E0120`, `E0121`, `E0122` |
| `E100` | `E0027`, `E0028`, `E0082` |
| `E101` | (none emitted) |
| `E102` | `E0030`, `E0031`, `E0032`, `E0033`, `E0034`, `E0035`, `E0036`, `E0037`, `E0125`, `E0129` |
| `E200` | `E0003`, `E0004`, `E0007`, `E0012`, `E0018`, `E0023`, `E0024`, `E0025`, `E0026`, `E0029`, `E0031`, `E0034`, `E0036`, `E0038`, `E0039`, `E0040`, `E0041`, `E0042`, `E0043`, `E0044`, `E0045`, `E0046`, `E0047`, `E0048`, `E0049`, `E0050`, `E0051`, `E0052`, `E0053`, `E0054`, `E0055`, `E0056`, `E0057`, `E0058`, `E0059`, `E0060`, `E0061`, `E0062`, `E0063`, `E0064`, `E0065`, `E0066`, `E0067`, `E0068`, `E0069`, `E0070`, `E0071`, `E0072`, `E0073`, `E0074`, `E0075`, `E0076`, `E0077`, `E0078`, `E0079`, `E0080`, `E0081`, `E0082`, `E0083`, `E0084`, `E0085`, `E0086`, `E0087`, `E0088`, `E0089`, `E0090`, `E0091`, `E0092`, `E0093`, `E0094`, `E0095`, `E0096`, `E0097`, `E0098`, `E0099`, `E0100`, `E0101`, `E0102`, `E0103`, `E0104`, `E0108`, `E0110`, `E0114`, `E0123`, `E0131` |
| `E201` | (none emitted) |
| `E202` | `E0105`, `E0106`, `E0107` |
| `E300` | `E0033`, `E0034`, `E0108`, `E0109`, `E0110`, `E0111`, `E0112`, `E0113`, `E0114`, `E0115`, `E0116`, `E0117`, `E0118`, `E0119`, `E0120`, `E0121`, `E0123`, `E0124` |
| `E400` | (none emitted) |
| `E401` | (none emitted) |
| `E402` | `E0125`, `E0126`, `E0127`, `E0128`, `E0131`, `E0133` |
| `E500` | `E0130` |
| `E501` | `E0131` |
| `E502` | (none emitted) |
| `E503` | `E0132` |

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
         Std prebuilt (<path>)
   Compiling foo (path/to/foo)
   Stage: front end (134ms)
      Reused: 12 of 107 files parsed, 3 of 4 bodies lowered
    Building ./target/debug/foo
       Units 2 reused, 1 compiled
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

An optional context-sensitive stage builds with `-C profile-use=app.profdata -C cs-profile-generate=cs`, runs again, and merges `cs` with `app.profdata` into the final profile. `wasm32-wasip2` is supported; `wasm32-unknown-unknown` is not.

### Tracy / Chrome trace (optional)

Through the standard library's tracing surface, traces compatible with `tracy` or Chrome's `chrome://tracing` can be produced. The choice of a specific library is outside the scope of this specification and is covered by a separate library spec.

## 9.9.8. Memory / GC tools

The runtime reads the following environment variables.

| Variable                  | Effect                                                                       |
|---------------------------|------------------------------------------------------------------------------|
| `ROKR_GC_LOG=1`           | Prints the kind (minor / full), pause time, mark count and sweep result of each collection to stderr. A concurrent full collection logs its start pause (a minor collection) and its finish pause separately; the sweep result is the live bytes before -> after |
| `ROKR_GC_STATS=1`         | At exit, prints collection counts (minor / full), pause time and collector CPU time to stderr. The second line gives block count / resident size / evacuation count / bytes moved |
| `ROKR_GC_CONCURRENT=auto\|always\|off` | Concurrent marking for full collections. `auto` applies when old objects are 32 MiB or more ([7.5](../007-runtime/705-concurrent-collection.md)) |
| `ROKR_GC_EVAC=always`     | For testing. Evacuation on every full collection ([7.3.1](../007-runtime/703-gc-algorithm.md#731-immix-mark-region)) |
| `ROKR_GC_STRESS=N`        | For testing. Collects every N allocations (every 8th of those is full)       |
| `ROKR_HEAP_DUMP=on_oom`   | On OOM, prints to stderr the totals, the top 32 types by bytes and the 32 largest large objects. Best effort: it allocates nothing and takes no lock. Format is runtime-defined |
| `ROKR_WORKERS=<n>`        | Number of executor workers N (default: the online CPUs) ([7.6.8](../007-runtime/706-scheduler.md#768-thread-per-core-option)) |
| `ROKR_THREAD_PER_CORE=1`  | Thread-per-core: tasks pinned to their worker, no stealing (7.6.8) |
| `ROKR_DYNAMIC_WORKERS=1`  | Idle workers park and are taken back when every active worker is busy ([7.7.8](../007-runtime/707-io-and-timer.md#778-dynamic-adjustment-of-worker-count)) |

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
| `:save <file>`       | Writes the session as a script: the items, then the cells that ran, as typed; a name declared again is a redeclaration there too ([5.1.4](../005-modules/501-variable-declaration.md#514-redeclaration-in-the-same-scope)). `it` is declared only where a later cell reads it. The first line is `// rok repl session (:save); :load replays it cell by cell` and a `// :cell` line comes before each cell: `:load` of the script runs the cells again one by one, so a name declared again keeps its last binding; `rokc run` runs it as a program |
| `:opt [<level>]`     | Prints, or sets for later cells, the optimization level (`0`, `1`, `2`, `3`, `s`, as `-O`) |
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
rokc build --emit=expansions    # → target/<profile>/build/<name>.expanded.rok
```

This option prints the result of [meta expansion](../006-meta-system/609-template-tokens.md) of the user package as source text. Each expansion's generated source follows a comment like the one below; when generated code is expanded again, that result follows the outer expansion.

```
// expanded from this #[derive(MyHash)] (src/geom.rok:12:1)
impl MyHash for Point { ... }
```

To see the full AST after all meta expansion and desugaring as Rok source, use `--emit=source` (`<name>.ast.rok`).

## 9.9.11. Benchmarks

Microbenchmarks are run with the following command.

```
rok bench
```

`rokc bench` builds the benchmark harness and `rok bench` ([9.10](910-rok-frontend.md)) runs it ([9.1.2](901-rokc-cli.md#rokc-bench-benchmarks)), measuring the functions marked `#[bench]` one at a time in an `-O2` build. Each takes a `Bench` handle and times a closure through `b.iter`, so setup outside the closure is not measured. The measurement is:

1. **Clock.** A monotonic nanosecond clock. Its fixed overhead is amortized by timing `iters` calls per sample, never a single call.
2. **Calibration.** Unless `--iters` is given, `iters` is raised until one sample takes about 100 ms, so clock quantization and overhead are negligible.
3. **Warm-up.** Calls are run for about one sample's worth of time and discarded, to settle caches, branch predictors and paging.
4. **Sampling.** `--samples` samples (default 10) are taken; each records the total time of `iters` calls, and the per-call time is that total divided by `iters`.
5. **Statistics.** The median, the quartiles and the median absolute deviation (MAD) of the samples are computed. Samples outside the Tukey fences (1.5 IQR beyond the quartiles) are counted as outliers. The reported figure is the **median (+/- MAD)** of all samples, which an occasional slow sample barely moves.
6. **GC awareness.** The runtime's collection counter is read around each sample, and the samples during which a collection ran are counted. The human line ends with `[<n> outliers, <m> gc]` when either count is not zero; `--format=rokbench` always gives both.

`black_box(x)` is an identity the optimizer cannot see through; apply it to the inputs and result of the timed closure so the work is neither hoisted out nor folded away. `b.set_bytes(n)` / `b.set_items(n)` turn the per-call time into a throughput (`MiB/s`, `Melem/s`).

Output is aligned and in human units, to a terminal or not; `--format=rokbench` gives a stable one-line-per-bench machine form for regression tools ([9.1.2](901-rokc-cli.md#rokc-bench-benchmarks)), and `--format=json` gives JSON records ([9.9.12](#9912-ci--test-integration)). A benchmark that never calls `b.iter` is reported as having measured nothing, a warning and not a failure. `--iters` / `--samples` adjust the sampling.

## 9.9.12. CI / test integration

### CI mode

In CI environments, machine-readable output is recommended.

```
rokc build --message-format=json --no-color > build.log
```

This output is suitable for CI systems to parse.

### Test and bench records

`rok test --format json` and `rok bench --format json` write one JSON object per line on stdout. The human text, the output of the tests, and anything a benchmark prints itself go to stderr, so stdout holds only records. Each record has `schema` (now `1`), `type` and `kind` (`test` or `bench`).

```json
{"schema":1,"type":"case","kind":"test","name":"math::adds","result":"ok","ms":3.215}
{"schema":1,"type":"case","kind":"bench","name":"sum","result":"ok","ms":270.1,"median_ns":6.833,"mad_ns":0.167,"samples":10,"outliers":0,"gc":0,"bytes":0,"items":0}
{"schema":1,"type":"summary","kind":"test","result":"failed","passed":2,"failed":1,"filtered_out":0,"ms":120.4}
```

| `type`    | Fields                                                              |
|-----------|---------------------------------------------------------------------|
| `case`    | `name`; `result`: `ok`, `failed`, or `ignored` (a benchmark that measured nothing); `ms`, the wall time of its process. A benchmark with `result` `ok` also has `median_ns`, `mad_ns`, `samples`, `outliers`, `gc`, `bytes` and `items`, as in the `#rokbench` line ([9.1.2](901-rokc-cli.md#rokc-bench-benchmarks)) |
| `summary` | `result` (`ok` or `failed`), `ms`, and the counts: for tests `passed`, `failed`, `filtered_out`; for benchmarks `run`, `failed`, `ignored`, `filtered_out` |

A summary follows the cases of each harness, so a package with both root files has two when its library's run passes. Records follow the `schema` rule of the JSON diagnostics ([9.9.4](#994-diagnostics--error-messages)) with their own number, still `1`: a reader ignores fields it does not know. With `--message-format=json`, the build's diagnostics are JSON lines on the same stdout; their `type` (`error`, `warning`, ..) tells them apart from records.

### Exit codes

The exit codes of the [`rokc` CLI interface](901-rokc-cli.md) are defined as follows.

| Code  | Meaning          |
|-------|------------------|
| `0`   | Success          |
| `1`   | User code error  |
| `2`   | CLI error        |
| `70`  | rokc internal panic |

`rokc test` builds the harness; it does not run the tests. `rok test` runs them and exits with `1` if any fails. Each test runs in a separate process, so a test's panic (`101`) does not become the runner's `101`.

CI can apply different handling per code (retry, notification, etc.).

## 9.9.13. LSP server

`rokc` provides an LSP (Language Server Protocol) mode. The same server is also provided as the standalone binary `rokls` (= `rok lsp`). It communicates over stdio and provides diagnostics, hover (including members), go-to-definition (returning the range of the defining name), references, rename, document symbols, member / path completion, signature help, quick fixes, formatting and semantic tokens (macro calls, user-defined keywords, literal suffixes, operators and attributes, which the static grammar cannot recognize). References and rename are buffer-local.

```
rokc lsp
rokls
```

In this mode, LSP clients such as VS Code, Vim and Emacs receive completion, hover, go-to-definition, diagnostics and so on. Internally it uses the incremental results of `rokc check`.

- **Completion.** After `x.`, the fields and methods of `x`'s type. Of a type from the standard library or another package only the `pub` ones are offered, without `#[doc(hidden)]` members and without the methods of operator traits (those written as `a + b`, `xs[i]` and so on). After `a::`, also in `use a::b::`, the modules and items under `a`. Elsewhere, the locals in scope at the cursor, the buffer's items, the names in scope without a path (the core library's and those the `use` items import) and keywords. The REPL and the Jupyter kernel complete with the same engine ([9.7.8](907-jit-cells.md#978-input-unit-repl-recognition)).
- **Hover and definition.** Hover on a function, method or field, the standard library's included, shows its signature and then its doc comment. Go-to-definition on `x.name` looks `name` up in the type of `x`; on a standard library item it goes to the item in the library's source.
- **Signature help.** Inside the parentheses of a call (asked on `(` and `,`), the callee's signature and doc comment, with the parameter under the cursor active. A method is looked up in its receiver's type, and its `self` is not counted.
- **Quick fixes.** A diagnostic that carries an edit (for example ``convert with `as i32` ``) offers it as a `quickfix` code action. Without one, a help line that says what to write becomes one: ``add `;` `` inserts `;` at the end of the diagnostic's span, ``add `use a::b;` `` inserts that line after the file's last top-level `use` (with none, after its leading `//!` lines and inner attributes), and ``rok add <name>`` becomes the command `rok.add`. On `workspace/executeCommand` the server runs `rok.add` as `rok add <name>` in the package directory (for a script, `rok add <name> <file>` in the file's directory), with the `rok` named by `$ROK` or else the one beside the server, then checks the open buffers again. It runs no other command and takes no flags.
- **Rename.** `prepareRename` returns the range of the name under the cursor when rename would take it. On a name declared outside the buffer it fails with a message that says where, for example ``cannot rename `println`: it is defined in the standard library``; on a keyword, `` `let` is a keyword ``.
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
rokc version: 0.2.0 (<compiler id>)
ROKHIR version: 9
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

`rokc inspect --dump <file>` prints the ROKHIR's version, producer, meta and all interface files (9.9.2).

This gives an at-a-glance view of what the package exposes externally.

## 9.9.15. Summary

The tools above form the core of Rok's development and debugging experience. Integration with standard tools (gdb, perf, jupyter) combined with rok-specific tools (`rokc demangle`, `rokc inspect`, etc.) provides a rich diagnostic environment.

This concludes chapter 9 (execution and tools) and completes the overall picture of this specification.
