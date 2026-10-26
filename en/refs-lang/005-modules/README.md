# 5. Variables / modules / visibility / packages overview

This part covers Rok's name space. Its goal is to define where variables are visible, how modules are grouped, how visibility levels work, and where the boundaries between packages lie.

## Contents

- [5.1. Variable declarations](501-variable-declaration.md)
- [5.2. `const`](502-const.md)
- [5.3. `static`](503-static.md)
- [5.4. Modules](504-modules.md)
- [5.5. `use` (imports)](505-use.md)
- [5.6. Visibility](506-visibility.md)
- [5.7. Paths](507-paths.md)
- [5.8. Re-exports (`pub use`)](508-re-export.md)
- [5.9. `extern` blocks (FFI declarations)](509-extern-block.md)

## The four levels of the name space

The name space has four levels:

```
package
  └── module
        └── item
              └── local variable
```

Containment runs top to bottom. A package has many modules, a module has many items (functions, types, traits, `const`, `static`), and an item (in particular a function body) has many local variables.

The boundary rules for each level are defined in the following documents.

## Packages: the largest unit

The package defined in [source units](../000-meta/003-source-units.md) is the outermost level of the name space. Each package is identified by a content-based hash.

All items in the same package belong to the same code-authority domain. This is why the orphan rule of [`impl`](../002-types/209-impl.md) applies per package.

## Modules: grouping units within a package

A package has many modules. Each module is either a single file (`foo.rok`) or a body file plus a folder of the same name (`foo.rok` + `foo/`). This is the same model as Rust 2018+. The detailed rules are defined in [modules](504-modules.md).

```
my_pkg/
└── src/
    ├── lib.rok                  <- root module body
    ├── geom.rok                 <- geom module    (single-file: geom.rok)
    ├── parsing.rok              <- parsing module body
    └── parsing/                 <- parsing's submodules
        ├── lexer.rok                parsing::lexer
        └── parser.rok               parsing::parser
```

## Items: named definitions within a module

Each module has items: functions, types, traits, `impl` blocks, `const`, `static`, and other module declarations.

Every definition of a [function declaration](../004-functions/401-function-declaration.md), [`struct`](../002-types/203-struct.md), [`enum`](../002-types/204-enum.md), [`trait`](../002-types/208-trait.md) and [`impl`](../002-types/209-impl.md) is an item within a module.

## Local variables: variables within functions / blocks

Within functions, blocks and other expressions, local variables are declared with the `let` keyword ([keywords](../001-lexical/105-keywords.md)).

```rok
fn area(r: f64) -> f64 {
    let pi = 3.14159f64;          // local variable
    pi * r * r
}
```

The visibility scope of a local variable is determined per block of a [block expression](../003-expressions/309-block-expressions.md). The detailed rules are defined in [variable declarations](501-variable-declaration.md).

## Visibility: exposing items

Each item has a visibility. The visibility level ([visibility](506-visibility.md)) determines how far the item is visible.

```
private (default)    within the same module only
pub(sub)             within submodules only
pub(pkg)             within the same package only
pub                  also from external packages
```

## Name resolution rules

Determining which item a name in the code refers to is called name resolution.

It considers the following. First, items locally visible in the current module are candidates. Second, external items imported with `use` ([`use`](505-use.md)) are also candidates. Third, the automatically imported `rok::**` ([standard library tiers](../000-meta/005-standard-libraries-tiers.md)) is consulted as well. Finally, a visibility check is performed, and only visible items remain as candidates.

Name resolution happens at compile time. Method dispatch on dynamic slots happens at runtime, but resolving the name itself is always completed at compile time.
