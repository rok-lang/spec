# 5.9. `extern` blocks ([FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) declarations)

The `extern` keyword (a contextual keyword that is a keyword only at the start of an item, [1.5.4](../001-lexical/105-keywords.md#154-contextual-keywords)) combined with an [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) specifier (`"C"` etc.) defines a block of declarations for external functions and globals. It is used to interface with external libraries, system calls, and other languages.

```
extern_block    ::= "extern" string_lit "{" extern_item* "}"
extern_item     ::= visibility? "fn" identifier "(" param_list? ")" return_type? ";"
                  | visibility? "static" "mut"? identifier ":" type_expr ";"
```

## 5.9.1. Basic form

Basic usage:

```rok
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;
    fn free(p: Ptr<u8>);
    fn strlen(s: ConstPtr<c_char>) -> usize;

    static errno: i32;
    static mut atexit_count: u64;
}
```

A block consists of `extern`, an ABI string (`"C"`, `"system"`, etc.; the meaning of each ABI is defined in [Calling Convention](../008-abi/806-calling-convention.md)), and `{ ... }`. The body may contain only function declarations with a signature and no body, and `static` declarations.

> [!IMPORTANT]
> Function declarations inside an `extern "C"` block are **implicitly unsafe**. The caller may call them only inside an `unsafe { ... }` block; the effect is the same without writing `unsafe` in the signature. See [4.15 unsafe](../004-functions/415-unsafe.md) for the full definition.

## 5.9.2. ABI specifiers

The supported ABIs:

| ABI        | Meaning                                                                          |
|------------|----------------------------------------------------------------------------------|
| `"C"`      | The platform C ABI. Always the same regardless of symbol name.                   |
| `"system"` | The OS system-call ABI. Differs from `"C"` on some platforms (e.g. Win32 `stdcall`). |
| `"rok"`    | The ABI of the runtime (`rokr`). See below.                                      |

On targets where `"system"` and `"C"` coincide (every Tier 1 target but 32-bit x86 Windows), `"system"` is `"C"`. On 32-bit x86 Windows it is stdcall ([5.9.10](#5910-extern-fn-definitions-exposed-externally)).

`extern "rok"` is the ABI of runtime functions declared by the toolchain; those declarations live in `rok::core::rt` as `pub(pkg)`. `#[no_safepoint]` on an `extern "rok"` block declares that its functions neither allocate nor block; calls to them are not safepoints. It is toolchain-internal and an error on `extern "C"` blocks.

| Value                           | Passing under `extern "rok"`             |
|---------------------------------|------------------------------------------|
| `str`                           | two words `(ptr, len)`                   |
| `fn` value                      | `{code, env}` ([8.5.0a](../008-abi/805-extern-symbols.md#850a-rokfn---c-abi-form-of-function-values)) |
| `extern "C" fn(..)` parameter   | C function pointer without env           |
| Anything else                   | The value as is (same as the C ABI)      |

> [!NOTE]
> ABIs of other languages such as `"Rust"` are generally unstable and are not targets of `extern`. Interfacing with other languages is possible only over a stable ABI both sides agree on (usually `"C"`).

> [!NOTE]
> Rok propagates errors as return values and does not use stack unwinding ([8.6.5](../008-abi/806-calling-convention.md#865-return-of-throwing-functions-value-based-error-propagation)). Exceptions or unwinding from a foreign language crossing a Rok frame are therefore undefined behavior; foreign functions must handle exceptions within the call boundary.

A plain `extern { ... }` without an ABI is a compile error; the ABI must be specified.

## 5.9.3. Items permitted in the body

### External function declarations

An external function is declared as:

```rok
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;
}
```

The signature has the same form as an ordinary function declaration, but has no body and ends with `;`. Parameters and return types may use only FFI-compatible types such as primitive integers, floating-point, and pointers.

### External `static`

A global variable defined externally is referenced as:

```rok
extern "C" {
    static errno: i32;             // read-only
    static mut atexit_count: u64;  // mutable
}
```

This form references a global variable defined externally; `static mut` is allowed only inside `extern` blocks ([`static`](503-static.md)).

### Not permitted

`struct`, `enum`, `impl`, `fn` bodies, `const`, etc. do not appear in the body of an `extern` block. The block only declares the boundary with an external ABI.

## 5.9.4. FFI-compatible types

The types of parameters, returns, and `static`s must all be FFI-compatible. The safe candidates are:

- Primitive integers: `i8` to `i64`, `u8` to `u64`, `usize`, `isize`.
- Primitive floating-point: `f32`, `f64`.
- Primitive character type: `c_char` (1 byte).
- Pointers: `Ptr<T>`, `ConstPtr<T>` (standard library, the `unsafe` area of [standard library tiers](../000-meta/005-standard-libraries-tiers.md)).
- `#[repr(C)]` `struct` / `union`: user-defined types marked C ABI compatible.
- Function pointers: `extern "C" fn(...) -> ...` (see "C function pointer types" below).
- Function-typed parameters: an `fn(..) -> R` parameter of an `extern "C"` declaration is the same as `extern "C" fn(..) -> R`.
- `str`: converted by the rule below.

A `str` parameter in an `extern "C"` signature is a single NUL-terminated `char*`. If a NUL follows the bytes, the bytes are passed as is; otherwise (a slice, etc.) a copy pinned for the duration of the call is passed. C must not retain that pointer after the call returns. A `str` return is a `char*` whose length is measured with `strlen`. If it contains an interior NUL, the C side sees only up to the first NUL. If the returned `char*` is null the behavior is undefined, so a C string that may be null is received as `CPtr<c_char>` ([stdlib 1.7.2](../../refs-standard-libraries/100-bare/107-ffi.md#172-cstr--cstring---null-terminated-string)).

A `pub` function of an `extern` block in module `m` is also reachable as `m::f`; its symbol is unprefixed. A `rok_*` name in an `extern "C"` block may not take or return `str` ([8.5](../008-abi/805-extern-symbols.md)).

Writing an incompatible type produces a compile warning or error. The exact checking rules are defined in [Calling Convention](../008-abi/806-calling-convention.md).

### C function pointer types

```
c_fn_type ::= "unsafe"? "extern" ("\"C\"" | "\"system\"") "fn" "(" (type_expr ("," type_expr)*)? ")" ("->" unary_type)?
```

- The ABI string may only be `"C"` or `"system"`. Anything else is an error.
- `extern "system" fn(..) -> R` follows the same rules as `extern "C" fn(..) -> R`, so Win32 callbacks are expressible. It is a distinct type only on 32-bit x86 Windows, where it is stdcall; elsewhere the two are the same type.
- Values of this type are made only from named C-ABI functions. The value of a C declaration in an `extern` block has type `unsafe extern "C" fn(..)` and cannot be put in a safe (non-`unsafe`) position; a Rok `extern "C" fn` definition (its C entry point, 5.9.10) yields a safe `extern "C" fn(..)` value.
- Rok `fn` values (including closures and variables of function type) and C function pointers do not convert to each other.
- Calling through a type without `unsafe` is safe; calling through `unsafe extern "C" fn` requires `unsafe`.
- A non-`none` C function pointer value is not null. Receiving NULL into an `extern "C" fn` position without `| none` (parameter, return, field) is therefore undefined behavior, and a C API that may return NULL must be declared with `| none`.

#### Nullable C function pointers: `extern "C" fn(..) -> R | none`

A sum consisting only of one C function pointer type and `none` (in any order, no other members) uses the null niche ([2.7.2](../002-types/207-niche-optimization.md#272-automatic-niches)).

- It is pointer-sized, and `none` is null.
- It has the same ABI as a C nullable function pointer. It can be used as is in extern parameters / returns, struct fields exchanged with C, and extern `static`s.
- Zero-filled storage reads as `none`.
- A larger sum containing it uses the ordinary tagged layout, and conversion between them is ordinary sum type conversion.

```rok
extern "C" {
    fn cmp_i32(a: ConstPtr<u8>, b: ConstPtr<u8>) -> i32;
    fn qsort(base: Ptr<u8>, n: usize, size: usize, cmp: unsafe extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32);
}
let f: unsafe extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32 = cmp_i32;   // a C declaration's value is unsafe
let g: extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32 = cmp_i32;          // ERROR: safe position
```

### Name space of extern functions

Functions declared in `extern` blocks form a single global name space and can be called by unprefixed name from any module. This mirrors C symbols, which live in a single global space at link time. It is an exception to the "no implicit search of ancestor modules" rule for ordinary items ([5.7.1](507-paths.md#571-starting-identifier-of-a-path)) and applies only to extern functions. Declaring an extern function of the same name twice with different signatures is a compile error. The toolchain's runtime declarations in `rok::core::rt` count: a user `extern` that redeclares a `rok_*` runtime function must match its signature.

## 5.9.5. `Ptr<T>` / `ConstPtr<T>`

Raw pointer types defined in the `unsafe` tier of the [standard library tiers](../000-meta/005-standard-libraries-tiers.md).

```rok
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;
    fn free(p: Ptr<u8>);
    fn read_byte(p: ConstPtr<u8>) -> u8;
}
```

`Ptr<T>` is a mutable raw pointer corresponding to C's `T*`, and `ConstPtr<T>` is a read-only raw pointer corresponding to C's `const T*`. Both are nullable, and `null` is handled as a niche automatically ([niche optimization](../002-types/207-niche-optimization.md)).

## 5.9.6. Calls / use

Functions declared in an `extern` block are called like ordinary Rok functions.

```rok
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;
    fn free(p: Ptr<u8>);
}

fn use_malloc() {
    let p = unsafe { malloc(1024usize) };
    if !p.is_null() {
        // ... use ...
        unsafe { free(p) };
    }
}
```

User-declared `extern "C"` functions are unsafe and are called inside `unsafe { ... }` ([4.15](../004-functions/415-unsafe.md)). Runtime functions declared by the toolchain (`rok_print_*`, `rok_panic_str`, etc.) can be called safely. Otherwise an `extern` function call differs from an ordinary call only in that it is an external symbol. Rok's ordinary semantics apply to the result type. An `extern static` can likewise be read and (if mut) written like an ordinary variable.

## 5.9.7. Visibility

Each item of an `extern` block may carry a visibility keyword.

```rok
extern "C" {
    pub fn public_api(x: i32) -> i32;       // public
        fn _internal(x: i32) -> i32;        // module-private
    pub static VERSION: i32;
}
```

The `extern "C"` block itself takes no visibility keyword; visibility is given per item.

## 5.9.8. Position of the `extern` block itself

An `extern "C" { ... }` block may appear only at module level. It cannot be placed inside a function or an `impl`.

```rok
fn f() {
    extern "C" { ... }                     // ERROR: not inside a function
}
```

## 5.9.9. `#[link("libname")]`: naming the library

To name the external library, use the `#[link]` attribute.

```rok
#[link("m")]
extern "C" {
    fn sin(x: f64) -> f64;
    fn cos(x: f64) -> f64;
    fn sqrt(x: f64) -> f64;
}
```

`#[link("x")]` or `#[link(name = "x", kind = "static" | "dylib")]` on an extern block passes `-lx` to the linker. The library search path is the `-L <dir>` directories, then `LIBRARY_PATH`. On Apple targets, `kind = "static"` links the `lib<name>.a` found there by its path. rokc's `-l` and a native package's `links` add libraries the same way. On wasm32, `#[link(wasm_import_module = "m")]` imports the block's functions from core module `m`, and `#[link_name = "sym"]` on a function sets its linked symbol or import name ([6.2.3](../006-meta-system/602-built-in-attributes.md#623-link-naming-an-external-library)). Without `#[link]`, symbols are resolved at link time through system defaults (`libc`, etc.) or other mechanisms. The behavior of `#[link]` is defined in [built-in attributes](../006-meta-system/602-built-in-attributes.md).

## 5.9.10. `extern fn` definitions (exposed externally)

The forms in the preceding sections call external functions from rok. The reverse form exposes a Rok function through an external ABI.

```rok
#[export_name("rok_compute")]
pub fn compute(x: i32, y: i32) -> i32 {
    x + y
}

extern "C" fn callback(arg: i32) -> i32 {
    arg * 2
}
```

An `extern "ABI"` qualifier on an ordinary function declaration marks the function to be compiled compatible with that ABI. An `extern "C" fn` definition keeps its mangled symbol name ([9.5](../009-tools/905-symbol-mangling.md)); if a name visible from C is needed, add `#[export_name("...")]` as well.

An `extern "ABI" fn` definition takes `"C"` or `"system"`; any other ABI string is an error. On 32-bit x86 Windows an `extern "system"` declaration or definition is stdcall: the callee pops its arguments, and the symbol is `_name@N` as with MSVC.

`extern "C"` and `extern "system"` fn definitions have these restrictions:

- They cannot be generic, `async`, or `throws`.
- Parameter and return types must be FFI-compatible types from 5.9.4; non-FFI types such as `str` cannot be used. (The `str` conversion of 5.9.4 applies only to declarations that *call* C functions.)

```rok
#[export_name("on_event")]
extern "C" fn on_event(code: i32) -> i32 { code }     // visible from C as on_event
extern "C" fn bad(s: str) { }                          // ERROR: str cannot be used
```

Calling external functions and exposing functions externally together form the two directions of FFI.

## 5.9.11. Calling rules of `extern fn`

`extern "C"` functions follow the C calling convention and may differ in part from the ordinary function call rules covered in [execution engines and user surface](../000-meta/001-execution-modes.md). The differences in calling convention are covered in the runtime ABI specification (chapters 7 / 8).

## 5.9.12. Responsibility for FFI safety

Every item of an `extern` block is the user's responsibility (consistent with the `unsafe` spirit of the [standard library tiers](../000-meta/005-standard-libraries-tiers.md)). The compiler verifies:

- The signature form.
- ABI-compatible types (as far as possible).

The compiler does not verify:

- The actual behavior of external functions (null returns, error returns, memory leaks, etc.).
- Concurrency safety of external `static`s.
- True ABI compatibility of external libraries.

The user must guarantee these.
