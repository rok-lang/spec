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

- `i32 + i32`: panics on overflow (checks on in every profile; wraps under `-C overflow-checks=no`)
- `f64 + f64`: IEEE 754 addition
- `Int + Int`: arbitrary precision; never overflows

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

## 0.5.10. Error types

Every error type in the standard libraries follows these rules:

1. **Private fields.** An error is built by a constructor (`new(..)`, or a named one such as `IoError::from_os_code`), never by a struct literal, and read through getters named after the fact (`kind()`, `position()`, `path()`, `os_code()`). Every error type has `message() -> str`: what went wrong, without the context its `Display` may add.
2. **Trait set.** Every error type implements `Display`, `Debug`, `Error`, `PartialEq`, `Eq` and `Clone`. An error that carries a value of a type parameter (`SendError<T>`, `TrySendError<T>`) implements the first three.
3. **Kinds.** A kind getter returns an enum (`IoErrorKind`, `TlsErrorKind`, `JoinErrorKind`), never a `str` or an integer, and the enum is `#[non_exhaustive]`. An OS error code is `os_code() -> i32 | none`, `none` when the error did not come from the OS.
4. **Unit structs.** An error with no fields is a unit struct (`struct TimeoutError;`), written `TimeoutError`, not `TimeoutError {}`.
5. **One error per failure space.** core keeps the shared errors (`ValueError`, `ParseError`, `OverflowError`, `ConversionError`, `TypeError`, `Utf8Error`); a std module with its own failure space has one error type (`IoError`, `TlsError`, `RokonError`). Every failed UTF-8 check throws `Utf8Error`.
6. **Messages.** A message starts in lower case (proper nouns and type names excepted), with no trailing period and no `error:` prefix. An operator's panic uses Rust's text (`index 5 out of bounds for length 3`). A method's panic starts `Type::method: ` (a free function's `module::fn: `), then the problem, then `; <what to write instead>` when there is one: `i32::isqrt: the number is negative; use `checked_isqrt` to get `none` instead`. A public function that panics on a bad argument is `#[track_caller]`.

## 0.5.11. Naming

Every name in the standard library follows this table. A name that breaks it is a defect of the library, not a precedent.

| Kind | Rule |
|---|---|
| constructors | `new` (required arguments), `with_<x>` (an extra argument), `from_<src>` (a conversion, `Formatter::from_sink`), `empty()` / `zeroed()` for special values; never `to_<x>` |
| conversions | `as_` a free view, `to_` a new owned value (costly), `into_` consumes; `str` is the string type, so `to_str`, `read_to_str`, `from_str`, never `to_string` |
| numeric conversions | lossless and checked ones go through `From` / `TryFrom` / `SaturatingFrom` / `WrappingFrom`; an inherent `to_<num>` only for a lossy rounding (`to_f64`) |
| getters | no `get_` prefix; `get` only for a keyed or indexed lookup, or the value of a wrapper (`NonZero::get`) |
| predicates | `is_` / `has_`, or Rust's established word (`success`, `readonly`) |
| setters | `set_<x>(ref self, ..)` returns unit; a builder uses the bare field name and returns `Self` |
| sizes | `len` = element or byte count of a container; `<x>_len` = a length property; `count` = something counted (`count_ones`); `size` = memory size |
| add / remove | `push` / `pop` at the ends, `insert` / `remove` keyed or indexed, `extend` many, `clear`, `retain`, `take` moves out |
| iteration | `iter()` returns `<Type>Iter`; any other iterator method returns its own name in CamelCase, in the defining module (below) |
| reflection | the plain form is the type's answer (`T::type_name()`, `T::size()`), the `_of_val` form the value's answer at run time (`x.type_name_of_val()`, `x.size_of_val()`, `x.field_names_of_val()`) |
| async | `<name>_async` is the twin of a blocking `<name>`; never an `async_` prefix |
| abbreviations | spelled out, except the fixed set: `len`, `fd`, `io`, `utf8`, `ns` / `us` / `ms` / `secs` / `mins`, `tls`, `der` / `pem`, `mmap`, `pid` |
| constants | `SCREAMING_SNAKE`; no function twin of a constant (`f64::INFINITY`) |
| modules | a singular noun or area (`fs`, `net`, `iter`); `collections` and `ops` stay plural |

