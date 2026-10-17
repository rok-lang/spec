# 9.2. AOT build pipeline

This chapter defines the step-by-step flow of `rokc build`. It makes the overall picture given in [Execution engines and user surfaces](../000-meta/001-execution-modes.md) and [Compile pipeline](../000-meta/002-pipeline.md) concrete in terms of commands and file flow.

## 9.2.1. Steps

The build proceeds in 13 steps.

```
1. package discovery       (find package root)
2. dependency resolve      (resolve deps to sources or prebuilt .rhir + .a)
3. source parse            (.rok → AST)
4. name / visibility       (resolve)
5. type check              (type check)
6. rok-hir build           (HIR build)
7. specialize              (collect call-sites + monomorphize)
8. rok-mir build
9. standard MLIR text emit
10. mlir-opt invoke        (external tool)
11. mlir-translate invoke  (external tool)
12. clang / lld invoke     (external tool)
13. artifact .a / executable
```

> In order: package discovery, dependency resolution, source parsing, name / visibility resolution, type checking, rok-hir construction, specialization (call-site collection and monomorphization), rok-mir construction, standard MLIR text emission, invocation of the external tools `mlir-opt`, `mlir-translate` and `clang` / `lld`, and finally the output: an `.a` archive or an executable.

The input, output and responsibility of each step are kept clearly separate.

## 9.2.2. Package discovery

Starting from the path of `rokc build [path]` and searching upward, `main.rok` or `lib.rok` is located. If `[path]` itself is `lib.rok` or `main.rok`, that is the package; otherwise (starting from the containing directory if path is a file) the search moves up one directory at a time.

If found, that directory is the package root (see [Source units](../000-meta/003-source-units.md)). The other cases:

