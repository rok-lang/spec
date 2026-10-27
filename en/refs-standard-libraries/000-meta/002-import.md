# 0.2. Import policy - implicit `use rok::core::**;`

> This document defines the implicit import policy of the standard libraries. Under the Rok virtual namespace, only the `rok::core` area is made visible automatically; `rok::bare` and `rok::std` require explicit imports.

Rok **implicitly** prepends the following single line to every user source file.

```rok
use rok::core::**;
```

Here `**` is a **recursive glob**: it brings in at once the pub items of the `rok` module itself and of all its submodules (`rok::core::ops`, `rok::core::iter`, `rok::core::collections`, `rok::core::sync`, `rok::core::task`, ...).

> [!NOTE]
> Rok **has no separate `rok::core::prelude` module**. The convention of a **re-export module** in the style of Rust's `std::prelude::v1::*` or Haskell's `Prelude` was rejected, and the `**` recursive glob takes its place **directly**. The mechanism becomes simpler and the extra indirection disappears.

## 0.2.1. Core design decision

| Model                                         | Languages      | Adopted by Rok |
|-----------------------------------------------|----------------|----------------|
| Separate `prelude` module + automatic import  | Rust / Haskell | **Rejected**   |
| Explicit use only                             | C / C++        | (verbose)      |
| **Implicit `use rok::core::**;` + explicit additions** | Rok itself | **Adopted** |

### Why a separate prelude module is rejected

A traditional prelude has several drawbacks. First, there is the **indirection** problem: a **re-export path** such as `rok::core::prelude::v1::List` becomes separated from the actual definition path. Next comes the burden of versioning per epoch, such as `prelude::v1` / `prelude::v2` (the Rust 2024 prelude additions are a case in point). Self-contradiction is also unavoidable: if the standard library itself imports the prelude, a cycle arises, requiring a separate escape such as `#![no_prelude]`. Above all, if `**` already provides a recursive glob, a prelude module is merely a **trivially redundant** layer.

Rok's solution is clean. Because `**` itself is brought in at the language level, both the automatic application and explicit user imports run on the same `**` mechanism.

## 0.2.2. `**` recursive glob

```rok
use foo::bar;            // single item
use foo::{a, b};         // multiple
use foo::*;              // single-level glob (only foo's direct children)
use foo::**;             // recursive glob (foo + all submodules)
```

The normative definition of the `**` syntax is given in [refs-lang 5.5 use](../../refs-lang/005-modules/505-use.md).

## 0.2.3. Implicit application - `use rok::core::**;`

```rok
// user code
fn main() {
    let xs: List<i32> = [1, 2, 3];        // rok::core::collections::List
    let total: i32 = xs.iter().sum();      // rok::core::iter::Iterator
    println(f"{total}");                   // rok::core::print::println

    let m = Mutex::new(0);                 // rok::core::sync::Mutex
    task::spawn(async { /* ... */ });              // rok::core::task::spawn

    let n = Atomic::new(0);                // rok::core::sync::Atomic
}
```

The code above works without any use statement, because the compiler **implicitly** prepends `use rok::core::**;`.

### Name collision policy

A user-defined item whose name `**` also brings in shadows it in that module: lookup takes the module's own items and its `use`s before the implicit `use rok::core::**` ([refs-lang 5.7.1](../../refs-lang/005-modules/507-paths.md#571-starting-identifier-of-a-path)). The core item stays reachable by its path or an alias.

```rok
struct List<T> { ... }                  // OK - shadows rok::core::collections::List here
let xs: rok::core::collections::List<i32> = [1, 2];   // the core List, by path
```

## 0.2.4. Tiers **not** applied implicitly

Only `use rok::core::**;` is implicit. unsafe and std are **explicit only**.

```rok
// unsafe - danger signal
use rok::bare::ptr::{Ptr, ConstPtr};
use rok::bare::ffi::{CStr, CString};

// std - target-dependency signal
use rok::std::fs;
use rok::std::time::Instant;
```

This policy matches the following table of the [0.1 Tier model](001-tier-model.md).

| Tier            | Implicit import | Reason                          |
|-----------------|-----------------|---------------------------------|
| `rok::core::**` | ✓               | All targets / safe              |
| `rok::bare::**` | ✗               | Danger signal - explicit only   |
| `rok::std::**` | ✗               | Target-dep - explicit only      |

This way, which areas a user is entering is immediately visible from the top of the file.

## 0.2.5. Opting out - `#![no_implicit_rok]`

In special situations the implicit `use rok::core::**;` itself can be turned off.

```rok
#![no_implicit_rok]   // top of the file

// no implicit import in this file
use rok::core::ops::Add;
let r = a + b;   // OK (Add explicit)

let m = Mutex::new(0);   // ERROR - Mutex not imported
```

Literals are compiler-defined and stay available; only names (and the operators of traits not imported, [refs-lang 6.7.8](../../refs-lang/006-meta-system/607-operator-magic.md#678-visibility-of-mappings)) are lost.

Uses include the following. The standard library itself needs this attribute, since importing itself would create a cycle, as do `no_std` or `no_rok` environments (rare, but embedded). It is also useful in teaching or demo code to show at a glance where each item comes from.

## 0.2.6. Visibility rules of `**`

`**` exposes **only pub items**. Internal implementation (`pub(pkg)` / private) never leaks to external users.

```rok
mod sync {                          // rok::core::sync
    pub struct Mutex<T> { ... }
    pub fn lock<T>(ref m: Mutex<T>) -> MutexGuard<T> { ... }

    pub(pkg) struct Internal { ... }    // pkg-internal only
    fn helper() { ... }                  // private
}

// user code - after the implicit use rok::core::**;
let m: Mutex<i32> = Mutex::new(0);       // OK
rok::core::sync::Internal::new();              // ERROR - pub(pkg)
rok::core::sync::helper();                     // ERROR - private
```

Names are not flattened; module paths are preserved as they are.

```rok
// after the implicit use rok::core::**;
let m = Mutex::new(0);              // OK - Mutex exposed directly (reachable as pub at rok::core::sync)
let s = task::spawn(async { /* */ });      // OK - the task module itself is reachable too; spawn is task::spawn

// both mean the same thing.
rok::core::sync::Mutex::new(0);
Mutex::new(0);
```

## 0.2.7. Comparison with other languages

| Language | Model                                          | Indirection                          |
|----------|------------------------------------------------|--------------------------------------|
| C        | Every #include explicit                        | 0 (direct)                           |
| Rust     | `std::prelude::v1::*` re-export module + automatic | 1 (prelude module)               |
| Haskell  | `Prelude` re-export module + automatic         | 1                                    |
| Python   | built-ins (language magic)                     | 0 (langauge-level)                   |
| **Rok**  | Implicit `use rok::core::**;`                  | **0** (direct - `**` is the mechanism) |

The biggest point where Rok diverges from other languages is that there is no prelude module indirection. Because `**` provides a recursive glob **at the language level**, users and the standard library share the same mechanism.

This single decision satisfies both principles: explicit > implicit, and no extra indirection.