Iterator types. `str`: `chars()` -> `Chars`, `bytes()` -> `Bytes`, `char_indices()` -> `CharIndices`, `lines()` -> `Lines`, `split()` -> `Split`, `splitn()` -> `SplitN`, `rsplit()` -> `RSplit`, `rsplitn()` -> `RSplitN`, `split_whitespace()` -> `SplitWhitespace`, `match_indices()` -> `MatchIndices`. `Dict`: `keys()` -> `Keys`, `values()` -> `Values`. `List` / `Slice`: `windows()` -> `slice::Windows`, `chunks()` -> `slice::Chunks` (beside the adapters `iter::Windows` / `iter::Chunks`). `Receiver::iter()` -> `ReceiverIter`. A leaf name that two modules share is written with its module (`slice::Windows`, `str::Bytes`).

## 0.5.12. Trait coverage, `#[non_exhaustive]` and `#[sealed]`

Every public type of the standard library implements the traits of its class:

| Type class | Implements |
|---|---|
| every public type | `Debug` |
| value types (addresses, paths, ids, times, kinds, algorithms, ranges, specs) | `Clone`, `PartialEq`; `Eq` when equality is total; `Hash` when `Eq`; `PartialOrd` / `Ord` when the values are ordered; `Copy` when small |
| types with an obvious empty or zero value | `Default` |
| text-like types | `Display` |

`Debug` output by kind:

- a value prints its contents: `0..3`, `0..=3`, `Start(3)`, `SocketAddrV4 { ip: 127.0.0.1, port: 80 }`, `Permissions { mode: 0o644, readonly: false }`; a `StrBuf` prints as its text, quoted;
- a handle prints its type and what names it: `File { path: "a.txt" }`, `Child { pid: 42 }`, `Thread { id: ThreadId(3), name: "worker" }`; a handle with nothing cheap to show prints its type name (`TcpStream`, `Mutex`);
- an iterator, stream or future prints its type name (`Map`, `Chars`, `AsyncFuture`);
- a value holding a secret never prints it: `PrivateKey { .. }`, `Hmac { .. }`, `RsaPrivateKey { bits: 2048, .. }`.

Value traits added in 0.2.0 (used as `Dict` keys and `Set` elements, compared and ordered):

| Type | Traits |
|---|---|
| `Path` | `Clone`, `Eq`, `Hash`, `PartialOrd`, `Ord` (by the text), `Default` (empty) |
| `Ipv4Addr`, `Ipv6Addr` | `Clone`, `Copy`, `Eq`, `Hash`, `PartialOrd`, `Ord` |
| `SocketAddr`, `SocketAddrV4`, `SocketAddrV6` | `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash`, `Debug` |
| `Ordering` | `Clone`, `Copy`, `Hash`, `PartialOrd`, `Ord` (`Less < Equal < Greater`) |
| `IoErrorKind` | `Clone`, `Copy`, `Eq`, `Hash`, `Display` (`not found`) |
| `TlsErrorKind`, `CipherSuite`, `AeadAlgorithm`, `EcCurve`, `HashAlgorithm` | `Hash` |
| `Signal`, `FileType`, `JoinErrorKind` | `Clone`, `Copy`, `Eq`, `Hash` |
| `Component`, `RawStr`, `RawPath` | `Clone`, `Eq`, `Hash` |
| `SeekFrom`, `Shutdown`, `Alignment`, `Sign`, `LoadOrdering`, `StoreOrdering`, `UpdateOrdering` | `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash`, `Debug` |
| `ThreadId` | `Clone`, `Copy`, `Eq`, `Hash`, `Debug` |
| `Permissions`, `Layout`, `ExitStatus` | `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash` |
| `CStr`, `CString`, `Certificate` (by its DER bytes), `FormatSpec` | `Clone`, `PartialEq`, `Eq`, `Hash` |
| `Range` | `Debug`, `Hash` |
| `RangeInclusive`, `RangeFrom`, `RangeTo`, `RangeToInclusive`, `RangeFull` | `Clone`, `PartialEq`, `Eq`, `Hash`, `Debug` |
| `Duration` (`ZERO`), `StrBuf` (empty), `FormatSpec` (`FormatSpec::new()`) | `Default` |

