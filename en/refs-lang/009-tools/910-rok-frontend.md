# 9.10. `rok` — package / build front-end

`rok` is a cargo-style front-end. It owns the manifest `rok.rokon` and lockfile `rok.lock`, dependencies, environments, the toolchain pin and script dependencies, and delegates compilation to the compiler `rokc` ([9.1](901-rokc-cli.md)). The runtime is `rokr` (`librokr.a`).

A package without dependencies builds by file conventions alone, without a manifest ([0.3.4](../000-meta/003-source-units.md#034-the-manifest-rokrokon-is-optional)). In that case `rok build` is equivalent to `rokc build`.

## 9.10.1. Subcommands and global options

| Subcommand                                                        | Action                                  |
|-------------------------------------------------------------------|-----------------------------------------|
| `new`, `init`                                                     | Create a package (9.10.2)               |
| `add`, `remove`, `update`, `deps`                                 | Dependencies (9.10.4)                   |
| `search`, `publish`, `yank`                                       | Stones (9.10.4a)                        |
| `env`                                                             | Environments (9.10.5)                   |
| `toolchain`                                                       | Toolchain (9.10.6)                      |
| `build`, `check`, `run` (and `run -i`), `test`, `bench`, `emit`   | Delegated to `rokc` with dependencies attached |
| `doc`, `lint`, `repl`, `lsp`                                      | Delegated to `rokc` with dependencies attached |
| `fmt`, `clean`, `demangle`                                        | Delegated to `rokc` without dependencies |
| `jupyter`                                                         | Running the kernel (`--connection-file`) is delegated; `jupyter install [--env <name>]` installs the kernelspec ([9.8.3](908-jupyter-integration.md#983-kernelspec-installation)) |
| `version`, `help`                                                 |                                         |
| (none)                                                            | Same as `rok repl` — starts the REPL    |

A delegated subcommand runs the `rokc` subcommand of the same name. `rok` consumes only its own global options (`--env` / `--offline` / `--locked` / `--cwd`, anywhere before `--`), passes the remaining arguments through unchanged, and adds the following.

- `--extern name=<source directory>` for each dependency package. They are prepared according to `rok.lock`, with dependencies before their dependents (the order `rok deps` prints). Prebuilding is done by `rokc` ([9.2.3](902-aot-pipeline.md#prebuilt-dependency-packages)).
- The manifest's `lints` as `-A` / `-W` / `-D` (before the arguments).
- Outside a package (environments, scripts), `--target-dir` set to the `target/` of that environment or script.

Options, output and exit codes follow those of `rokc` ([9.1](901-rokc-cli.md)). `rokls` is a standalone executable equivalent to `rok lsp`.

Running just `rok [options]` with no subcommand starts the REPL, like `rok repl`.

```
rok                 # = rok repl
rok --env sci       # = rok repl --env sci
```

The global options of `rok` are:

| Option           | Meaning                                                           |
|------------------|-------------------------------------------------------------------|
| `--env <name>`   | Use that named environment instead of the project's environment (9.10.5) |
| `--offline`      | Do not use the network. git dependencies must be in the cache     |
| `--locked`       | If `rok.lock` would need to change, do not change it and fail (for CI) |
| `--cwd <dir>`    | Change directory before running                                   |
| `-q`, `--quiet`  | Reduce output                                                     |

Environment variables are `ROK_HOME` (default `~/.rok`), `ROK_ENV` (9.10.5), `ROK_TOOLCHAIN` (9.10.6) and `ROK_REGISTRY` (9.10.4a).

## 9.10.2. `rok new` / `rok init`: package creation

```
rok new [--lib | --bin] <name>
rok init [--lib | --bin] [dir]
```

`rok new` creates the directory `<name>` and fills in default files. `--bin` (the default) creates a `main.rok` that prints a greeting; `--lib` creates a `lib.rok` with an example function and one `#[test]`. It is an error if `<name>` already exists.

`rok init` turns an existing directory (the current directory if omitted) into a package. It is an error if that directory already contains `main.rok` or `lib.rok`.

Both commands also write `rok.rokon` in canonical form ([RokON 3.9.4](../../refs-standard-libraries/300-std/309-rokon.md#394-canonical-form)) and a `.gitignore` containing `target/`, and run `git init` if not inside a git repository. `init` uses the directory name, adjusted to the naming rules (9.10.3), as the package name.

## 9.10.3. `rok.rokon` / `rok.lock`

`rok.rokon` at the package root is the manifest, and `rok.lock` in the same place is the lockfile. Both are in [RokON](../../refs-standard-libraries/300-std/309-rokon.md) format. The names are spelled exactly this way (all lowercase). Files differing only in case (`Rok.rokon` etc.) are ignored with a warning and not read.

The manifest is a single RokON value, a `Package { .. }` struct (the root name `Package` may be omitted). It is only read, never compiled or executed.

```rokon
Package {
    name: "geom",                    // string, optional
    version: "0.3.1",                // string, optional
    toolchain: "0.4",                // string, optional — toolchain pin (9.10.6)
    description: "2D geometry",      // string, optional
    license: "MIT OR Apache-2.0",    // string, optional (SPDX expression)
    repository: "https://example.com/geom.git",   // string, optional
    deps: {                          // package name (string) -> Dep
        "util": Dep::path("../util"),
        "parser": Dep::git("https://example.com/parser.git", tag: "v1.2"),
        "json": "1.4",                                   // stone (= Dep::version("1.4"))
        "csv": Dep::version("^0.3", registry: "https://example.com/index"),
    },
    dev_deps: {                      // same form as deps. test / bench only
        "fixtures": Dep::path("tests/fixtures"),
    },
    lints: { "dead_code": "allow", "nonstandard_style": "deny" },
}
```

- Values are RokON literals (strings, `{ "k": v }` dicts, the empty dict `{}`) and `Dep::path("..")` / `Dep::git("url", rev | tag | branch: "..")` / `Dep::version("req", registry: "url")`. `Dep::git` takes at most one of `rev` / `tag` / `branch`, and `registry` of `Dep::version` is optional. A dependency value that is a single string is shorthand for `Dep::version`.
- `Dep::registry` is for `rok.lock` only; using it in `rok.rokon` is an error.
- Package names (keys of `deps`, `name`) are lowercase ASCII `[a-z0-9_-]+` beginning with a letter (beginning with a digit, `-` or `_` is an error). Keys are strings (`"rok-syntax": Dep::version("^0.1")`).
- In code, the name with `-` replaced by `_` is the top-level path referring to that package (`rok-syntax` → `use rok_syntax::..`, `--extern rok_syntax=..`).
- In the registry and within one dependency graph, names are compared treating `-` and `_` as equal and ignoring case. So `rok-syntax` and `rok_syntax` cannot coexist.
- A dependency package must be a library package with a `lib.rok`. Transitive dependencies are the `deps` of the dependency package (`dev_deps` are not transitive).
- `lints` sets lint levels (lint or group name → `"allow"` | `"warn"` | `"deny"`, [6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control)). They are treated as if prepended to the CLI flags of `rokc lint`, so CLI `-A/-W/-D` and attributes in code override them.
- Errors: unknown fields / `Dep` kinds / named arguments, duplicates, package names violating the rules, syntax that is not RokON, anything other than a single `Package { .. }`, trailing tokens. Diagnostics show the location with `file:line:col` and a caret.

| Dependency source | Form                                                         | Pinned in `rok.lock` |
|-------------------|--------------------------------------------------------------|----------------------|
| path        | `Dep::path("..")` (path relative to the manifest)            | No (always current content) |
| git         | `Dep::git("url")`, optionally one of `rev:` / `tag:` / `branch:` | Commit and tree hash |
| stone       | `Dep::version("req"[, registry: "url"])` or the string `"req"` ([9.10.4a](#9104a-stones-and-rokstonesio)) | Version, commit, tree hash |

The following are reserved; using them is a "reserved, not supported yet" error.

- `Dep::native` (conda-style C libraries)
- The `native` field

Within one build, one name is one package. If the same name is required from different sources (different paths, or different urls / commits), it is an error showing both requirers.

### `rok.lock` format

`rok.lock` is RokON (canonical form) written deterministically by `rok`. After a header comment it holds one `Locked` per git package and stone, in name order. Path dependencies are not pinned and do not appear.

```rokon
// rok.lock: generated by `rok`; do not edit. Commit it with your sources.
Lock {
    version: 1,
    packages: [
        Locked {
            name: "parser",
            source: Dep::git("https://example.com/parser.git", tag: "v1.2"),
            commit: "4f0c2a9e1b7d3c58a6e09f21d4b8c7a3e5f60123",
            tree: "9a1e7c3b5d2f4a6081c9e3b7d5f2a4c6e8b0d1f3",
        },
        Locked {
            name: "json",
            vers: "1.4.2",
            source: Dep::registry("https://github.com/rok-lang/rokstones.io-index"),
            commit: "0b6d..",
            tree: "77e1..",
        },
    ],
}
```

| Field     | Meaning                                                              |
|-----------|----------------------------------------------------------------------|
| `version` | Lock format version (`1`)                                            |
| `name`    | The `deps` key in `rok.rokon`                                        |
| `vers`    | Stones only. The resolved version                                    |
| `source`  | For git, the `Dep::git(..)` exactly as written in `rok.rokon`; for stones, `Dep::registry("<index url>")` |
| `commit`  | The resolved commit (40 hex digits)                                  |
| `tree`    | The git tree hash of that commit. Used to verify the cached checkout |

- The `source` of `Locked` is `Dep::git(..)` or `Dep::registry(..)`.
- A missing `packages` field is read as an empty list.
- If `version` is not `1`, it is an "unsupported lockfile version" error.
- Every error raised while reading the lock appends "(delete rok.lock and run `rok update` to make it again)".

## 9.10.4. Dependencies: `rok add` / `remove` / `update` / `deps`

```
rok add <name> --path <dir> [--dev] [--env <env>]
rok add <name> --git <url> [--rev <rev> | --tag <tag> | --branch <branch>] [--dev] [--env <env>]
rok add <name>[@<req>] [--registry <url>] [--dev] [--env <env>]      # stone
rok remove <name>
rok update [name...]
rok deps [--dev]
```

- `add` / `remove` edit `rok.rokon`, preserving comments and layout. `--dev` adds to `dev_deps`, and `--env` edits the `rok.rokon` of that named environment. Adding a stone without a requirement writes `^<newest version>`.
- Builds use the commits and versions in `rok.lock`. Newly declared dependencies are added to the lock and removed ones are cleaned up, but only `update` moves something already pinned. Given names, `update` re-resolves only those packages; otherwise it re-resolves all git dependencies and stones.
- `deps` prints the resolved dependency packages as lines `<code name>=<lib.rok path>` (the code name has `-` replaced by `_`), dependencies first (`--dev` includes dev dependencies).
- `rok.lock` is committed to version control.

### Cache

Fetched git sources are kept in `$ROK_HOME/cache/` and shared by all projects.

| Path                                       | Contents                                                |
|--------------------------------------------|---------------------------------------------------------|
| `cache/git/db/<repo>-<hash>/`              | Bare clone of the repository (one per url)              |
| `cache/registry/db/<repo>-<hash>/`         | Bare clone of the stone index (one per index)           |
| `cache/git/checkouts/<commit>/`            | Checkout of a commit. Verified by tree hash before use, and re-extracted if different |
| `cache/scripts/<hash>/rok.lock`            | Lockfile of a single-file script (9.10.7)               |

## 9.10.4a. Stones and rokstones.io

A package in a registry is called a **stone**. The default registry is **rokstones.io**, whose index is the separate git repository `https://github.com/rok-lang/rokstones.io-index`.

### Version requirements

Same as Cargo.

| Requirement              | Meaning                                         |
|--------------------------|-------------------------------------------------|
| `1.2.3`, `^1.2.3`        | `>=1.2.3, <2.0.0` (`^0.2.3` is `<0.3.0`, `^0.0.3` is `<0.0.4`) |
| `~1.2.3`                 | `>=1.2.3, <1.3.0`                               |
| `=1.2.3`                 | Exactly                                         |
| `>=` `>` `<` `<=`        | Comparison                                      |
| `*`, `1.*`, `1.2.x`      | Wildcard                                        |
| `>=1.2, <1.5`            | Comma means all must be satisfied               |

A pre-release version (`1.3.0-beta.1`) matches only when the requirement names a pre-release of the same major.minor.patch.

### Which index

The order is `ROK_REGISTRY` > `Config { registry: "<url>" }` in `$ROK_HOME/config.rokon` > the default (the rokstones.io index). The `registry:` of `Dep::version` and `rok add --registry` set the index for that one dependency.

### Index format

The root of the index repository has a `config.rokon` (`Registry { name, dl, api }`). A stone is one file at the shard path of its published lowercase name (`-` kept as is).

| Name length | Path                     |
|-------------|--------------------------|
| 1           | `1/a`                    |
| 2           | `2/ab`                   |
| 3           | `3/a/abc`                |
| 4 or more   | `ab/cd/abcd..`           |

Each line of the file is one version, as single-line RokON in canonical form. Older versions come first.

```rokon
Stone { name: "json", vers: "1.4.2", deps: [{ name: "util", req: "^0.3", kind: "normal" }], source: Dep::git("https://example.com/json.git", rev: "0b6d.."), cksum: "77e1..", yanked: false }
```

- The `kind` in `deps` is `"normal"` or `"dev"`; `registry` is optional.
- `description`, `license` and `repository` from the manifest appear as optional trailing fields.
- The package is at the root of the `source` repository. `cksum` is the git tree hash of that commit, verified after checkout.

### Resolution

- Among matching versions, the highest non-yanked one is chosen. A version pinned in the lock is kept as long as it still matches (even if yanked since).
- In one dependency graph, one name is one version. Transitive dependencies follow the `deps` of the index entry.
- If two requirements do not overlap, it is an error stating both. There is no backtracking; writing a requirement at the root that both accept resolves it.
- The lock is checked against the index. If the `rev` of a pinned version has changed, it is a "run `rok update`" error.
- `--offline` uses the cached index and respects `--locked`.

### `rok search` / `publish` / `yank`

```
rok search <query>
rok publish [--index <checkout>] [--commit]
rok yank --vers <version> [--undo] --index <checkout> [name]
```

- `search` searches the index by name. Index lookups accept either spelling, `-` or `_`.
- `publish` is a dry run by default. It requires a clean work tree, the package at the repository root, and a `lib.rok`. Normal path / git dependencies are rejected, and path / git dev dependencies are dropped. It checks for conflicts with the same version and with names equal under `-` / `_` and case folding. A name already owned under a different spelling is rejected. With `--commit`, it creates a commit "Publish <name> <vers>" in the index checkout; it does not push.
- `yank` sets (or with `--undo` clears) `yanked` for that version in the index checkout and commits.

## 9.10.5. Environments

Environments follow uv / conda. There is no activation step.

- Every directory with a `rok.rokon` (a project) is its own environment. `rok` commands inside it use its dependencies.
- A named environment keeps `rok.rokon` / `rok.lock` in `$ROK_HOME/envs/<name>/`. Path dependencies in it are written as absolute paths.
- The environment used outside a project is chosen in the order `--env <name>` > environment variable `ROK_ENV` > the one set by `rok env use` (`$ROK_HOME/envs/CURRENT`) > `default`.

```
rok env create <name>
rok env list                      # marks the current environment with *
rok env remove <name>
rok env use <name>                # changes the default environment outside projects
rok env export <name> <dir>       # writes rok.rokon / rok.lock to <dir>
rok env import <name> <dir | rok.rokon>    # creates from a project or export (relative paths made absolute)
rok env shell [name]              # $SHELL with ROK_ENV=<name>
rok repl --env <name>
```

`:add <dep> ..` in the REPL adds dependencies to the current environment (same arguments as `rok add`). After adding, it re-reads the dependencies and attaches them to the session.

## 9.10.6. toolchain

`rok` / `rokc` / `rokls` in `$ROK_HOME/bin/` are proxies that run the same executable of the active toolchain (rustup-style). The active toolchain is chosen in this order:

1. `+<version>` on the command line (`rok +0.4 build`)
2. The environment variable `ROK_TOOLCHAIN`
3. `toolchain: "X"` in the manifest (the nearest `rok.rokon`)
4. The default (`Config { registry, toolchain }` in `$ROK_HOME/config.rokon`)

A version prefix matches versions under it (`0.3` is the highest `0.3.*`). If no toolchain matches, it is an error suggesting `rok toolchain install`.

```
rok toolchain list
rok toolchain install <version> --from <dir | archive | url>
rok toolchain default [<version>]          # without an argument, prints the current default
rok toolchain remove <version>
```

- An installation is `$ROK_HOME/toolchains/<version>-<triple>/` and is self-contained ([9.2.19](902-aot-pipeline.md#9219-installation-layout)). `install` requires `--from`.
- `install` verifies with `bin/rokc --version --verbose`; the version and host must match.
- The first installed toolchain becomes the default, and the `bin/` proxies are created at that point.

### Distribution

- A release archive is `rok-<version>-<triple>.tar.gz` (and `.sha256`), containing the toolchain layout.
- `install.sh` (<https://rok-lang.org/install.sh>) installs into `$ROK_HOME` from an archive or from GitHub releases of the `rok-lang` organization.

## 9.10.7. Inline manifest for single-file scripts

An anonymous package ([4.16.8](../004-functions/416-entry-point.md#4168-single-file-execution)) declares dependencies with a ```` ```rokon ```` block holding `Package { deps: { .. } }` inside the `//!` lines at the very start of the file (like uv's inline script metadata). The block may contain only `deps`. Code blocks of other languages are ignored. Paths are relative to the script directory. The script's lockfile is kept in the cache (`cache/scripts/`).

````rok
//! ```rokon
//! Package {
//!     deps: { "util": Dep::git("https://example.com/util.git", tag: "v1") },
//! }
//! ```

use util::greet;
greet();
````
