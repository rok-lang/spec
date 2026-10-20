# 2.15. Strings — `str` / `char` / `c_char`

The `rok::core::str` module provides the authoritative definitions of the string and character types. rok follows a **UTF-8 first** policy: every in-process string is handled as UTF-8, and `c_char` is used only at FFI boundaries.

This chapter defines three of the 48 lang slots. First, `str_type` is bound to `rok::core::str::str`, a lowercase struct that is a language built-in. Second, `char_type` is bound to `rok::core::str::Char`, a standard library addition carrying `#[keyword("char")]`. Third, `cchar_type` is bound to `rok::core::str::c_char`, likewise a language built-in.

## 2.15.1. String model — immutable `str` and mutable `StrBuf`

`str` is an immutable UTF-8 string value; the mutable buffer for building strings is the separate type `StrBuf`.

| Type     | Meaning                                            | Memory representation        | Mutability          |
|----------|----------------------------------------------------|------------------------------|---------------------|
| `str`    | UTF-8 string value                                 | `(ptr, len)` value — bytes are an immortal constant or a GC object | **immutable** |
| `StrBuf` | growable UTF-8 buffer                              | GC heap object               | mutable             |
| `char`   | Unicode scalar value (PascalCase struct + keyword) | 4B (u32)                     | (value — cannot change) |
| `c_char` | C ABI 1-byte char (FFI primitive)                  | 1B (i8 or u8 — target-dependent) | (value)         |

### Rust / Python / rok model comparison

How the three languages handle strings:

| Aspect         | Rust             | Python             | rok                        |
|----------------|------------------|--------------------|----------------------------|
| Value          | `&str`           | `str`              | `str` (immutable `(ptr, len)`) |
| Mutable buffer | `String`         | `io.StringIO`, etc. | `StrBuf`                  |
| Memory         | borrow checker   | refcount           | GC                         |

rok's `str` is close to Python's immutable `str` in needing no lifetime tracking, and is represented like Rust's `&str` as a `(ptr, len)` pair. The bytes of strings created at runtime live in a GC object (type id `1`, [refs-lang 8.2.12](../../refs-lang/008-abi/802-scan-table.md#8212-runtime-reserved-type-ids)).

## 2.15.2. `str` — immutable UTF-8 string

```rok
#[lang("str_type")]
struct str {
    ptr: ConstPtr<u8>;
    len: usize;            // byte length (NOT char count)
}
```

`str` is a language built-in recognized directly by the compiler, so it needs no `#[keyword]`. The lowercase struct name is the same as the type notation in user code.

`str` is a `(ptr, len)` pair passed by value, and the bytes it points to never change. Every operation that modifies a string returns a new `str`.

### Core methods

```rok
impl str {
    fn from_utf8(bytes: List<u8> | Slice<u8>) throws (Utf8Error) -> str;

    fn len(self) -> usize;                  // byte length
    fn is_empty(self) -> bool;
    fn as_bytes(self) -> Slice<u8>;         // read-only view without copying
    fn byte(self, i: usize) -> u8;          // O(1), panics when out of range
    fn as_ptr(self) -> ConstPtr<u8>;        // FFI

    // reading methods
    fn chars(self) -> CharsIter;            // Unicode scalar iterator
    fn bytes(self) -> BytesIter;            // u8 iterator
    fn char_indices(self) -> CharIndicesIter;
    fn lines(self) -> LinesIter;            // strips a trailing \r, does not yield a final empty line
    fn split(self, sep: str) -> SplitIter;  // keeps empty pieces ("a,,b" → "a", "", "b")
    fn split_whitespace(self) -> SplitWhitespaceIter;   // splits on Unicode White_Space (same set as trim)

    fn starts_with(self, pat: str) -> bool;
    fn ends_with(self, pat: str) -> bool;
    fn contains(self, pat: str) -> bool;
    fn find(self, pat: str) -> usize | none;
    fn rfind(self, pat: str) -> usize | none;

    fn strip_prefix(self, prefix: str) -> str | none;   // the rest after removing the prefix, if present
    fn strip_suffix(self, suffix: str) -> str | none;
    fn split_once(self, sep: str) -> (str, str) | none; // before / after the first sep
    fn repeat(self, n: usize) -> str;

    fn trim(self) -> str;                   // strips Unicode White_Space
    fn trim_start(self) -> str;
    fn trim_end(self) -> str;

    fn to_lowercase(self) -> str;           // new str (full Unicode mapping — 1:N, including context rules)
    fn to_uppercase(self) -> str;
    fn replace(self, from: str, to: str) -> str;   // empty from matches at every char boundary: "ab".replace("", "-") == "-a-b-"

    fn parse<T: FromStr>(self) throws (T::Err) -> T;

    // UTF-8-safe slicing — validates byte boundaries
    // (the s[a:b] desugaring dispatches to this method)
    fn slice(self, range: Range<usize>) -> str;        // same as s[a:b], shares bytes
}
```

