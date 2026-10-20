# 1.7. String literals

A string literal is a token representing a sequence of characters. This document covers only the lexical form; which library type the result evaluates to is decided by the standard library's literal handlers.

## 1.7.1. Ordinary strings

```
string_lit  ::= '"' string_item* '"'
string_item ::= string_char | escape_seq
string_char ::= <any unicode codepoint except '"' or '\'>
escape_seq  ::= "\\" ("\\" | "\"" | "'" | "n" | "r" | "t" | "0")
              | "\\x" hex_digit hex_digit
              | "\\u" "{" hex_digit ("_"? hex_digit){0,5} "}"
              | "\\" line_terminator whitespace*
```

An ordinary string is enclosed in double quotes (`"`).

```rok
"hello"
"hello, 세계"
""
"a string
spanning
several lines"
```

It may contain any Unicode code point, including line breaks as is. A string that is never closed is a compile error: the closing `"` must be in the same file, so a string left unclosed through the end of the file is an error.

### Escape sequences

The following escape sequences are recognized.

| Sequence   | Meaning                                                              |
|------------|----------------------------------------------------------------------|
| `\\`       | Backslash (`U+005C`)                                                 |
| `\"`       | Double quote (`U+0022`)                                              |
| `\'`       | Single quote (`U+0027`)                                              |
| `\n`       | Line feed (`U+000A`)                                                 |
| `\r`       | Carriage return (`U+000D`)                                           |
| `\t`       | Tab (`U+0009`)                                                       |
| `\0`       | NULL (`U+0000`)                                                      |
| `\xHH`     | One byte (two hex digits, only `U+0000`-`U+007F` allowed — ASCII range) |
| `\u{H...}` | Any Unicode code point (1–6 hex digits, at most `U+10FFFF`)          |

Hex digits inside `\u{...}` may be separated with `_` (e.g. `\u{1_F600}`). Any backslash sequence other than those listed above is a lexical error; an unknown escape is never silently left as its original characters.

### Line continuation

When a backslash in a string immediately precedes a line terminator, the backslash, the line terminator and the leading whitespace of the next line are all omitted.

```rok
"a long string that \
 reads as one line"
// resulting contents: "a long string that reads as one line"
```

This rule lets a long string be split across several lines in source code without inserting line breaks into the resulting value.

## 1.7.2. Raw strings

A raw string is written with the `r` prefix followed by double quotes.

```rok
r"raw\n"
r"C:\path\to\file"
```

No escapes are processed inside a raw string, so `\n` is kept as two characters. If the contents of a raw string need a closing `"`, the delimiter can be changed by surrounding it with the same number of `#` on both sides.

```rok
r#"a string may contain "double quotes""#
r##"even r#"this form"# works"##
```

