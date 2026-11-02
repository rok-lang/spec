# 0.2. Import policy - the prelude `rok::core::prelude`

> This document defines the implicit import policy of the standard libraries. Every user file sees the names of one module, `rok::core::prelude`, without a `use`; everything else in `rok::core`, and all of `rok::bare` and `rok::std`, is named by path or imported explicitly.

Rok **implicitly** prepends the following single line to every user source file.

```rok
use rok::core::prelude::*;
```

`rok::core::prelude` is an ordinary module of `pub use` re-exports ([refs-lang 5.8](../../refs-lang/005-modules/508-re-export.md)). Its list is fixed below (0.2.3) and is normative: a name is in the prelude only if this document lists it.

The prelude holds no enum variants. A bare variant name resolves from the type the position expects ([refs-lang 2.4.8](../../refs-lang/002-types/204-enum.md#248-bare-variants-from-the-expected-type)): `let o: Ordering = Less;`, `x.load(SeqCst)`, `return Ok(v);`.

## 0.2.1. Core design decision

| Model                                         | Languages             | Adopted by Rok |
|-----------------------------------------------|-----------------------|----------------|
| Explicit use only                             | C / C++               | (verbose)      |
| Every public name of the core library         | Swift, Python builtins | Rejected      |
| **A short, curated prelude module**           | Rust, Gleam, Elm      | **Adopted**    |

### Why a curated list

An implicit glob over all of `rok::core` would make every public core item a bare name in every file: about 160 types, 100 traits, 90 functions and every enum variant. Several of them would collide (`Map`, `Filter`, `Zip` in `iter` and `stream`; `join` in `task` and `par`; the variant `Equal` in `Ordering` and `fmt::Alignment`), and each new core item would change name resolution in user code.

The prelude keeps what code writes in nearly every file: the collection and core types, the traits written in `impl` headers and bounds, the printing and panic functions, and the top-level core modules, so `task::sleep` and `par::map` stay short. Every name in it is unique in its namespace (a library test checks this on every target), so a bare prelude name is never ambiguous. Because the implicit import has the lowest priority (0.2.4), adding a name to the prelude never breaks a program that defines or imports the same name.

## 0.2.2. `**` recursive glob

```rok
use foo::bar;            // single item
use foo::{a, b};         // multiple
use foo::*;              // single-level glob (only foo's direct children)
use foo::**;             // recursive glob (foo + all submodules)
```

The normative definition of the `**` syntax is given in [refs-lang 5.5 use](../../refs-lang/005-modules/505-use.md).

## 0.2.3. The prelude list

The names below are exactly the names of `rok::core::prelude`. Modules, types and traits share one namespace and functions another, as everywhere in Rok, so `channel`, `panic` and `print` are each a module and a function.

**Modules (29).** The top-level modules of `rok::core`, except `prelude` itself and `hash` (imported with a `use`, [2.5.11](../200-core/205-cmp.md)):

`array` `bench` `channel` `cmp` `collections` `convert` `crypto` `encoding` `error` `fmt` `future` `hint` `index` `io` `iter` `marker` `num` `ops` `panic` `par` `print` `range` `reflect` `str` `stream` `sync` `task` `time` `tuple`

**Types (32).**

| Group | Names |
|---|---|
| Collections | `List` `Dict` `Set` `Deque` `BinaryHeap` `Slice` `StrBuf` |
| Numbers and ranges | `Range` `RangeInclusive` `Int` `Float` |
| General | `Result` `Ordering` `Duration` `Instant` |
| Synchronization and channels | `Mutex` `RwLock` `Atomic` `Cell` `Sender` `Receiver` `Task` |
| Formatting | `Formatter` `FmtError` |
| Core errors | `ParseError` `ValueError` `OverflowError` `ConversionError` `TypeError` `StackOverflowError` `OutOfMemoryError` `InternalError` |

**Traits (30).**

| Group | Names |
|---|---|
| Markers | `Clone` `Copy` `Drop` `Default` `Send` `Sync` |
| Comparison | `PartialEq` `Eq` `PartialOrd` `Ord` `Hash` |
| Formatting and errors | `Debug` `Display` `Error` |
| Iteration and async | `Iterator` `IntoIterator` `FromIterator` `TryIterator` `Future` `IntoFuture` |
| I/O | `Read` `Write` `AsyncRead` `AsyncWrite` |
| Conversion | `From` `Into` `TryFrom` `TryInto` `FromStr` |
| Reflection | `Inspect` (also its derive, [refs-lang 6.8.11](../../refs-lang/006-meta-system/608-meta-blocks.md)) |

The operator traits are not in the prelude: write `ops::Add`, or `use rok::core::ops::Add;`. Operators themselves need no name and work everywhere ([refs-lang 6.7.8](../../refs-lang/006-meta-system/607-operator-magic.md#678-visibility-of-mappings)), and so do the methods of every `rok::core` trait ([refs-lang 3.5](../../refs-lang/003-expressions/305-member-access-and-method-call.md)).

**OS extension traits.** On the targets that have them, the prelude also names `PermissionsExt`, `CommandExt` and `ExitStatusExt` (unix) and `ChildExt` and `TcpStreamExt` (linux), so their methods need no `use` there ([3.11.1](../300-std/311-os.md#3111-ext-traits-in-the-prelude)). They are not counted above.

**Functions (11).**

`print` `println` `eprint` `eprintln` `panic` `assert` `assert_eq` `assert_ne` `unreachable` `channel` `channel_bounded`

**Variants.** None. See [refs-lang 2.4.8](../../refs-lang/002-types/204-enum.md#248-bare-variants-from-the-expected-type).

```rok
// user code
fn main() {
    let xs: List<i32> = [1, 2, 3];        // prelude: List
    let total: i32 = xs.iter().sum();      // methods of core traits: no import
    println(f"{total}");                   // prelude: println

    let m = Mutex::new(0);                 // prelude: Mutex
    task::spawn(async { /* ... */ });      // prelude module task
    task::sleep(Duration::ms(10));         // prelude: Duration

    let n = Atomic::new(0);
    n.store(1, SeqCst);                    // variant from the parameter type
}
```

Everything else in `rok::core` is named through a prelude module or imported:

```rok
let p: future::Poll<i32> = future::Poll::Pending;
impl ops::Add for V { ... }
use rok::core::sync::LoadOrdering;
let h = encoding::hex::encode("rok");
```

A name that is not in scope gets an error that says what to write: the shortest path that works (`task::sleep`, `ops::Add`, `Ordering::Less`), and the `use` that imports it.

## 0.2.4. Name collision policy

The implicit import has the lowest priority: lookup takes the module's own items and its `use`s first ([refs-lang 5.7.1](../../refs-lang/005-modules/507-paths.md#571-starting-identifier-of-a-path)). A user item named like a prelude name shadows it in that module, and the core item stays reachable by path.

```rok
struct List<T> { ... }                  // OK: shadows the prelude's List here
let xs: rok::core::collections::List<i32> = [1, 2];   // the core List, by path
```

A `use` path may start with a prelude name: `use task::sleep;` imports `rok::core::task::sleep`.

## 0.2.5. Tiers **not** applied implicitly

Only the prelude is implicit. unsafe and std are **explicit only**.

```rok
// unsafe: danger signal
use rok::bare::ptr::{Ptr, ConstPtr};
use rok::bare::ffi::{CStr, CString};

// std: target-dependency signal
use rok::std::fs;
use rok::std::time::SystemTime;
```

This policy matches the following table of the [0.1 Tier model](001-tier-model.md).

| Tier            | Implicit import         | Reason                          |
|-----------------|-------------------------|---------------------------------|
| `rok::core`     | ✓ the prelude (0.2.3)   | All targets / safe              |
| `rok::bare`     | ✗                       | Danger signal, explicit only    |
| `rok::std`      | ✗                       | Target-dep, explicit only       |

This way, which areas a user is entering is immediately visible from the top of the file.

## 0.2.6. Opting out - `#![no_implicit_rok]`

In special situations the implicit import itself can be turned off.

```rok
#![no_implicit_rok]   // top of the file

// no implicit import in this file
use rok::core::prelude::*;   // the same import, written out
```

Literals are compiler-defined and stay available, and so do operators and the methods of core traits ([refs-lang 6.7.8](../../refs-lang/006-meta-system/607-operator-magic.md#678-visibility-of-mappings)); only bare names are lost.

It is useful in teaching or demo code to show at a glance where each item comes from.

The standard library's own modules do not import the prelude: each tier sees every public item of `rok::core` by its leaf name (an internal `use rok::core::**;`), which user code does not get.

## 0.2.7. Visibility

The prelude re-exports `pub` items only. Internal implementation (`pub(pkg)` / private) never leaks to users: `rok::core::collections::raw` is `pub(pkg)`, so `GcBuf` and the `buf_*` functions are reached only as `rok::bare::mem::..` with an explicit `use`.

Names are not flattened beyond the list; module paths are preserved as they are.

```rok
let m = Mutex::new(0);              // prelude: Mutex
let s = task::spawn(async { /* */ });      // prelude module task; spawn is task::spawn

// both mean the same thing.
rok::core::sync::Mutex::new(0);
Mutex::new(0);
```

## 0.2.8. Comparison with other languages

| Language | Implicit scope                                     | Variants                                 |
|----------|----------------------------------------------------|------------------------------------------|
| Rust     | `std::prelude::rust_20xx`, an explicit list        | `Some` `None` `Ok` `Err` only            |
| Go       | the universe block: about 20 builtins and types    | qualified (`time.Second`)                |
| Swift    | every public declaration of `Swift`                | `.less` from the contextual type         |
| Kotlin   | default package imports                            | context-sensitive resolution (2.2)       |
| **Rok**  | `rok::core::prelude`, an explicit list (0.2.3)     | bare, from the expected type (any enum)  |