`as_ptr` returns a pointer that stays valid while the `str` is held in a live local. A string reachable only through another object may be moved by the GC; it is not pinned ([stdlib 1.6](../100-bare/106-gc-pin.md)).

### `StrBuf` — mutable buffer

Use `StrBuf` to build a string piece by piece.

```rok
struct StrBuf { /* GC heap — ptr / len / capacity */ }

impl StrBuf {
    fn new() -> StrBuf;
    fn push(self, c: char);                 // appends one char (UTF-8 encoded)
    unsafe fn push_byte(self, b: u8);       // caller keeps the contents UTF-8; the safe byte path is str::from_utf8
    fn push_str(self, s: str);
    fn capacity(self) -> usize;
    fn to_str(self) -> str;                 // immutable str of the current contents
}
```

Like a collection, `StrBuf` is a reference handle, so its methods take `self` ([2.9](209-collections.md)).

```rok
let buf = StrBuf::new();
for x in xs {
    buf.push_str(f"{x},");
}
let s: str = buf.to_str();
```

### String literals — `"hello"`

```rok
let s: str = "hello";        // a str pointing to immortal constant bytes
```

A `"hello"` literal lowers to a `str` value pointing to constant bytes in the immortal region. Because `str` is immutable, there is no copying or copy-on-write.

### Trait implementations

| Trait                       | impl                           | Behavior                                   |
|-----------------------------|--------------------------------|--------------------------------------------|
| `Display`                   | ✓                              | self as is                                 |
| `Debug`                     | ✓                              | `"..."` with escapes                       |
| `Clone`                     | ✓                              | same value (immutable, so bytes are shared) |
| `Copy`                      | ✓                              | copies the `(ptr, len)` value              |
| `PartialEq`                 | ✓                              | content comparison (byte-wise)             |
| `Eq`                        | ✓                              |                                            |
| `PartialOrd`                | ✓                              | lexicographic by byte                      |
| `Ord`                       | ✓                              |                                            |
| `Hash`                      | ✓                              |                                            |
| `IntoIterator`              | **none**                       | explicit `.chars()` — avoids byte vs char ambiguity |
| `Add<str>` (`s + t`)        | ✓ → str (concat — new str)     |                                            |
| `AddAssign<str>` (`s += t`) | ✓ → `s = s + t` (replaced by a new str) |                                   |
| `FromIterator<char>`        | ✓                              | `(chars).collect::<str>()`                 |
| `FromIterator<str>`         | ✓                              | `(strs).collect::<str>()`                  |

> [!NOTE]
> `str` is an immutable value, so `let t = s;` only copies `(ptr, len)`, and neither side can change the contents.

## 2.15.3. `s[a:b]` slicing — UTF-8 safe

```rok
let s: str = "hello, 세계!";
s.len();                       //→ 14  (byte length — "hello, " 7 + Korean 3 bytes × 2 + "!" 1)
s.chars().count();             //→ 10  (char count)

let s2 = s[7:13];              //→ "세계"  (byte boundary OK)
let s3 = s[8:9];               // panic — byte in the middle of a UTF-8 sequence
let s4 = s[7:100];             //→ "세계!" (end clamped to the length)
let s5 = s[-1:];               //→ "!"
```

