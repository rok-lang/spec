# 0.7. Glossary

The same concepts recur across many chapters of this specification, so it helps readers to collect frequently used terms in one place. This document gathers the core terms that appear throughout the Rok specification, so that a reader can consult it directly whenever a concept becomes unclear while reading the body. Each entry gives a short definition together with its specific Rok context, indicating not just the dictionary meaning but the place the term occupies within the specification.

## ABI (Application Binary Interface)

The ABI is the binary-level contract that compiled code must follow at link and run time. It includes object layout, calling conventions, symbol naming and so on. Rok's ABI is defined in [Chapter 8: ABI / MLIR lowering](../008-abi/README.md).

## AOT (Ahead-of-Time)

AOT is the mode that compiles source ahead of time into a static artifact (`.a` or an executable), in contrast to JIT. Rok supports both the JIT and AOT engines, and the two share the same pipeline up to rok-mir.

## archive

An archive is the output of a compiled package: a `.a` file in standard ar format. It holds the objects for concrete code; the library's interface (ROKHIR) is the `<name>.rokhir` file beside it.

## block

A block is a group of statements and expressions enclosed in `{ ... }`. It is itself an expression, and its last expression becomes the block's value.

## block (Immix)

An Immix block is the GC's memory unit, corresponding to a 32 KB OS memory page. In the Immix algorithm it is divided into lines and serves as the unit of mark / sweep. It is a different concept from the code block above.

## concrete fn / generic fn

A concrete fn is a function whose types at the call site are all concrete; a generic fn is a function with remaining type parameters. Under AOT, a generic fn is specialized at the call site into a concrete fn.

## dispatch

Dispatch is the process of determining which implementation a name, operator or method call is bound to. Rok uses only static dispatch; because of the trait-as-type ban, there are no indirect vtable calls.

## extern "C" symbol

An extern "C" symbol is a symbol exposed or called through the C ABI; such symbols form the interface between the compiler and the runtime.

## FFI (Foreign Function Interface)

FFI is the interface for calling functions in other languages (mainly C) or being called from other languages. It is defined with `extern "C"` blocks.

## GC (Garbage Collector)

The GC is the automatic memory management mechanism. Rok's GC combines Immix mark-region with Sticky Mark-Bit generational collection.

## generic

A generic is a function or type that takes types as parameters. It is monomorphized at call and instantiation sites.

## HIR (High-level IR) / MIR (Mid-level IR)

HIR and MIR are intermediate representations of the compilation pipeline. The overall flow is source → AST → HIR → MIR → machine code (native backend) or standard MLIR → LLVM IR (MLIR → LLVM backend).

## Immix

Immix is the GC algorithm published by Blackburn and McKinley in 2008. It uses mark-region in units of 32 KB blocks and 128 B lines, and handles fragmentation with opportunistic evacuation.

## JIT (Just-in-Time)

JIT is the mode that compiles code immediately at run time, with the native backend. The REPL and Jupyter use this mode; it contrasts with AOT.

## lang slot / magic

A lang slot is the mechanism by which the language identifies specific library items. `#[lang(slot_name)]` connects a standard library item to a compiler slot.

## LOS (Large Object Space)

The LOS is the GC's space dedicated to large objects. Objects at or above the threshold (8 KB / 2 KB) are kept in the LOS without being moved.

## monomorphization

Monomorphization is the process of compiling a generic function separately for each concrete type at its call sites. It gains zero indirect-call cost at the price of compile time and binary size.

## ROKHIR (Rok HIR bytecode)

ROKHIR is the interface file (`<name>.rokhir`) of a prebuilt library package. It sits beside the archive and lets downstream packages read declarations and specialize generic fns. Its body is the package source with concrete function bodies stripped out ([9.4](../009-tools/904-rokhir-bytecode.md)).

## niche optimization

Niche optimization is an optimization that eliminates the tag slot by using unused bit patterns of a sum type or enum. The canonical example is `none = null pointer` in `Ptr | none`.

## scan_table

A scan_table is the meta table the GC uses to find which slots of an object are GC pointers. It is mapped `type_id → scan_table` and registered by the compiler at build time.

## shadow stack

The shadow stack is an auxiliary stack the GC uses to trace live objects. The compiler directly emits code that links and unlinks, at function entry and exit, a frame holding the slots of GC object variables. Frames live on the machine stack, and the head of the chain is kept per thread in TLS.

## specialize

