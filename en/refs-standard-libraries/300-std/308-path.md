# 3.8. `path::` — UTF-8 Path (Python-style)

The `rok::std::path` module defines path representation. The key point is that it puts a UTF-8 newtype first, to avoid the ergonomic burden of Rust's split between `Path` and `OsStr` while adopting the intuitiveness of Python's `pathlib.Path`.

## 3.8.1. Two types — `Path` / `RawPath`

rok's path representation is split into two types.

| Type      | Meaning                                       | Frequency of use      |
|-----------|-----------------------------------------------|-----------------------|
| `Path`    | UTF-8 path (95%+)                             | Default for ordinary code |
| `RawPath` | OS native bytes (Unix bytes / Windows WTF-16) | 1% (non-UTF-8 file names) |

Most code needs only `Path`. `RawPath` is needed only where legacy bytes are truly required, such as when dealing with old unix systems.

## 3.8.2. `Path` — UTF-8

`Path` is an immutable value. `join`, `/` and `with_*` all return a new `Path`; there is no `PathBuf` / `push` that modifies a path in place. `p /= q` is `p = p / q` under the ordinary compound assignment rule.

The `Path` interface is as follows.

```rok
struct Path { /* UTF-8 str newtype, GC heap */ }

impl Path {
    fn new() -> Path;
    fn from(s: str) -> Path;
    fn from_components<I: Iterator<Item = str>>(components: I) -> Path;

    // decomposition
    fn parent(self) -> Path | none;
    fn file_name(self) -> str | none;
    fn file_stem(self) -> str | none;
    fn extension(self) -> str | none;
    fn components(self) -> Components;   // implements Iterator<Item = Component>
    fn ancestors(self) -> Ancestors;     // implements Iterator<Item = Path>

    // manipulation
    fn join(self, other: Path) -> Path;
    fn with_file_name(self, name: str) -> Path;
    fn with_extension(self, ext: str) -> Path;
    fn strip_prefix(self, base: Path) throws (StripPrefixError) -> Path;

    fn is_absolute(self) -> bool;
    fn is_relative(self) -> bool;
    fn has_root(self) -> bool;
    fn starts_with(self, base: Path) -> bool;
    fn ends_with(self, suffix: Path) -> bool;
    fn normalize(self) -> Path;             // resolve "." / ".." (lexical)

    // OS interaction (convenience — fs::* internally)
    fn exists(self) -> bool;
    fn is_file(self) -> bool;
    fn is_dir(self) -> bool;
    fn is_symlink(self) -> bool;
    fn metadata(self) throws (IoError) -> Metadata;
    fn canonicalize(self) throws (IoError) -> Path;     // realpath (verified against the filesystem)

    fn as_str(self) -> str;
    fn to_raw(self) -> RawPath;
}

enum Component {
    RootDir,                    // "/"
    Prefix(str),                // "C:" (Windows)
    CurDir,                     // "."
    ParentDir,                  // ".."
    Normal(str),                // "data"
}
```

## 3.8.3. The `/` operator — Python pathlib style

The `/` operator is introduced to make path joining more readable.

```rok
impl Div<str> for Path {
    type Output = Path;
    #[infix("/")]
    fn div(self, rhs: str) -> Path { self.join(Path::from(rhs)) }
}

impl Div<Path> for Path { ... }
```

`p /= "x"` and `p /= q` (`q: Path`) are also allowed and mean `p = p / ..`.

The **effect** is clear from the following example.

```rok
let p = Path::from("/etc") / "nginx" / "nginx.conf";
                                          //→ Path("/etc/nginx/nginx.conf")

let home = env::home_dir()? / ".config" / "myapp" / "settings.toml";
```

This provides **the same ergonomics** as Python's `Path("/etc") / "nginx" / "nginx.conf"`, and avoids the verbosity of Rust's `PathBuf::join` and `push`.

## 3.8.3a. `AsPath` — path arguments

```rok
trait AsPath {
    fn as_path(self) -> Path;
}
impl AsPath for str { ... }
impl AsPath for Path { ... }
```

