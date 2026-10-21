# 9. Execution / tools overview

This block specifies the rok compiler `rokc`, the front-end `rok`, and the execution environments (REPL / Jupyter kernel). It covers how the two engines ([JIT](../000-meta/007-glossary.md#jit-just-in-time) / [AOT](../000-meta/007-glossary.md#aot-ahead-of-time)) and the three user surfaces (REPL / Jupyter / direct execution of AOT output) defined in [Execution engines and user surfaces](../000-meta/001-execution-modes.md) are implemented.

## Contents

- [9.1. `rokc` CLI interface](901-rokc-cli.md)
- [9.2. AOT build pipeline](902-aot-pipeline.md)
- [9.3. Package archive (.a) format](903-package-archive.md)
- [9.4. ROKHIR format](904-rokhir-bytecode.md)
- [9.5. Symbol mangling scheme](905-symbol-mangling.md)
- [9.6. JIT engine (RokJitSession)](906-jit-engine.md)
- [9.7. JIT cell execution flow and state management](907-jit-cells.md)
- [9.8. Jupyter kernel integration](908-jupyter-integration.md)
- [9.9. Debug / diagnostic tools](909-debug-tools.md)
- [9.10. `rok` — package / build front-end](910-rok-frontend.md)

## Tool surface

The tool surface is organized as follows.

```
rok  (front-end, cargo-like)
  │   new / init, add / remove / update / deps (rok.rokon, rok.lock), env, toolchain
  │   build / check / run / test / bench / emit / fmt / lint / doc / clean / repl / lsp / demangle
  │        └── resolve deps, then call rokc ... --extern <dep roots>
  ▼
rokc  (compiler)
  │
  ├── build / run
  │   ├── rokc build      : AOT compile — .rok source to .a / executable
  │   ├── rokc run        : AOT compile then run immediately
  │   ├── rokc test       : collect + run test functions (#[test])
  │   ├── rokc bench      : collect + run benchmarks (#[bench])
  │   ├── rokc check      : type / diagnostics only (no native build)
  │   └── rokc clean      : remove build artifacts
  │
  ├── tools
  │   ├── rokc fmt        : formatter
  │   ├── rokc lint       : linter
  │   ├── rokc doc        : API documentation
  │   └── rokc lsp        : LSP server (IDE integration)
  │
  ├── interactive
  │   ├── rokc repl       : terminal REPL (each cell runs once)
  │   └── rokc jupyter    : Jupyter kernel (REPL cell session + ZMQ)
  │
  ├── debug / diagnostic
  │   ├── rokc demangle   : symbol demangle
  │   ├── rokc rokhir-text  : ROKHIR interface text dump
  │   └── rokc inspect    : package inspect
  │
  └── rokc version / help

rokls  (= rok lsp, for editors)
librokr.a  (runtime, rokr)
```

The toolchain consists of the compiler `rokc` (the counterpart of rustc), the cargo-style front-end `rok`, the editor LSP executable `rokls`, and the runtime `librokr.a`. `rok` owns the manifest `rok.rokon` / `rok.lock`, dependencies, environments and the toolchain pin; build and tool commands are delegated to `rokc` with the resolved dependency packages attached via `--extern` ([9.10](910-rok-frontend.md)). There are no separate tools (`rok-fmt` etc.).

## Implementation of the two engines

The two engines defined in [Execution engines and user surfaces](../000-meta/001-execution-modes.md) are divided as follows.

```
JIT engine  ──  used by rokc repl / jupyter
AOT engine  ──  used by rokc build / run / test
```

Both engines go through the same compile pipeline (rok-hir → rok-mir → standard MLIR) and diverge only after the last intermediate representation. See [Execution engines and user surfaces](../000-meta/001-execution-modes.md) and [Compile pipeline](../000-meta/002-pipeline.md) for details.

## Implementation of the three user surfaces

The three user surfaces and the commands that implement them are:

```
surface              command
────────             ──────
REPL                 rokc repl
Jupyter              rokc jupyter        (invoked by Jupyter, not directly)
AOT direct exec      rokc build → run artifact
```

The implementation of each surface is covered in the following documents.

## Package build output

A library package of a [source unit](../000-meta/003-source-units.md) compiles to `libfoo.a` and the interface `foo.rokhir`.

```
foo/
├── lib.rok                  <- input
└── ...
       v rokc build
foo/target/debug/
├── libfoo.a                 <- artifact (static archive)
├── foo.rokhir                 <- interface (ROKHIR)
├── build/                   <- intermediates (foo.o, ...)
└── ...
```

The per-profile layout is defined in [9.2.15](902-aot-pipeline.md#9215-output), and the [archive](../000-meta/007-glossary.md#archive) format in [Package archive](903-package-archive.md).

## ROKHIR: portable IR for generic functions

Generic functions must be [monomorphized](../000-meta/007-glossary.md#monomorphization) at the call site, so binary code alone is not enough. An interface file, ROKHIR, therefore sits next to the library. ROKHIR holds the package source with concrete function bodies stripped (because monomorphization takes the AST as input).

The detailed format is defined in [ROKHIR format](904-rokhir-bytecode.md).

## Single dependency: `librokr.a`

Build output consists of the user package's `libfoo.a`, the runtime `librokr.a`, and the standard libraries `librok_*.a`. There are no further dependencies beyond external OS libraries.

The detailed build flow is defined in [AOT build pipeline](902-aot-pipeline.md).