`#[non_exhaustive]` ([refs-lang 6.2.10](../../refs-lang/006-meta-system/602-built-in-attributes.md)) marks every public enum that can gain variants and every options record that can gain fields. A `match` on one outside its package needs a `_` arm, and a struct pattern needs `..`:

- enums: `IoErrorKind`, `TlsErrorKind`, `CipherSuite`, `AeadAlgorithm`, `EcCurve`, `HashAlgorithm`, `rokon::Value`, `Stdio`, `FileType`, `Signal`, `JoinErrorKind`;
- records: `tls::ClientConfig`, `tls::ServerConfig`, `FieldInfo`, `MethodInfo`.

Closed enums stay exhaustive: `Ordering`, `Poll`, `Result`, `SeekFrom`, `Shutdown`, `Alignment`, `Sign`, `SocketAddr`, `Component`, `LoadOrdering`, `StoreOrdering`, `UpdateOrdering`, `SendError`, `TrySendError`.

`#[sealed]` ([refs-lang 6.2](../../refs-lang/006-meta-system/602-built-in-attributes.md)) marks the traits whose implementations the library relies on for soundness or meaning; only the defining package implements them, and bounds and calls are unaffected: `AtomicPrimitive` (2.16), `NicheInt`, `NicheFloat` (2.2), `Pinnable` (1.6). `Step` (2.7) stays open for user range types. `Pod` (1.3) is not sealed (`#[derive(Pod)]` checks a user struct), and `bool` and `char` are not `Pod`.

## 0.5.13. Async twins

`x_async` is the async twin of the blocking `x`: the same arguments and the same errors, awaited (`try await s.x_async(..)`) and suspending only the task. A twin takes `ref self` where `x` works on its receiver in place, and `self` where `x` consumes it (`close_async`).

