# 9.10. `rok` - package / build front-end

`rok` is a cargo-style front-end. It owns the manifest `rok.rokon` and lockfile `rok.lock`, dependencies, environments, the toolchain pin and script dependencies, and delegates compilation to the compiler `rokc` ([9.1](901-rokc-cli.md)). The runtime is `rokr` (`librokr.a`).

A package without dependencies builds by file conventions alone, without a manifest ([0.3.4](../000-meta/003-source-units.md#034-the-manifest-rokrokon-is-optional)). In that case `rok build` is equivalent to `rokc build`.

## 9.10.1. Subcommands and global options

| Subcommand                                                        | Action                                  |
|-------------------------------------------------------------------|-----------------------------------------|
| `new`, `init`                                                     | Create a package (9.10.2)               |
| `add`, `remove`, `update`, `deps`                                 | Dependencies (9.10.4)                   |
| `search`, `publish`, `yank`, `login`, `logout`                    | Stones (9.10.4a)                        |
| `env`                                                             | Environments (9.10.5)                   |
| `toolchain`                                                       | Toolchain (9.10.6)                      |
| `install`, `uninstall`                                            | Libraries for the REPL / Jupyter, and executables (9.10.5a) |
| `build`, `check`, `run`, `emit`                                   | Delegated to `rokc` with dependencies attached |
| `test`, `bench`                                                   | `rokc` builds the harness with dependencies attached; `rok` runs it ([9.1](901-rokc-cli.md)) |
| `doc`, `lint`, `repl [path]`, `lsp`                               | Delegated to `rokc` with dependencies attached |
| `fmt`, `clean`, `demangle`, `profdata`                            | Delegated to `rokc` without dependencies |
| `jupyter`                                                         | Running the kernel (`--connection-file`) is delegated; `jupyter install [--env <name>]` installs the kernelspec ([9.8.3](908-jupyter-integration.md#983-kernelspec-installation)) |
| `version`, `help`                                                 |                                         |
| (none)                                                            | Same as `rok repl`: starts the REPL     |

A delegated subcommand runs the `rokc` subcommand of the same name. `rok` consumes only its own global options (`--env` / `--offline` / `--locked` / `--cwd`, anywhere before `--`), passes the remaining arguments through unchanged, and adds the following. For `test` and `bench`, `rok` has `rokc` build the harness (`--emit-harness`) rather than run it, then runs the harness itself ([9.1](901-rokc-cli.md)); the runner is not part of the compiler.

- `--extern name=<source directory>` for each dependency package. They are prepared according to `rok.lock`, with dependencies before their dependents (the order `rok deps` prints). Prebuilding is done by `rokc` ([9.2.3](902-aot-pipeline.md#prebuilt-dependency-packages)).
- The package's own features as `--cfg feature=<f>`, and each dependency's as `--extern-feature <name>=<f>` (9.10.4c). The feature options `--features <f,..>` (repeatable; `<dep>/<f>` too), `--all-features` and `--no-default-features` of build / run / test / bench / check / lint / doc / emit / repl / lsp / jupyter / deps are consumed by `rok` (before `--`).
- The manifest's `lints` as `-A` / `-W` / `-D` (before the arguments). The manifest is that of the project the path selects: for a `.rok` file, the nearest directory at or above the file containing `src/main.rok`, `src/lib.rok` or `rok.rokon`. A file in the `src/` of a package selects that package ([9.1.1](901-rokc-cli.md#911-general-form)). A lone file elsewhere in a directory that has `rok.rokon` (a script beside a package's `src/`, or a directory with only `rok.rokon`) is compiled as a single-file package with that manifest's `lints` (and `lints_from`) and dependencies.
- Outside a package (environments, scripts), `--target-dir` set to the `target/` of that environment or script.

`rok repl <path>` resolves dependencies and lints from the project of that path, as `run` does, and passes the manifest name (`--name`).

Options, output and exit codes follow those of `rokc` ([9.1](901-rokc-cli.md)). `rok` treats a closed standard output or standard error the same way as `rokc` ([9.1.6](901-rokc-cli.md#916-exit-codes)). `rokls` is a standalone executable equivalent to `rok lsp`.

Running just `rok [options]` with no subcommand starts the REPL, like `rok repl`.

```
rok                 # = rok repl
rok --env sci       # = rok repl --env sci
```

`rok [options] <file> [args..]` runs a script: it is `rok run -q <file> -- args..`. Every argument after the file is passed to the program unchanged, and the exit code is the program's. The word is a file when it contains `/` (also `\` on Windows) or has the extension `.rok`; no subcommand does, so a subcommand is never taken for a file. Together with a shebang line ([1.2.3](../001-lexical/102-lexical-structure.md#shebang-line)), `./script.rok args` runs the script.

The path of a delegated command is its first argument that is neither an option nor an option's value, so options may come before it (`rok run --release t.rok`).

The global options of `rok` are:

| Option           | Meaning                                                           |
|------------------|-------------------------------------------------------------------|
| `--env <name>`   | Work in that named environment (the REPL, Jupyter, `rok install`); an error on a command that works on a package ([9.10.5](#9105-environments)) |
| `--offline`      | Do not use the network. git dependencies must be in the cache     |
| `--locked`       | If `rok.lock` would need to change, do not change it and fail (for CI) |
| `--cwd <dir>`    | Change directory before running                                   |
| `-q`, `--quiet`  | Reduce output                                                     |

Environment variables are `ROK_HOME` (default `~/.rok`), `ROK_ENV` (9.10.5), `ROK_TOOLCHAIN` (9.10.6), `ROK_REGISTRY` (the index) and `ROK_REGISTRY_TOKEN` (the registry API token) (9.10.4a), and `ROK_DIST` (the release server, 9.10.6).

## 9.10.2. `rok new` / `rok init`: package creation

```
rok new [--lib] [--bin] <name>
rok init [--lib] [--bin] [dir]
rok init --script <file.rok>
```

`rok new` creates the directory `<name>` and fills in default files. `--bin` (the default) creates `src/main.rok`, which prints a greeting; `--lib` creates `src/lib.rok` with an example function and one `#[test]`; both together create both, `src/main.rok` then calling the library's function through `use <name>::add;` ([0.3.1](../000-meta/003-source-units.md#031-executables-and-libraries)). It is an error if `<name>` already exists.

`rok init` turns an existing directory (the current directory if omitted) into a package. It is an error if that directory already contains `src/main.rok` or `src/lib.rok`.

`rok init --script <file.rok>` turns a script ([9.10.7](#9107-inline-manifest-for-single-file-scripts)) into a package in the script's directory, named after the file (without `.rok`). `rok.rokon` gets the merged `deps` (and `native`) of the script's blocks, paths rewritten relative to it; the script becomes `src/main.rok` and every file its `mod`s load moves under `src/` at the same relative path, each without its block. It is an error if the script is inside a package, a moved file or `rok.rokon` already exists, or a submodule file is outside the script's directory.

Both commands also write `rok.rokon` in canonical form ([ROKON 3.9.4](../../refs-standard-libraries/300-std/309-rokon.md#394-canonical-form)) and a `.gitignore` containing `target/`, and run `git init` if not inside a git repository. `init` uses the directory name, adjusted to the naming rules (9.10.3), as the package name.

```
hello/
├── .gitignore        <- target/
├── rok.rokon
└── src/
    └── main.rok
```

## 9.10.3. `rok.rokon` / `rok.lock`

`rok.rokon` at the package root is the manifest, and `rok.lock` in the same place is the lockfile. Both are in [ROKON](../../refs-standard-libraries/300-std/309-rokon.md) format. The names are spelled exactly this way (all lowercase). Files differing only in case (`Rok.rokon` etc.) are ignored with a warning and not read.

The manifest is a single ROKON value, a `Package { .. }` struct (the root name `Package` may be omitted). It is only read, never compiled or executed.

```rokon
Package {
    name: "geom",                    // string, optional
    version: "0.3.1",                // string, optional
    toolchain: "0.4",                // string, optional (toolchain pin, 9.10.6)
    description: "2D geometry",      // string, optional
    license: "MIT OR Apache-2.0",    // string, optional (SPDX expression)
    repository: "https://example.com/geom.git",   // string, optional
    deps: {                          // package name (string) -> Dep
        "util": Dep::path("../util"),
        "parser": Dep::git("https://example.com/parser.git", tag: "v1.2"),
        "http": "0.9",                                   // stone (= Dep::version("0.9"))
        "json": Dep::version("^1.4", optional: true),    // only with a feature (9.10.4c)
        "csv": Dep::version("^0.3", registry: "https://example.com/index"),
    },
    dev_deps: {                      // same form as deps. test / bench only
        "fixtures": Dep::path("tests/fixtures"),
    },
    features: { "default": ["json"], "json": ["dep:json"], "tls": ["http/tls"], "full": ["json", "tls"] },
    native: { "zlib": "^1.3", "png": Dep::native(url: "vendor/png.rokon", sha256: "<64 hex>") },
    lints_from: "..",                // string, optional (starts from ../rok.rokon's lints)
    lints: { "dead_code": "allow", "nonstandard_style": "deny" },
}
```

- Values are ROKON literals (strings, `{ "k": v }` dicts, the empty dict `{}`) and `Dep::path("..")` / `Dep::git("url", rev | tag | branch: "..")` / `Dep::version("req", registry: "url")`. `Dep::git` takes at most one of `rev` / `tag` / `branch`, and `registry` of `Dep::version` is optional. A dependency value that is a single string is shorthand for `Dep::version`.
- `Dep::path` / `Dep::git` / `Dep::version` also take `optional: true`, `features: ["<f>", ..]` and `default_features: false` (9.10.4c); an optional dev dependency is an error.
- `Dep::registry` is for `rok.lock` only; using it in `rok.rokon` is an error.
- Package names (keys of `deps`, `name`) are lowercase ASCII `[a-z0-9_-]+` beginning with a letter (beginning with a digit, `-` or `_` is an error). The name `rok` is reserved for the standard library: a package, dependency, native package or registry name `rok` is an error (`` `rok` is reserved for the standard library; name the package something else, e.g. `rok_util` ``), and so is `rok new` / `rok init` / `rok add` with it. Code must be able to name the package too, so its code name (below) may not be a keyword (1.5: `fn`, `match`, `self`, ..) or a path start (5.5.7: `pkg`, `sup`); `use pkg::add;` in an executable always means its own root (`` `pkg` already names a package's own root (`pkg::`), so code can't name a package called that; name the package something else, e.g. `pkg_util` ``). Keys are strings (`"rok-syntax": Dep::version("^0.1")`).
- In code, the name with `-` replaced by `_` is the top-level path referring to that package (`rok-syntax` → `use rok_syntax::..`, `--extern rok_syntax=..`).
- In the registry and within one dependency graph, names are compared treating `-` and `_` as equal and ignoring case. So `rok-syntax` and `rok_syntax` cannot coexist.
- A dependency must have a `src/lib.rok`; that library is what its dependents get, whether or not the package also has `src/main.rok`. A dependency with only `src/main.rok` is an error that says it is a program, for `rok install`. Transitive dependencies are the `deps` of the dependency package (`dev_deps` are not transitive).
- `native` maps native package names (their own namespace: a package and a native package may share a name) to `Dep::native("req"[, registry: "url"])`, a requirement string, or `Dep::native(url: "<url or path relative to the manifest>", sha256: "<64 hex digits>")` (9.10.4b). `Dep::native` in `deps` / `dev_deps` is an error. Native dependencies of every package in the graph are linked into executables (those reached only through `dev_deps`, only for test / bench).
- `lints` sets lint levels (lint or group name → `"allow"` | `"warn"` | `"deny"`, [6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control)). They are treated as if prepended to the CLI flags of every `rokc` command `rok` runs, so CLI `-A/-W/-D` and attributes in code override them.
- `lints_from` names a directory (relative to this manifest's directory, or absolute) whose `rok.rokon` supplies starting lint levels. That manifest's `lints`, after its own `lints_from` chain, are applied first, and this manifest's `lints` follow, so an entry here overrides the same lint or group from the chain (all entries are passed as `-A`/`-W`/`-D` in this order; later flags win). It applies only to the package being built or checked (and to an environment's manifest); dependencies' manifests are never consulted for lints. It is an error if the directory does not exist, contains no `rok.rokon`, or if the chain returns to a manifest already in it (a cycle). `rok env import` rewrites a relative `lints_from` as an absolute path. A script's `//! ```rokon` block may not use it.
- Errors: unknown fields / `Dep` kinds / named arguments, duplicates, `features` that break the rules of 9.10.4c, package names violating the rules, syntax that is not ROKON, anything other than a single `Package { .. }`, trailing tokens. Diagnostics show the location with `file:line:col` and a caret.

| Dependency source | Form                                                         | Pinned in `rok.lock` |
|-------------------|--------------------------------------------------------------|----------------------|
| path        | `Dep::path("..")` (path relative to the manifest)            | No (always current content) |
| git         | `Dep::git("url")`, optionally one of `rev:` / `tag:` / `branch:` | Commit and tree hash |
| stone       | `Dep::version("req"[, registry: "url"])` or the string `"req"` ([9.10.4a](#9104a-stones-and-rokstonesio)) | Version, commit, tree hash |
| native      | `Dep::native("req")` / `Dep::native(url:, sha256:)` ([9.10.4b](#9104b-native-packages)) | Version and entry SHA-256 (index); none (the hash pins it) |

Rok has no build scripts: no code from a package runs at build time. C / C++ libraries come in as prebuilt native packages (9.10.4b).

Within one build, one name is one package. If the same name is required from different sources (different paths, or different urls / commits), it is an error showing both requirers.

### `rok.lock` format

`rok.lock` is ROKON (canonical form) written deterministically by `rok`. After a header comment it holds one `Locked` per git package and stone, in name order. Path dependencies are not pinned and do not appear.

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
- Features are not recorded: the lock pins every package of the graph, optional ones included, so it does not depend on the features a build enables.
- Native packages from an index are pinned in an optional `natives: [LockedNative { name, vers, source: Dep::registry("<index>"), sha256 }]`, sorted by name and written only when non-empty. `sha256` is the SHA-256 of the entry's canonical index line with `yanked: false`; a changed entry is a "run `rok update`" error.
- If `version` is not `1`, it is an "unsupported lockfile version" error.
- Every error raised while reading the lock appends "(delete rok.lock and run `rok update` to make it again)".

## 9.10.4. Dependencies: `rok add` / `remove` / `update` / `deps`

```
rok add <name> --path <dir> [--dev]
rok add <name> --git <url> [--rev <rev> | --tag <tag> | --branch <branch>] [--dev]
rok add <name>[@<req>] [--registry <url>] [--dev]      # stone
rok add <name> .. [--features <f,..>] [--no-default-features] [--optional]   # with any of the above
rok remove <name>
rok add .. <file.rok>                    # into a script's //! rokon block
rok remove [--native] <name> <file.rok>
rok update [name...]
rok deps [--dev] [--features <f,..>] [--all-features] [--no-default-features]
```

- `add` / `remove` edit `rok.rokon`, preserving comments and layout. `--dev` adds to `dev_deps`. `add` / `remove` edit the package's `rok.rokon`. Given a script file as the last argument, `add` / `remove` edit that file's `//! rokon` block instead (9.10.7): a missing block is created after the shebang line and the leading `//!` lines (written in canonical form; an existing block keeps its layout and comments), a block left empty is removed, a path is written relative to the script, and `--dev` / `--optional` are errors. A script inside a package is an error (it uses the package's `rok.rokon`). Outside a package without a script, or with `--env`, they are an error naming `rok add <name> <file.rok>` and `rok install` / `rok uninstall` ([9.10.5a](#9105a-rok-install--rok-uninstall)). Adding a stone without a requirement writes `^<newest version>`. `--features`, `--no-default-features` and `--optional` are written into the `Dep::..`; `--optional` also adds the feature `"<name>": ["dep:<name>"]` unless the package has a feature of that name.
- Builds use the commits and versions in `rok.lock`. Newly declared dependencies are added to the lock and removed ones are cleaned up, but only `update` moves something already pinned. Given names, `update` re-resolves only those packages; otherwise it re-resolves all git dependencies and stones.
- `deps` prints the resolved dependency packages as lines `<code name>=<path of its src/lib.rok>` (the code name has `-` replaced by `_`), dependencies first (`--dev` includes dev dependencies). Each such line is followed by its `--extern-feature=<code name>=<f>` lines; `deps` takes the feature options and `--dev`. For each native package (dependents first, for the host) it also prints `# native <name> <vers>` followed by its `rokc` arguments, one per line.
- `rok.lock` is committed to version control.
- `rok update` re-resolves `rok.lock`; `rok toolchain update` replaces the toolchain (9.10.6).

### Cache

Fetched git sources are kept in `$ROK_HOME/cache/` and shared by all projects.

| Path                                       | Contents                                                |
|--------------------------------------------|---------------------------------------------------------|
| `cache/git/db/<repo>-<hash>/`              | Bare clone of the repository (one per url)              |
| `cache/registry/db/<repo>-<hash>/`         | Bare clone of the stone index (one per index)           |
| `cache/git/checkouts/<commit>/`            | Checkout of a commit. Verified by tree hash before use, and re-extracted if different |
| `cache/scripts/<hash>/`                    | `rok.lock` and `target/` of a script; `<hash>` covers the script's path and the path and block of every file it reaches ([9.10.7](#9107-inline-manifest-for-single-file-scripts)) |
| `cache/native/<sha256>/`                   | A native archive, unpacked after its SHA-256 matched; never fetched again (9.10.4b) |
| `cache/native/meta/<sha256>.rokon`         | A `Dep::native(url:)` file                              |

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

The root of the index repository has a `config.rokon`, `Config { api: "<url>" }`: the base URL of the registry's API (`https://rokstones.io` for rokstones.io). It must be `https://`, except `http://127.0.0.1` and `http://localhost`. A stone is one file at the shard path of its published lowercase name (`-` kept as is).

| Name length | Path                     |
|-------------|--------------------------|
| 1           | `1/a`                    |
| 2           | `2/ab`                   |
| 3           | `3/a/abc`                |
| 4 or more   | `ab/cd/abcd..`           |

Each line of the file is one version, as single-line ROKON in canonical form. Older versions come first.

```rokon
Stone { name: "json", vers: "1.4.2", deps: [{ name: "util", req: "^0.3", kind: "normal" }], source: Dep::git("https://example.com/json.git", rev: "0b6d.."), cksum: "77e1..", yanked: false }
```

- The `kind` in `deps` is `"normal"`, `"dev"` or `"native"`; `registry` is optional.
- A stone with features has `features: { "<name>": ["<entry>", ..] }` after `deps`, checked as in 9.10.4c. A `normal` dependency may have `optional: true`, `features: [..]` and `default_features: false` (written only when not the default); `dev` / `native` ones may not be optional, and native ones have no features. `rok publish` writes them from `rok.rokon`; resolution reads them from the entry. After checkout the entry's `features` must equal those of the stone's `rok.rokon` at that commit (in any order); otherwise it is an error.
- `native/<shard path>` holds native packages, one `Native { .. }` per line ([9.10.4b](#9104b-native-packages)).
- `description`, `license` and `repository` from the manifest appear as optional trailing fields.
- The package is at the root of the `source` repository. `cksum` is the git tree hash of that commit, verified after checkout.

### Resolution

- Among matching versions, the highest non-yanked one is chosen. A version pinned in the lock is kept as long as it still matches (even if yanked since).
- In one dependency graph, one name is one version. Transitive dependencies follow the `deps` of the index entry.
- If two requirements do not overlap, it is an error stating both. There is no backtracking; writing a requirement at the root that both accept resolves it.
- The lock is checked against the index. If the `rev` of a pinned version has changed, it is a "run `rok update`" error.
- `--offline` uses the cached index and respects `--locked`.

### `rok search` / `publish` / `yank` / `login` / `logout`

```
rok search [--native] <query>
rok login [<token>]
rok logout
rok publish [--index <checkout>] [--commit]
rok yank [--native] --vers <version> [--undo] [--index <checkout>] [name]   # name required with --native
```

- `search` searches the index by name and prints one line per match: `<name> <newest version> - <description>` (the description part omitted when the entry has none). Index lookups accept either spelling, `-` or `_`.
- `publish` is a dry run by default. It requires a clean work tree, the package at the repository root, and a `src/lib.rok`, a `src/main.rok` or both. A stone with only `src/main.rok` is a program: `rok install` builds it, and it cannot be a dependency. Normal path / git dependencies are rejected, and path / git dev dependencies are dropped. It checks for conflicts with the same version and with names equal under `-` / `_` and case folding. A name already owned under a different spelling is rejected. A dry run without `--index` only prints the entry; with `--index` it is also checked against that checkout. The entry's source is the url of git remote `origin` as configured (before `insteadOf` rewrites); it must be `https://`, except `http://127.0.0.1` and `http://localhost`.
- With `--commit`, `publish` adds the entry: with `--index`, as a commit "Publish <name> <vers>" in that checkout (not pushed); without it, through the registry API of the configured index ("Which index"), which adds it to the index. The client never writes to the index repository itself.
- `yank` sets (or with `--undo` clears) `yanked` for that version: with `--index`, as a commit in that checkout; otherwise through the registry API.
- The registry API is at `api` of the index's `config.rokon`. Requests carry `Authorization: Bearer <token>`, and bodies are single-line ROKON (`application/rokon; charset=utf-8`):
  - `GET <api>/api/v1/me` → `Me { login: "..", publish: <bool> }`
  - `POST <api>/api/v1/publish`, body the index line (`Stone { .. }` / `Native { .. }`) → `Published { name, vers, file }`
  - `POST <api>/api/v1/yank`, body `Yank { name, vers, undo: <bool>, native: <bool> }` → `Yanked { name, vers, yanked }`

  A failure is a 4xx / 5xx status with `Error { message: ".." }`, whose message `rok` reports; 401 means the token is not valid.
- The registry API needs a token; an `--index` checkout needs none. The token is `ROK_REGISTRY_TOKEN` if set and non-empty, else the one `rok login` saved. Without either, it is an error "run `rok login` first".
- `rok login` takes a rokstones.io account token, created on the website (`<api>/me`, signed in with GitHub; currently only the registry's maintainers can create one). When the token is omitted it is read from standard input, not echoed (echo is restored on Ctrl-C). `login` asks `/api/v1/me`: it refuses a token the API rejects, and an account whose `publish` is false ("this account cannot publish yet: publishing is open to rokstones.io maintainers only"), saving nothing. Otherwise it saves `Credentials { token: ".." }` in `$ROK_HOME/credentials.rokon` (mode 0600 on Unix) and prints "Logged in as <login>". `rok logout` removes that file.
- The token reaches `curl` only on its standard input: never on a command line or in the environment, and never printed (errors show `***`).

## 9.10.4b. Native packages

Rok compiles no C. A native package is a prebuilt archive per target triple plus a value:

```rokon
Native {
    name: "png", vers: "1.6.43",
    deps: { "zlib": Dep::native("^1.3") },
    targets: {
        "x86_64-unknown-linux-gnu": Artifact {
            url: "https://example.com/png-1.6.43-x86_64-linux.tar.gz",
            sha256: "<64 hex>",
            links: { "png16": "static" },
            runtime: [],
        },
    },
    yanked: false, description: "PNG codec", license: "Libpng", repository: "https://example.com/png",
}
```

The value is one index line, or a file for `Dep::native(url:, sha256:)`, whose artifact urls may be relative to it.

- The archive (any tar-readable format) holds `lib/`: `lib<n>.a`, `lib<n>.dylib` / `lib<n>.so`, Windows `<n>.lib` (static or import library). It also holds the `runtime` files and license files. Nothing in it is run.
- Hashes are mandatory. An archive is checked before unpacking, and is refused if it has absolute or `..` paths, links leaving it, or lacks a named library or runtime file.
- On Apple targets runtime dylibs must have an `@rpath/` install name.
- Builds use the target triple's artifact; a missing one is an error listing the triples it has.
- `rok` passes `rokc` `-L <archive>/lib`, `-l kind=name` per link and `--native-runtime` per runtime file, dependents before dependencies.

```
rok add --native <name>[@<req>]
rok remove --native <name>
rok search --native <query>
rok publish --native <file> [--index <dir>] [--commit]
rok yank --native --vers <version> [--undo] [--index <dir>] <name>
```

`publish --native` fetches and checks every target, and requires absolute urls, a license file, and dependencies from an index only. `--commit` and `yank --native` follow the stone rules: an `--index` checkout, else the registry API with the token (`native: true` in `Yank`).

- On ELF targets a runtime library's soname must equal its file name. A `dylib` link for a `-musl` target is an error.
- `repl` / `jupyter` load native packages: shared libraries through the system loader, static archives linked into the session. `check` / `lint` / `doc` link nothing.
- Copies beside an executable are build outputs.

## 9.10.4c. Features

A package declares features in `features`, a dict from feature name to the list of what the feature enables. Names are `[a-z0-9][a-z0-9_-]*`, words that say what they are (`http1`, `http2`, `tls`). Each entry is one of:

| Entry       | Enables                                                              |
|-------------|----------------------------------------------------------------------|
| `<f>`       | Feature `f` of this package                                          |
| `dep:<d>`   | The optional dependency `d`                                          |
| `<d>/<f>`   | Feature `f` of dependency `d`, and `d` itself if it is optional      |
| `<d>?/<f>`  | Feature `f` of the optional dependency `d`, only if something else enables `d` |

- `default` is an ordinary feature that is on unless turned off: `--no-default-features` for the package being built, `default_features: false` on a dependency.
- An optional dependency (`optional: true`) is in the build only when a `dep:<d>` or `<d>/<f>` entry of an enabled feature enables it. There is no implicit feature of its name, and it is an error if no entry enables it.
- It is an error if an entry names a feature or dependency (of `deps`) the package does not have, if `dep:` or `?` names a non-optional dependency, or if features form a cycle. Diagnostics show `file:line:col`. It is an error if a dependency is asked for a feature it does not have.
- Resolution: the lock covers the whole graph. The build then starts from the root package's features (`default` unless turned off, `--features`, or every feature with `--all-features`) and follows every enabled dependency, with the features it asks of it plus its `default`. Features are additive and unified: each package has one feature set for the whole graph. The root's `dev_deps` and the features they ask count only for `test` / `bench`. The result does not depend on the order of anything.
- `--features` takes the root's feature names and `<dep>/<f>` for its direct dependencies.
- Each package is compiled with `cfg(feature = "<f>")` for exactly its enabled features ([6.3.6](../006-meta-system/603-cfg.md#636-where-parameters-come-from)). Native packages have no features.
- A script's ```` ```rokon ```` block has no `features` (9.10.7).

## 9.10.5. Environments

Environments replace conda / venv. There is no activation step.

- A package (a directory with `src/main.rok`, `src/lib.rok` or `rok.rokon`) uses the dependencies of its own `rok.rokon` and nothing else; it never sees libraries installed into an environment. Neither does a script. A command that works on a package is an error with `--env`.
- A named environment keeps `rok.rokon`, `rok.lock` and `target/` in `$ROK_HOME/envs/<name>/`. Its `deps` are the libraries installed with `rok install` ([9.10.5a](#9105a-rok-install--rok-uninstall)), for the REPL and Jupyter; only `rok install` / `rok uninstall` and `rok env import` change them. Path dependencies in it are absolute. Installed libraries are prebuilt into its `target/`.
- The environment used outside a package (the REPL, Jupyter, the language server, `rok install`) is chosen in the order `--env <name>` > `ROK_ENV` > `rok env use` (`$ROK_HOME/envs/CURRENT`) > `default`; `default` is created by the first `rok install` into it. Inside a package, `rok repl` / `jupyter` / `lsp` without a path use the package, or with `--env` that environment.

### Where dependencies come from

| Compiled as | Dependencies | Lock | Prebuilt into |
|---|---|---|---|
| A package | its `rok.rokon` `deps` (`dev_deps` for test / bench) | `<package>/rok.lock` | `<package>/target/<profile>/deps/` |
| A script | the merged `deps` of the `//! rokon` blocks of it and of the files its `mod`s load | `cache/scripts/<hash>/rok.lock` | `cache/scripts/<hash>/target/` |
| The REPL / Jupyter outside a package | the active environment's installed libraries | the environment's | the environment's `target/` |

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

`:add <dep> ..` in the REPL runs `rok install <dep> ..` into the current environment, or `rok add` in a package's session; it then re-reads the dependencies and attaches them to the session.

## 9.10.5a. `rok install` / `rok uninstall`

```
rok install <name>[@<req>] [--registry <url>]
rok install [<name>] --path <dir | file.rok>
rok install [<name>] --git <url> [--rev <rev> | --tag <tag> | --branch <branch>]
rok install .. [--features <f,..>] [--no-default-features] [--lib] [--force]
rok install --native <name>[@<req>] [--registry <url>]
rok install --list
rok uninstall [--native] <name>..
```

- The package at the source is fetched into the cache first ([9.10.4](#9104-dependencies-rok-add--remove--update--deps)). A package with `src/main.rok`, or a script file, is an executable: it is built with `--release` and its executable placed in `$ROK_HOME/bin/<name>`, as with `cargo install`. A git or stone package is built from a scratch copy of its checkout; a path package in place; a script file in place. `$ROK_HOME/installs.rokon` (`Package { deps: { "<name>": <source> } }`) records each executable and its exact source (a stone as `=<version>`, a git package with its `rev:`). When the package also has `src/lib.rok`, `rok install` says so and that `--lib` installs the library instead.
- A package with only `src/lib.rok`, or any package with `src/lib.rok` given `--lib`, is a library and goes into the active environment, as with `pip install`, for `rok repl`, `rok jupyter` and `:add` ([9.10.5](#9105-environments)): it is written into the environment's `rok.rokon` (a stone without a requirement as `^<newest version>`, a path as an absolute path), resolved into its `rok.lock`, and prebuilt into its `target/`. Scripts do not see it: a script gets a library with `rok add <name> <file.rok>`; the output line names this. If it does not build, nothing is installed. `--lib` on a package without `src/lib.rok`, or on a script, is an error.
- `<name>` is the library's top-level name, or the executable's file name. If omitted, it is the package's `name`, else the directory name (path), the repository name without `.git` (git), or the file stem (script). It follows the naming rules of [9.10.3](#9103-rokrokon--roklock).
- `--features` / `--no-default-features` go into the library's `Dep`, or to the executable's build; `--dev` / `--optional` are errors. `--native` adds a native package to the environment's `native` ([9.10.4b](#9104b-native-packages)).
- The proxies `rok`, `rokc`, `rokls` ([9.10.6](#9106-toolchain)) are never replaced, and neither is a file in `$ROK_HOME/bin` that `rok install` did not put there (an error). Installing an executable again: one from the same stone version or git commit with the same features is kept, not rebuilt (`` `tool` 0.1.0 is installed already (<path>); to rebuild it, run `rok install tool --force` ``), and `--force` rebuilds it (`Reinstalled`); another version replaces it (`Upgraded tool from 0.1.0 to 0.2.0`, `Downgraded ...`), and another git commit too (`Replaced <url> at <commit> with ...`). One from a path or a script file is always rebuilt, as with `cargo install --path`. Library installs are unchanged.
- `--list` prints ``libraries (environment `<env>`):`` and `executables (<bin dir>):`, each followed by `  <name> <source>` lines in install order.
- `rok uninstall <name>..` removes each from the active environment's libraries (re-resolving its lock), or else removes the executable `rok install` put in `$ROK_HOME/bin`; `--native` removes a native package from the environment. A name that is neither is an error.
- Apart from `rok install`, only `rok toolchain` (toolchains, proxies, `config.rokon`), `rok login` (`credentials.rokon`), `rok env` (environments) and the caches write under `$ROK_HOME`.

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
rok toolchain update [<version>]
rok toolchain clean                        # remove every installed toolchain but the default
```

- An installation is `$ROK_HOME/toolchains/<version>-<triple>/` and is self-contained ([9.2.19](902-aot-pipeline.md#9219-installation-layout)). `install` requires `--from`.
- `install` verifies with `bin/rokc --version --verbose`; the version and host must match.
- The first installed toolchain becomes the default, and the `bin/` proxies are created at that point.
- `rok toolchain update` replaces the default toolchain under `$ROK_HOME`. Without an argument it installs the latest release; with a version it installs that version. The installed toolchain becomes the default and the `bin/` proxies are rewritten; on Windows a running proxy is renamed before its copy replaces it. It downloads the release archive and verifies its SHA-256 (Distribution, below), unpacks it apart and moves it into place, then checks that the proxies run it; on failure the previous default and proxies are restored and the new toolchain is removed. Earlier toolchains stay installed. A version already installed is made the default without a download. `rok update` re-resolves `rok.lock`; `rok toolchain update` replaces the toolchain (9.10.4).
- `rok toolchain clean` removes every installed toolchain except the current default, an opt-in tidy-up for the earlier versions that `update` leaves installed. A version a project pins is removed too, so reinstall it if a build needs it; with no default set it is an error.

### Distribution

- A release archive is `rok-<version>-<triple>.tar.gz` (Windows `.zip`), containing the toolchain layout; outside Windows a `rok-<version>-<triple>.tar.xz` with the same files sits beside it. Each archive has `<archive>.sha256` beside it: the archive's SHA-256 in hex, optionally followed by the file name (sha256sum format).
- `install.sh`, `install.ps1` and `rok toolchain install --from <url>` download `<url>.sha256` from the same place and refuse an archive whose SHA-256 differs; a missing `.sha256` is an error. An archive given by path is checked against `<path>.sha256` when that file exists. Signatures are not defined; HTTPS vouches for the origin.
- `install.sh` (<https://rok-lang.org/install.sh>) installs into `$ROK_HOME` from an archive or from GitHub releases of the `rok-lang` organization. It downloads the `.tar.xz` where tar can read it (macOS, or `xz` on `PATH`) and the release has one, else the `.tar.gz`. `rok toolchain install` downloads the `.tar.gz` (on Windows the `.zip`).
- Releases are served from `$ROK_DIST` (default `https://github.com/rok-lang/rok/releases`): `<dist>/download/v<version>/<archive>` with its `.sha256`, and `<dist>/latest/download/rok-version` holding the latest release's version.

## 9.10.7. Inline manifest for single-file scripts

An anonymous package ([4.16.8](../004-functions/416-entry-point.md#4168-single-file-execution)) gets its dependencies only from ```` ```rokon ```` blocks holding `Package { deps: { .. } }` (and `native`) inside the `//!` lines at the very start (after a shebang line) of its file and of each file its `mod`s load (like uv's inline script metadata). Libraries installed into an environment are never used. A block may contain only `deps` and `native` (no `features`, so no optional dependencies). Code blocks of other languages are ignored. Paths in a block are relative to the file that holds it. The blocks are merged; one library given different entries in two files is an error naming both files and both entries. Running a submodule file directly uses only the blocks of the files that run reaches. `rok add <name> .. <file.rok>` / `rok remove <name> <file.rok>` write a block ([9.10.4](#9104-dependencies-rok-add--remove--update--deps)), and `rok init --script` turns a script into a package ([9.10.2](#9102-rok-new--rok-init-package-creation)). A script keeps its lockfile and builds in `cache/scripts/<hash>/`, keyed by its path and the path and block of every file it reaches. A script's executable is cached by everything it is built from, every file it includes and its libraries included. A `use x::..` of a package a script lacks is an error whose help is the command to run, ``run `rok add x <file>` ``, with a note when the active environment has `x`; in a package it suggests `rok add x`. `import x` (and `include` / `require`) is an error whose help is `mod x;` when `x.rok` is beside the file, `use` when `x` is a path root or a module the file declares, and otherwise `rok add x <file>`.

````rok
//! ```rokon
//! Package {
//!     deps: { "util": Dep::git("https://example.com/util.git", tag: "v1") },
//! }
//! ```

use util::greet;
greet();
````

Running a single-file script reuses its executable. `rokc run` of an anonymous package keeps the executable in `<target>/<profile>/scripts/<key>/`, where `<target>` is the script's `target/` (under `cache/scripts/<hash>/`, beside its `rok.lock`). `<key>` is a hash of everything the build is made from: the toolchain, the build options, lint levels and target, the source and every file it includes, the runtime and the C library the toolchain ships for the target, and the dependencies passed to it with their prebuilt libraries. When the key matches, nothing is compiled and the warnings of the build that made the executable are printed again. A new executable is written under a scratch name and renamed into place, so concurrent runs of one script are safe. An entry unused for a day is removed when another is built.
