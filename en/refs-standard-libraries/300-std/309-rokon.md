# 3.9. `rokon::` - the ROKON data format

ROKON (Rok Object Notation) is a data format that writes values in Rok's literal syntax. The file extension is `.rokon`. It fills the same role as Rust's RON and Zig's ZON; the package manifest `rok.rokon` and the lockfile `rok.lock` ([refs-lang 9.10.3](../../refs-lang/009-tools/910-rok-frontend.md#9103-rokrokon--roklock)) use this format.

## 3.9.1. Format

ROKON is a subset of Rok expression syntax. The library reads it with its own lexer and recursive-descent parser over that subset; the compiler's parser is not used at run time. One file holds one value.

| Form        | Example                                               | Notes                                                 |
|-------------|-------------------------------------------------------|-------------------------------------------------------|
| none        | `none`                                                |                                                       |
| bool        | `true`, `false`                                       |                                                       |
| Integer     | `42`, `-7`, `0xff`, `0b1010`, `0o17`, `1_000_000`      | No suffix                                             |
| Float       | `3.14`, `-1.5e-3`, `2E10`                              | No suffix                                             |
| char        | `'a'`, `'\n'`                                         | `c'..'` not allowed                                   |
| String      | `"text"`, raw strings, multi-line strings             | f-strings not allowed                                 |
| List        | `[a, b, c]`                                           |                                                       |
| dict        | `{ "k": v, 1: w }`                                    | Keys are values (strings, numbers, etc.)              |
| Tuple       | `(a, b)`, `(a,)`                                      | `(x)` is grouping, not a tuple                        |
| struct      | `Point { x: 1, y: 2 }`, `{ x: 1, y: 2 }`, `Pair(1, 2)`, `Unit {}` | The name is optional. Field names are identifiers. Tuple structs are `Name(a, b)` |
| enum variant | `Shape::Unit`, `Shape::Circle(1.0, at: p)`, `Shape::Rect { w: 2, h: 3 }` | Written with the full path            |

- A trailing comma is allowed. Comments are `//` and `/* */`.
- `{}` is both an empty struct and an empty dict (the target type decides). A single `{ .. }` may contain either fields (identifier keys) or keys (value keys), not both; mixing them is an error.
- Not allowed: bare identifiers, operators (only unary `-` before a number is allowed), method calls, `..`, blocks, f-strings, Set literals (`{1, 2}`; write a Set as a list), `c'..'` literals, generic paths (`List<i32>::..`), and tokens left after the value. All are errors showing `line:col`.
- NaN and infinity have no ROKON notation. `nan` / `inf` in the input are rejected as bare identifiers.

> [!NOTE]
> Known limitation: a dict whose first key is `none` (`{ none: 1 }`) is not supported, because the reader takes `none:` as a field name and reads the braces as a struct.

```rokon
// config.rokon
Config {
    name: "geom",
    retries: 3,
    ratio: 0.75,
    tags: ["a", "b"],
    limits: { "cpu": 2, "mem": 512 },
    mode: Mode::Fast,
    proxy: none,
}
```

## 3.9.2. Correspondence with Rok types

| ROKON          | Rok type                                                                     |
|----------------|------------------------------------------------------------------------------|
| `none`         | `none` of `T \| none`                                                        |
| `true` / `false` | `bool`                                                                     |
| Integer        | Every integer type, `int`. Also accepted where a float is expected (`f32` / `f64` / `float`) |
| Float          | `f32`, `f64`, `float`                                                         |
| char           | `char`                                                                        |
| String         | `str`                                                                         |
| List           | `List<T>`, `[T; N]`, `Set<T>`, `OrderedSet<T>`                                |
| dict           | `Dict<K, V>`, `OrderedDict<K, V>`                                             |
| Tuple          | Tuples of 1 ~ 6 elements                                                      |
| struct         | `struct` (`#[derive(Rokon)]`)                                                 |
| enum variant   | `enum` (`#[derive(Rokon)]`)                                                   |
| Any value      | `Value`                                                                       |

## 3.9.3. Reading

`parse` reads text into a `Value`.

