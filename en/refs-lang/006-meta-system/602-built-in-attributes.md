# 6.2. Built-in attributes

This section defines the built-in attributes the compiler processes directly. `#[cfg]` is covered separately in [conditional compilation](603-cfg.md), and magic attributes and meta blocks are described in the sections from [`#[lang(...)]` magic](604-lang-magic.md) onward.

```
derive_attr     ::= "#" "[" "derive" "(" trait_name ("," trait_name)* ","? ")" "]"
trait_name      ::= identifier
repr_attr       ::= "#" "[" "repr" "(" repr_arg ("," repr_arg)* ")" "]"
repr_arg        ::= "C" | "transparent" | "packed" | "align" "(" int_lit ")"
                  | identifier                          // primitive int repr
inline_attr     ::= "#" "[" "inline" ("(" ("always" | "never") ")")? "]"
deprecated_attr ::= "#" "[" "deprecated" ("(" deprecated_args ")" | "=" string_lit)? "]"
deprecated_args ::= string_lit | deprecated_kv ("," deprecated_kv)*
deprecated_kv   ::= ("since" | "note") "=" string_lit
future_unsafe   ::= "#" "[" "future_unsafe" ("(" string_lit ")" | "=" string_lit)? "]"
```

## 6.2.1. `#[derive(...)]` automatic trait implementation

```rok
#[derive(Clone, PartialEq, Eq, Hash, Debug)]
struct Point { x: i64; y: i64; }
```

The arguments of `#[derive(...)]` are trait names separated by commas. For each listed trait, an automatic implementation suited to the type is generated.

### Built-in derivable traits

Only `Clone`, `Copy`, `Debug`, `Default`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, and `Hash` can be derived. The generation rules and requirements for each trait are defined in [stdlib 2.13.11](../../refs-standard-libraries/200-core/213-marker.md#21311-derivable-traits).

- `Display` cannot be derived. The display format is decided by the type's author ([stdlib 2.10](../../refs-standard-libraries/200-core/210-fmt.md)).
- Deriving `Eq` / `Ord` without their supertraits (`PartialEq` / `PartialOrd`, `Eq`) is a compile error. `Copy` requires `Clone` and cannot be combined with `Drop`.
- `#[derive]` on a `union` is a compile error.

### Combining with user-defined derives

User-defined traits can also be derived, as in `#[derive(MyTrait)]`. This requires `MyTrait` to be defined by an `#[impl_derive(MyTrait)]` meta block ([6.8](608-meta-blocks.md)). The built-in derive names above cannot be redefined by a meta block.

Built-in and user-defined derives may be mixed freely in one `#[derive(...)]`.

```rok
#[derive(PartialEq, Eq, Clone, MyCustomTrait)]
struct Foo { ... }
```

## 6.2.2. `#[repr(...)]` explicit memory representation

```rok
#[repr(C)]
struct CCompatible { x: i32; y: i32; }

#[repr(transparent)]
struct Wrapped(InnerType);

#[repr(packed)]
struct Tight { a: u8; b: u32; }

#[repr(align(16))]
struct Aligned { ... }

#[repr(u32)]
enum Status { ... }
```

| Argument                    | Meaning                                                                                              |
|-----------------------------|------------------------------------------------------------------------------------------------------|
| `C`                         | C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) layout: declaration order, natural alignment. Same as the default `struct` layout |
| `transparent`               | Exactly one field, and no other `repr` argument. Same representation as that field, and crosses the C ABI as it |
| `packed`                    | Packs tightly, ignoring alignment ([FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) only). Cannot hold GC references; exclusive with `align(N)` |
| `align(N)`                  | Forces a minimum alignment of N bytes                                                                |
| `u8` to `u64` / `i8` to `i64` | The tag type of an `enum`. Widths other than 32 bits only for field-less enums; enums with data accept `i32` / `u32` |

Without `#[repr]`, a [`struct`](../002-types/203-struct.md) uses declaration order and natural alignment (the same as `repr(C)`); the layout of [`enum`](../002-types/204-enum.md) and sums is the compiler's ([2.7](../002-types/207-niche-optimization.md)).

## 6.2.3. `#[link(...)]` naming an external library

```rok
#[link("m")]
extern "C" {
    fn sin(x: f64) -> f64;
    fn cos(x: f64) -> f64;
}

#[link(name = "openssl", kind = "static")]
extern "C" { ... }
```

