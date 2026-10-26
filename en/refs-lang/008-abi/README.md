# 8. Runtime ABI / MLIR lowering overview

This block defines the shared contract between the compiler (`rokc`) and the runtime (`librokr.a`). The compiler and the runtime are connected through the text of this specification. The only code they share is a single definition of the GC ABI constants (`gc_abi`, [8.1](801-object-header-abi.md)).

## Contents

- [8.1. Object header ABI](801-object-header-abi.md)
- [8.2. ScanTable serialization format](802-scan-table.md)
- [8.3. Mutator TLS slot layout](803-tls-slots.md)
- [8.4. Safepoint protocol](804-safepoint.md)
- [8.5. Extern "C" symbols](805-extern-symbols.md)
- [8.6. Calling convention](806-calling-convention.md)
- [8.7. MLIR dialect usage](807-mlir-dialects.md)
- [8.8. Lowering: data types and expressions](808-lowering-types.md)
- [8.9. Lowering: control flow and GC integration](809-lowering-control.md)
- [8.10. Lowering: coroutines and generics](810-lowering-coroutine-generic.md)

## Two kinds of contract

### 1. Runtime ABI

The interface used when compiler-generated code is linked and run together with `librokr.a`. The memory layout of objects is specified so that the [GC](../000-meta/007-glossary.md#gc-garbage-collector) can interpret compiler-created objects exactly. [Extern "C" symbols](../000-meta/007-glossary.md#extern-c-symbol) are runtime functions the compiler may call; the thread-safety class of each symbol (`thread-safe` / `caller-thread-only` / `process-init`, etc.) is defined in [8.5 Extern "C" symbols](805-extern-symbols.md#850-thread-safety-classes). TLS slots are thread-local locations read and written by inline code the compiler emits. The calling convention specifies register and stack usage in function calls.

### 2. MLIR lowering

The rules, applied in the rok-mir → standard MLIR stage of the [compilation pipeline](../000-meta/002-pipeline.md), that translate Rok's high-level semantics into standard MLIR dialects. `tagged union`, `any` slots, pattern matching, try/catch and generics are all lowered to standard MLIR dialects. There is no custom Rok dialect.

## Why this specification text is authoritative

The compiler and the runtime share no code or header files other than `gc_abi`. The guarantee that both follow the same [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) comes from this specification text and from end-to-end verification in integration tests.

This model follows the same approach as Swift ABI Stability: the specification is authoritative and the implementation follows it.

## Stability

The contract in this block is part of the language version (SemVer). ABI changes come with an explicit version bump. Minor / patch releases may only add and must not break compatibility. Major releases may break compatibility, but do so in a way the package manager can detect.

The ROKHIR header of a prebuilt package records the `rokc` that built it ([9.4](../009-tools/904-rokhir-bytecode.md)). Using a package built by a different `rokc` is an error that asks for a rebuild.
