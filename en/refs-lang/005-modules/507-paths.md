# 5.7. Paths

A sequence of identifiers joined by the `::` token ([punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md)) is a path that designates a location in the module tree. This document defines paths as references to modules and items.

[Type paths](../002-types/213-type-paths.md) covers paths in type position; this document covers paths referring to values and modules. Both have the same form.

```
path            ::= path_root ("::" path_segment)*
path_root       ::= "pkg" | "sup" | "rok" | identifier
path_segment    ::= identifier
                  | identifier "::" "<" type_arg_list ">"      // turbofish
type_arg_list   ::= type_expr ("," type_expr)* ","?
```

The grammar above defines only the turbofish (`name::<T>`) of value and module position paths. Generics written as `name<T>` in type position are defined by `type_path` in [type paths](../002-types/213-type-paths.md).

## 5.7.1. Starting identifier of a path

The first identifier of a path is one of:

| Start token          | Meaning                                                  |
|----------------------|----------------------------------------------------------|
| `pkg`                | From the root of the current package                     |
| `sup`                | From the parent module                                   |
| `rok`                | The standard library virtual namespace                   |
| Ordinary identifier  | A currently visible name (child module, `use`d name)     |

`pkg` shares its stem with the visibility keyword `pub(pkg)`, and `sup` is the **mirror** direction of the visibility `pub(sub)` (sub means descendants, sup means ancestors). `rok` is a virtual root that groups the compiler-managed standard library packages under one name. All three are path roots only when immediately followed by `::`; elsewhere they are ordinary names ([1.5.4 Contextual keywords](../001-lexical/105-keywords.md)).

`self::` is not a path starting point. Children and items of the same module are reachable by ordinary identifiers (`child::item`).

