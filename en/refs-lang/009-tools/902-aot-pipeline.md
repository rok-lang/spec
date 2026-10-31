# 9.2. AOT build pipeline

This chapter defines the step-by-step flow of `rokc build`. It makes the overall picture given in [Execution engines and user surfaces](../000-meta/001-execution-modes.md) and [Compile pipeline](../000-meta/002-pipeline.md) concrete in terms of commands and file flow.

## 9.2.1. Steps

The build proceeds in 11 steps, all inside `rokc`.

```
1. package discovery       (find package root)
2. dependency resolve      (resolve deps to sources or prebuilt .rokhir + .a)
3. source parse            (.rok → AST)
4. name / visibility       (resolve)
5. type check              (type check)
6. rok-hir build           (HIR build)
7. specialize              (collect call-sites + monomorphize)
8. rok-mir build
9. code generation         (native at -O0; MLIR → LLVM at -O1 and above)
10. archive / link         (lld)
11. artifact .a / executable
```

> In order: package discovery, dependency resolution, source parsing, name / visibility resolution, type checking, rok-hir construction, specialization (call-site collection and monomorphization), rok-mir construction, code generation to objects, archiving or linking, and finally the output: an `.a` archive or an executable.

The input, output and responsibility of each step are kept clearly separate.

## 9.2.2. Package discovery

Starting from the path of `rokc build [path]` and searching upward, a directory containing `src/main.rok` or `src/lib.rok` is located (starting from the containing directory if path is a file); the search moves up one directory at a time.

If found, that directory is the package root and its `src/` the package's source (see [Source units](../000-meta/003-source-units.md)). The other cases:

- If path is a file outside the found package's `src/` (a script or tool beside the package's source), or no package is found walking upward, it is an anonymous executable package consisting of that single file ([4.16.8](../004-functions/416-entry-point.md#4168-single-file-execution)). Its file submodules are beside it.
- If path is a directory and there is no package, it is an error.

The package name is the name of the package directory (for an anonymous package, the file name without `.rok`) and becomes the output name (9.2.15). A package with both root files is built as two targets: its library first, then its executable, which mounts the library as dependency `<name>` ([0.3.1](../000-meta/003-source-units.md#library-and-executable-in-one-package)).

## 9.2.3. Dependency resolution

This step handles external package references (`use external_pkg::...`) in [paths](../005-modules/507-paths.md) and [`use`](../005-modules/505-use.md).

`rokc` does not resolve dependencies itself. The front-end `rok` ([9.10](910-rok-frontend.md)) prepares path / git dependency packages according to `rok.rokon` / `rok.lock` and passes the root of each package to `rokc` via `--extern`.

```
rokc build --extern util=../util --extern parser=target/debug/deps/libparser.a
```

`--extern name=<path>` mounts one dependency package at the top-level path `name`. `name` is the identifier obtained from the `deps` key in `rok.rokon` by replacing `-` with `_`. The path takes two forms.

| Path                              | Dependency package                                                           |
|-----------------------------------|------------------------------------------------------------------------------|
| `<dir>` / `<dir>/src/lib.rok`     | Source. That `src/lib.rok` is compiled together in the same build            |
| `<name>.rokhir` / `lib<name>.a`     | Prebuilt package. Given either one, the sibling file in the same directory is located. ROKHIR ([9.4](904-rokhir-bytecode.md)) is read for declarations and generics, and the archive is linked |

- User code uses that package's items via `use name::..`.
- `pkg::` inside a dependency package refers to that dependency package itself.
- A dependency package is a separate package. Only `pub` items are usable from outside; `pub(pkg)` / `pub(sub)` / private items are not ([5.7.14](../005-modules/507-paths.md#5714-paths-between-packages)).
- It is an error for a module of the user package to have the same name as a dependency package.

`rokc build` on a package with `src/lib.rok` produces `lib<name>.a` and `<name>.rokhir`, and with `src/main.rok` the executable `<name>`; a package with both produces all three, the executable linked against that library. `--name <n>` sets the package name (the output name and mount name); the default is the directory name.

### Prebuilt dependency packages

`rokc` prebuilds each source `--extern` **once**, in the order given, into `target/[<triple>/]<profile>/deps/`. Each dependency package sees all previously built dependency packages. Subsequent compilation uses these `lib<name>.a` and `<name>.rokhir` instead of the source.

```
target/<profile>/deps/
├── lib<name>.a
├── <name>.rokhir
└── <name>.fp               # fingerprint
```

If the fingerprint matches, it is not rebuilt. The fingerprint includes:

- The identity of `rokc` (the same value as the ROKHIR producer)
- Build options: `--release` / `-O*`, `-g`, `--target`, `--cfg` (without its `feature` keys), `--bootstrap`
- The dependency's features (`--extern-feature`)
- All files under that dependency package's `src/`, and its `rok.rokon`
- The fingerprints of dependency packages built before it

A prebuilt package's interface is read with the features it was built with (its ROKHIR meta, [9.4.6](904-rokhir-bytecode.md#946-meta)), not the consumer's.

Every command that compiles uses this path: `build` / `run` / `test` / `bench` / `check` / `lint` / `doc` / `lsp` / `repl` / `jupyter`, single-file scripts, and named environments. The REPL host also links the prebuilt archives. If there is neither a package nor `--target-dir`, they are placed under the lone-file cache in the system temporary directory (9.2.16).

The standard library is built from source ([9.3.6](903-package-archive.md#936-standard-library)) and kept prebuilt in `rokc`'s cache (9.2.19).

## 9.2.4. Source parsing

All `.rok` files are parsed into an AST according to the lexical / syntactic rules from [Source files](../001-lexical/101-source-files.md) through [Punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md).

A directory module of [Modules](../005-modules/504-modules.md) combines all its .rok files into the body of one module.

The manifest `rok.rokon` at the package root ([9.10.3](910-rok-frontend.md#9103-rokrokon--roklock)) is not a module file. The manifest is read as a single [ROKON](../../refs-standard-libraries/300-std/309-rokon.md) value (`Package { .. }`); top-level statement rules do not apply to it and it is not compiled.

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

HIR is an intermediate representation rich in type information, reflecting all results of steps 4-5 above. It is structurally compressed so that identical types and identifiers share a representation.

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

## 9.2.10. Backend selection

The optimization level selects the code generator by default ([0.2.4](../000-meta/002-pipeline.md#024-backends)); `-C backend=native|llvm` or `ROK_BACKEND` overrides it.

| Level                    | Backend      |
|--------------------------|--------------|
| `-O0`                    | native       |
| `-O1` `-O2` `-O3` `-Os`  | MLIR → LLVM  |

The native backend supports {aarch64, x86_64} × {macOS, Linux}. Other targets, modules that use [`__mlir_op` / `__mlir_type`](../../refs-standard-libraries/100-bare/109-mlir.md), builds with MLIR plugins, and builds with `-C profile-generate` / `-C profile-use` use the MLIR → LLVM backend at every level. The REPL follows the same selection (9.6.1). Both backends follow the runtime ABI of [8](../008-abi/README.md), so objects from either link together, including prebuilt dependencies built at another level.

## 9.2.11. Native backend

The native backend generates machine code and the object file (Mach-O, ELF) straight from rok-mir, with no intermediate IR, for aarch64 and x86_64 on macOS and Linux. It aims at compile speed: debug builds and REPL cells need no LLVM. With `-g` it emits DWARF line and frame information.

## 9.2.12. MLIR → LLVM backend

rok-mir is lowered to standard MLIR ([8.7](../008-abi/807-mlir-dialects.md)). `rokc` parses that MLIR in process, runs the MLIR pass pipeline down to the LLVM dialect, translates it to LLVM IR, optimizes it at the requested level and generates the object for the target triple. A module whose IR exceeds 1 MiB is split into parts that are code-generated in parallel. All of this runs in-process with no external tools; the intermediate forms are written only for `--emit` ([9.9.1](909-debug-tools.md#991-intermediate-output---emit)).

```
target/debug/build/foo.o
```

## 9.2.13. Archives

A library package puts its objects, one per codegen unit (9.2.16), into a standard `ar` archive written by `rokc` itself, and writes the interface file `foo.rokhir` ([ROKHIR format](904-rokhir-bytecode.md)) next to it.

```
target/debug/libfoo.a
target/debug/foo.rokhir
```

## 9.2.14. Linking (for executables)

An executable is linked by the embedded lld:

```
foo.o + <deps>/lib*.a + <sysroot>/lib/librokr.a + target libc  →  target/debug/foo
```

A Mach-O executable links against the macOS SDK (Xcode Command Line Tools). A Windows executable links against import libraries `rokc` builds from module definitions it carries (derived from mingw-w64), with Rok's own startup over the UCRT; no MSVC tools or Windows SDK are needed. Rok's Windows startup runs the C initializers in `.CRT$XCA`..`.CRT$XCZ` before `main`. For x86 Windows the import names come from mingw-w64's `lib32` definitions, with stdcall `@N` decoration, and images link with `/SAFESEH:NO`. Linux targets, native or cross, always link the shipped C library in `<sysroot>/lib/rok/targets/<triple>/`: musl statically, glibc through link stubs (executables need glibc 2.34 or later). Other libraries are found through `-L` and `LIBRARY_PATH`. The dynamic loaders named for 32-bit glibc targets are `/lib/ld-linux.so.2` (i686) and `/lib/ld-linux-armhf.so.3` (armv7hf). If `ROK_CC` is set, that C compiler driver performs the final link instead (`$ROK_CC <objects> <archives> -o <out>`); this is an optional override. Shared libraries given with `--native-runtime` are copied beside the executable, which gets rpath `@loader_path` (Mach-O) / `$ORIGIN` (ELF); Windows loads DLLs from the executable's directory. Executables are dead-stripped: unreferenced functions and data are removed; a `release` build also folds identical functions ([9.2.15](#9215-output)). When the executable has debug information on Apple targets, the embedded `dsymutil` (LLVM's, linked into `rokc`) writes a `.dSYM` bundle next to the executable.

## 9.2.15. Output

Output is placed in **per-profile** directories, as in cargo. The profiles are `debug` (default) and `release`; the profile is `release` when the optimization level is `-O2` / `-O3` / `-Os` (including `--release`).

The profiles differ as in cargo:

| Profile | Optimization | Debug information | Link |
|---|---|---|---|
| `debug` | `-O0`, `-O1` | on | unreferenced code and data removed |
| `release` | `-O2`, `-O3`, `-Os` | only with `-g` | also identical functions folded (not one whose address an object marks as taken); without `-g`, no symbols beyond what the loader needs |

Debug information belongs to the executable or archive a build makes (`build`, `run`, `test`), not to `--emit` output or REPL and Jupyter cells. A panic names its source location in either profile. A build's objects are linked from one place beside the output, so the same inputs make the same executable. Profiles have no settings in `rok.rokon`.

```
target/
├── debug/                      # profile (default)
│   ├── foo                     # executable (src/main.rok)
│   ├── libfoo.a                # library archive (src/lib.rok)
│   ├── foo.rokhir              # library interface (9.4)
│   ├── build/                  # intermediates and --emit output (foo.mlir, ...)
│   └── deps/                   # prebuilt dependencies (lib<dep>.a, <dep>.rokhir, <dep>.fp)
├── release/                    # --release
│   └── ...
├── aarch64-unknown-linux-gnu/  # cross build (--target=<triple>)
│   └── debug/ ...
└── doc/                        # rok doc / rokc doc
```

| Output          | Path                                                              |
|-----------------|-------------------------------------------------------------------|
| Executable      | `target/<profile>/<name>`                                         |
| Library         | `target/<profile>/lib<name>.a`                                    |
| `--emit=<kind>` | `target/<profile>/build/<name>.<ext>` (`hir` `mir` `mlir` `ll` `s` `o`) |
| Intermediates   | `target/<profile>/build/`                                         |
| Dependencies    | `target/<profile>/deps/` (`lib<dep>.a`, `<dep>.rokhir`, `<dep>.fp`, 9.2.3) |
| Cross build     | `target/<triple>/<profile>/...` (same structure as above)         |
| Documentation   | `target/doc/`                                                     |
| Incremental     | `target/<profile>/incremental/<name>/` (an executable's codegen units), `.../lib<name>/` (a library's), and `.../<name>-<hash>/` per target root (the front end), 9.2.16 |

A package with both root files has both outputs, `target/<profile>/<name>` and `target/<profile>/lib<name>.a` (+ `<name>.rokhir`); `-o` names the executable.

The name `foo` is the package name (9.2.2). `-o` changes the path of a single output; `--target-dir=<dir>` and the environment variable `ROKC_TARGET_DIR` change the location of `target/` itself. `rokc clean` / `rok clean` delete the whole `target/`. A library's ROKHIR is `foo.rokhir` next to the archive ([9.4](904-rokhir-bytecode.md)).

## 9.2.16. Incremental build

Incremental builds work at three levels: a package whose inputs did not change is not built at all; a package that is built takes the parses and lowered bodies of its unchanged files and functions from the last build (front end, below); and its code is compiled by codegen units, of which only the changed ones are compiled again. If the fingerprint of the previous build equals the current one, `rokc build` / `rokc run` print `Fresh` and skip the build. The fingerprint includes:

- The compiler's identity (the same value as the ROKHIR producer), build options, diagnostic format (`--message-format`, color), input / output paths, `ROK_CC` if set
- The files under `src/` of the package and its dependencies, their `rok.rokon`, and every other file the build read (`#[path]` / `include_str!` outside `src/`)
- Files read by `include_str!` / `include_bytes!` ([6.11](../006-meta-system/611-include.md)), with their contents
- The lint levels (`-A` / `-W` / `-D`, and the manifest's `lints` through them)
- The runtime library, the C runtime objects, C library and linker the toolchain ships for the target ([9.2.19](#9219-installation-layout)), the `.a` of prebuilt dependencies, MLIR plugin files, a `-C profile-use` file and the profile runtime, each by its size, modification time and file identity (inode and status change time; on Windows its creation time), so a copy that keeps the modification time is a change
- The size, modification time and file identity of the outputs

Warnings emitted by a build, lint warnings and the back end's included, are recorded, and a `Fresh` build prints them again.

A native build without `--cfg`, `--target`, MLIR plugins, profile options or `-O3` links the prebuilt standard library from `lib/rok-cache/` (9.2.19), one folder per hash of toolchain, options and sources, built on first use under a lock. At `-O3` the standard library is compiled with the program (whole-program optimization). The REPL compiles the standard library from source.

### Front end

A build of a package (`check`, `build`, `run`, `test`, `bench`, `doc`, `lint`) keeps what its front end computed under `target/<profile>/incremental/<name>-<hash>/`, where `<hash>` names the package root (each target of a package with both root files has its own, `<hash>` naming its root), with one subdirectory per kind of build (target, cfg, test or not, which libraries are prebuilt). The next build of the same kind takes from it:

- The parse of each file whose text and parse inputs (the build's extension operators, the package's keywords, meta blocks and `#[literal]` patterns, and cfg) are unchanged.
- The lowering (name resolution, type checking, generic instantiation) of each plain function body: a non-generic, non-`const` function, or a method of a non-generic `impl`. It is reused when its own text and position are unchanged, the program's interface is unchanged, every signature, instance and other fact the lowering read is the same now, and the text before every position its result names outside the body is unchanged.
- The interface is every item except the plain bodies whose return type is written (or `()`) and the plain functions' parameters and return types; a body whose return type is inferred is part of it (its callers read the type). A body edit lowers that body again, and an edit of a body whose return type is inferred also lowers the bodies that read its type; a plain function's signature edit lowers it and the bodies that read its signature; any other change to an item (a type, a trait, an `impl` header, a generic function, a `use`) lowers every body again.
- Functions, types and instances a reused lowering made are numbered as in a clean build; diagnostics, the program and its symbols are those of a clean build.

Signatures, `impl` registration, coherence and the whole-program checks run in every build. What collecting the library's items made (the standard library's and the dependencies'), and the library files' pre-pass parses, are kept in the toolchain's cache (`lib/rok-cache/library/`, one file per hash of toolchain, build flags and library sources) for every build, a lone file's too; `ROKC_INCREMENTAL` neither moves nor disables them. Nothing kept is required: a missing, damaged or stale entry is computed again. A lone file outside any package keeps no parses or bodies of its own unless `ROKC_INCREMENTAL` names a directory. `ROKC_INCREMENTAL=<dir>` keeps the results under `<dir>/<name>-<hash>/`; `ROKC_INCREMENTAL=0` turns them off. `ROKC_INCR_VERIFY=1` also lowers every body again and fails the build if a reused lowering differs. `-v` reports how many files' parses and bodies were reused.

### Codegen units

Within a package, a build that is not `Fresh` reuses compiled code by codegen units. rokc divides a package's code (an executable's or a library's) into units: each module of the package is one, and so is each module of a library whose instances it compiles (instances may be spread over several units by their type arguments), plus one unit for what the program has once (statics, the C `main`, GC type registration).

- A unit is compiled only when its key changed since the last build into the same target directory. Otherwise its result, kept under `target/<profile>/incremental/<name>/` (a library's: `lib<name>/`), is reused; then everything is linked.
- An anonymous file's units are kept in the target directory used without a package (`--target-dir`, `ROKC_TARGET_DIR`, else the lone-file cache), under `[<triple>/]<profile>/incremental/<stem>-<hash of its path>/`. The lone-file cache is `<system temporary directory>/rokc-target-<uid>/` on Unix, `<uid>` the effective user ID, and `%TEMP%\rokc-target\` on Windows, whose temporary directory is the user's own. It is a cache: at most once a day, rokc removes its files that no build wrote for a week.
- rokc runs what it finds in the lone-file cache, so on Unix it is private: rokc creates it with mode `0700`, and before using it checks that it is a directory (not a symbolic link), owned by the effective user, with no permission bits for group or others. Otherwise the command fails with exit code `1` and ``rokc: error: <dir> is not private to you (owner uid N, mode 0755); remove it or set ROKC_TARGET_DIR``. `--target-dir` and `ROKC_TARGET_DIR` are used as given.
- A unit's key covers its functions' code, the signatures and code-generation facts of what it uses from other units (a function's signature change recompiles its callers' units), the statics and GC types it uses, the target, the optimization level, the code generation options, the compiler's identity and, under `-g`, the paths of the source files its debug info names (adding a file recompiles only the units that refer to it).
- Symbols and GC type ids do not depend on what other units contain (a unit reads GC type ids from symbols the package's shared unit defines; [9.5.7](905-symbol-mangling.md#957-collision-avoidance-typehash-stability)), so the output is the one a clean build makes.
- Optimized builds still inline across units (ThinLTO, whose per-unit results are cached too).
- Profile-guided builds (`-C profile-generate`, `-C profile-use`), and builds with MLIR plugins or passes are compiled whole.
- `-v` reports how many units were reused and compiled; `-vv` names the compiled ones.

## 9.2.17. Separation of compiler and runtime

`rokc` uses zero third-party crates, plus one pinned, statically linked LLVM (22.1.8).

The pipeline keeps the compiler and runtime apart: `rokc` does not reference the runtime (`librokr.a`) at build time.

Instead it only emits calls to `extern "C"` symbols ([Extern "C" symbols](../008-abi/805-extern-symbols.md)), and at link time the user build links `librokr.a` alongside.

This separation lets `rokc` and `librokr.a` evolve under independent SemVer.

## 9.2.18. Embedded toolchain

`rokc` is self-contained: LLVM 22.1.8 with MLIR and lld is built into it, not looked up on `PATH`. It runs no external compiler, assembler or linker. Cross-compilation is built in: the toolchain ships the C library (musl, glibc link stubs) for its supported Linux targets.

| Needed from the system | Target                                   |
|------------------------|------------------------------------------|
| macOS SDK (Xcode Command Line Tools; not redistributable) | Apple targets |
| nothing (shipped C library) | Linux targets                   |
| nothing (import libraries built from shipped module definitions; UCRT) | Windows targets |
| `ROK_CC` (optional)    | Overrides the final link (9.2.14)        |

## 9.2.19. Installation layout

The installation layout follows rustup / cargo. The default of `ROK_HOME` is `~/.rok` (`%USERPROFILE%\.rok` on Windows).

```
$ROK_HOME/
├── bin/                         # proxy: runs the same-named executable of the active toolchain
│   ├── rok
│   ├── rokc
│   ├── rokls
│   └── <name>                   # an executable installed by rok install (9.10.5a)
├── toolchains/
│   └── <version>-<triple>/      # self-contained toolchain
│       ├── bin/{rok, rokc, rokls}
│       └── lib/
│           ├── librokr.a        # runtime
│           ├── libclang.{so,dylib,dll}  # libclang, loaded only by rokc bindgen (std 1.7.6)
│           ├── clang/<major>/include/   # clang's built-in headers (its resource directory)
│           ├── rok/lldb/rok_lldb.py      # lldb formatters for -g programs (9.9.6)
│           ├── rok/targets/<triple>/   # what --target links: the runtime (librokr.a / rokr.lib), Linux C library,
│           │                           #   libclang_rt.profile.a (Windows clang_rt.profile.lib); the host's included
│           └── rok/{core, std, bare}/src/   # standard library sources: each tier is laid out as a package, root src/lib.rok (5.4)
├── cache/                       # git, registry, native, scripts (9.10.4)
├── envs/<name>/, envs/CURRENT   # named environments: rok.rokon, rok.lock, target/ (9.10.5)
├── installs.rokon               # what rok install put in bin/, and where from
└── config.rokon                 # Config { registry, toolchain }
```

`rokc` finds the standard library and runtime relative to its own executable (sysroot = the executable's `..`) and does not look at `ROK_HOME`. Each tier of the standard library is laid out like a library package: its root module is `lib/rok/<tier>/src/lib.rok` and its modules are under `lib/rok/<tier>/src/`. A `rokc` built in the implementation repository reads the same layout from the repository's `library/<tier>/src/`. Its own cache is the toolchain's `lib/rok-cache/` (a repository build uses `target/<profile>/rok-cache/`). It holds the prebuilt standard library, keyed by toolchain, build options and sources (9.2.16), and in `.facts/` the cached target data layout and tool facts. Selection of the active toolchain is defined by [9.10.6](910-rok-frontend.md#9106-toolchain).

A Linux (glibc) toolchain needs glibc 2.34 or later, as the executables it builds do (9.2.14).

A release may be accompanied by an MLIR plugin development archive (`rok-mlir-dev-<version>-<triple>`: the pinned LLVM / MLIR headers and, on Windows, `rokc.lib`) ([stdlib 1.10.2](../../refs-standard-libraries/100-bare/110-mlir-plugin.md)).
