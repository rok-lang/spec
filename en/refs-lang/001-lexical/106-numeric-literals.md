# 1.6. Numeric literals

This section covers the tokenization rules for numeric character sequences written directly in source code, and the rules that determine the type of each literal.

## 1.6.1. Basic form

The grammar of numeric literals is defined as follows.

```
numeric_lit ::= int_lit | float_lit
int_lit     ::= dec_lit | hex_lit | bin_lit | oct_lit
dec_lit     ::= dec_digit ("_"? dec_digit)*
hex_lit     ::= ("0x" | "0X") hex_digit ("_"? hex_digit)*
bin_lit     ::= ("0b" | "0B") bin_digit ("_"? bin_digit)*
oct_lit     ::= ("0o" | "0O") oct_digit ("_"? oct_digit)*
float_lit   ::= dec_lit "." dec_digit_seq? exponent?
              | dec_lit exponent
exponent    ::= ("e" | "E") ("+" | "-")? dec_digit_seq
dec_digit_seq ::= dec_digit ("_"? dec_digit)*
dec_digit   ::= [0-9]
hex_digit   ::= [0-9a-fA-F]
bin_digit   ::= [01]
oct_digit   ::= [0-7]
```

### Radix

The supported radixes and their notation are as follows.

| Radix       | Prefix         | Allowed digits            | Example        |
|-------------|----------------|---------------------------|----------------|
| Decimal     | none           | `0`-`9`                   | `42`, `1024`   |
| Hexadecimal | `0x` or `0X`   | `0`-`9`, `a`-`f`, `A`-`F` | `0xFF`, `0X1a` |
| Binary      | `0b` or `0B`   | `0`, `1`                  | `0b1010`       |
| Octal       | `0o` or `0O`   | `0`-`7`                   | `0o755`        |

Non-decimal radixes are used only for integer literals; floating-point literals allow decimal notation only.

A form with a leading zero (e.g. `0123`) has exactly one meaning: decimal `123`. The "leading 0 = octal" rule common in C and Java is not adopted; octal must be written with the `0o` prefix.

### Digit separators

Underscores (`_`) may be placed between digits for readability.

```rok
1_000_000
0xFFFF_FFFF
0b1010_0101
3.141_592_653
```

The rules for underscores are as follows.

- `_` may appear only between digits; the first or last character must not be `_`.
- `_` must not appear immediately after a radix prefix (`0x_FF`) or immediately before or after the decimal point (`1_.0`, `1._0`).
- Consecutive underscores (`1__000`) are rejected.
- `_` is ignored during evaluation, so `1_000` and `1000` denote the same value.

### Floating-point forms

A literal is a floating-point literal if it matches one of the following two forms; otherwise it is an integer literal or is handled as separate tokens.

```
digit+  '.'  digit*  exponent?         // 1.5, 1., 1.5e10, 0.5
digit+              exponent           // 2e10  (an exponent without a point is still floating-point)
```

A floating-point literal must have digits before the decimal point. It is written `0.5`; `.5` is not a floating-point literal. A `.` followed by digits is always a tuple index (`t.0`, `t.5`).

The exponent part (`exponent`) consists of `e` or `E`, an optional sign (`+` or `-`), and one or more decimal digits. Thus `1.`, `0.5`, `1.5e10`, `2e-3` and `6.022e23` are all valid floating-point literals. A point alone with both sides empty (`.`) is not a floating-point literal but a separate token.

#### Distinguishing `1.` from `1.foo`

The form `digit+ '.'` (`1.`) is tokenized as floating-point only when no identifier starts after the point. The cases are as follows.

| Input                                       | Tokenization                                            |
|---------------------------------------------|---------------------------------------------------------|
| `1.5`                                       | One floating-point token `1.5`                          |
| `1.` (followed by end/whitespace/punctuation) | One floating-point token `1.`                         |
| `1.foo`                                     | Three tokens: integer `1`, `.`, identifier `foo`        |
| `1 .foo`                                    | Same (whitespace does not affect the token split)       |
| `1..2`                                      | Three tokens: integer `1`, `..`, integer `2` (`..` is the longer match) |
| `1.+2`                                      | Three tokens: floating-point `1.`, `+`, integer `2`     |

In one line: if, in the middle of a number, the first character after `.` is `XID_Start` or another `.`, that `.` is not part of a floating-point literal; otherwise (punctuation, operator character, end of input) it is absorbed into the floating-point literal.

## 1.6.2. Type suffixes

A suffix after a literal specifies which concrete type the literal evaluates to.

```rok
42i32
3.14f64
2i              // imaginary unit (e.g. complex numbers)
0xFFu8
1_000_000i64
```