| Input                        | `Value`                                                        |
|------------------------------|----------------------------------------------------------------|
| `{ ident: v, .. }`           | Unnamed `Struct`                                               |
| `{ "k": v, .. }`, `{}`       | `Dict`                                                         |
| `Name { f: v }`              | Named `Struct`                                                 |
| `A::B { f: v }`              | `Variant` (values in `named`)                                  |
| `A::B(a, k: b)`, `A::B`      | `Variant` (`args`, `named`)                                    |
| `Name(a, b)`                 | `Variant` with a single-segment path (tuple struct)            |
| Integer                      | `Int` (exact `int`)                                            |

In typed reads (`from_str`, `decode`), the target type determines the interpretation.

- A struct target accepts `Struct`, `{}` and `A::B { .. }`, so struct names with a module path (`geom::Point { .. }`) are accepted too.
- A written name must match the target. For structs, the last path segment is compared; for enum variants, the last two segments.
- An unknown field is an error at the position of that field name. `#[rokon(deny_unknown_fields)]` on the type says so explicitly and keeps it when 0.2.0 makes reads skip unknown fields by default.
- A missing field takes the field's `= default` if it has one, `none` if its type is `T | none`, and is an error otherwise (`FromRokon::from_missing`).
- An integer value outside the range of the target integer type is an error.
- Named variants can also be read as `E::V(a, k: b)`. Positional arguments fill the fields in order, followed by named arguments.

## 3.9.4. Canonical form

The canonical form emitted by `rokon::to_string` and by `rok` (when editing the manifest or writing `rok.lock`) is as follows. The canonical form is also the readable form, so there is no separate pretty function.

- Indentation is 4 spaces.
- A value is written on one line if it fits within 100 columns; otherwise on multiple lines with a comma after each element.
- Floats use the shortest notation that reads back to the same value (`1.0`, `1.0e300`). Attempting to write NaN / infinity panics ("NaN/infinity has no ROKON form").
- A variant with only named values has one canonical form, `E::V { .. }` (`Value::Variant` keeps no form flag).
- Bidi controls (U+202A-U+202E, U+2066-U+2069) in strings and chars are written escaped (`\u{202e}`).
- Reading the canonical form back yields the same value (round-trip). The toolchain's reader (`rok.rokon`, `rok.lock`, `rokc fmt` on a `.rokon` file) rejects float literals out of the `f64` range (`1e999`), naming the range: ``float `1e999` is out of range (ROKON floats are f64, up to about 1.8e308)``. `rokon::parse` reads such a literal into `Value::Float`, whose `float` has no such limit.

`rokc fmt` checks that a `.rokon` file is ROKON and only re-indents it. It does not rewrite it into canonical form, so comments and layout are preserved.

## 3.9.5. Library: `rok::std::rokon`

```rok
use rok::std::rokon;

pub enum Value {
    None,
    Bool(bool),
    Int(int),
    Float(float),
    Char(char),
    Str(str),
    List(List<Value>),
    Dict(OrderedDict<Value, Value>),
    Tuple(List<Value>),
    Struct { name: str | none; fields: OrderedDict<str, Value>; },
    Variant { path: str; args: List<Value>; named: OrderedDict<str, Value>; },
}
impl Value {
    fn to_string(self) -> str;       // canonical form
    fn kind(self) -> str;            // kind name for error messages (below)
}                                    // + Display, Debug, PartialEq, Eq, Hash

pub struct RokonError {
    #[deprecated(since = "0.1.6", note = "use `RokonError::message()`")]
    pub msg: str;
    #[deprecated(since = "0.1.6", note = "use `RokonError::line()`")]
    pub line: u32;
    #[deprecated(since = "0.1.6", note = "use `RokonError::col()`")]
    pub col: u32;
    #[deprecated(since = "0.1.6", note = "use `RokonError::path()`")]
    pub path: str;
}
impl RokonError {
    pub fn message(self) -> str;     // without the position Display adds
    pub fn line(self) -> u32;        // 1-based; 0 when the value did not come from text
    pub fn col(self) -> u32;         // 1-based, in code points; 0 as for line
    pub fn path(self) -> str;        // ".servers[2].port"; empty at the root
}

pub trait ToRokon { fn to_rokon(self) -> Value; }
pub trait FromRokon {
    fn from_rokon(v: Value) throws (RokonError) -> Self;
    fn from_missing() throws (RokonError) -> Self;   // default: error. none for T | none
}

pub fn parse(s: str) throws (RokonError) -> Value;
pub fn to_string<T: ToRokon>(v: T) -> str;                        // panics on a NaN / infinite float
pub fn try_to_string<T: ToRokon>(v: T) throws (RokonError) -> str;  // throws there instead, with the path
pub fn from_str<T: FromRokon>(s: str) throws (RokonError) -> T;
pub fn encode<T: ToRokon>(x: T) -> Value;
pub fn decode<T: FromRokon>(v: Value, seg: str) throws (RokonError) -> T;
```

