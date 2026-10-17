# 9.1. `rokc` CLI interface

This section defines the command-line interface of the compiler `rokc`. The toolchain consists of three binaries.

| Binary   | Role                                                                                  |
|----------|---------------------------------------------------------------------------------------|
| `rokc`   | Compiler (the counterpart of rustc). Contains build, run, test, formatter, linter, REPL and LSP (this section) |
| `rok`    | Cargo-style front-end. Owns the manifest, dependencies, environments and toolchain pin, and invokes `rokc` ([9.10](910-rok-frontend.md)) |
| `rokls`  | LSP server executable for editors. Equivalent to `rok lsp`                            |

The runtime is `librokr.a` (`rokr`). There are no separate tools (`rok-fmt` etc.). `rokc` does not resolve dependencies; dependency packages are passed by `rok` via `--extern` ([9.2.3](902-aot-pipeline.md#923-dependency-resolution)).

## 9.1.1. General form

`rokc` is invoked in the following general form.

```
rokc <subcommand> [options] [args]
```

As grammar rules:

```
cli_invocation  ::= "rokc" subcommand option* arg*
subcommand      ::= "build" | "emit" | "run" | "test" | "bench" | "check"
                  | "fmt" | "lint" | "doc" | "repl" | "jupyter" | "lsp"
                  | "clean" | "demangle" | "rhir-text" | "inspect" | "version" | "help"
option          ::= "--" identifier ("=" value)?
                  | "-" letter
```

`--name value` is equivalent to `--name=value`. Global options (9.1.5) may appear before or after the subcommand.

| Area                | Subcommands                                           |
|---------------------|-------------------------------------------------------|
| Build / run         | `build`, `emit`, `run`, `test`, `bench`, `check`      |
| Package             | `clean`                                               |
| Tools               | `fmt`, `lint`, `doc`, `lsp`                           |
| Interactive         | `repl`, `jupyter`                                     |
| Debug / diagnostic  | `demangle`, `rhir-text`, `inspect`                    |
| Other               | `version`, `help`                                     |

### Package selection (`path`)

The `[path]` of `build` / `run` / `test` / `check` / `clean` is one of the following. If omitted, it is the current directory.

| `path`                       | Package                                                                                                 |
|------------------------------|---------------------------------------------------------------------------------------------------------|
| Package directory            | That directory. If it has no `main.rok` or `lib.rok`, the search moves upward (9.2.2)                   |
| File inside a package        | The first directory containing `main.rok` / `lib.rok`, walking upward from the file                     |
| File not in any package      | An anonymous executable package consisting of that single file ([4.16.8](../004-functions/416-entry-point.md#4168-single-file-execution)) |

It is an error for one directory to contain both `main.rok` and `lib.rok` ([0.3](../000-meta/003-source-units.md)). It is an error if a directory is given and no package is found walking upward. An anonymous package cannot have file submodules (`mod foo;`).

The package name (the output name) is the name of the package directory; for an anonymous package it is the file name without `.rok`.

## 9.1.2. Core subcommands

### `rokc build`: [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) build

`rokc build` builds a package AOT.

```
rokc build [path]
```

`[path]` follows the package selection rules above. Output is placed in the per-profile `target/` of [9.2.15](902-aot-pipeline.md#9215-output). An executable package produces `target/<profile>/<name>`; a library package produces `target/<profile>/lib<name>.a` (and `<name>.o`) plus the interface `target/<profile>/<name>.rhir` ([9.4](904-rhir-bytecode.md)). The profile is `release` for `-O2` / `-O3` / `-Os` (including `--release`) and `debug` otherwise; with `--target=<triple>` the directory is `target/<triple>/<profile>/`. If `--target-dir=<dir>` or `ROKC_TARGET_DIR` is set, that directory is used instead of `target/`.

Options:

- `--release` performs an optimized build (equivalent to `-O2`) in the `release` profile; the default is a debug build (`-O0`) in the `debug` profile.
- `--target-dir=<dir>` relocates `target/` (equivalent to `ROKC_TARGET_DIR`).
- `--extern <name>=<dir | lib.rok | <name>.rhir | lib<name>.a>` mounts one dependency package at the top-level path `name` ([9.2.3](902-aot-pipeline.md#923-dependency-resolution)). Source (`dir` / `lib.rok`) is compiled together in the same build; a prebuilt package (`.rhir` / `.a`; given one, its sibling is located) has its RHIR read and its archive linked. May be given multiple times and is normally filled in by `rok`. `name` is an identifier: the package name with `-` replaced by `_` ([9.10.3](910-rok-frontend.md#9103-rokrokon--roklock)).
- `--name <n>` sets the package name (naming rules of [9.10.3](910-rok-frontend.md#9103-rokrokon--roklock)). The outputs are `lib<n>.a` / `<n>.rhir` and the mount name is the name with `-` replaced by `_`. The default is the directory name. `--extern a-b=..` is likewise bound as `a_b`.
- `--bootstrap <file>` gives a standard library root to preload (may be repeated); `--rokr <lib>` gives the runtime to link. The defaults are the sysroot's `lib/rok/{core,std,bare}/lib.rok` and `lib/librokr.a` ([9.2.19](902-aot-pipeline.md#9219-installation-layout)). If not found, it is an error that lists the paths searched.
- `-O0` / `-O1` / `-O2` / `-O3` / `-Os` select the optimization level explicitly.
- `--target=<triple>` selects the target platform, one of the 13 Tier 1 targets in [Platform targets](../000-meta/006-platform-targets.md).
- `-o` / `--out=<path>` selects the output location.
- `--cfg <flag>` / `--cfg <key>=<value>` sets a cfg flag for [conditional compilation](../006-meta-system/603-cfg.md). May be repeated.
- `--emit=<kind>[,<kind>..]` writes intermediate output (`ast`, `hir`, `mir`, `mlir`, `llvmir`, `asm`, `obj`, `expanded`) to `target/<profile>/build/<name>.<ext>` ([9.9.1](909-debug-tools.md#991-intermediate-output---emit)). Several kinds may be given at once separated by commas; in that case `-o` cannot be used. With `--emit`, no executable or archive is produced.
- `--load-mlir-plugin <path|name>` passes an MLIR dialect plugin to `mlir-opt --load-dialect-plugin`, `--load-mlir-pass-plugin <path|name>` passes a pass plugin as `--load-pass-plugin`, and `--mlir-pass <pass>` passes `--<pass>` before conversion to the LLVM dialect ([rok::bare::mlir::plugin](../../refs-standard-libraries/100-bare/110-mlir-plugin.md#1103-plugin-loading)). `rokc` does not link MLIR, so it does not load plugins itself. All may be repeated, and `rok` passes them through unchanged.
- `-g` / `--debug` emits DWARF debug information ([9.9](909-debug-tools.md#dwarf-debug-information)).
- `-C overflow-checks[=yes|no]` turns overflow checks (panic) for fixed-width integer arithmetic on or off. The default is on only at `-O0`. An omitted value means `yes`. Operations on constant operands that cannot overflow skip the check.

If nothing has changed since the previous build, `rokc build` / `rokc run` print `Fresh` and skip the build ([9.2.16](902-aot-pipeline.md#9216-incremental-build)).

### `rokc emit`: intermediate output

```
rokc emit [path] [--emit=<kind>[,<kind>..]]
```

Equivalent to `rokc build --emit=..`. If `--emit` is omitted, it is `mlir`.

### `rokc run`: build and run

`rokc run` runs the output immediately after building.

```
rokc run [path] [-- args...]
```

It builds an executable package and runs it at once; arguments after `--` are passed to the executable unchanged. Options are the same as `rokc build`. A library package has nothing to run, so it is an error.

A package is built to `target/<profile>/<name>`; an anonymous package (single file) is built in a temporary directory, so no `target/` is created next to the file. `rokc build <file>.rok` builds even an anonymous package into `target/`.

```
rokc run hello.rok -- arg1 arg2
```

The exit code of `rokc run` is the exit code of the program. If the program terminates by signal `n`, it is `128 + n`.

### `rokc test`: tests

`rokc test` collects test functions, builds them and runs them.

```
rokc test [path] [filter]
```

It collects functions marked `#[test]`, builds a test executable and runs it. `#[test]` implies `#[cfg(test)]` ([6.2.5](../006-meta-system/602-built-in-attributes.md#625-test-test-function-marker)), so test functions are not part of a normal build. A test build uses an entry point that selects tests instead of the package's entry point.

If there is a single argument and it is not an existing path, it is taken as `filter`. `filter` matches the test name including its module path (`math::adds`).

- It is a small regular expression using `^` `$` `.` `*` `+` `?` and `\` escapes. The search is unanchored, so without `^` / `$` it may match any part of the name.
- Without these characters it is equivalent to a plain substring search.

**Each test runs in its own process.** If a test panics (exit code 101) or ends with an uncaught error (exit code 1), only that test fails and the rest keep running. Termination by signal is also a failure.

Options are the same as `rokc build`, plus two more:

- `--threads=N` sets the number of test processes run concurrently. The default is `ROKC_NUM_JOBS`, or the CPU count if that is unset.
- `--show-output` also shows the stdout / stderr of passing tests. Output of failing tests is always shown.

Output goes to stdout in the following form.

```
running 3 tests
test math::adds ... ok
test math::subs ... FAILED
test util::trims ... ok

failures:

---- math::subs ----
<stdout / stderr of that test>
(panicked (exit 101))

failures:
    math::subs

test result: FAILED. 2 passed; 1 failed; 0 filtered out; finished in 0.12s
```

If any test fails, the exit code is `1`.

### `rokc bench`: benchmarks

```
rokc bench [path] [filter]
```

Collects functions marked `#[bench]`, runs them in an optimized build (`-O2` unless `-O` is given) and measures them ([9.9.11](909-debug-tools.md#9911-benchmarks)). Like `#[test]`, `#[bench]` implies `#[cfg(test)]`. The `path` and `filter` rules are the same as `rokc test`.

- `--iters=N` is the number of calls per sample. If omitted, it is calibrated so that one sample takes about 10 ms.
- `--samples=N` is the number of samples measured (default 10). One warm-up sample is discarded before them.

Benchmarks run one at a time, and print in cargo form the mean ns per call and the standard deviation across samples.

```
bench fib_20 ... bench:       41,503 ns/iter (+/- 312)
```

### `rokc repl`: terminal REPL

`rokc repl` launches an interactive shell in the terminal.

```
rokc repl [--replay] [--package <path>]
```

This mode runs as a terminal interactive shell. Cell semantics follow the rules of [9.7.0](907-jit-cells.md#970-rules-of-the-cell-session): each cell runs exactly once, top-level `let` bindings and items carry over to later cells, and a failed cell leaves no trace in the bindings. An expression cell prints `type: value`.

- The default mode turns each cell into LLVM IR and hands it to a JIT host process (libLLVM's ORC) that lives for the session; the host JIT-compiles it and runs it once ([9.6](906-jit-engine.md)). It is an error if libLLVM is unavailable. Bindings, `static` state and `any` values carry over between cells. A binding whose type cannot be named at top level (closures, private types, etc.) is rejected when created, with a hint pointing to `--replay`.
- `--replay` recompiles the whole session for every cell and JIT-runs it from the start in a throwaway child process forked by the host (no executable is produced). It keeps such bindings too, but the side effects of earlier cells are repeated every time.
- `--package <path>` attaches a library package via `--extern` and inserts `use <name>::*;` before the first cell. It is inserted again after `:reset`. A `main.rok` package (executable package) is rejected.

`rok` and `rok repl` ([9.10](910-rok-frontend.md)) run the same REPL with dependencies attached, and pass `--replay` through.

REPL commands (`:type`, `:reset`, `:add`, `:help`, `:quit`) and debugging aids are in [9.9.9](909-debug-tools.md#999-repl--jupyter-debugging).

### `rokc jupyter`: Jupyter kernel

`rokc jupyter` provides the kernel mode invoked by Jupyter.

```
rokc jupyter [--connection-file <path>]
```

This command is the kernel mode invoked by Jupyter, so users do not run it directly. `--connection-file` takes the path of the ZMQ connection file provided by Jupyter, and build options such as `--extern` are accepted as well. The kernel runs on the same cell session as the REPL ([9.7.0](907-jit-cells.md#970-rules-of-the-cell-session)). To run with dependencies resolved, use `rok jupyter --connection-file <path>` ([9.10](910-rok-frontend.md)). Integration details are defined in [Jupyter integration](908-jupyter-integration.md).

The kernel is registered by `rok jupyter install [--env <name>]` ([9.8.3](908-jupyter-integration.md#983-kernelspec-installation)).

### `rokc fmt`: formatter

`rokc fmt` applies the standard format.

```
rokc fmt [path]
```

`[path]` accepts files or directories (several allowed; the current directory if omitted) and rewrites them in place to the [standard format](#9111-standard-format-rokc-fmt). Directories are searched recursively for `.rok` files, skipping `target/` and directories beginning with `.`. `.rokon` files ([RokON](../../refs-standard-libraries/300-std/309-rokon.md#394-canonical-form)) are checked for being RokON and only re-indented (they are not rewritten in canonical form, so comments are preserved).

Options:

- `--check` does not modify files; it prints the files that would change and their diff. If any file would change, the exit code is `1`; suitable for CI.
- `--diff` does not modify files and prints only the diff.

A file with lexical errors is left untouched and the error is reported, with exit code `1`.

### `rokc check`: diagnostics

```
rokc check [path] [--check-library]
```

Runs the front end, lints and code generation (up to standard MLIR) and reports diagnostics. No native build (assemble, link) is performed. Options are the same as `rokc build`. `--check-library` lowers and checks every body of a library regardless of reachability.

### `rokc lint`: linter

```
rokc lint [path] [--list]
```

`rokc check` without code generation. The diagnostic format is 9.1.8; lint diagnostics have code `E503`. The list of lints and groups is in [6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control). `--list` prints the lints, groups and default levels.

Lint levels are set by the global options `-A` / `-W` / `-D` (9.1.5) and by attributes in code. If any `deny`-level lint fires, the exit code is `1`.

### `rokc doc`: documentation generation

```
rokc doc [path]
```

Generates HTML API documentation in `target/doc/` from the public items and docs (`///`, `//!`, `#[doc]`, [6.2.8](../006-meta-system/602-built-in-attributes.md#628-doc---doc-comment-form)) of the package (and `--extern` dependency packages).

```
target/doc/
├── index.html                 # package list
├── <pkg>/<mod>/index.html     # one page per module
└── search-index.js            # search index
```

Doc text is interpreted as a subset of markdown (headings, fenced code blocks, inline code, links, `**bold**`, lists, paragraphs).

### `rokc lsp`: LSP server

`rokc lsp` runs in Language Server Protocol server mode.

```
rokc lsp
```

This mode is designed to be invoked from LSP clients such as VS Code, Vim and Emacs. The standalone editor binary `rokls` is equivalent to `rok lsp` (that is, `rokc lsp`). `.rokon` buffers are checked for syntax only.

## 9.1.3. Debug / diagnostic subcommands

### `rokc demangle`: symbol demangle

`rokc demangle` decodes mangled symbols into a human-readable form.

```
rokc demangle <symbol>...
```

For each symbol it prints a block in the [9.5.4](905-symbol-mangling.md) format, separated by blank lines ([9.9.3](909-debug-tools.md#993-symbol-demangle)). If any argument is not a rok symbol, the exit code is `1`.

### `rokc rhir-text`: [RHIR](../000-meta/007-glossary.md#rhir-rok-hir-bytecode) text output

`rokc rhir-text` prints the RHIR of a prebuilt library as text.

```
rokc rhir-text path/to/foo.rhir      # or libfoo.a — reads the sibling foo.rhir
```

It prints the header fields, producer, meta and interface files; the format is described in [RHIR format](904-rhir-bytecode.md).

### `rokc inspect`: package inspection

`rokc inspect` gives an overview of an archive's contents.

```
rokc inspect path/to/libfoo.a        # or foo.rhir
```

It shows the package name and hash, rokc version, RHIR version, GC type id range, exports, [generic](../000-meta/007-glossary.md#generic) items, and archive members with their sizes ([9.9.14](909-debug-tools.md#9914-package-inspection)).

## 9.1.4. Auxiliary subcommands

### `rokc clean`: remove build output

`rokc clean` removes build output.

```
rokc clean [path]
```

Deletes the package's whole `target/` directory (including all profiles and cross builds; the `--target-dir` / `ROKC_TARGET_DIR` directory if set). Does nothing if it does not exist.

### `rokc version`: version

```
rokc version
rokc --version            # = -V
rokc -V -v                # adds host line
```

Prints one line `rokc <version> (<commit> <date>)` to stdout. With `-v`, a `host: <triple>` line is appended.

### `rokc help`: help

```
rokc help [subcommand]
rokc <subcommand> --help
```

Given a subcommand, shows the help for that command.

## 9.1.5. Global options

The global options are listed below. They may appear before or after the subcommand.

| Option                           | Meaning                                                     |
|----------------------------------|-------------------------------------------------------------|
| `--verbose` / `-v`               | Show per-phase timing ([9.9.5](909-debug-tools.md#995-logging--progress-display)) |
| `-vv`                            | More detailed logging                                       |
| `--quiet` / `-q`                 | Do not print progress lines (diagnostics are still printed) |
| `--color=auto\|always\|never`    | Colored diagnostic output                                   |
| `--no-color`                     | Equivalent to `--color=never`                               |
| `--cwd=<path>`                   | Change working directory before running                     |
| `--message-format=human\|json`   | Diagnostic format. `json` writes one line per diagnostic to stdout (9.1.8) |
| `-A` / `-W` / `-D <lint \| group>` | Lint level (`--allow` / `--warn` / `--deny`). Applies to `check` / `lint` / `lsp`. May be repeated; the later one wins |

The lint level is determined in the order: the lint's default level < `-A` / `-W` / `-D` < `#[allow(..)]` / `#[warn(..)]` / `#[deny(..)]` in code (innermost wins). `rok` prepends the manifest's `lints` as these flags ([9.10.1](910-rok-frontend.md#9101-subcommands-and-global-options)).

`--color=auto` (the default) uses color when stderr is a terminal and the environment variable `NO_COLOR` is not set.

Examples:

```
rokc --verbose build
rokc test -q
rokc --cwd=examples/hello run
```

## 9.1.6. Exit codes

Exit codes are defined as follows.

| Code  | Meaning                                              |
|-------|------------------------------------------------------|
| `0`   | Success                                              |
| `1`   | User code error (compile error / test failure, etc.) |
| `2`   | Usage error (invalid option, unknown subcommand, etc.) |
| `101` | rokc internal error (panic)                          |

`rokc` does not read `ROK_HOME` (the sysroot is relative to the executable). If rokc itself panics (internal compiler error), it prints a message and exits with `101`. `rokc run` exits with the exit code of the program it ran (9.1.2).

## 9.1.7. Environment variables

The environment variables that affect `rokc` are:

| Variable          | Meaning                                                                                  |
|-------------------|------------------------------------------------------------------------------------------|
| `ROK_CC`          | C compiler that performs the final link. Default is `xcrun clang` on Apple, LLVM's `clang` elsewhere ([9.2.18](902-aot-pipeline.md)) |
| `ROKC_VERBOSE`    | Equivalent to `--verbose`                                                                |
| `ROKC_TARGET_DIR` | Location of `target/` (default: inside the package)                                      |
| `ROKC_CACHE_DIR`  | [JIT](../000-meta/007-glossary.md#jit-just-in-time) cache directory (default `~/.cache/rok/`) |
| `ROKC_NUM_JOBS`   | Default for `rokc test --threads` (CPU core count if unset)                              |
| `NO_COLOR`        | If set, `--color=auto` does not use color                                                |

## 9.1.8. Output format

### Diagnostic messages

Diagnostic messages are printed in the following format.

```
error: 'count' is not declared
  --> src/main.rok:5:10
   |
 5 |     let x = count + 1;
   |             ^^^^^ unknown identifier
   |
   = note: did you mean 'counter'?
```

Diagnostics use the file + line + column units defined in [Source files](../001-lexical/101-source-files.md).

### Build progress

Progress is shown on stderr with right-aligned verbs. Nothing is printed with `-q`.

```
   Compiling foo (path/to/foo)
    Building target/debug/foo
    Finished in 1.23s
     Running `target/debug/foo`
```

| Verb        | When                                          |
|-------------|-----------------------------------------------|
| `Compiling` | Front end starts (package name and directory) |
| `Building`  | Executable / archive produced (`build`)       |
| `Wrote`     | One `--emit` output written                   |
| `Finished`  | Build done (elapsed time)                     |
| `Running`   | Program run (`run`)                           |
| `Created`   | Package created (`new`)                       |
| `Removed`   | `target/` deleted (`clean`)                   |

### JSON output (for tool integration)

For tool integration, output can also be produced as JSON.

```
rokc build --message-format=json
```

In this mode each diagnostic becomes one line of JSON object on stdout, easy for IDEs and CI tools to parse. The shape of the object is defined in [9.9.4](909-debug-tools.md#994-diagnostics--error-messages).

## 9.1.9. Standard input

Writing `-` in an argument position, as in `rokc check -`, means standard input, for use with pipes or scripts. Source from standard input is checked as the `main.rok` of an anonymous executable package.

## 9.1.10. Compatibility with external tools

`rokc` is a low-level compiler; dependency resolution is the job of the front-end `rok` ([9.10](910-rok-frontend.md)). `rok` invokes `rokc`, passing the roots of resolved dependency packages via `--extern` ([9.2.3](902-aot-pipeline.md#923-dependency-resolution)). Other build systems may integrate by invoking `rokc` directly in the same way. Changes that break CLI compatibility are managed under SemVer rules.

## 9.1.11. Standard format (`rokc fmt`)

The standard format of `rokc fmt` is a **token-preserving layout rule**. It does not change tokens or comments within a line; it only sets line indentation and blank lines.

1. Indentation is 4 spaces per bracket (`(` `[` `{` `#[`) depth. A line beginning with a closing bracket takes the depth of the line that opened it. Tabs become spaces.
2. A continuation line, i.e. one whose preceding token is not `;` `,` or a bracket (such as a line splitting a long expression), keeps the amount by which it was indented beyond the block indentation in the original file.
3. Trailing whitespace is removed.
4. Consecutive blank lines are collapsed to one, and blank lines at the start of the file are removed.
5. The file ends with exactly one newline.

The formatter does not move tokens between lines, does not split or join lines, and does not change spacing within a line. Comments therefore stay exactly as written and the token sequence is unchanged. The interior of multi-line strings and block comments is also left as is.
