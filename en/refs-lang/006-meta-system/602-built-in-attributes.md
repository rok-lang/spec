# 6.2. Built-in attributes

This section defines the built-in attributes the compiler processes directly. `#[cfg]` is covered separately in [conditional compilation](603-cfg.md), and magic attributes and meta blocks are described in the sections from [`#[lang(...)]` magic](604-lang-magic.md) onward.

```
derive_attr     ::= "#" "[" "derive" "(" trait_name ("," trait_name)* ","? ")" "]"
trait_name      ::= identifier
repr_attr       ::= "#" "[" "repr" "(" repr_arg ("," repr_arg)* ")" "]"
repr_arg        ::= "C" | "transparent" | "packed" | "align" "(" int_lit ")"
                  | identifier                          // primitive int repr
inline_attr     ::= "#" "[" "inline" ("(" ("always" | "never") ")")? "]"
deprecated_attr ::= "#" "[" "deprecated" ("(" string_lit ")")? "]"
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
| `transparent`               | Exactly one non-zero-size field. Same representation as that field, and crosses the C ABI as it      |
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

`assert_eq` is in [stdlib 2.14.5](../../refs-standard-libraries/200-core/214-error.md#assert--assert_eq--assert_ne). `#[test]` on a function marks it as a test function. The test runner (`rokc test`, etc.) collects and runs such functions automatically. In ordinary builds they are excluded, as if `#[cfg(test)]` were also attached. A test function is run whatever its own visibility and that of its enclosing modules; `#[test]` does not change visibility for other code.

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
```

Using an item marked `#[deprecated]` produces a warning, not an error. A message given with `= "..."` is included in the warning.

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

## 6.2.9. `#[allow(...)]` / `#[deny(...)]` / `#[warn(...)]` lint control

```rok
#[allow(unused)]
fn _placeholder() { }

#[deny(deprecated)]
fn strict_zone() { ... }
```

These attributes change the diagnostic level of the linter (`rokc lint`, [9.1.2](../009-tools/901-rokc-cli.md#rokc-lint-linter)) within the scope of the item they are attached to. `allow` ignores the diagnostic, `warn` emits it as a warning, and `deny` emits it as an error. Inner forms such as `#![allow(..)]` apply to the enclosing module (the whole file at the file head). The argument is a lint name or a group name.

The level is determined in the following order, later overriding earlier: each lint's default level < the `lints` field of `rok.rokon` (after its `lints_from` chain) ([9.10.3](../009-tools/910-rok-frontend.md#9103-rokrokon--roklock)) < CLI `-A` / `-W` / `-D` < attributes in source (the innermost wins). `lints` is treated as if prepended to the CLI flags, so CLI flags written after it override it. The code of lint diagnostics is `E503`.

The default level of every lint is `warn`.

| lint                     | Checks                                                              |
|--------------------------|---------------------------------------------------------------------|
| `unused_variables`       | Local variables or parameters that are never read                   |
| `unused_imports`         | A `use` whose name the module never uses                            |
| `dead_code`              | Non-`pub` functions nobody calls                                    |
| `unreachable_code`       | Statements after `return` / `break` / `continue` / `throw` / `panic` |
| `unused_must_use`        | Discarding the result of a `#[must_use]` function                   |
| `ignored_error`          | A `try? f();` statement silently discarding an error                |
| `shadowed_loop_var`      | A `let` in a `for` body rebinding the loop variable                 |
| `nan_comparison`         | Comparison with NaN (always false; `!=` always true)                |
| `none_comparison`        | `x == none` instead of `x is none`                                  |
| `bool_comparison`        | Comparing a `bool` with `true` / `false`                            |
| `self_assignment`        | `x = x;`                                                            |
| `unclosed_resource`      | A `File` / `Dir` / `ReadDir` / `TcpStream` / `TcpListener` / `UdpSocket` / `Child` not released by `using` (warn) |
| `while_true`             | `while true` instead of `loop`                                      |
| `deprecated`             | Use of a `#[deprecated]` function                                   |
| `redundant_cast`         | `x as T` where `x` is already `T`                                   |
| `redundant_clone`        | `.clone()` on a value of a primitive type                           |
| `non_snake_case`         | Function, local variable, parameter, or module names not in snake_case |
| `non_camel_case_types`   | Type, trait, or variant names not in UpperCamelCase                 |
| `non_upper_case_globals` | `const` / `static` names not in UPPER_SNAKE_CASE                    |
| `confusable_idents`      | Identifiers that resemble each other ([1.4](../001-lexical/104-identifiers.md)) |

| Group               | Includes                                                                                     |
|---------------------|----------------------------------------------------------------------------------------------|
| `unused`            | `unused_variables`, `unused_imports`, `dead_code`, `unreachable_code`, `unused_must_use`     |
| `nonstandard_style` | `non_snake_case`, `non_camel_case_types`, `non_upper_case_globals`                           |
| `all`               | Every lint                                                                                   |

`rokc lint --list` prints the lints, groups, and default levels.

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