`#[link(...)]` names the external library an [`extern` block](../005-modules/509-extern-block.md) links against; it is attached to `extern` blocks.

| Argument form               | Meaning                     |
|-----------------------------|-----------------------------|
| `"libname"`                 | Plain library name          |
| `name = "..."`              | Library name (key-value)    |
| `kind = "static" / "dylib"` | Link kind                   |
| `wasm_import_module = "m"` | wasm32: the block's functions are imported from core module `m`; ignored elsewhere |

### `#[link_name = "sym"]`

`#[link_name = "sym"]` on an extern function gives the symbol it links to, or its import name inside a `wasm_import_module` block.

```rok
#[link(wasm_import_module = "env")]
extern "C" {
    #[link_name = "log_line"]
    fn host_log(ptr: ConstPtr<u8>, len: usize);
}
```

## 6.2.4. `#[inline]` inlining hints

```rok
#[inline]
fn small_helper(x: i32) -> i32 { x + 1 }

#[inline(always)]
fn very_hot() { ... }

#[inline(never)]
fn never_inline() { ... }
```

| Argument | Meaning                                       |
|----------|-----------------------------------------------|
| (none)   | Inlining recommended (compiler decides)       |
| `always` | Inlining forced                               |
| `never`  | Inlining forbidden                            |

`#[inline]` is only an optimization hint; it does not affect the program's meaning.

## 6.2.5. `#[test]` test function marker

```rok
#[test]
fn test_addition() {
    assert_eq(1i32 + 1i32, 2i32);
}
```