Stream operations get their twins from traits, not from methods of each type: a type that implements `Read`, `BufRead`, `Write`, `Seek` or `Close` gets `read_async`, `fill_buf_async`, `write_async`, `flush_async`, `seek_async` or `close_async` by implementing `AsyncRead`, `AsyncBufRead`, `AsyncWrite`, `AsyncSeek` or `AsyncClose` ([2.23.5](../200-core/223-io.md#2235-async-traits)). Only operations that are not stream I/O are inherent `_async` methods: opening and connecting (`open_async`, `connect_async`), `accept_async`, `shutdown_async`, datagram and fd passing calls, TLS control and process waits (`wait_async`, `output_async`, `status_async`).

| Type | Async traits | Inherent twins |
|---|---|---|
| `File` | `AsyncRead`, `AsyncWrite`, `AsyncSeek`, `AsyncClose` | `open_async` and the other open functions, `sync_all_async`, `set_len_async`, `metadata_async`, `lock_*_async` |
| `TcpStream` | `AsyncRead`, `AsyncWrite`, `AsyncClose` | `connect_async`, `shutdown_async` |
| `TcpListener` | `AsyncClose` | `accept_async` |
| `UdpSocket` | `AsyncClose` | `connect_async`, `send_async`, `recv_async`, `send_to_async`, `recv_from_async` |
| `Socket` | `AsyncRead`, `AsyncWrite`, `AsyncClose` | `connect_async`, `accept_async`, `shutdown_async`, `send_to_async`, `recv_from_async` |
| `UnixStream` | `AsyncRead`, `AsyncWrite`, `AsyncClose` | `connect_async`, `shutdown_async`, `send_with_fd_async`, `recv_with_fd_async` |
| `UnixListener` | `AsyncClose` | `accept_async` |
| `UnixDatagram` | `AsyncClose` | `connect_async`, `send_async`, `recv_async`, `send_with_fd_async`, `recv_with_fd_async` |
| `tls::Client`, `tls::Server` | `AsyncRead`, `AsyncWrite`, `AsyncClose` | `new_async`, `with_config_async`, `close_notify_async`, `key_update_async` |
| `ChildStdin` | `AsyncWrite`, `AsyncClose` | |
| `ChildStdout`, `ChildStderr` | `AsyncRead`, `AsyncClose` | |
| `Child` | | `wait_async` |
| `Command` | | `output_async`, `status_async` |
| `BufReader`, `BufWriter`, `LineWriter`, `Cursor` | as their inner stream ([2.23.6](../200-core/223-io.md#2236-buffers-and-cursor)) | |

A `Sender`'s `close` is not I/O and has no twin.

## 0.5.14. Docs and attributes

Every public item of the standard libraries has a `///` doc comment: each function, type, trait, trait method and inherent method, `const fn` and `async fn` ones and those behind a `cfg` included. A doc says what the item does and what it returns; a function that throws, returns `none` or panics says when, and an `unsafe` function says what its caller must uphold. The library's own check (`rokc check --check-library`) denies `missing_docs` ([refs-lang 6.2.9](../../refs-lang/006-meta-system/602-built-in-attributes.md)).

A function whose result has an order states it:

| Item | Order |
|---|---|
| `Dict`, `Set` and their iterators | insertion order |
| `Set::union`, `Set::symmetric_difference` | the members from `self` in its order, then those only in `other` in its order |
| `Set::intersection`, `Set::difference` | `self`'s order |
| `fs::walk` | depth first in pre-order (a directory before its entries), each directory's entries sorted by name |
| `net::resolve` | the system resolver's order, its preferred address first |
| `env::vars`, `env::vars_os` | the order the OS keeps the environment in; not sorted |
| `task::race` | the first task to finish wins; tasks found finished at the same wake-up go to the lowest index; the others are cancelled |
| `task::select` | polled in list order; a tie goes to the lower index |

`#[must_use]` ([refs-lang 6.2.11](../../refs-lang/006-meta-system/602-built-in-attributes.md#6211-unused-results-unused_result-discardable-must_use)) marks these, each type with a message that says how to use the value:

| Kind | Items |
|---|---|
| Iterators | those of `str` (`Chars`, `CharIndices`, `Split`, `Lines`, ...), `List` (`ListIter`), `Slice` (`SliceIter`, `Windows`, `Chunks`), `SliceMut` (`SliceMutIter`), `Dict` (`Keys`, `Values`, `DictIter`), `Set` (`SetIter`), `Deque` (`DequeIter`), `BinaryHeap` (`HeapIter`) and `Receiver` (`ReceiverIter`); the `iter` adapters and the `TryIterator` adapters (`TryMap`, `MapErr`, `Results`, ...); `GenIter`, `TryGenIter`; `env::Vars`, `env::VarsOs`; `fs::ReadDir`, `fs::Walk`; `io::Lines`, `io::Split`, `io::Bytes`; `net::Incoming`; `path::Components`, `path::Ancestors` |
| Streams | `AsyncStream`, `TryAsyncStream`; the `stream` adapters and combinators and the `TryStream` adapters; `io::BytesStream`, `io::LinesStream`; `fs::ReadDirStream` |
| Futures | the value of an `async fn` call or `async` block; `LockFuture`, `ReadFuture`, `WriteFuture`, `CondvarWait`, `RecvFuture`, `SendFuture`, `task::Join`, `SelectFuture` |
| Builders and results | `process::Command`, `thread::Builder`, `thread::JoinHandle`, `Result` |
| Functions | the value methods of `str` and `Path`; every `Duration` function and `Instant`'s arithmetic; `Set::union`, `intersection`, `difference`, `symmetric_difference`; every method of the integer and float primitives, `Int` and `Float` |

A `Task`, what `spawn` and `spawn_blocking` return, is not `#[must_use]`: the work runs whether or not it is awaited. A guard needs no mark: dropping a `MutexGuard` in a statement is an error. The `#[discardable]` functions are those refs-lang 6.2.11 lists.