Specialize is another name for [monomorphization](#monomorphization). It is used as the name of a stage in the AOT / JIT pipelines.

## stone

A Rok package published to a registry. Its name is `[a-z0-9_-]+` (starting with a letter), and in code `-` is written as `_`. The registry treats `-` / `_` and differing case as the same name. In `rok.rokon` it is required with `Dep::version("req")` or the string `"req"` ([9.10.4a](../009-tools/910-rok-frontend.md#9104a-stones-and-rokstonesio)).

## sticky mark-bit

The sticky mark-bit is the generation-tracking bit of the generational GC. It is set on the first mark, making the object an "old object". A young GC skips sticky objects to reduce cost.

## sum type

A sum type is a type of the form `A | B` that holds one of the alternatives; it is a form of tagged union. It is a closed slot: all variants are explicit, so exhaustiveness checking of pattern matching is possible.

## TLAB (Thread-Local Allocation Buffer)

A TLAB is a thread-local memory buffer owned by each worker thread. Fast-path allocation is done with only a bump pointer, with zero synchronization cost.

## work-stealing

Work-stealing is the scheduler's load-balancing mechanism. Each worker has a local deque, and a worker that runs out of work steals tasks from the end of another worker's deque.

## unsafe block / unsafe fn

The `unsafe { ... }` block expression and the `unsafe fn` function modifier are Rok's single mechanism for visibly confining regions outside the guarantees of safe abstractions. Raw pointer dereference, calls to `unsafe fn`, direct calls of MLIR ops, etc. are allowed only inside an unsafe scope. See [4.15 unsafe](../004-functions/415-unsafe.md) for details.

## MLIR plugin

An MLIR plugin is the mechanism for dynamically registering an externally built MLIR dialect with the MLIR embedded in the Rok compiler. It is loaded via the `--load-mlir-plugin=<path>` CLI option or a package manifest declaration, and loaded ops become callable through `rok::bare::mlir::__mlir_op`. It is used in domain-specific compiler areas (cryptography, signal processing, new accelerator ISAs, etc.). See [refs-standard-libraries 1.10](../../refs-standard-libraries/100-bare/110-mlir-plugin.md) for details.

## rokc / rok / rokls / rokr

The executables and runtime that make up the toolchain. `rokc` is the compiler (the counterpart of rustc, [9.1](../009-tools/901-rokc-cli.md)), with LLVM, MLIR and lld built in; `rok` is the cargo-like front-end that handles the manifest, dependencies, environments and toolchain pins and invokes `rokc` ([9.10](../009-tools/910-rok-frontend.md)); `rokls` is the LSP server executable for editors (`rok lsp`); and `rokr` is the runtime (`librokr.a`). The executables in `$ROK_HOME/bin/` are proxies to the active toolchain (`$ROK_HOME/toolchains/<version>-<triple>/`), and `rokc` finds the standard libraries and runtime in a sysroot relative to its own executable ([9.2.19](../009-tools/902-aot-pipeline.md#9219-installation-layout)).

## ROKON

Rok Object Notation. A data format for writing values using a subset of Rok's literal syntax, with the extension `.rokon`. The manifest `rok.rokon` and the lockfile `rok.lock` use this format ([stdlib 3.9](../../refs-standard-libraries/300-std/309-rokon.md)).

## rokstones.io

Rok's default stone registry. Its index is the separate git repository `rokstones.io-index` ([9.10.4a](../009-tools/910-rok-frontend.md#9104a-stones-and-rokstonesio)).

## rok.rokon / rok.lock

The manifest and lockfile at the package root. The names are spelled exactly this way (all lowercase), and both use the [ROKON](#rokon) format. A package without dependencies does not need them ([0.3.4](003-source-units.md#034-the-manifest-rokrokon-is-optional)). `rok.rokon` declares dependencies and toolchain pins as a single `Package { .. }` value, and `rok.lock` pins the results resolved by `rok update`.

## environment

The unit in which `rok` keeps a dependency set. Every directory containing `rok.rokon` is an environment, and there is no activation step. Outside a project, a named environment in `$ROK_HOME/envs/` (default `default`) is used ([9.10.5](../009-tools/910-rok-frontend.md#9105-environments)).

The entries collected here are not mere word translations; they also form a compact index of the foundations Rok stands on: the compilation pipeline, memory management, the type system, the concurrency model and so on. Whenever a concept blurs while reading the body, returning to this chapter clarifies the place each term occupies in the specification as a whole.