`assert_eq` is in [stdlib 2.14.5](../../refs-standard-libraries/200-core/214-error.md#assert--assert_eq--assert_ne). `#[test]` on a function marks it as a test function. `rokc test` collects such functions into a harness and the test runner (`rok test`, etc.) runs them automatically. In ordinary builds they are excluded, as if `#[cfg(test)]` were also attached. A test function is run whatever its own visibility and that of its enclosing modules; `#[test]` does not change visibility for other code.

`#[bench]` marks a benchmark function and, like `#[test]`, implies `#[cfg(test)]`. `rokc bench` collects and runs them ([9.1.2](../009-tools/901-rokc-cli.md#rokc-bench-benchmarks)). A benchmark function is run whatever its own visibility and that of its enclosing modules; `#[bench]` does not change visibility for other code.

## 6.2.6. `#[cold]` rarely called functions

```rok
#[cold]
fn handle_error() { ... }
```

`#[cold]` tells the compiler that the function is rarely called. The code generator then places the function outside the hot code path to improve cache friendliness.

## 6.2.7. `#[deprecated]` discouraged-use marker

```rok
#[deprecated]
fn old_api() { ... }

#[deprecated = "use new_api instead"]
fn old_api2() { ... }

impl Config {
    #[deprecated(since = "0.1.6", note = "use `Config::try_new`")]
    pub fn new() -> Config { ... }
}
```

A use of an item marked `#[deprecated]` produces a warning (lint `deprecated`), not an error. A message, given as `= "..."`, `("...")` or `note = "..."`, is shown with the warning; it should say what to write instead. `since = "x.y.z"` names the release that deprecated the item.

The warning covers these items, wherever they are declared (the package itself, a dependency, the standard library):

- functions and methods, called or used as a value; a trait method's mark covers every impl of it;
- fields, when read, assigned, set in a struct literal or named in a pattern;
- structs, enums, unions, type aliases and traits, named in a type or as the start of a path (`Old::new`);
- constants and statics.

Only uses in the package being built warn ([6.2.9](#629-allow--deny--warn-lint-control)). A type's own `impl` does not warn about the type, and a struct's own methods do not warn about its fields. A type is recognized by its full path, or by a last segment no other type shares. The attribute is accepted on any item; on others (variants, impls, modules) it has no effect.

A deprecation whose `since` is newer than the baseline of the `warnings` group is outside that group: `"warnings": "deny"` leaves it a warning, while `deprecated` and `all` reach it. The language server tags the use as deprecated, so editors strike it through.

## 6.2.8. `#[doc = "..."]` doc comment form

```rok
#[doc = "Returns the sum of two integers."]
fn add(x: i32, y: i32) -> i32 { x + y }
```

A `///` doc comment from [lexical structure](../001-lexical/102-lexical-structure.md#doc-comments) is this attribute.

```rok
/// This function ...
fn f() { ... }

// same meaning
#[doc = "This function ..."]
fn f() { ... }
```

The inner form `#![doc]` likewise has the same effect as `//!`.

### `#[doc(hidden)]`

```rok
/// What a dict spread checks (lowered code calls it).
#[doc(hidden)]
pub const fn spread_misfit<V>(d: Dict<str, V>, key: str, why: str) { .. }
```

`#[doc(hidden)]` keeps a public item out of the API docs (`rokc doc`) and out of the language server's completion. It is for items that are `pub` only so code the compiler generates can call them; the item is still public and compiles and resolves as before. On a module it hides everything inside. It may stand beside the item's doc comment, which still serves hover. Any other `#[doc(..)]` form is an error: write `#[doc = ".."]` or `#[doc(hidden)]`.

## 6.2.9. `#[allow(...)]` / `#[deny(...)]` / `#[warn(...)]` lint control

```rok
#[allow(unused)]
fn _placeholder() { }

#[deny(deprecated)]
fn strict_zone() { ... }
```

These attributes change the level of lint diagnostics within the scope of the item they are attached to. Lints run in every command that compiles the package: `rokc build`, `run`, `test`, `bench`, `emit`, `check`, `lint` ([9.1.2](../009-tools/901-rokc-cli.md#rokc-lint-linter)) and the language server (not `doc`). They check the package being built, not its dependencies. A `warn` lint is printed and does not fail a build; a `deny` lint is an error, and the build fails with exit code 1. `allow` ignores the diagnostic, `warn` emits it as a warning, and `deny` emits it as an error. Inner forms such as `#![allow(..)]` apply to the enclosing module (the whole file at the file head). The argument is a lint name or a group name. An item's `#[allow]` also covers its variants, fields and methods.

The level is determined in the following order, later overriding earlier: each lint's default level < the `lints` field of `rok.rokon` (after its `lints_from` chain) ([9.10.3](../009-tools/910-rok-frontend.md#9103-rokrokon--roklock)) < CLI `-A` / `-W` / `-D` < attributes in source (the innermost wins). `lints` is treated as if prepended to the CLI flags, so CLI flags written after it override it. The code of lint diagnostics is `E503`.

The default level of every lint is `warn`. The Since column is the release that added the lint.

| lint                     | Since   | Checks                                                              |
|--------------------------|---------|---------------------------------------------------------------------|
| `unused_variables`       | 0.1.0   | Local variables or parameters that are never read. A `ref` parameter that is assigned, or captured by `ref` (`[ref n]`), is used: the write reaches the caller |
| `unused_imports`         | 0.1.0   | A `use` whose name the module never uses                            |
| `dead_code`              | 0.1.0   | Non-`pub` functions nobody calls                                    |
| `unreachable_code`       | 0.1.0   | Statements after `return` / `break` / `continue` / `throw` / `panic` |
| `unused_must_use`        | 0.1.0   | Discarding the result of a `#[must_use]` function, or a value of a `#[must_use]` type ([6.2.11](#6211-must_use-warning-on-ignored-results)) |
| `unused_doc_comments`    | 0.1.0   | A doc comment on what is not documented (a statement, match arm, struct literal or tuple field, expression); use `//` for a plain comment |
| `write_to_copy`          | 0.1.0   | A field assigned on a copy that nothing reads afterwards: a by-value parameter or `self`, a `for` binding, or a `let` of an element (`xs[i]`, [2.3.8](../002-types/203-struct.md#238-value-semantics)), when no later read, loop iteration, return or call can see the change. A read that only feeds another write to the same variable does not count; a variable a closure or coroutine block mentions is not checked |
| `ignored_error`          | 0.1.0   | A `try? f();` statement silently discarding an error; `let _ = try? f();` marks a deliberate best-effort call and is not reported |
| `shadowed_loop_var`      | 0.1.0   | A `let` in a `for` body rebinding the loop variable                 |
| `nan_comparison`         | 0.1.0   | Comparison with NaN (always false; `!=` always true)                |
| `none_comparison`        | 0.1.0   | `x == none` instead of `x is none`                                  |
| `bool_comparison`        | 0.1.0   | Comparing a `bool` with `true` / `false`                            |
| `self_assignment`        | 0.1.0   | `x = x;`                                                            |
| `unclosed_resource`      | 0.1.0   | A `File` / `Dir` / `ReadDir` / `TcpStream` / `TcpListener` / `UdpSocket` / `Child` not released by `using` (warn) |
| `while_true`             | 0.1.0   | `while true` instead of `loop`                                      |
| `deprecated`             | 0.1.0   | Use of a `#[deprecated]` function, method, field, type, constant or static ([6.2.7](#627-deprecated-discouraged-use-marker)) |
| `deprecated_safe`        | 0.1.6   | A call of a `#[future_unsafe]` function, or a use of such a static, outside `unsafe` ([6.2.14](#6214-future_unsafe-safe-today-unsafe-later)) |
| `redundant_cast`         | 0.1.0   | `x as T` where `x` is already `T`                                   |
| `redundant_clone`        | 0.1.0   | `.clone()` on a value of a primitive type                           |
| `non_snake_case`         | 0.1.0   | Function, local variable, parameter, or module names not in snake_case |
| `non_camel_case_types`   | 0.1.0   | Type, trait, or variant names not in UpperCamelCase: no lowercase first letter, no `__`, and `_` only between characters without case (`V1_0`) |
| `non_upper_case_globals` | 0.1.0   | `const` / `static` names not in UPPER_SNAKE_CASE                    |
| `confusable_idents`      | 0.1.0   | Identifiers that resemble each other ([1.4](../001-lexical/104-identifiers.md)) |
| `missing_f_prefix`       | 0.1.0   | A plain string literal containing `{name}` where `name` is a local in scope: the `f` prefix is missing (warn) |
| `literal_spread_mismatch` | 0.1.6   | A `List` or Dict literal spread into fixed parameters that cannot fill them: a length that differs, or a key that names no parameter left ([4.2](../004-functions/402-parameters.md)); the call panics at run time (warn) |

| Group               | Includes                                                                                     |
|---------------------|----------------------------------------------------------------------------------------------|
| `unused`            | `unused_variables`, `unused_imports`, `dead_code`, `unreachable_code`, `unused_must_use`, `unused_doc_comments` |
| `nonstandard_style` | `non_snake_case`, `non_camel_case_types`, `non_upper_case_globals`                           |
| `warnings`          | Every lint added up to the baseline release, 0.1.5                                           |
| `all`               | Every lint, including those later releases add                                               |

The help of `write_to_copy` suggests `ref self` / `ref p`, an index loop (`for i in 0..xs.len() { xs[i].f = .. }`), or writing the element back.

`warnings` is the group to deny for a build that must stay clean across releases: `"warnings": "deny"` in `lints`, `-D warnings` or `#![deny(warnings)]`. A lint added after the baseline is not in it, and a deprecation (`deprecated`) of an item that a release after the baseline deprecated keeps its default level under it, so a later release of the same series never turns code that built into an error. To check those too, name them, or deny `all`: denying `all` opts in to every lint later releases add. The baseline moves only with a release that may break code (0.2.0).

Some warnings come from the compiler itself rather than from a lint pass, and still belong to a lint: a confusable identifier is `confusable_idents` and a misplaced doc comment is `unused_doc_comments`. Their level is set by the lint's name, as for any other lint.

Each lint diagnostic says where its level came from: `` `#[warn(unused_variables)]` on by default ``, `` `-D unused` on the command line ``, `` `"unused_variables": "deny"` in `lints` of rok.rokon `` for a level from the manifest, or a label on the attribute that set it. An unknown lint or group name is a warning that names the closest known one and points to `rokc lint --list`.

`rokc lint --list` prints the lints with their default levels and the release that added each, then the groups.

## 6.2.10. `#[non_exhaustive]` opting out of external exhaustiveness

```rok
#[non_exhaustive]
pub enum ApiError {
    NotFound,
    Unauthorized,
}
```

`#[non_exhaustive]` may be attached to an `enum` or `struct`. When present, a `match` on the type from an external package requires an additional catch-all (`_ => ...`) even if it handles every variant. This lets library authors add new variants later without breaking SemVer compatibility.

## 6.2.11. `#[must_use]` warning on ignored results

```rok
#[must_use]
fn calculate() -> i32 { ... }
```

Discarding the return value of a `#[must_use]` function unused produces a warning. To ignore the result intentionally, write `_ = ...` as described in [parameters](../004-functions/402-parameters.md).

`#[must_use]` may also be attached to a `struct` or `enum`. Then an expression statement whose value has that type, or a local of that type that is never read, produces the `unused_must_use` warning with the attribute's message. The value of an `async fn` call or `async` block (a future never awaited never runs) and `thread::JoinHandle` ([stdlib 3.4.2](../../refs-standard-libraries/300-std/304-thread.md#342-api); a thread never joined may not finish before the program ends; `detach` says it runs on its own) are `#[must_use]`.

## 6.2.12. `#[track_caller]` passing the call site

```rok
#[track_caller]
fn expect_positive(n: i32) -> i32 {
    if n <= 0 { panic("not positive") }      // location: where expect_positive was called
    n
}
```

- The function receives the call site as a hidden trailing `str` argument. Panics inside it report that location ([4.14.13](../004-functions/414-throws.md#41413-panics-and-fatal-errors)).
- When a tracked function calls another tracked function, it passes on the location it received.
- A function value of a tracked function reports the location where the value was created.
- On an `async fn`, only the call is tracked, not the coroutine body.
- On an `extern` (runtime) declaration, the symbol takes an extra location argument ([8.5.7](../008-abi/805-extern-symbols.md#857-diagnostic-area)).
- It cannot be attached to `main`, `extern "C" fn`, or `#[export_name]` functions.

Standard library functions that panic on invalid input (`panic`, `unreachable`, `assert*`, indexing, slicing, `unwrap`, `List::insert` / `remove`, etc.) are `#[track_caller]`.

## 6.2.13. Summary

The attributes above are built-in attributes recognized directly by the compiler and tools; they need no user definition through meta blocks.

Magic attributes such as `#[lang]` and `#[literal]` are bound to the meaning of the language itself and are defined in separate documents.

### Other built-in attributes

The compiler also recognizes the following attributes. Each is specified where its feature is.

| Attribute                      | Placement                     | Meaning |
|--------------------------------|-------------------------------|---------|
| `#[path("file.rok")]`          | `mod name;`                   | Loads the module from that file instead of `name.rok` ([5.4.6](../005-modules/504-modules.md#546-custom-paths-path)) |
| `#![no_implicit_rok]`          | file (inner)                  | Turns off the implicit `use rok::core::**;` for that file ([stdlib 0.2.5](../../refs-standard-libraries/000-meta/002-import.md#025-opting-out---no_implicit_rok)) |
| `#[export_name("sym")]`        | function                      | Exports the function under the unmangled symbol `sym` (a wasm export of that name on wasm32); a root for unreachable-function elimination ([9.5.8](../009-tools/905-symbol-mangling.md)) |
| `#[no_mangle]`                 | function                      | Exempts the function from the naming and unused-function lints. It does not change the symbol; use `#[export_name]` for that |
| `#[wit::export]`               | function, `pub struct`        | Implements a WIT export: a function, or a resource ([stdlib 1.8](../../refs-standard-libraries/100-bare/108-wit.md)) |
| `#[niche(..)]`                 | newtype struct                | Declares values the type never holds, used as the niche of `T \| none` ([2.7.4](../002-types/207-niche-optimization.md#274-opt-in-niches-niche)) |
| `#[intrinsic("rok.OP.TYPE")]`  | standard library only         | Binds a function or method to a compiler operation; an error in user code ([stdlib 0.3](../../refs-standard-libraries/000-meta/003-intrinsics.md)) |
| `#[no_safepoint]`              | `extern "rok"` block          | Toolchain-internal: the block's functions neither allocate nor block, so calls to them are not safepoints; an error on other blocks ([5.9.2](../005-modules/509-extern-block.md#592-abi-specifiers)) |

## 6.2.14. `#[future_unsafe]` safe today, `unsafe` later

```rok
#[future_unsafe("`ptr` must point at a NUL-terminated string")]
pub fn from_ptr(ptr: Ptr<u8>) -> CStr { ... }
```

`#[future_unsafe("note")]` marks a library function, method, extern function or static that a later release makes `unsafe`. A call of it, or a use of the static, outside an `unsafe` block or `unsafe fn` warns with lint `deprecated_safe` and the note: wrap the call in `unsafe { .. }`, which compiles now and stays correct when the item becomes `unsafe`. Inside `unsafe` there is no warning.

It is for the standard library and library packages. On a function of the program being built it is an error: make that function `unsafe fn` instead. On an item other than a function, method, extern function or static it is an error.