An opening `r##"` must end with a closing `"##` with exactly the same number of `#`; the count that matters is the number of `#` on the opening side. Raw strings do not support line continuation (`\` + line terminator), since by definition a raw string preserves every character as is.

## 1.7.3. Format strings

Format strings come in two forms, `f"..."` and `f"""..."""` (the same rules as Python 3.12 / PEP 701).

```rok
f"x = {x}"
f"sum = {a + b}, diff = {a - b}"
f"""
  name: "{name}"
  total: {items
      .len()}
"""
```

- `{{` and `}}` in text are a single brace.
- The text of `f"""..."""` may contain `"` as is, and the code in a hole may span several lines. A hole in `f"..."` is a single line. There is no `"""` string without `f`.

### Holes

The inside of a hole `{ .. }` consists of ordinary tokens read by the ordinary lexer.

- The outer quotes may be reused (`f"{s.split(" ")}"`, `f"{d["k"]}"`). f-strings nest to any depth.
- A string literal inside a hole keeps its own escapes (`f"{"a\tb"}"`).
- A `\` outside a string literal is an error. The diagnostic advises writing `"` as is.
- The expression ends at a `}` or `:` at parenthesis depth 0. `::` is a path separator.

```rok
f"name = {user.name ?? "anon"}"     // OK
f"name = {user.name ?? \"anon\"}"   // ERROR — \ inside a hole
f"{\"ab\".replace("a", "b")}"  // ERROR — write f"{"ab".replace("a", "b")}"
```

### Format specifiers

A format specifier may follow the expression after a colon (`:`).

```rok
f"{x:.2f}"        // two decimal places
f"{n:0>5d}"       // 5 digits, zero-filled, right-aligned
f"{val:?}"        // debug format
```

The grammar of format specifiers is as follows.

```
spec  := [fill][align][sign]['#'][0][width]['.'precision][type]
align := '<' | '>' | '^' | '='
sign  := '+' | '-' | ' '
type  := 'd' | 'g' | 's' | 'f' | 'b' | 'o' | 'x' | 'X' | 'e' | 'E' | '?'
```

If `width` is present and `align` is not, numbers (integers and floating-point) are right-aligned and other types are left-aligned. When `<` `>` `^` is given (with a fill character), any type follows that alignment. An f-string is turned into formatting code at the stage where the types of the holes are known, so the alignment default and spec checking are determined by type. Holes are written in sequence into the buffer of a single `Formatter`, without creating an intermediate string per hole ([stdlib 2.10](../../refs-standard-libraries/200-core/210-fmt.md#2101-dispatch)).

| Part         | Meaning                                                                                      |
|--------------|----------------------------------------------------------------------------------------------|
| `fill`       | Fill character (default space). Must be immediately followed by `align`                     |
| `align`      | `<` left, `>` right, `^` center, `=` pad between sign and digits (numbers only)              |
| `sign`       | `+` shows `+` for positives too, space shows a space for positives, `-` is the default (`-` for negatives only) |
| `#`          | `0b` / `0o` / `0x` prefix for `b` / `o` / `x` / `X`                                          |
| `0`          | Zero-pad numbers (same as `=` alignment with fill `0`)                                       |
| `.precision` | Digits after the decimal point for floating-point (without conversion); maximum length (truncation) for strings etc.; ignored for integers |

| `type`          | Applies to              | Meaning                                                           |
|-----------------|-------------------------|-------------------------------------------------------------------|
| (none), `d`, `g`, `s` | All types         | `Display`                                                         |
| `f`             | Numbers                 | Fixed point (`precision` digits)                                  |
| `b` `o` `x` `X` | Integers                | Base 2 / 8 / 16, at the width of the integer type (`{-1i8:x}` is `ff`) |
| `e` `E`         | Integers, floating-point | Exponent notation                                                |
| `?`             | All types               | `Debug`. May be combined with width / align                       |

A static spec is checked at compile time against the hole's type; a mismatch is an error pointing at that hole (`{s:x}` with `s: str`, etc.). The trait invoked by each type letter is defined by the standard library formatting specification ([stdlib 2.10](../../refs-standard-libraries/200-core/210-fmt.md)).

### Dynamic format specifiers

Expressions may be embedded inside a format specifier. Both `{expr}` (Python style) and `${expr}` are accepted.

```rok
f"{x:.{prec}f}"           // dynamic precision
f"{x:>${width}.${prec}f}" // dynamic width + precision
f"{val:{full_spec}}"      // entire specifier dynamic
```

- An embedded expression is evaluated, converted to a string, and becomes part of the specifier text. The expanded result must satisfy the `spec` grammar and the hole's type; otherwise it is a runtime panic.
- Nested holes have no specifiers of their own.
- `$$` inside a specifier is a single `$`. A lone `$` not forming `${` is kept as is.

### Lexical processing

The lexer splits a format string into the following tokens.

```
FStrStart                     f" or f"""
  FStrText    "x = "          ({{ / }} folded into braces)
  HoleStart
    (ordinary tokens — expression)
    HoleSpec                  (optional) :
      SpecText / nested hole
  HoleEnd
  FStrText    ", y = "
FStrEnd
```

The expression inside a hole is processed by the ordinary expression parser.

## 1.7.4. Automatic concatenation of adjacent strings

C- and Python-style automatic concatenation of adjacent strings (`"abc" "def"` → `"abcdef"`) is not adopted. Concatenation requires an explicit operator or method.
