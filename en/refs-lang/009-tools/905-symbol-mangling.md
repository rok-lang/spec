# 9.5. Symbol mangling scheme

As a result of [specialize](../000-meta/007-glossary.md#specialize) in the [AOT build pipeline](902-aot-pipeline.md), one function splits into a separate function per concrete type. This section defines the rules that give these functions unique symbol names.

## 9.5.1. Form

The overall grammar of a symbol is:

```
mangled_symbol  ::= "_Y" pkg_hash "_" mod_path "_" item_name "_" type_hash
pkg_hash        ::= [0-9a-f]{8}            // package SHA-256 prefix
mod_path        ::= identifier ("_" identifier)*
item_name       ::= identifier
type_hash       ::= [0-9a-f]{16}           // generic args SHA-256 prefix
                  | ""                       // concrete (non-generic) item
```

Written out on one line:

```
_Y <pkg_hash_8> _ <path> _ <name> _ <typehash_16>
```

A real example:

```
_Y3f8a1c2b_std_vec_push_a7c91d8e44b2f190
   |       |       |    |
   |       |       |    +-- typehash (16 chars, hash of generic args)
   |       |       +-- name (push)
   |       +-- path (std_vec)
   +-- pkg hash (8 chars)
```

## 9.5.2. Meaning of each part

### `_Y`: rok symbol prefix

Every rok symbol begins with `_Y`. Names beginning with underscore + uppercase letter are reserved for the implementation in C, so they do not collide with ordinary C symbols; `_Y` was chosen so as not to overlap with widely used mangling schemes either (C++ Itanium's `_Z`, Rust v0's `_R`, D's `_D`, Swift's `$s`). It also makes symbols emitted by rok easy to spot in `nm` or linker error output. The prefix is immediately followed by the lowercase hexadecimal `<pkg_hash_8>`, so it is distinguished by the first character alone from the reserved prefixes followed by an uppercase letter (`_YK` etc., [9.5.5](#955-reserved-prefixes)).

### `<pkg_hash_8>`

The package hash of [Source units](../000-meta/003-source-units.md): the first 8 characters of a content-based SHA-256 hash.

```
3f8a1c2b
```

A package with the same content yields the same hash wherever it is built. Consequently, same-named functions in different packages never collide.

### `<path>`: module path

The module path defined in [Paths](../005-modules/507-paths.md), with `::` replaced by `_`. Standard library paths drop the `rok::` root segment (`rok::std::vec` → `std_vec`), and user paths drop `pkg::`.

```
rok::std::vec      → std_vec
pkg::geom   → geom               (pkg:: is already identified by the package hash)
foo::inner::sub → foo_inner_sub
```

### `<name>`: item name

An identifier as defined in [Identifiers](../001-lexical/104-identifiers.md). Each segment of `<path>` is escaped by the same rule.

Since `_` is the separator, characters in an identifier other than `[A-Za-z0-9]` are escaped as follows.

| Source | Encoding |
|---|---|
| `[A-Za-z0-9]` | unchanged |
| `_` | `_0` |
| Any other character (including Unicode) | `_1<lowercase hex code point>_` |

Identifiers do not begin with a digit, so `0` / `1` never follows a separator `_`. Escapes and separators are therefore always distinguishable.

```
mod a { fn b }   → _Y<pkg>_a_b_
fn a__b          → _Y<pkg>__a_0_0b_
fn α             → _Y<pkg>___13b1__
```

### Method names

A method is mangled with `<type>.<method>` as its name and the module where the type is defined as its path. If several traits give the same type a method of the same name, `__by_<trait>` is appended. Methods of dependency packages ([9.2.3](902-aot-pipeline.md#923-dependency-resolution)) are also mangled by this stable path `<module>::<type>.<method>[__by_<trait>]`, so separately built packages and the program produce the same symbol. `.` and `_` are escaped per the table above.

### Nested functions

A nested function ([4.10](../004-functions/410-nested-functions.md)) is mangled in the module of its enclosing item with `<outer>/<name>` as its name, where `<outer>` is the enclosing function's name (`<type>.<method>` for a method, `<trait>.<method>` for a trait default body, recursively `<outer>/<name>` for a nested one). Two nested functions with the same `<outer>/<name>` in one source file get `/2`, `/3`, … in source order. `/` occurs in no other item name, so the symbol is distinct from a module path and from a method. `/` is escaped per the table (`_12f_`).

```
mod a { fn outer() { fn helper() } }   → _Y<pkg>_a_outer_12f_helper_
```

### `<typehash_16>`: type tuple hash

The concrete type tuple of the [generics](../002-types/210-generics.md) is hashed with SHA-256 and the first 16 characters are used.

```
fn id<T>(x: T) -> T

call id(42i32)   → typehash = SHA-256("(i32)")[0..16]
call id(3.14f64) → typehash = SHA-256("(f64)")[0..16]
```

### typehash of a concrete `fn`

For a function that takes no generic arguments, the typehash is left empty or omitted entirely.

```
_Y3f8a1c2b_geom_compute_main           (no typehash)
or
_Y3f8a1c2b_geom_compute_main_           (empty typehash)
```

The exact notation is chosen by `rokc`. This specification only requires that the absence of generic arguments be unambiguously identifiable in the typehash.

## 9.5.3. Comparison: Rust `_ZN`

Rust's Itanium-style mangling looks like this.

```
_ZN3std3vec10push_inner17h2d4c3e9d8f1a2b3cE
```

- Every component carries a length prefix (`3std3vec`).
- An [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) identifier hash is included (17h...).
- It ends with `E`.

It has the advantage of Itanium compatibility, but is hard to read and length prefixes are inefficient to encode.

By contrast, rok's `N...` is easy to read and has no length prefixes (components are separated by `_`). Demangling stays simple too.

## 9.5.4. demangle tool

To decode a symbol into a human-readable form:

```
rokc demangle <symbol>
```

The output looks like this:

```
$ rokc demangle _Y3f8a1c2b_std_vec_push_a7c91d8e44b2f190
package:  3f8a1c2b
path:     rok::std::vec
name:     push
generics: a7c91d8e44b2f190 (type tuple hash)
```

## 9.5.5. Reserved prefixes

The prefixes in use and reserved are:

| Prefix   | Meaning                                                                        |
|----------|--------------------------------------------------------------------------------|
| `_Y...`  | Ordinary rok symbol                                                            |
| `rok_*`  | `extern` "C" runtime functions ([Extern "C" symbols](../008-abi/805-extern-symbols.md)) |
| `_YK...` | (reserved) keyword handlers                                                    |
| `_YL...` | (reserved) literal handlers                                                    |
| `_YI...` | (reserved) infix / prefix / postfix magic                                      |
| `_YM...` | (reserved) generated code of meta blocks (impl_derive etc.)                    |

`_YK` / `_YL` / `_YI` / `_YM` only reserve the namespace; all ordinary symbols are emitted with the `_Y` prefix.

## 9.5.6. Symbol length

Most rok symbols have the following length distribution.

```
_Y + 8 + _ + path (tens) + _ + name (a few to tens) + _ + 16
≈ 30 ~ 80 characters
```

A very long path combined with deep [generics](../000-meta/007-glossary.md#generic) may exceed 100 characters. Debuggers and tools handle such lengths without trouble (2KB or less is generally recommended).

## 9.5.7. Collision avoidance: typehash stability

The type tuple of the generic arguments is serialized deterministically and then hashed with SHA-256.

```
type tuple → canonical serialization → SHA-256 → first 16 characters
```

The same types yield the same hash however many times the build is redone. The same specialization in a different build also gets the same symbol name. Names rokc gives to closures and other generated functions derive from the item they are written in (for an instance, also its type arguments), so they too are the same in every build.

This stability is what makes separate builds, incremental builds, and symbol matching between dependency packages possible.

### Canonical serialization rules

Serialization of a type tuple is deterministic.

- Primitive types are written as short identifiers (`i32` / `f64` etc.).
- User types take the form `{pkg_hash}**{path}**{name}` plus their generic arguments.
- Sum types have their members order-normalized and then joined (following the set semantics of [Sum types (`A | B`)](../002-types/206-sum-types.md)).
- A `struct` consists of its field types (field names are irrelevant).

The exact normalization algorithm is chosen by `rokc`. This specification only requires that it be deterministic.

## 9.5.8. `extern` "C" functions: no mangling

External functions declared in an [`extern` block](../005-modules/509-extern-block.md) use the standard naming convention of that ABI unchanged. rok mangling is not applied.

```rok
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;     // symbol: "malloc" (as is)
}
```

In the other direction, `#[export_name("name")]` specifies the name directly when exposing a rok function externally (see [`extern` block ([FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) declarations)](../005-modules/509-extern-block.md)). Such items are roots for unreachable-function elimination and are always emitted ([9.2.8](902-aot-pipeline.md#928-specialize-monomorphization)).

```rok
#[export_name("rok_compute")]
pub fn compute(x: i32, y: i32) -> i32 { x + y }
// symbol: "rok_compute" (not mangled)
```

These two mechanisms provide integration between rok and C.

## 9.5.9. Symbol of the main function

The entry-point function of [Source units](../000-meta/003-source-units.md) uses the platform's ordinary main symbol unchanged.

```
main                                (Linux / macOS)
mainCRTStartup                      (Windows console)
WinMainCRTStartup + WinMain         (Windows GUI, -C subsystem=windows)
_start                              (no-OS)
__main_void                         (wasm32-wasi*; called by wasi-libc's _start)
_initialize, then main              (wasm32-unknown-unknown; reactor exports called by the host)
```

On `wasm32-unknown-unknown` the module is a reactor: the host calls `_initialize`, then the export `main`, and the runtime stays up afterwards (no shutdown). `#[export_name("x")]` functions are wasm exports named `x`. On every wasm32 target the runtime starts on the first of `__main_void`, `main` or an exported call.

On Windows, rok's own startup object provides `mainCRTStartup` (console) or `WinMainCRTStartup` and `WinMain` (GUI), which call the rokc-generated `main` wrapper.

The platform entry point calls the user-defined main function inside a wrapper generated by rokc. The wrapper calls `rok_runtime_init` and the type registration function `rok.register_types` ([8.2.13](../008-abi/802-scan-table.md#8213-registration-time)) in order, then calls the entry function, determines the exit code by the rules of [4.16.4](../004-functions/416-entry-point.md#4164-exit-codes-and-unhandled-exceptions), and returns via `rok_runtime_shutdown`.

When `main.rok` consists of top-level statements ([4.16.2](../004-functions/416-entry-point.md#4162-meaning-of-top-level-statements-implicit-main)), the implicit `main` synthesized by the compiler is the entry function. The implicit `main` is mangled by the same rules as an ordinary function but cannot be referenced by name from user code; only the wrapper calls it.

## 9.5.10. Debug information

Each mangled symbol carries its original name in the DWARF and PDB debug information.

```
debugger output.
  fn distance<f64> at geom.rok:42
    (mangled: _Y3f8a1c2b_geom_distance_a7c91d8e44b2f190)
```

This lets tools such as gdb, lldb and VS Code display rok functions by their original names.

## 9.5.11. Stability

The symbol mangling scheme is also part of the ABI. Changes must therefore follow the stability policy in the [Overview](../008-abi/README.md).

- Compatibility is maintained within the same major version.
- Compatibility breaks across major versions, and the ABI hash changes as well.