Path arguments of `fs` are taken as `AsPath`, so both `str` and `Path` can be passed (`fs::read("a.txt")`, `fs::read(dir / "a.txt")`).

## 3.8.4. UTF-8 invariant

`Path` is **always UTF-8**. At `Path::from(s)`, `s` is already UTF-8, so no separate validation is needed.

There are rare situations, however, where the OS returns non-UTF-8 file names.
- On Unix, arbitrary bytes are possible, so `Path::from(...)` performs a **lossy conversion** (replacement char `U+FFFD`).
- On Windows, names are WTF-16, and surrogate pairs are handled separately.

When a strict conversion is needed, write:

```rok
let raw: RawPath = ...;
let p: Path = try Path::try_from(raw);    // throws NonUtf8Error if not UTF-8
let q: Path = try raw.try_into();          // same
```

This conversion is `impl TryFrom<RawPath> for Path { type Error = NonUtf8Error; ... }` ([2.11.4](../200-core/211-conversion.md#2114-tryfromu-trait--fallible-conversion)).

## 3.8.5. `RawPath` — bytes / WTF-16

`RawPath` exists for places that need the OS's native representation as is.

```rok
struct RawPath { /* unix: Vec<u8>, windows: Vec<u16> */ }

impl RawPath {
    fn from_bytes(bytes: List<u8>) -> RawPath;        // unix
    fn from_wide(wide: List<u16>) -> RawPath;          // windows

    fn as_bytes(self) -> Slice<u8>;                    // unix only
    fn as_wide(self) -> Slice<u16>;                    // windows only

    fn try_to_path(self) throws (NonUtf8Error) -> Path;
    fn to_path_lossy(self) -> Path;
}
```

On Windows, `NonUtf8Error.valid_up_to` counts UTF-16 units. `to_path_lossy` replaces each maximal ill-formed subsequence (Unix) / each lone surrogate (Windows) with one U+FFFD.

Most code never encounters this type. It appears only where code exchanges syscalls directly with the OS.

## 3.8.6. canonicalize vs. normalize

These two functions look similar but differ clearly in behavior and cost.

| Function         | Meaning                       | File system access  |
|------------------|-------------------------------|---------------------|
| `normalize()`    | Purely lexical cleanup (resolves "..") | **No**     |
| `canonicalize()` | Resolves symlinks + converts to an absolute path | **Yes** (`realpath`) |

```rok
Path::from("/a/b/../c").normalize();              //→ "/a/c" (lexical)
try Path::from("/a/b/../c").canonicalize();       //→ absolute path with symlinks resolved
```

`normalize` is fast and makes no syscalls, so it does the job in 90% of places.

## 3.8.7. WASI compatibility

`Path` also integrates naturally with WASI Preview 2's **capability paths**.

```rok
let dir: Dir = try Dir::root().open_dir("data");
let f = try dir.open(Path::from("config.toml"));
                                          // Path is relative to dir — escape rejected
```

The capability model is covered in detail in [3.1 fs](301-fs.md).

## 3.8.8. Comparison with other languages

To pin down the design decision, compare with neighboring languages.

| Aspect        | Rust                      | Python `pathlib` | Go `filepath`   | rok                         |
|---------------|---------------------------|------------------|-----------------|-----------------------------|
| Type          | `Path` / `PathBuf`        | `Path`           | `string`        | `Path`                      |
| owned vs view | borrowed `&Path` / owned  | **immutable**    | **immutable**   | **immutable** (`/` gives a new Path) |
| `/` operator  | **none** (explicit `.join()`) | `/`          | **none**        | `/`                         |
| non-UTF-8     | `OsStr` (mixed)           | `bytes` (separate) | `string` (confusing) | separate `RawPath`   |
| canonicalize  | `.canonicalize()`         | `.resolve()`     | `filepath.Abs`  | `.canonicalize()`           |
| Method chaining | OK (mostly)             | excellent        | OK              | excellent                   |

In summary, rok's **biggest decision** is the combination of a UTF-8 default and the `/` operator, which brings Python pathlib's ergonomics together with Rust-level performance.
