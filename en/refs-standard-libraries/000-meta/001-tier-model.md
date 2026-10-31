# 0.1. The 3-tier model

The Rok standard libraries are divided into three tiers. Each tier may depend on a different
area, and the import rules in user code differ as well.

```
rok::bare   (base, depends only on runtime extern "C")
  ↓
rok::core     (foundation, safe, all targets; uses bare internally)
  ↓
rok::std   (target-dep; uses foundation + bare)
```

All three tiers live under the virtual namespace `rok::` ([refs-lang 5.7.4](../../refs-lang/005-modules/507-paths.md#574-rok-the-standard-library-virtual-namespace)).

The reasons for the three-way split are clear. Raw operations that can break memory safety are
easier to audit when confined to a narrow area, and core abstractions that must behave identically
on every platform can guarantee portability only when they are not mixed with OS-dependent code.
Features whose availability varies by platform must expose that fact in their signatures and import
sites so that users can handle wasm builds and native builds consistently. Each of the three tiers
is responsible for one of these three requirements.

## 0.1.1. Responsibilities per tier

### `rok::bare`

The bottom tier, `rok::bare`, depends only on the runtime's `extern "rok"` symbols.
It exposes raw tools such as `Ptr<T>`, `ConstPtr<T>`, `transmute`, raw allocation, pin/unpin and
volatile, so using them always requires an explicit `use`. Being explicit is the danger signal:
it is a deliberate device so that dangerous areas are visible at a glance during code review.

```rok
use rok::bare::ptr::CPtr;
use rok::bare::alloc::{alloc, Layout};
let layout = try Layout::array::<u8>(1024);
let p: CPtr<u8> = unsafe { alloc(layout) };    // null on failure (see 1.4 alloc)
```

### `rok::core` (foundation)

The middle, foundation tier, `rok::core`, holds modules such as `rok::core::collections`, `rok::core::iter` and `rok::core::ops`, and depends on `rok::bare` only internally. Nearly every abstraction needed for everyday use of the language (numeric types, arithmetic traits,
iter, collections, fmt, conversion, sync, task, future, etc.) is gathered in this tier. In user code,
**`use rok::core::**;` is applied implicitly and automatically** (see [0.2 Import policy](002-import.md)
for the detailed policy), so it can be used directly without any import.

```rok
let xs: List<i32> = [1, 2, 3];        // List used directly, brought in by the implicit use rok::core::**;
let r = a + b;                         // Add trait automatic
let m = Mutex::new(0);                 // sync::Mutex automatic too
task::spawn(async { /* ... */ });              // task module automatic too
```

Note that there is no separate `rok::core::prelude` module for this automatic application.
The `**` recursive glob itself plays the role of the prelude, leaving no room for an additional
indirection.

### `rok::std`

The top tier, `rok::std`, uses both the foundation and `rok::bare` as needed. This tier holds
platform-dependent items such as `fs`, `net`, `time`, `thread` (OS threads), `io`, `env`, `process`
and `path`. Because availability varies by target, users must always import them explicitly, making
clear which platform features they are stepping into.

```rok
use rok::std::fs;
use rok::std::time::Instant;
```

## 0.1.2. Dependency rules (enforced)

The following dependencies are allowed.

| Dependent   | May depend on                                             |
|-------------|-----------------------------------------------------------|
| `rok::bare`  | runtime extern "C"                                       |
| `rok::core`  | `rok::bare`                                              |
| `rok::std`  | `rok::core`, `rok::bare`                                  |
| User code   | `rok::core` (implicit), `rok::std` (explicit), `rok::bare` (explicit) |

Dependencies in the following directions are **forbidden**. The foundation (`rok::core`) must not
import `rok::std`: the foundation must build on every target, and portability collapses the moment
it pulls in platform-dependent code. `rok::bare` must not import the foundation or `rok::std`
either: once a low-level tier starts depending on a high-level one, the layering itself loses its
meaning. Finally, circular dependencies among individual `rok::std::*` modules are forbidden, so
each std module remains a unit that can be built independently.

The rule governs what a tier writes: its `use`s and paths. The implicit `use rok::core::**` still supplies every tier the language's vocabulary (the 0.4 slot items, `List`, `Slice`, `Result`, `panic`) that bare's own signatures use (1.6, 1.7).

## 0.1.3. Build units

Each of the three tiers is processed as a separate build unit (a separate archive) and can be cached per compilation.
The link order runs `rok::std` → `rok::core` → `rok::bare` → runtime, with the user package on top
of everything. The compiler links them automatically through the `-NL <tier>` mechanism, so users do
not need to add them to dependency declarations (see [0.5.4 Distribution form and linking mechanism](../../refs-lang/000-meta/005-standard-libraries-tiers.md#054-distribution-form-and-linking-mechanism)).

## 0.1.4. Target availability

`rok::bare` and the foundation (`rok::core`) build on every [Tier 1 platform](../../refs-lang/000-meta/006-platform-targets.md),
including wasm32. `rok::std`, by contrast, has availability that varies by target. For example,
`rok::std::thread` is not supported on wasm32, because the browser environment runs main thread
only. `rok::std::fs` is likewise unavailable on wasm32-unknown-unknown and is provided only on
the WASI targets (`wasm32-wasi`, `wasm32-wasi-threads`). Such availability is stated in the availability section of each `rok::std::*` module spec.
The API still builds on every target: an operation a target lacks fails at run time with
`IoErrorKind::Unsupported`. In the browser, stdout / stderr go to the host console and stdin is at EOF.

## 0.1.5. Evolution policy

The three tiers also differ in stability policy. The `rok::bare` API is not stable and
evolves together with the compiler and runtime, so user code is advised to pin the version.
The foundation (`rok::core`) API aims for SemVer stability, and direct changes are accompanied by a major
version bump. The `rok::std` API is as volatile as its per-target availability differences, but the
signatures themselves remain stable.

Within a release series an item is never removed or changed incompatibly. An item that is replaced
keeps working and is marked `#[deprecated(since = "..", note = "..")]`
([6.2.7](../../refs-lang/006-meta-system/602-built-in-attributes.md#627-deprecated-discouraged-use-marker)),
the note naming what to use instead. A safe function that must become `unsafe` is marked
`#[future_unsafe]` ([6.2.14](../../refs-lang/006-meta-system/602-built-in-attributes.md#6214-future_unsafe-safe-today-unsafe-later)).
The next breaking release removes the item or makes it `unsafe`.
