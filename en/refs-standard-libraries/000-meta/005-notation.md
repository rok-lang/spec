# 0.5. Notation

The notational conventions of this specification follow [refs-lang 0.8 Notation](../../refs-lang/000-meta/008-notation.md) as is. This chapter defines the additional conventions that apply only to the standard library specification.

## 0.5.1. Trait signature notation

Each trait spec begins with a signature block that gathers all methods, associated types and supertraits in one place.

```rok
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}
```

This form is identical to how the standard library is actually written, so users can use the same format in their own code.

## 0.5.2. Method signature tables

A trait defining several methods is also summarized in table form, as follows.

| Method | Signature                      | Meaning             |
|--------|--------------------------------|---------------------|
| `add`  | `fn(self, rhs: Rhs) -> Output` | Arithmetic addition |
| `sub`  | `fn(self, rhs: Rhs) -> Output` | Arithmetic subtraction |

## 0.5.3. Trait implementation matrix

At the end of each trait spec, a table states which standard library types implement the trait. The columns are only the type and the binding mechanism; implementation file paths are left outside the spec.

```markdown
## X.Y.Z. Trait implementation matrix

| Type  | binding                          | Notes                          |
|-------|----------------------------------|--------------------------------|
| `i32` | `#[intrinsic("rok.add.i32")]`    | Direct to compiler PrimOp      |
| `i64` | `#[intrinsic("rok.add.i64")]`    |                                |
| `f32` | `#[intrinsic("rok.add.f32")]`    |                                |
| `f64` | `#[intrinsic("rok.add.f64")]`    |                                |
| `int` | (no intrinsic, auto-promote)    | Defined by the standard library |
```

## 0.5.4. Semantics

The meaning of each method is written out in prose, covering not only form and examples but also limits and error cases.

```markdown
### `Add::add(self, rhs)`

Arithmetic addition. Returns the sum of the two values.

- `i32 + i32`: 2's complement wrap (debug=panic, release=wrap)
- `f64 + f64`: IEEE 754 addition
- `int + Int`: unbounded precision; promotes to BigInt when the result exceeds the i64 range

Overflow handling follows the fixed-width arithmetic rules of [2.1 Numeric types](../200-core/201-numbers.md).
Saturating / checked / wrapping variants are exposed as separate methods (`saturating_add`, etc.).
```

## 0.5.5. Examples

Each trait and method is accompanied by one or more working examples. Examples assume the implicit prelude import, `use rok::core::prelude::*;`, is in effect, so they must be code that runs as is without any imports.

```rok
let a: i32 = 5;
let b: i32 = 3;
let c = a + b;        //→ 8 (i32)

let x: f64 = 1.5;
let y: f64 = 2.5;
let z = x + y;        //→ 4.0 (f64)
```

## 0.5.6. Cross-reference

References to other documents use one of two forms depending on location. Within this specification, write `[2.7 iter](../200-core/207-iter.md)` or refer informally to "refs-standard-libraries 2.7"; references to refs-lang follow the form `[refs-lang 6.4](../../refs-lang/006-meta-system/604-lang-magic.md)`.

## 0.5.7. Separation of specification and implementation

This document defines only the **specification**. The actual implementations of the compiler and the standard libraries are maintained in separate repositories; in case of conflict with this specification, this specification takes precedence.

## 0.5.8. The `try_` prefix

A `try_` name marks one of two forms, and only these:

1. A non-blocking attempt: it does now what the plain form waits for, and reports that it could not (`Mutex::try_lock`, `Receiver::try_recv`, `Sender::try_send`).
2. A form that passes on the throw of the closure or future it takes (`Iterator::try_fold`, `Iterator::try_for_each`, `par::try_scope`, `task::try_block_on`).

A throwing form of a trait method that cannot throw keeps the `try_` name too: `TryFrom::try_from`, `TryInto::try_into`, `File::try_clone`. No other function takes the prefix: a function that fails throws under its plain name (0.5.9), so there is no `try_` twin of it. `try_` is a prefix only; no name ends in `_try`.

## 0.5.9. Error model

Each function reports failure through exactly one channel, chosen by what the failure is:

| Failure | Channel | Examples |
|---|---|---|
| An ordinary "no answer" with one cause | `-> T \| none` | `List::get`, `Iterator::find`, `Path::parent`, `Path::strip_prefix`, `checked_*`, `char::from_u32`, `NonZero::new` |
| Caused by the input, the environment or I/O, or with several causes the caller may tell apart | `throws (E)` | `FromStr::from_str`, `str::from_utf8`, `Int::parse_radix`, `FormatSpec::parse`, `ClientConfig::new`, the `fs` and `net` operations |
| A caller bug the caller can check cheaply first | panic, `#[track_caller]` | an operator index out of range, a radix outside 2..=36, a zero chunk size |

