# 0.3. Source units

rok's compilation unit is the **package**. A package is recognized on the file system by a fixed convention (`src/main.rok` / `src/lib.rok`). The manifest `rok.rokon` is present only when declaring dependencies and the like; a package without dependencies does not need one (0.3.4).

## 0.3.1. Executables and libraries

A package has an executable, a library, or both. Which it has is decided solely by the root files in its source directory `src/`: `src/main.rok` is the executable's root, `src/lib.rok` the library's.

A package directory holds the package's source in `src/` and everything else beside it: the manifest `rok.rokon` and lockfile `rok.lock` (0.3.4), `target/` (build output, [9.2.15](../009-tools/902-aot-pipeline.md#9215-output)), and files such as `README`, `LICENSE` or test data. Only the files under `src/` are the package's source; a `.rok` file outside `src/` is not part of the package (it is not compiled with it, is not in its identifier (0.3.3), and run on its own it is an anonymous package, [4.16.8](../004-functions/416-entry-point.md#4168-single-file-execution)).

### Executable package: `src/main.rok`

A directory containing `src/main.rok` is recognized as an executable package.

```
my_app/
├── rok.rokon         <- manifest (optional, 0.3.4)
├── src/
│   ├── main.rok      <- entry point
│   └── ...           <- other module files
└── target/           <- build output
```

`main.rok` is the entry module and provides the entry point in one of two forms: an explicit `fn main` or top-level statements. The exact form of the entry point and the meaning of top-level statements are defined in [4.16 Entry point](../004-functions/416-entry-point.md). When a single file that belongs to no package is run, that file is treated as the `main.rok` of an anonymous package ([4.16.8](../004-functions/416-entry-point.md#4168-single-file-execution)).

### Library package: `src/lib.rok`

A directory containing `src/lib.rok` is recognized as a library package.

```
my_lib/
├── rok.rokon
└── src/
    ├── lib.rok       <- library root
    └── ...
```

A library package has no entry point; it serves as a set of definitions used from outside. Every module file other than `main.rok`, including `lib.rok`, may contain only items and must not contain top-level statements, so importing a module never executes any code ([4.16.6](../004-functions/416-entry-point.md#4166-modules-are-definition-only)).

### Library and executable in one package

A package may have both `src/main.rok` and `src/lib.rok`. They are two targets built from one `src/`: the library, and an executable that uses it. The executable sees the library as a dependency named after the package (`use my_app::parse;`; the code name of [9.10.3](../009-tools/910-rok-frontend.md#9103-rokrokon--roklock), `-` replaced by `_`). The library is a separate package from the executable: only its `pub` items are visible there, not `pub(pkg)`, `pub(sub)` or private ones ([5.7.14](../005-modules/507-paths.md#5714-paths-between-packages)), and none of its items is in scope until the executable `use`s it. `pkg::` in either target is that target's own root. Each target has its own module tree, so both may declare `mod util;` for the same `src/util.rok`, and each gets its own copy. A package that depends on this one gets the library only.

## 0.3.2. Files and modules

An additional module inside a package takes one of two forms, as files under `src/`.

```
foo.rok                <- single-file module (no submodules)
```

Or it can be accompanied by a folder:

```
foo.rok                <- module body
foo/                   <- foo's submodules
├── bar.rok                foo::bar submodule
└── baz.rok                foo::baz submodule
```

In the folder form, the body file (`foo.rok`) sits alongside a folder of the same name (`foo/`). Only submodules declared in the body with `mod name;` are mapped to files in the folder. This is the same rule as Rust 2018+; the detailed mapping rules are defined in [Modules](../005-modules/504-modules.md).

Alternatively, an inline module can be placed in the body without a separate file, in the form `mod name { ... }` or `pub mod name { ... }`. It behaves like Rust's inline modules; its detailed form and mapping are likewise defined in [Modules](../005-modules/504-modules.md).

## 0.3.3. Package identifier

Each package is assigned a content-based hash. The hash input is the concatenation of all of the package's source files, sorted in byte order of their paths relative to `src/` (`/` separator), with each file serialized as `path + \0 + contents + \0`. The first 8 characters of its SHA-256 are used as the identifier.

The hash has the following properties: a package with the same contents produces the same hash wherever it is built, and changing even a single character produces a new hash.

The hash is used in artifact symbol names and cache keys; the specific uses are defined in later documents.

## 0.3.4. The manifest (`rok.rokon`) is optional

What constitutes a package is determined solely by the presence of files. If `src/main.rok` or `src/lib.rok` exists in a directory, that directory is a package, and which of them exist determines whether it has an executable, a library, or both. The manifest does not change this convention.

Only when information such as dependencies, dev dependencies or a toolchain pin is needed is a manifest `rok.rokon` ([9.10.3](../009-tools/910-rok-frontend.md#9103-rokrokon--roklock)) placed at the package root, beside `src/`; the results resolved by the tool are recorded in the lockfile `rok.lock`. The two file names are spelled exactly this way (all lowercase). A name differing in case (`Rok.rokon`, etc.) is a warning. Both files use the [ROKON](../../refs-standard-libraries/300-std/309-rokon.md) format; the manifest is only read as a single `Package { .. }` value and is never compiled or executed.

This decision enables two things. First, the compiler can process a package without dependencies by looking at its files alone. Second, the manifest is purely additive and never breaks an existing package definition.
