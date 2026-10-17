# 0. Introduction

## Contents

- [0.1. Execution engines and user surfaces](001-execution-modes.md)
- [0.2. Compilation pipeline](002-pipeline.md)
- [0.3. Source units](003-source-units.md)
- [0.4. Boundary between the language and the standard libraries](004-language-vs-standard-libraries.md)
- [0.5. Standard library tiers](005-standard-libraries-tiers.md)
- [0.6. Platform targets](006-platform-targets.md)
- [0.7. Glossary](007-glossary.md)
- [0.8. Notation](008-notation.md)

## What rok is

**rok** is a general-purpose programming language aimed primarily at science and data analysis. In science it covers numerical computation, simulation and modeling. In data analysis it covers tabular data processing, statistics and visualization. As a general-purpose language it is not limited to these areas: system tools, web backends and CLIs can reasonably be written in it as well.

Real workflows cut across all three. Code that starts in a notebook becomes part of a static artifact, then is packaged as a library and called from another notebook. rok is designed so that this flow continues within a single language.

## Design slogan

> [!IMPORTANT]
> **"A small language, rich libraries."**

The language specification keeps its own scope narrow. Numeric operations, conversions, collections, I/O, concurrency and the like are all delegated to the standard libraries. The language core stays small, and domain semantics are decided by libraries.

There are two motivations for this split. For modifiability, domain-specific decisions for areas such as science and data can evolve at the library level. For verifiability, the smaller the language specification, the lower the cost of implementing, verifying and teaching it.

The exact boundary between the language and the libraries is declared in [0.4. Boundary between the language and the standard libraries](004-language-vs-standard-libraries.md).

## Freedom within control

> **"The clearer the rules, the freer you are within them."**

rok does not shy away from constraints. Static types, closed sum types, the trait-as-type ban, the ban on implicit promotion, and AOT-only compilation are such decisions. They narrow the user's options. Within those narrower options, however, the user writes code that is predictable and deeply controllable.

This philosophy comes from two directions. On one side are the areas the language controls: types, dispatch, the memory model and evaluation order. The language decides these once, rather than the user deciding each time. On the other side are the areas where the user is free: library design, domain modeling, algorithm choice and code structure. The language imposes nothing there, and the libraries and the meta-system permit deep expressiveness.

The coexistence of gradual `any` slots with static types, the guarantee of identical semantics between JIT and AOT, and extending the language itself through the meta-system are all concrete forms of this principle. Free areas are possible only on top of a well-controlled core.

## Non-goals

The following are not priority areas for rok. This does not mean they are excluded forever; it means the initial specification and implementation do not compromise for them. They are: mobile app development (iOS, Android); bare-metal embedded and environments without an operating system; and 16-bit and 8-bit microcontrollers.

These areas already have sufficiently mature language ecosystems of their own and lie off the core path of science and data workflows.