1. **No twins.** No two functions differ only in the failure channel. `try? f()` is the `T | none` form of every throwing call ([refs-lang 3.14.3](../../refs-lang/003-expressions/314-error-control-flow.md)), so the library has no `_or_none` twin of a throwing function and no panicking twin of one.
2. **No sentinels.** A result never encodes failure in a value of its own type (`0`, `-1`, `""`): it is `T | none`, or the function throws.
3. **Closing throws.** `close()` of an OS resource (a file, a socket, a TLS connection) throws what the OS reports. `Drop` never throws: a value released by `Drop` drops the error.
4. An operator and its named method may differ: `xs[i]` panics where `xs.get(i)` returns `none`.

The forms 0.2.0 removed under these rules: `FormatSpec::try_parse` (`FormatSpec::parse` throws), `Float::parse_or_none` and `Float::try_parse` (`try? s.parse()`), `task::try_id` (`task::current_id`), `task::block_on_try` (renamed `task::try_block_on`), `ClientConfig::try_new` (`ClientConfig::new` throws), the sockets' `try_close` (`close` throws), and `Path::strip_prefix_or_none` with `StripPrefixError` (`Path::strip_prefix` returns `Path | none`).

## 0.5.10. Error types

Every error type in the standard libraries follows these rules:

1. **Private fields.** An error is built by a constructor (`new(..)`, or a named one such as `IoError::from_os_code`), never by a struct literal, and read through getters named after the fact (`kind()`, `position()`, `path()`, `os_code()`). Every error type has `message() -> str`: what went wrong, without the context its `Display` may add.
2. **Trait set.** Every error type implements `Display`, `Debug`, `Error`, `PartialEq`, `Eq` and `Clone`. An error that carries a value of a type parameter (`SendError<T>`, `TrySendError<T>`) implements the first three.
3. **Kinds.** A kind getter returns an enum (`IoErrorKind`, `TlsErrorKind`, `JoinErrorKind`), never a `str` or an integer, and the enum is `#[non_exhaustive]`. An OS error code is `os_code() -> i32 | none`, `none` when the error did not come from the OS.
4. **Unit structs.** An error with no fields is a unit struct (`struct TimeoutError;`), written `TimeoutError`, not `TimeoutError {}`.
5. **One error per failure space.** core keeps the shared errors (`ValueError`, `ParseError`, `OverflowError`, `ConversionError`, `TypeError`, `IndexError`, `Utf8Error`); a std module with its own failure space has one error type (`IoError`, `TlsError`, `RokonError`). Every failed UTF-8 check throws `Utf8Error`.
6. **Messages.** A message starts in lower case (proper nouns and type names excepted), with no trailing period and no `error:` prefix. An operator's panic uses Rust's text (`index 5 out of bounds for length 3`). A method's panic starts `Type::method: ` (a free function's `module::fn: `), then the problem, then `; <what to write instead>` when there is one: `i32::isqrt: the number is negative; use `checked_isqrt` to get `none` instead`. A public function that panics on a bad argument is `#[track_caller]`.

The forms 0.2.0 removed under these rules: the public fields of every error type (and of `Ipv4Addr` / `Ipv6Addr`), `IoError::new(message)`, `IoError::with_kind`, `IoError::from_os` and `IoError::code` (`IoError::new(kind, message)`, `IoError::from_os_code`, `IoError::os_code`), `JOIN_LOST` / `JOIN_TAKEN` (`JoinError::kind`), `NonUtf8Error` (`Utf8Error`), and `process::signal(i32, handler)` (`process::on_signal(Signal, handler)`).
