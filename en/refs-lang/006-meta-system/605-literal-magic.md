# 6.5. `#[literal(...)]` magic

The `#[literal(...)]` attribute registers a user-written syntax pattern as a literal that is dispatched to a library handler.

## 6.5.1. Motivation

Beyond the lexical literals from [numeric literals](../001-lexical/106-numeric-literals.md) through [boolean and sentinel literals](../001-lexical/109-other-literals.md) — numbers, strings, characters, booleans — rok registers compound literals such as collections, ranges, and slices in the standard library.

```
[1, 2, 3]                 → List<T> instance (fixed array if the expected type is [T; N] — compiler)
{"a": 1, "b": 2}          → Dict<K, V>
{1, 2, 3}                 → Set<T>
(1, "hi", true)            → Tuple (compiler)
0..5                       → Range<T>
0..=5                      → RangeInclusive<T>
1:3                        → Slice<T> (compiler)
2i                         → Complex<T>
```

The `(compiler)` marker denotes literals the compiler builds directly without a handler. Fixed arrays, tuples, and the slice `a[i:j]` inside an index are intrinsic to the language and cannot be registered (6.5.7, 6.5.9).

The syntax of these literals consists of tokens from [punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md) and forms from [type expressions](../002-types/201-type-expressions.md), but which library type is constructed is decided by the library.

The `#[literal]` magic attaches that decision to a trait or function.