- `Display` of `RokonError` is `line:col: msg` if there is a position, otherwise `at <path>: msg`. Errors from typed reads carry paths such as `.servers[1].port` or `.limits["cpu"]`, and `from_str` converts that path to the `line:col` of the input. The `seg` of `decode` is the first segment of the path.
- `Value::kind` is the kind name used in error messages: in order, `"none"`, `"bool"`, `"integer"`, `"float"`, `"char"`, `"string"`, `"list"`, `"dict"`, `"tuple"`, `"struct"`, and for `Variant` ``"variant `<path>`"``.
- `==` and `hash` on `Value` are structural. Dict entries and struct fields are compared regardless of order, and floating-point values are compared by identity (NaN == NaN, -0.0 != 0.0). An unnamed empty struct and an empty dict are equal.
- `Debug`, `==` and `hash` on `Value` never panic.
- Standard implementations: every integer type, `f32` / `f64`, `int` / `float`, `bool` / `char` / `str`, `Value`, `T | none`, `List`, `Set`, `Dict`, `OrderedDict`, `OrderedSet`, tuples of 1 ~ 6 elements, `[T; N]`.

The module also exports `Fields`, `struct_args`, `variant_of`, `variant_args`, `unknown_variant`, `RokonError::new` / `within` and the submodules `read` / `write`. They support the code `@derive` generates for `#[derive(Rokon)]` and are not a stable API: all but `RokonError::new` / `within` are `#[doc(hidden)]`, and 0.2.0 makes them private to the standard library.

### `#[derive(Rokon)]`

Implements both `ToRokon` and `FromRokon` for structs and enums (including generic ones).

| Definition           | Written form           |
|----------------------|------------------------|
| Struct with named fields | `Name { f: v }`    |
| Tuple struct         | `Name(a, b)`           |
| Unit struct          | `Name {}`              |
| Unit variant         | `E::V`                 |
| Tuple variant        | `E::V(a)`              |
| Named variant        | `E::V { f: v }`        |

A field's `= default` is used for missing fields (3.9.3). The derive is built with the meta tokens `@Self.name`, `@Self.is_unit` / `is_tuple` / `is_named`, `@field.has_default` / `@field.default` ([refs-lang 6.9](../../refs-lang/006-meta-system/609-template-tokens.md)).

```rok
#[derive(Rokon)]
struct Server { host: str; port: u16 = 80; tls: bool | none; }

#[derive(Rokon)]
struct Config { name: str; servers: List<Server>; limits: OrderedDict<str, i32>; }

let cfg: Config = try rokon::from_str(text);   // errors are "line:col: msg"
print(rokon::to_string(cfg));
```

### Limitations

- `Dict` / `Set` are written in hash order. To preserve order, use `OrderedDict` / `OrderedSet`.
- `[T; 0]` cannot be read.
- Tuples are limited to 6 elements.
- A dict whose first key is `none` cannot be read (3.9.1).

## 3.9.6. Comparison with other formats

| Format | struct notation             | Notes                             |
|--------|-----------------------------|-----------------------------------|
| RON    | `Name(field: v)`            | A dedicated syntax different from Rust |
| ZON    | `.{ .name = "x" }`          | Zig's anonymous struct literal    |
| ROKON  | `Name { field: v }`         | Rok's own literal syntax as is    |

ROKON adds no separate syntax and uses Rok literals directly, so a value written in source and a value written in a file look the same.