- It is an error for one directory to contain both `main.rok` and `lib.rok`.
- If path is a file and no package is found walking upward, it is an anonymous executable package consisting of that single file ([4.16.8](../004-functions/416-entry-point.md#4168-single-file-execution)). File submodules (`mod foo;`) are an error.
- If path is a directory and there is no package, it is an error.

The package name is the name of the root directory (for an anonymous package, the file name without `.rok`) and becomes the output name (9.2.15).

## 9.2.3. Dependency resolution

This step handles external package references (`use external_pkg::...`) in [paths](../005-modules/507-paths.md) and [`use`](../005-modules/505-use.md).

`rokc` does not resolve dependencies itself. The front-end `rok` ([9.10](910-rok-frontend.md)) prepares path / git dependency packages according to `rok.rokon` / `rok.lock` and passes the root of each package to `rokc` via `--extern`.

```
rokc build --extern util=../util --extern parser=target/debug/deps/libparser.a
```

`--extern name=<path>` mounts one dependency package at the top-level path `name`. `name` is the identifier obtained from the `deps` key in `rok.rokon` by replacing `-` with `_`. The path takes two forms.

| Path                              | Dependency package                                                           |
|-----------------------------------|------------------------------------------------------------------------------|
| `<dir>` / `<dir>/lib.rok`         | Source. That `lib.rok` is compiled together in the same build                |
| `<name>.rhir` / `lib<name>.a`     | Prebuilt package. Given either one, the sibling file in the same directory is located. RHIR ([9.4](904-rhir-bytecode.md)) is read for declarations and generics, and the archive is linked |

- User code uses that package's items via `use name::..`.
- `pkg::` inside a dependency package refers to that dependency package itself.
- A dependency package is a separate package. Only `pub` items are usable from outside; `pub(pkg)` / `pub(sub)` / private items are not ([5.7.14](../005-modules/507-paths.md#5714-paths-between-packages)).
- It is an error for a module of the user package to have the same name as a dependency package.

`rokc build` on a library package produces both `lib<name>.a` and `<name>.rhir`. `--name <n>` sets the package name (the output name and mount name); the default is the directory name.

### Prebuilt dependency packages

`rokc` prebuilds each source `--extern` **once**, in the order given, into `target/[<triple>/]<profile>/deps/`. Each dependency package sees all previously built dependency packages. Subsequent compilation uses these `lib<name>.a` and `<name>.rhir` instead of the source.

```
target/<profile>/deps/
├── lib<name>.a
├── <name>.rhir
└── <name>.fp               # fingerprint
```

If the fingerprint matches, it is not rebuilt. The fingerprint includes:

- The identity of `rokc` (the same value as the RHIR producer)
- Build options: `--release` / `-O*`, `-g`, `--target`, `--cfg`, `--bootstrap`
- All `.rok` files and `rok.rokon` of that dependency package
- The fingerprints of dependency packages built before it

Every command that compiles uses this path: `build` / `run` / `test` / `bench` / `check` / `lint` / `doc` / `lsp` / `repl` / `jupyter`, single-file scripts, and named environments. The REPL host also links the prebuilt archives. If there is neither a package nor `--target-dir`, they are placed under `rokc-target/` in the system temporary directory.

The standard library is not prebuilt; it is bootstrapped from source on every build ([9.3.6](903-package-archive.md#936-standard-library)).

## 9.2.4. Source parsing

All `.rok` files are parsed into an AST according to the lexical / syntactic rules from [Source files](../001-lexical/101-source-files.md) through [Punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md).

A directory module of [Modules](../005-modules/504-modules.md) combines all its .rok files into the body of one module.

The manifest `rok.rokon` at the package root ([9.10.3](910-rok-frontend.md#9103-rokrokon--roklock)) is not a module file. The manifest is read as a single [RokON](../../refs-standard-libraries/300-std/309-rokon.md) value (`Package { .. }`); top-level statement rules do not apply to it and it is not compiled.

### Scanner pre-pass

The scanner pre-pass of the [`#[keyword]` magic](../006-meta-system/606-keyword-magic.md) is also performed in this step. All .rok files are first scanned shallowly, `#[keyword(...)]`, `#[infix(...)]`, `#[literal(...)]` registrations are collected, and the registrations are passed to the real lexer / parser.

## 9.2.5. Name / visibility resolution

Every path and identifier is mapped to an item according to the rules of [`use` (imports)](../005-modules/505-use.md), [Visibility](../005-modules/506-visibility.md), [Paths](../005-modules/507-paths.md) and [Re-export (`pub use`)](../005-modules/508-re-export.md).

This step applies the implicit `rok::core::**` import (see [Standard library tiers](../000-meta/005-standard-libraries-tiers.md); `rok::bare::*` and `rok::std::*` are excluded as they require explicit import), performs visibility checks reporting improper access as errors, and checks for import conflicts.

## 9.2.6. Type checking

The type of every expression is determined by the rules of [2. Type system](../002-types/README.md).

This includes deciding between gradual and static slots (see [`any` slot](../002-types/202-any-slot.md)), checking trait bounds (see [Generics](../002-types/210-generics.md)), and fixing the result types of branches (see [3. Expressions](../003-expressions/README.md)). Narrowing of sum types and `any` slots is handled here as well.

## 9.2.7. rok-hir construction

The type-checked AST is converted into the rok-hir ([HIR](../000-meta/007-glossary.md#hir-high-level-ir--mir-mid-level-ir)) data structure.

HIR is an intermediate representation rich in type information, reflecting all results of steps 4–5 above. It is structurally compressed so that identical types and identifiers share a representation.

## 9.2.8. specialize: monomorphization

Monomorphization of [generics](../002-types/210-generics.md) and [const generics](../002-types/211-const-generics.md) is performed.

```
1. Collect all call sites.
2. Determine the concrete type tuple of each call.
3. Create a separate specialized HIR per (item_id, type_tuple).
4. Cache. Each (item_id, type_tuple) only once.
```

Functions unreachable from the program's roots (`main`, exported symbols, `#[export_name]` items, symbols referenced by the runtime) are not emitted. The same holds for library builds: functions not reachable from the package's `pub` items and those roots are not put into the archive.

This step is what lets [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) make direct calls without vtables (see [trait-as-type ban](../002-types/212-trait-as-type-banned.md)).

## 9.2.9. rok-mir construction

The [specialized](../000-meta/007-glossary.md#specialize) HIR is converted into rok-mir (MIR).

MIR is an IR one level lower than HIR. In this step operator desugaring is completed (see [Operator dispatch](../003-expressions/301-operator-dispatch.md)), the decision tree of pattern matching is determined (see [`match` expression](../003-expressions/312-match-expression.md)), and the frame layout of closures and coroutines is fixed.

## 9.2.10. Standard MLIR text output

`.mlir` text is emitted using the standard MLIR dialects of [Compile pipeline](../000-meta/002-pipeline.md) and [Use of MLIR dialects](../008-abi/807-mlir-dialects.md).

```
target/debug/build/foo.mlir
```

All rok semantics are converted into standard dialects according to the lowering rules of [8. Runtime ABI / MLIR lowering](../008-abi/README.md).

`rokc` does not use MLIR library bindings (C++, Rust, etc.). It emits MLIR as text, and the following steps invoke the external tools `mlir-opt`, `mlir-translate` and `clang` as processes.

This is the end of `rokc`'s area of responsibility.

## 9.2.11. Invoking mlir-opt

The standard MLIR tool `mlir-opt` is invoked to apply cleanup passes and conversion passes to the LLVM dialect. `mlir-opt` has no optimization level options such as `-O2`, so passes are listed by name.

```
mlir-opt --canonicalize --cse \
         --convert-scf-to-cf --convert-cf-to-llvm --convert-arith-to-llvm \
         --convert-func-to-llvm --reconcile-unrealized-casts \
         target/debug/build/foo.mlir > target/debug/build/foo.opt.mlir
```

The exact pass list is implementation-defined; the above is a representative configuration. Optimization levels such as `-O0` – `-O3` apply in the later `clang` step.

## 9.2.12. Invoking mlir-translate

The standard tool `mlir-translate` translates MLIR to LLVM IR.

```
mlir-translate --mlir-to-llvmir target/debug/build/foo.opt.mlir > target/debug/build/foo.ll
```

## 9.2.13. Object generation

LLVM IR is turned into an object by the `clang` of the same LLVM.

```
clang -O2 -c target/debug/build/foo.ll -o target/debug/build/foo.o
ar rcs target/debug/libfoo.a target/debug/build/foo.o
```

A library package also writes the interface file `foo.rhir` ([RHIR format](904-rhir-bytecode.md)) next to the archive.

## 9.2.14. Linking (for executables)

```
$ROK_CC target/debug/build/foo.o <deps>/lib*.a <sysroot>/lib/librokr.a -o target/debug/foo
```

Linking is done with `ROK_CC`. If unset, it is `xcrun clang` on Apple and LLVM's `clang` elsewhere. On Apple with `-g`, `dsymutil` is run after linking.

## 9.2.15. Output

Output is placed in **per-profile** directories, as in cargo. The profiles are `debug` (default) and `release`; the profile is `release` when the optimization level is `-O2` / `-O3` / `-Os` (including `--release`).

```
target/
├── debug/                      # profile (default)
│   ├── foo                     # executable (for main)
│   ├── libfoo.a                # package archive (for lib)
│   ├── foo.rhir                # interface (for lib, 9.4)
│   ├── foo.o                   # the archive's object (for lib)
│   ├── build/                  # intermediates and --emit output (foo.mlir, ...)
│   └── deps/                   # prebuilt dependencies (lib<dep>.a, <dep>.rhir, <dep>.fp)
├── release/                    # --release
│   └── ...
├── aarch64-unknown-linux-gnu/  # cross build (--target=<triple>)
│   └── debug/ ...
└── doc/                        # rok doc / rokc doc
```

| Output          | Path                                                              |
|-----------------|-------------------------------------------------------------------|
| Executable      | `target/<profile>/<name>`                                         |
| Library         | `target/<profile>/lib<name>.a` and `target/<profile>/<name>.o`    |
| `--emit=<kind>` | `target/<profile>/build/<name>.<ext>` (`hir` `mir` `mlir` `ll` `s` `o`) |
| Intermediates   | `target/<profile>/build/`                                         |
| Dependencies    | `target/<profile>/deps/` (`lib<dep>.a`, `<dep>.rhir`, `<dep>.fp`, 9.2.3) |
| Cross build     | `target/<triple>/<profile>/...` (same structure as above)         |
| Documentation   | `target/doc/`                                                     |

The name `foo` is the package name (9.2.2). `-o` changes the path of a single output; `--target-dir=<dir>` and the environment variable `ROKC_TARGET_DIR` change the location of `target/` itself. `rokc clean` / `rok clean` delete the whole `target/`. A library's RHIR is `foo.rhir` next to the archive ([9.4](904-rhir-bytecode.md)).

## 9.2.16. Incremental build

Incremental builds are per package. If the fingerprint of the previous build equals the current one, `rokc build` / `rokc run` print `Fresh` and skip the build. The fingerprint includes:

- The compiler's identity (the same value as the RHIR producer), build options, diagnostic format (`--message-format`, color), input / output paths, `PATH`
- All `.rok` files and `rok.rokon` readable from the package and its dependencies
- The `.a` of prebuilt dependencies, MLIR plugin files
- The size and modification time of the outputs

Warnings emitted by a build are recorded, and a `Fresh` build prints them again.

## 9.2.17. Separation of compiler and runtime

This pipeline keeps the zero-code-dependency principle: `rokc` does not reference the runtime (`librokr.a`) at build time.

Instead it only emits calls to `extern "C"` symbols ([Extern "C" symbols](../008-abi/805-extern-symbols.md)), and at link time the user build links `librokr.a` alongside.

This separation lets `rokc` and `librokr.a` evolve under independent SemVer.

## 9.2.18. External tool dependencies

| Tool             | Role                 |
|------------------|----------------------|
| `mlir-opt`       | MLIR optimization    |
| `mlir-translate` | MLIR → LLVM IR       |
| `clang`          | LLVM IR → object (from the same LLVM) |
| `ROK_CC` / `xcrun clang` | Final link   |

LLVM 21–22 is used. Tools are looked up in `PATH` first, then in known locations (Homebrew, apt.llvm.org). `rokc` only invokes these tools as processes and does not link LLVM libraries ([0.2.4](../000-meta/002-pipeline.md#024-emitting-mlir-text-directly)). If a tool is missing, it is an error stating the locations searched.

## 9.2.19. Installation layout

The installation layout follows rustup / cargo. The default of `ROK_HOME` is `~/.rok`.

```
$ROK_HOME/
├── bin/                         # proxy: runs the same-named executable of the active toolchain
│   ├── rok
│   ├── rokc
│   └── rokls
├── toolchains/
│   └── <version>-<triple>/      # self-contained toolchain
│       ├── bin/{rok, rokc, rokls}
│       └── lib/
│           ├── librokr.a        # runtime
│           └── rok/{core, std, bare}/   # standard library sources
├── cache/                       # git, registry, script locks (9.10.4)
├── envs/<name>/, envs/CURRENT   # named environments (9.10.5)
└── config.rokon                 # Config { registry, toolchain }
```

`rokc` finds the standard library and runtime relative to its own executable (sysroot = the executable's `..`) and does not look at `ROK_HOME`. Selection of the active toolchain is defined by [9.10.6](910-rok-frontend.md#9106-toolchain).