The type of numeric literals is decided by the language, not by this magic ([1.6.2](../001-lexical/106-numeric-literals.md#162-type-suffixes), [1.6.3](../001-lexical/106-numeric-literals.md#163-unsuffixed-literals)). For numbers, `#[literal]` handles only suffixes the language does not define, such as `2i`.

## 6.5.2. Form

The pattern is written **directly as tokens** inside the attribute parentheses, not as a string. Quotes are used only to wrap a single literal token within the pattern.

```
literal_attr    ::= "#" "[" "literal" "(" pattern ("," "priority" "=" "-"? int_lit)? ")" "]"
pattern         ::= pat_elem+
pat_elem        ::= pat_atom pat_rep?
pat_atom        ::= string_lit                          // a single literal token
                  | "expr" | "number" | "string" | "ident"  // captures
                  | "{" pattern "}" | "(" pattern ")"     // groups
pat_rep         ::= ("%" string_lit)? ("*" | "+")
                  | "?"
```

```rok
#[literal("[" {expr} % "," * "]")]
fn list_literal<T>(items: ...T) -> List<T> { ... }

#[literal("{" {expr ":" expr} % "," * "}")]
fn dict_literal<K: Hash, V>(pairs: ...(K, V)) -> Dict<K, V> { ... }

#[literal("{" {expr} % "," + "}")]
fn set_literal<T: Hash>(items: ...T) -> Set<T> { ... }

#[literal(expr ".." expr)]
fn range_literal<T>(start: T, end: T) -> Range<T> { ... }

#[literal(expr "..=" expr)]
fn range_inclusive_literal<T>(start: T, end: T) -> RangeInclusive<T> { ... }

#[literal(number "i")]
fn imaginary_literal(n: f64) -> Complex<f64> { ... }

#[literal("m" "[" {{expr} % "," +} % ";" + "]", priority = 2)]
fn matrix_literal<const C: usize>(rows: ...[f64; C]) -> Matrix { ... }   // [f64; C] per row
```

The basic form is `#[literal(pattern)]`, optionally followed by `, priority = N` (may be negative, default 0) (6.5.5). The handler is a function, and a literal matching the pattern becomes a call to that function.

Scope of registration: registrations in the standard library are visible in every package; registrations in a user package are visible only within that package. The compiler collects registrations before parsing, so they apply regardless of where they are registered.

## 6.5.3. Format of syntax patterns

Syntax patterns form a small query language.

| Token      | Meaning                                                    |
|------------|------------------------------------------------------------|
| `"string"` | Literal token (punctuation or keyword). Matched as is      |
| `expr`     | Any expression                                             |
| `number`   | Numeric literal                                            |
| `string`   | String literal                                             |
| `ident`    | Identifier                                                 |
| `{ ... }`  | Group                                                      |
| `( ... )`  | Group (same as `{ }`)                                      |
| `% ","`    | Separator (comma, etc.). Must be followed by `*` / `+`     |
| `+`        | One or more repetitions                                    |
| `*`        | Zero or more repetitions                                   |
| `?`        | Optional (zero or one)                                     |

The example decomposes as follows.

```
"[" {expr} % "," * "]"
 │   │      │       │
 │   │      │       └── close token
 │   │      └── zero or more
 │   ├── separated by comma
 │   └── expr group
 └── open token
```

> The open token is the opening token, the close token is the closing token, the expr group is the group of `expr`, "separated by comma" means comma-separated, and "zero or more" means zero or more repetitions.

This pattern matches every comma-separated list: `[]`, `[1]`, `[1, 2]`, `[1, 2, 3]`, etc. Separated repetition allows one trailing separator (`[1, 2,]`).

The content of quotes must be a single token. Several tokens such as `"ab cd"` is an error; quote each token separately.

## 6.5.4. Matching and dispatch

When `[1, 2, 3]` appears in source, it is processed in these steps.

1. The parser finds a matching pattern among the registered `#[literal]` patterns.
2. On a match, the matched subexpressions are passed as arguments to the handler function.
3. The handler function constructs and returns the value.

```rok
[1, 2, 3]
   ↓
list_literal(1, 2, 3)          // items: Slice<i32>
   ↓
List<i32> instance
```

Captures become handler arguments in order.

| Pattern part                       | Argument                                                                            |
|------------------------------------|-------------------------------------------------------------------------------------|
| A single capture (`expr` etc.)     | One value                                                                           |
| Captures inside a `?` group        | A value or `none` per capture                                                       |
| `*` / `+` repetition               | Elements in one argument. Each element is a value, a tuple if there are several captures, an array if repetition is nested |
| Repetition in a variadic `...T` position | The elements become the variadic arguments ([4.2.5](../004-functions/402-parameters.md#425-variadic-t)) |

The compiler checks that the handler signature matches the number of captures and the repetition positions.

## 6.5.5. Multiple handlers (ambiguity)

When several patterns match the same input, the **longest match** wins first (among user patterns). If match lengths are equal or the same pattern has several handlers, the following rules apply in order.

1. Choose the handler whose return type equals the type of the receiving slot. If the receiving slot is `Set<T>`, the set handler is preferred.
2. Among those remaining, choose the handler with the highest `priority`.
3. If two or more still remain, it is an ambiguity error.

The slot type also determines the type parameters of the chosen handler. In `let s: Set<i64> = {1, 2};`, the `T` of `set_literal` is `i64`.

Registering the same pattern twice with the same `priority` and the same return type cannot be resolved by the rules above, so it is an error at registration, not at use.

## 6.5.6. Dict and Set ambiguity

```
{ "a": 1 }              // Dict (matches the `:` pattern)
{ 1, 2 }                 // Set (values only)
{ 1, }                   // Set (single element — trailing comma)
{ 1 }                    // always a block
{ stmt; stmt; expr }    // block ([block expression](../003-expressions/309-block-expressions.md))
```

Since `{ ... }` is also used as a [block expression](../003-expressions/309-block-expressions.md), the following rules distinguish them.

- If there are statements, as in `{ expr ; ... }`, it is a [block](../000-meta/007-glossary.md#block).
- If there is a `:`, as in `{ expr : expr , ... }`, it is a Dict.
- If there is at least one comma, as in `{ expr , ... }`, it is a Set. A single-element Set is written `{ expr , }`.
- `{ expr }` is always a block, regardless of the type of the receiving slot.

The set pattern `"{" {expr} % "," + "}"` textually also matches `{e}`, but `{e}` is a block by the rules above and does not go to the handler. A single-element set is `{e,}`.

The parser decides by looking at the contents. An explicit prefix for sets such as `s{1, 2}` is not introduced, for readability.

## 6.5.7. User-defined literals

Though rare, users can register their own literals.

```rok
struct Coord { x: f64; y: f64; }

#[literal("(" expr "," expr ")" "@")]
fn coord_literal(x: f64, y: f64) -> Coord { Coord { x, y } }

let p = (3.14, 2.71)@;        // → coord_literal(3.14, 2.71) → Coord
```

Three kinds of patterns can be registered.

| Pattern                                                      | Processing                                                                 |
|--------------------------------------------------------------|----------------------------------------------------------------------------|
| Built-in shapes (`[..]`, `{k: v}`, `{a, b}`, the six range forms) | The parser parses by the language rules and calls the handler registered for that shape |
| `number "suffix"`                                            | A numeric suffix the language lacks (`2i`, [1.6.2](../001-lexical/106-numeric-literals.md#162-type-suffixes)) |
| Any other pattern starting with a quoted token               | Tried before built-in syntax at every expression start; longest match      |

User patterns must start with a quoted token. Only the built-in range shapes (`expr ".." expr` etc.) may start with a capture. The following are errors:

- Registering the tuple shape `"(" expr "," {expr} % "," * ")"`. Tuples are intrinsic to the language.
- Registering the language's numeric suffixes (`i32`, `f64`, etc.).

A pattern starting with an identifier token (`"m" "["`) is tried before indexing of a variable of the same name (`m[i]`), so it hides that name.

## 6.5.8. `const` compatibility of literal handlers

```rok
#[literal("[" {expr} % "," * "]")]
const fn list_literal<T>(items: ...T) -> List<T> { ... }
```

- A `const fn` handler is evaluated at compile time if all its arguments are constants ([5.2.4](../005-modules/502-const.md#524-constant-expressions-and-the-evaluator)). A panic or `throw` inside it is a compile error at the literal's location.
- If the result uses the heap (`List` etc.), it is only checked at compile time and constructed at runtime ([5.2.5](../005-modules/502-const.md#525-const-fn-functions-evaluable-at-compile-time)).
- If any argument is a runtime value, it is an ordinary call.
- A handler that is not a `const fn` is called at runtime.

## 6.5.9. Slices

`a[i:j]` / `a[i:j:k]` inside an index is not a literal but the language's slicing syntax, and cannot be registered with `#[literal]`. Its meaning and trait mapping are defined by [indexing and slicing](../003-expressions/306-indexing-and-slicing.md).

## 6.5.10. When it is evaluated

`#[literal]` mapping happens at compile time, in this order.

1. The parser tokenizes the input.
2. At each expression position of the input, the registered patterns are tried.
3. When a matching pattern is found, the handler call is placed at that expression position.
4. Subsequent passes, name resolution and type checking, then process the result.

Which function call `[1, 2, 3]` becomes is thus determined unambiguously at compile time.

## 6.5.11. Magic and the standard library

`#[literal]` is used almost exclusively by the standard library; rok's collection literals are its main registrations.

Users may register literals in their own libraries, but must choose patterns carefully so they do not collide with basic tokens.
