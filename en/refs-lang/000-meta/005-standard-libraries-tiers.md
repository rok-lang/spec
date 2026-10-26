# 0.5. Standard library tiers

The standard libraries are divided into three package tiers under the virtual namespace `rok::*`. They are distinguished by the environments in which they are available and by the risk of the behavior they expose.

## 0.5.1. Three tiers

The overall structure is as follows.

```
  rok::bare   ←   no safety guarantees. Raw memory, FFI, transmutable bit casts, direct MLIR access.
     ↑
  rok::core   ←   safe foundation available on every target. Uses rok::bare internally.
     ↑
  rok::std   ←   target-dependent tier. File system, network, OS threads, etc.
```

The arrows show the direction of dependency. `rok::core` may use `rok::bare` in its internal implementation, and `rok::std` may use both `rok::core` and `rok::bare`. Dependencies in the opposite direction are forbidden.

All three tiers live under the virtual namespace `rok::` ([5.7.4 rok::](../005-modules/507-paths.md#574-rok-the-standard-library-virtual-namespace)). `rok` itself is a path-root contextual keyword ([1.5.4](../001-lexical/105-keywords.md)) and does not collide with user identifiers.

## 0.5.2. Responsibilities of each tier

### `rok::bare`

`rok::bare` is the low-level, dangerous area. Behavior whose safety the compiler cannot guarantee gathers here; representative items include raw pointer forms (the raw pointer abstraction) and arbitrary bit casts (`transmute`-like). Pinning, volatile memory access, and the primitives of C [ABI](007-glossary.md#abi-application-binary-interface)-compatible memory allocation are also part of `rok::bare`. Direct calls into MLIR dialects (`rok::bare::mlir`) and registration of user-defined dialect plugins (`rok::bare::mlir::plugin`) also live in this tier and serve as the channel through which external tensor / SIMD / GPU libraries reach the acceleration domain.

The caller of a function in this tier is responsible for safety, and the path itself makes that fact clear.

### `rok::core` (foundation)

`rok::core` is the most frequently used safe library; every module of the foundation lives here (e.g. `rok::core::vec`, `rok::core::iter`, `rok::core::ops`). It must be available identically on every build target this specification guarantees. Its contents are very broad: numeric, character and string types and their methods, and collections (lists, maps, sets, tuples). It has the trait definitions for arithmetic, comparison, logical and bitwise operators, and also provides iterators, streams and futures. Concurrency primitives (atomic, mutex, channel) and the API surface for tasks and parallel execution also belong to this tier. Finally, it covers formatting, display, debug, reflection and inspection.

Acceleration domains such as tensors / automatic differentiation / SIMD / GPU are not part of `rok::core`; external libraries implement them directly on top of `rok::bare::mlir`. The standard libraries are responsible only for the surface semantics of ordinary code and leave the acceleration domain to decisions at the library layer.

The items of `rok::core` are made visible by the compiler implicitly applying `use rok::core::**;` at the start of every Rok source. That is, `rok::core::vec::Vec` can be written simply as `Vec`. There is no separate prelude module; the implicit recursive glob itself plays that role.

Items of `rok::core` are `pub` by default. The surface exposed to external packages is exactly the automatically visible surface, so no separate step is needed.

### `rok::std`

`rok::std` is the area that is meaningful only in particular environments (mainly on top of an OS). It contains the file system and networking, time and clocks, standard I/O, environment variables, processes, and direct control of OS threads.

Functions of `rok::std` may not exist on some targets. In environments where some areas are meaningless (for example, a notebook kernel running inside a browser), the corresponding modules are not built.

## 0.5.3. Visibility conventions

The visibility rules are as follows.

| Area            | Implicit import | Explicit import required | Notes                                         |
|-----------------|-----------------|--------------------------|-----------------------------------------------|
| `rok::core::**` | ✓ automatic     | ✗ not required           | `use rok::core::**;` applied automatically    |
| `rok::bare::**` | ✗               | ✓ required               | Makes the danger signal visible in code       |
| `rok::std::**` | ✗               | ✓ required               | Makes target dependency visible in code       |
| User packages   | ✗               | ✓ required               | Items of other packages                       |

The concrete syntax for "bringing in" (`use`, `import`, etc.) is defined in the module system specification.

## 0.5.4. Distribution form and linking mechanism

The three tiers are provided as prebuilt static archives distributed with the compiler.

```
librok_bare.a       (-NL bare)
librok_core.a       (-NL core)
librok_std.a       (-NL std)
```

Each archive has its own package ABI, and the compiler links it automatically at build time through the `-NL <name>` mechanism. This path is separate from the ordinary dependencies of user packages, so a user package that happens to have the same name (e.g. `mod bare`, `mod std`) does not collide.

Like the compiler itself, the three archives follow the zero-dependency principle. Building a user package links only the tiers it needs, and the resulting artifact requires no additional external libraries.

## 0.5.5. Why three tiers

With only two tiers (safe, dangerous), "safe anywhere" could not be distinguished from "safe only when an OS is present", and the split would break down in environments lacking some OS features (browser notebooks, etc.). Conversely, four or more tiers increase the rules users must memorize and diminish the value of module boundaries.

Three tiers is a split well proven in practice, and the simplest one that is still meaningful.

Grouping them all under the virtual namespace `rok::` further lowers the learning cost. Users need only remember the one-line rule "the standard libraries are all under `rok::`", and learn the division of responsibility among `bare` / `core` / `std` gradually.