Name resolution does not implicitly search ancestor modules (exception: `extern` functions, [5.9.4](509-extern-block.md#name-space-of-extern-functions)). An ordinary identifier is looked up only among the current module's items, child modules, and names imported with `use` (including the implicit `use rok::core::**`). Items of the parent or higher modules must be named with `sup::` or imported with `use`.

Lookup order is as follows; the first match wins.

1. Items and child modules of the current module
2. `use`s of the current module. Named imports take precedence over globs (`*`, `**`).
3. The implicit `use rok::core::**`

Functions, generic or not, share a single value name space. The nearest declaration therefore wins regardless of kind (a non-generic `max` in the current module hides the generic `max` of core).

```rok
// inside pkg::a::b
helper();              // ERROR: no implicit search even if helper is in pkg::a
sup::helper();         // OK
```

## 5.7.2. `pkg::`: from the package root

```rok
pkg::geom::Point
pkg::config::DEFAULT_TIMEOUT
pkg::main_module::run
```

`pkg::` starts from the package's root module (`src/main.rok` or `src/lib.rok`) and is written the same way anywhere in the package. It is the standard form for safe absolute references in large codebases.

## 5.7.3. `sup::`: from the parent module

```rok
sup::sibling_function
sup::parent_const
```

`sup::` starts from the current module's parent module, so it suits references to items of sibling and parent modules. To go up several levels, chain it as `sup::sup::...`.

```rok
sup::sup::ancestor_item
```

## 5.7.4. `rok::`: the standard library virtual namespace

```rok
rok::core::collections::List
rok::core::iter::Iterator
rok::bare::ptr::Ptr
rok::std::fs::File
```

`rok::` designates the compiler-managed set of standard library packages. Under it, three tiers exist as separate packages ([0.5 Standard library tiers](../000-meta/005-standard-libraries-tiers.md)).

| Path           | Meaning                              | Link mechanism |
|----------------|--------------------------------------|----------------|
| `rok::bare::*` | raw / unsafe area                    | `-NL bare`     |
| `rok::core::*` | foundation / commonly used traits    | `-NL core`     |
| `rok::std::*`  | OS integration / I/O etc.            | `-NL std`      |

The three packages are separate archives the compiler links automatically at build time, handled by the `-NL` mechanism distinct from user package dependencies. In source, `rok` merely presents them under one path root; the build system separates the package boundaries.

`rok::bare` and `rok::std` are not automatically imported and require an explicit import. The `rok::core` area, by contrast, is visible automatically through the implicit `use rok::core::**;` defined in [5.5 use](505-use.md).

## 5.7.5. Starting with an ordinary identifier

```rok
foo::bar::baz                   // OK if foo is a currently visible name
geom::Point                      // if the geom module is a child or imported
Point                            // a single name, resolved automatically
```

If the first identifier is a name visible in the current module, it can be used as is. A visible name here means a child module or an item imported with `use`.

## 5.7.6. Meaning of each path segment

```rok
pkg::geom::shapes::Circle
└──┘  └──┘  └────┘   └────┘
root  mod   module   item
```

Each segment is a module or (for the last one) an item. The last identifier is the final item being designated; the preceding segments are modules or types along the path.

## 5.7.7. Paths by item kind

| Designates                                   | Form                  |
|----------------------------------------------|-----------------------|
| Function                                     | `path::function_name` |
| Type (`struct` / `enum` / `union` / `trait`) | `path::TypeName`      |
| `const`                                      | `path::CONST_NAME`    |
| `static`                                     | `path::STATIC_NAME`   |
| Module itself                                | `path::module_name`   |
| Static method                                | `Type::method_name`   |
| Associated `const`                           | `Type::CONST`         |
| Associated type                              | `T::AssocType`        |

These are the same as the associated item access forms of [type paths](../002-types/213-type-paths.md).

## 5.7.8. Expression position and type position

The same path form can be used both in expression position (designating a value) and in type position (designating a type).

```rok
// expression position
let f = pkg::geom::distance;          // function value
let p = pkg::geom::Point::origin();   // static method call
let v = pkg::config::DEFAULT;           // const value

// type position
let p: pkg::geom::Point = ...;
fn make() -> pkg::geom::Point { ... }
```

The difference between the two positions is whether generic arguments are written with the turbofish (`::<...>`). Expression position uses the turbofish; type position uses plain `<...>`. Details are in [type paths](../002-types/213-type-paths.md).

## 5.7.9. Combining paths and globs

Combined with the import forms of [`use`](505-use.md), various import notations are available.

```rok
use pkg::geom::{Point, Circle, distance};
use sup::{a, b as alias};
use foo::*;
```

`*` (glob) and `**` (recursive glob) behave as defined in [`use` (imports)](505-use.md).

## 5.7.10. Empty paths / a lone `::`

An empty path (only `::` with no identifier) is a compile error.

A leading `::`, as in `::A`, is also an ERROR. Absolute paths are written explicitly as `pkg::A`.

## 5.7.11. Shortening with `use`

A name imported with `use` can be used by its short name in the current module.

```rok
use pkg::geom::Point;

fn f() {
    let p = Point { ... };          // short name
    let q: Point = ...;
}
```

Gathering frequently used items with `use` at the top of a module keeps code bodies concise.

## 5.7.12. When paths are resolved

Path resolution happens in the compile-time name resolution phase and has no effect on runtime behavior (method dispatch on dynamic slots, however, happens separately at runtime; see [`any` slot](../002-types/202-any-slot.md)).

## 5.7.13. Referring to modules dynamically (none)

```rok
let m = "foo";
m::bar();                            // ERROR: no dynamic module references
```

The module part of a path is always static and determined at compile time. Receiving a module as a variable at runtime and calling functions in it is not provided.

This is consistent with the principle in [modules](504-modules.md) that the module tree is a static structure.

## 5.7.14. Paths between packages

Items of another package (a dependency) are referenced by paths starting with that package's name.

```rok
use parser::Token;
let t = parser::lex(src);
```

- The name of a dependency package is a top-level path. The name is the `deps` key in `rok.rokon` with `-` replaced by `_` (`rok-syntax` → `rok_syntax::..`), and the build tool registers it with `rokc --extern name=..` ([9.2.3](../009-tools/902-aot-pipeline.md#923-dependency-resolution), [9.10.3](../009-tools/910-rok-frontend.md#9103-rokrokon--roklock)). No separate declaration such as Rust's `extern crate` is needed.
- `pkg::` inside a dependency package is that dependency's own root.
- A module of the user package with the same name as a dependency package is an error.
- A dependency that resolves to the package being built is an error; a package names its own items with `pkg::`.

### Package boundaries

Each of the following is a separate package. The visibilities `pub(pkg)` / `pub(sub)` ([5.6](506-visibility.md)) do not cross these boundaries.

- The package being built
- Each dependency package
- Each tier of `rok::` (`rok::bare`, `rok::core`, `rok::std`)

Thus private / `pub(sub)` / `pub(pkg)` items of a dependency cannot be used from outside it; only `pub` items can.