The bounds of `str` / `StrBuf` are byte offsets. Bounds are first clamped as in Python (negative counts from the end, [refs-lang 3.6.2](../../refs-lang/003-expressions/306-indexing-and-slicing.md#362-slicing-arrstartend)); if the resulting position is in the middle of a UTF-8 sequence, it panics. The result is a `str` sharing the original bytes (below). `str` slices have no step; `s[::2]` is a type error.

### Direct indexing rejected

```rok
let s: str = "hello";
let c = s[0];                  // ERROR — `Index<usize>` is not defined
                               //         reason: byte vs char ambiguity + UTF-8 safety
```

The intent must be stated explicitly, as in `s.chars().next()` or `s.byte(0)`.

### Substrings are not copied

The results of `s[a:b]`, `lines`, `split*`, `split_whitespace`, `split_once`, `trim*` and `strip_*` are `str` values sharing the original bytes, in O(1) ([refs-lang 8.8.1](../../refs-lang/008-abi/808-lowering-types.md#881-primitive-integers--floating-point)). A small piece therefore keeps a large original alive. There is no API to detach it. `as_bytes()` is likewise a read-only `Slice<u8>` view without copying.

## 2.15.4. `char` — Unicode scalar value (standard library addition)

```rok
#[keyword("char")]
#[lang("char_type")]
struct Char {
    value: u32;        // Unicode code point — excluding surrogates
}
```

`char` is a **standard library addition** type registered with `#[keyword("char")]`, so user code writes it as lowercase `char`. The struct itself is PascalCase `Char`.

The Unicode scalar value range is the union of 0..=0xD7FF and 0xE000..=0x10FFFF. The surrogate range 0xD800..=0xDFFF is used as an **automatic niche**, so `char | none` can be encoded in 4B.

### Literals / methods

```rok
let c: char = 'A';                   //→ U+0041
let k: char = '한';                  //→ U+D55C
let e: char = '😀';                  //→ U+1F600

c.to_lowercase();                    //→ 'a'
c.is_alphabetic();                   //→ true
c.is_ascii();                        //→ true
try '1'.to_digit(10);                //→ 1 (u32, ValueError on failure)
c.len_utf8();                        //→ 1 (number of bytes when UTF-8 encoded)
e.len_utf8();                        //→ 4
```

```rok
impl char {
    fn from_u32(n: u32) -> char | none;                  // none for a surrogate / out of range
    unsafe fn from_u32_unchecked(n: u32) -> char;        // no check — the caller guarantees a scalar value
    fn from_digit(d: u32, radix: u32) -> char;           // panics if d >= radix or radix is outside 2..=36
    fn to_digit(self, radix: u32) throws (ValueError) -> u32;
    fn is_digit(self, radix: u32) -> bool;
    fn len_utf8(self) -> usize;
    fn len_utf16(self) -> usize;

    // Unicode properties
    fn is_alphabetic(self) -> bool;
    fn is_lowercase(self) -> bool;
    fn is_uppercase(self) -> bool;
    fn is_whitespace(self) -> bool;
    fn is_numeric(self) -> bool;
    fn is_alphanumeric(self) -> bool;
    fn is_control(self) -> bool;
    fn to_lowercase(self) -> char;       // 1:1 mappings only
    fn to_uppercase(self) -> char;

    // ASCII
    fn is_ascii(self) -> bool;
    fn is_ascii_uppercase(self) -> bool;     fn is_ascii_lowercase(self) -> bool;
    fn is_ascii_alphabetic(self) -> bool;    fn is_ascii_digit(self) -> bool;
    fn is_ascii_alphanumeric(self) -> bool;  fn is_ascii_hexdigit(self) -> bool;
    fn is_ascii_whitespace(self) -> bool;    fn is_ascii_control(self) -> bool;
    fn is_ascii_graphic(self) -> bool;       fn is_ascii_punctuation(self) -> bool;
    fn to_ascii_uppercase(self) -> char;     fn to_ascii_lowercase(self) -> char;
    fn eq_ignore_ascii_case(self, other: char) -> bool;
}
```

`char`'s `to_lowercase` / `to_uppercase` return a single character, so only 1:1 mappings apply. A character that maps to several characters is left unchanged (`'ß'.to_uppercase() == 'ß'`). `str`'s `to_lowercase` / `to_uppercase` apply the full mapping, including 1:N mappings and context rules (Greek final sigma `Σ` → `ς`) (`"ß".to_uppercase() == "SS"`).

The Unicode property and case tables are UCD 18.0.0.

### Conversion

```rok
char::from_u32(0x41);                //→ 'A' (char | none — none for a surrogate)
char::from_digit(5, 10);             //→ '5'

let n: u32 = c as u32;               //→ 65 (lossless)
let c2: char = n as char;            //→ ERROR — only u8 can `as char`
let b: u8 = 0x41;
let c4: char = b as char;            //→ 'A' (a u8 is always a scalar value)
let c3 = try char::try_from(n);      //→ char (TryFrom<u32> — via from_u32, OverflowError if none)
```

### Trait implementations

`char` implements every cmp / fmt trait as well as `Copy`, `Clone` and `Hash`. The niche applies automatically, so `char | none` is 4B.

## 2.15.5. `c_char` — C ABI 1-byte (language built-in)

```rok
#[lang("cchar_type")]
struct c_char {
    value: i8 or u8;     // target-dependent (signedness differs by OS / arch)
}
```

`c_char` is an FFI primitive that is a language built-in, a lowercase struct recognized directly by the compiler.

It is ABI-compatible with C's `char`; note that it is a raw byte, not a UTF-8 encoding unit. Its use is FFI only:

```rok
extern "C" {
    fn strlen(s: ConstPtr<c_char>) -> usize;
    fn strcmp(a: ConstPtr<c_char>, b: ConstPtr<c_char>) -> i32;
}
```

### Literals

```rok
let c: c_char = c'A';        //→ 65 (i8/u8 — target-dependent)
let n: c_char = c'\n';       //→ 10
```

The prefixed form `c'...'` separates it from `char`.

### char vs c_char

Comparing the two types makes the difference clear.

| Aspect       | `char`                              | `c_char`                      |
|--------------|-------------------------------------|-------------------------------|
| Size         | 4 bytes                             | 1 byte                        |
| Meaning      | Unicode scalar (code point)         | C ABI byte                    |
| Range        | 0..=0x10FFFF (excluding surrogates) | 0..=255 or -128..=127         |
| Multibyte    | a single char covers both BMP / SMP | a multibyte char = byte sequence |
| Literal      | `'A'`                               | `c'A'`                        |
| Registration | standard library addition (`#[keyword]`) | language built-in         |
| Used for     | general string processing           | FFI / byte-level processing   |

Standard library functions therefore use **`char` first** as a rule, and `c_char` only at explicit FFI boundaries.

## 2.15.6. UTF-8 invariant — every in-process string

A `str` is always guaranteed to be valid UTF-8, for these reasons. Constructors validate: `str::from_utf8(bytes)` throws `Utf8Error` on an invalid byte sequence (`throws (Utf8Error)`). Slicing validates byte boundaries and rejects positions in the middle of a multibyte sequence. In-place modifications such as `push` or `insert` also preserve UTF-8 integrity. Only `rok::bare::str_from_bytes_unchecked` can bypass validation, and it belongs to the unsafe tier.

### Why validation bypass is restricted

Consider what goes wrong if the UTF-8 invariant **breaks**: `chars()` may throw `IndexError` or loop forever, a truncated multibyte sequence may display incorrectly, and security issues such as overlong encodings may arise.

Because the standard library guarantees validation at construction time, `str.chars()` and the like always behave safely in user code. The invariant cannot be broken without entering unsafe code.

### byte ↔ str conversion

```rok
let bytes: List<u8> = [0x68, 0x65, 0x6C, 0x6C, 0x6F];
let s: str = try str::from_utf8(bytes);
                                          //→ "hello"

let invalid: List<u8> = [0xFF, 0xFE];
let s: str = try str::from_utf8(invalid);
                                          //→ throw Utf8Error { ... }
```

## 2.15.7. `FromStr` trait — parsing

```rok
trait FromStr {
    type Err;
    fn from_str(s: str) throws (Err) -> Self;
}
```

`str.parse::<T>()` is `throws (T::Err)` and dispatches through this trait. For numeric types and `bool`, `Err` is core's `ParseError { input, pos, reason }` ([2.14.4](214-error.md#2144-built-in-error-types)). Numeric parsing is implemented on top of the runtime helpers `rok_str_int_status` / `rok_str_to_i64` / `rok_str_to_u64` / `rok_str_float_ok` / `rok_str_to_f64` ([refs-lang 8.5](../../refs-lang/008-abi/805-extern-symbols.md#string-helpers)).

```rok
let n: i32 = try "42".parse();         // FromStr for i32
let f: f64 = try "3.14".parse();       // FromStr for f64
let b: bool = try "true".parse();      // FromStr for bool

// user-defined
impl FromStr for Color {
    type Err = ParseColorError;
    fn from_str(s: str) throws (ParseColorError) -> Color { ... }
}
let c: Color = try "red".parse();
```

`FromStr` is **separate** from `TryFrom<str>` in [2.11 conversion](211-conversion.md). It handles only `str → T` conversion and makes fallibility explicit.

## 2.15.8. Module mapping

How the types are registered and where they are implemented:

| Type     | struct name          | Registration         | Implementation   |
|----------|----------------------|----------------------|------------------|
| `str`    | `str` (lowercase)    | language built-in     | standard library |
| `StrBuf` | `StrBuf`             | ordinary type        | standard library |
| `c_char` | `c_char` (lowercase) | language built-in     | standard library |
| `char`   | `Char` (PascalCase)  | `#[keyword("char")]` | standard library |

As for intrinsics, some `str` method bodies and some `char` methods carry `#[intrinsic("rok.str.*")]` or `#[intrinsic("rok.char.*")]`.

## 2.15.9. Comparison with other languages

| Aspect          | Rust              | Python 3                | Java                    | rok                                |
|-----------------|-------------------|-------------------------|-------------------------|------------------------------------|
| Type split      | `&str` + `String` | `str` (single)          | `String`                | `str` + `StrBuf`                   |
| Mutability      | only `String` mutable | **immutable**       | **immutable**           | `str` immutable, `StrBuf` mutable  |
| Encoding        | UTF-8             | unspecified (implementation-dependent) | UTF-16   | UTF-8                              |
| char            | `char` (4B)       | `str` (single-char)     | `char` (2B = code unit) | `char` (4B Unicode scalar)         |
| FFI byte        | `c_char` / `u8`   | `bytes`                 | `byte`                  | `c_char` (1B, target-dependent)    |
| Meaning of len  | byte len          | char count              | UTF-16 code unit        | byte len (explicit `.chars().count()`) |
| index `s[i]`    | **none**          | char (Python str)       | UTF-16 code unit        | **none** (explicit chars/bytes)    |

`str` is an immutable `(ptr, len)` value, and the mutable buffer is split off as `StrBuf`; with a GC, no lifetime distinction is needed, unlike Rust. `len()` means **byte length**, matching Rust; the char count is obtained separately with `chars().count()`. By **not providing** `s[i]`, rok matches Rust and avoids byte vs char ambiguity. Defining `char` as a 4B Unicode scalar avoids the surrogate pitfalls caused by Java's 2B UTF-16 code units. Finally, `c_char` is a separate type, keeping a clear ABI distinction from `char`.