The suffix format follows these rules. A suffix is attached immediately after the last character of the literal without whitespace and consists only of identifier characters (`XID_Continue`). Lexically it is kept as part of one token: the lexer groups the numeric body and the suffix string into a single token.

Numeric type name suffixes (`i8` `i16` `i32` `i64` `i128` `isize` `u8` `u16` `u32` `u64` `u128` `usize` `c_char` `f32` `f64`) are defined by the language and set the literal's type to that type. The `f32` / `f64` suffixes may also be attached to integer forms (`1f64`). Other suffixes (`2i`, etc.) are interpreted by handlers registered by the standard libraries or users through [`#[literal(...)]` magic](../006-meta-system/605-literal-magic.md).

## 1.6.3. Unsuffixed literals

The type of an unsuffixed literal (`42`, `3.14`) is determined by the expected type.

- Integer literal: if the expected type is an integer or floating-point type, it is that type. With no expected type, it is `i32`.
- Floating-point literal: if the expected type is a floating-point type, it is that type. With no expected type, it is `f64`. It cannot appear where an integer type is expected.
- The arbitrary-precision std types `int` and `float` also count: an integer literal may take `int` or `float`, and a floating-point literal may take `float`. These types hold any value, so no range check applies.
- In a binary operation, if the type of one operand is statically fixed, that type becomes the expected type for the literal side.
- When an integer literal is placed where a floating-point type is expected and that floating-point type cannot represent the value exactly, it is a compile error (`let f: f32 = 16777217;`).
- If the expected type is a sum type with an integer type member, an integer literal takes that integer type (the `1` in `let v: i64 | f64 = 1;` is `i64`).
- Arithmetic among unsuffixed literals (`1 + 2`, `-1`, `2 * (3 + 4)`) remains untyped until it meets a typed operand or an expected type. When it does, the whole expression takes that type; if it never does, it takes the defaults above (`i32` / `f64`).
- If one branch of `if` / `else` is an unsuffixed integer literal and the other branch has an integer type, the literal branch follows the other branch's type.

The expected type comes from a `let` type annotation, the parameter type in argument position, the return type in return position, and the binary operation rule above.

```rok
let a = 42;              // i32
let b = 3.14;            // f64
let x: f64 = 1;          // f64
let y: u8 = 200;         // u8
let r = a * 2;           // 2 is i32 (follows a)
let s: f64 = 2.0;
let t = s * 2;           // 2 is f64 (follows s)
let e: i32 = 1.5;        // ERROR - floating-point literal cannot be an integer type
let w: i64 = 7i64;
let k = w + (1 + 2);     // 1 + 2 is i64 (meets w)
let m: u8 = 2 * 100;     // 2 * 100 is u8 (expected type)
let n = 1 + 2;           // i32 (never fixed → default)
let v = if flag { 0 } else { w };   // 0 is i64 → v: i64
```

## 1.6.4. Range checking

A literal value outside the range of its type is a compile error. Hexadecimal, octal and binary literals follow the same range rules as decimal.

```rok
let a: u8 = 256;         // ERROR - out of u8 range
let b = 0xFFi8;          // ERROR - 255 is out of i8 range
let c = 0xFFu8 as i8;    // OK - convert with as when a bit pattern is needed (-1)
```

Arithmetic consisting only of literals and constants is folded at compile time; if the result exceeds the type's range or divides by zero, it is a compile error ([5.2.4](../005-modules/502-const.md#524-constant-expressions-and-the-evaluator)).

```rok
let m: u8 = 2 * 200;     // ERROR - 400 is out of u8 range
```

## 1.6.5. Rejected forms

The following inputs are errors immediately at the lexical stage.

| Input   | Reason for rejection             |
|---------|----------------------------------|
| `0x`    | No digits after hex prefix       |
| `0b`    | No digits after binary prefix    |
| `0o`    | No digits after octal prefix     |
| `1__0`  | Consecutive `_`                  |
| `1_`    | Ends with `_`                    |
| `0x_FF` | `_` immediately after radix prefix |
| `1e`    | No digits in exponent            |

Note that `_1` is tokenized as an identifier, not a numeric literal, so it is not a rejected input but a token of a different kind.

## 1.6.6. Negative numbers

There is no separate negative literal token. `-42` consists of two tokens, the unary operator `-` and the literal `42`; a literal itself is always tokenized in non-negative form.

There is one exception to range checking. A literal immediately after unary `-` is allowed up to `MAX + 1` of its type, so `-128i8` is the minimum value of `i8`. Operands of binary operations have no such exception.

```rok
let m = -128i8;          // OK - i8::MIN
let n = a - 128i8;       // ERROR - 128 is out of i8 range
```
