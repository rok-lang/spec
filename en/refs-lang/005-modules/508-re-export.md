# 5.8. Re-exports (`pub use`)

Combining `use` with a visibility keyword makes a re-export: the imported item is exposed again from the current module. The grammar is:

```
re_export       ::= visibility "use" use_tree ";"
```

## 5.8.1. Form

A re-export combines a visibility keyword, `use`, a path, and a semicolon:

```rok
pub use foo::bar;
pub(pkg) use pkg::internal::helper;
pub(sub) use sup::shared;
```

Writing `pub` or another visibility keyword with `use`, a path, and `;` exposes the imported item again from the current module with that visibility.

## 5.8.2. Meaning

```rok
// pkg::deep::Item
mod deep {
    pub struct Item { ... }
}

// root
pub use deep::Item;             // Item is pub at the root too
```

External packages can then reach the same item through two paths.

```rok
use my_pkg::deep::Item;          // original path - OK
use my_pkg::Item;                // re-export path - OK
```

The same item is visible through two paths.

## 5.8.3. Uses

### Facade pattern

Used to hide the internal structure and offer a flat API externally.

```rok
// pkg::core::geom::Point
mod core {
    pub mod geom {
        pub struct Point { ... }
    }
}

// root - flat for external users
pub use core::geom::Point;
pub use core::geom::distance;
```

External users access `my_pkg::Point` and need not know the internal structure (`core::geom::`).

### Building a prelude

A prelude module that gathers frequently used items in one place is built the same way.

```rok
// pkg::prelude
pub use pkg::types::{Point, Circle, Polygon};
pub use pkg::ops::{distance, area};
```

Users bring in the frequently used items at once with the single line `use my_pkg::prelude::*;`.

### Version compatibility

If an internal item moves but the re-export path is kept, the old path keeps working for external users.

```rok
// v1: pkg::Item
pub struct Item { ... }

// v2: moved
pub mod core { pub struct Item { ... } }
pub use core::Item;             // keeps the v1 path
```

## 5.8.4. Combining visibility levels

Re-exports combined with visibility keywords:

```
pub use         → exposed to external packages
pub(pkg) use    → exposed within the package
pub(sub) use    → exposed to submodules
use             → module-private (not a re-export)
```

Visibility is ordered private < `pub(sub)` < `pub(pkg)` < `pub`. The original item's visibility must be ≥ the re-export visibility; a `pub use` wider than the original is an error.

```rok
mod inner {
    fn private_fn() { ... }      // private
}

pub use inner::private_fn;       // ERROR - cannot expose a private item as pub
```

Glob re-exports (`pub use mod::*`, `pub use mod::**`) behave differently: items whose original visibility is narrower than the re-export visibility are silently filtered out, and this is not an ERROR ([5.8.12 Glob re-exports](#5812-glob-re-exports-and-visibility)).

## 5.8.5. Multiple re-exports

Several items can be re-exported at once, optionally with aliases.

```rok
pub use foo::{a, b, c};
pub use foo::{
    Point,
    Circle,
    distance as dist,
};
```

The multiple-import and alias forms of [`use`](505-use.md) apply unchanged.

## 5.8.6. Glob re-exports

To re-export a whole module or subtree at once:

```rok
pub use foo::*;
pub use foo::**;
```

The whole module or the subtree below it is exposed at once.

> [!WARNING]
> A name conflict is an error at the re-export.

## 5.8.7. Combining with aliases

An alias shortens the externally visible name.

```rok
pub use foo::very_long_name as short_name;
```

Externally only the short name (`short_name`) is visible; using the original name (`very_long_name`) requires a separate import.

## 5.8.8. Re-exporting a module itself

A module itself, not only items, can be re-exported.

```rok
pub use foo::geom;               // re-exports the module itself
```

External users can then access `my_pkg::geom::Point`. Paths through a re-exported module resolve as through the module.

## 5.8.9. The `prelude` pattern

By convention, library authors build a `prelude` module gathering frequently used items to curate the external surface. The compiler does not pull it in automatically; users must import it explicitly.

```rok
// pkg::prelude
pub use pkg::collections::{List, Dict, Set};
pub use pkg::ops::{Add, Sub, Mul};
pub use pkg::iter::Iterator;
```

On the user side it is imported with one line:

```rok
// user side
use my_pkg::prelude::*;
```

The compiler gives the name `prelude` no special meaning; it is treated as an ordinary module.

## 5.8.10. Circular re-exports (none)

Cycles of re-exports pointing at each other are not allowed.

```rok
mod a { pub use pkg::b::Item; }
mod b { pub use pkg::a::Item; }
// ERROR - circular dependency
```

Such a cycle is an ERROR, detected by the compiler.

## 5.8.11. Implementation

A re-export only adds an entry to the name table. The actual code and memory exist only at the original item; there is no wrapper function or copy, so the cost is zero. A chain of re-exports of any length resolves to the original item; there is no step limit.

## 5.8.12. Glob re-exports and visibility

Whether a glob re-export exposes an item depends on the original item's visibility.

```rok
mod source {
    pub fn a() { ... }
    pub(pkg) fn b() { ... }
}

pub use source::*;               // a is exposed externally; b is pub(pkg), so visible only within the package
```

`pub use ...::*` imports only items for which the re-export visibility reaches the caller and the original visibility permits the re-export visibility. An item with a narrow reach cannot be widened by a re-export ([5.8.4](#584-combining-visibility-levels)).

## 5.8.13. Summary

`pub use` is the key tool for separating internal structure from the external API. Users need know only the public location, and library authors may reorganize internal locations freely.
