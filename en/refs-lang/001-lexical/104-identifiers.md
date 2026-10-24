# 1.4. Identifiers

An identifier is a token that names a variable, function, type, module, etc.

## 1.4.1. Format

Identifiers are the default definition of Unicode Standard Annex #31 (UAX #31) (`XID_Start` / `XID_Continue`) plus the following: first, the underscore `_`; second, the mathematical notation profile; third, primes; fourth, emoji.

```
identifier          ::= identifier_start identifier_continue*
identifier_start    ::= xid_start | "_" | math_start | emoji | flag
identifier_continue ::= xid_continue | math_continue | prime | emoji | emoji_component
```

> [!IMPORTANT]
> **Variable names can be written in human languages and mathematical symbols, as long as the first character is not a digit.**

### Unicode version

The properties `XID_Start`, `XID_Continue`, `ID_Compat_Math_Start`, `ID_Compat_Math_Continue` and `Extended_Pictographic` follow the tables of **Unicode 18.0.0** (UCD 18.0.0).

### Mathematical notation profile

This follows UAX #31 §7.1 "Mathematical Compatibility Notation Profile". `math_start` is `ID_Compat_Math_Start` plus the n-ary operator symbols `∑ ∏ ∫`, and `math_continue` is `ID_Compat_Math_Continue` plus the same three symbols. The table below is an informative example (as of UCD 18.0); the properties are authoritative.

| Set             | Code points                                                                                                 |
|-----------------|-------------------------------------------------------------------------------------------------------------|
| `math_start`    | `∂` U+2202, `∇` U+2207, `∞` U+221E, their mathematical style variants U+1D6C1, U+1D6DB, U+1D6FB, U+1D715, U+1D735, U+1D74F, U+1D76F, U+1D789, U+1D7A9, U+1D7C3 (`𝛁 𝛛 𝛻 𝜕 𝜵 𝝏 𝝯 𝞉 𝞩 𝟃`), and `∑` U+2211, `∏` U+220F, `∫` U+222B |
| `math_continue` | All of `math_start`, superscripts `¹ ² ³ ⁰ ⁴`-`⁹` `⁺ ⁻ ⁼ ⁽ ⁾` (U+00B2, U+00B3, U+00B9, U+2070, U+2074-207E), subscripts `₀`-`₉` `₊ ₋ ₌ ₍ ₎` (U+2080-208E) |

`∑ ∏ ∫` are added by rok so that they can be used as function names; for this reason they cannot be operator tokens.

### Primes

`′ ″ ‴ ⁗` (U+2032-2034, U+2057) may appear only in `identifier_continue` (`f′`, `x″`).

### Emoji

- `emoji`: code points with the `Extended_Pictographic` property, excluding those at or below U+00FF (`©`, `®`, etc.). They may appear in both start and continue positions.
- `emoji_component`: may appear only in continue positions.
  - ZWJ (U+200D) - only between two pictographs
  - Variation selectors U+FE0E / U+FE0F - removed by normalization (1.4.2)
  - Skin tone modifiers U+1F3FB-1F3FF
  - Regional indicators U+1F1E6-1F1FF (flags)
  - Tags U+E0020-E007F

- `flag`: a flag made of two regional indicators (`🇰🇷`) may also appear at the start of an identifier.

A keycap sequence (`1️⃣`) starts with a digit and is therefore not an identifier.

### `x²` is a name

`x²` is not `x` squared but a single name, `x²`. Squaring is written `x * x` or `x.pow(2)`.

### Valid examples

```rok
count
my_var
α
β2
Δt
σ²
x₀
f′
∂u
∇φ
∑
변수1
データ
🚀count
_internal
```

### Invalid examples

```rok
2x         // first character is a digit
my-var     // '-' is not an identifier character
my var     // whitespace
′x         // a prime cannot be the first character
1️⃣         // keycap - starts with a digit
```

## 1.4.2. Normalization

Identifiers are compared after NFC (Normalization Form C) normalization followed by removal of the variation selectors U+FE0E / U+FE0F. The precomposed `é` and the combining form `e + ◌́` are therefore the same identifier, as are `❤️` and `❤`. The compiler applies this normalization immediately after tokenizing an identifier.

This decision ensures searchability and consistency of diagnostic messages.

### Confusable warnings

The compiler emits a **warning** (not an error) in the following cases.

- When distinct names within the same package (regardless of file) have the same skeleton. The skeleton is NFKC normalization plus confusable folding; an implementation may use only a subset of the UTS #39 confusable table. Examples: `σ²` and `σ2`; Latin `a` and Cyrillic `а`.
- When Cyrillic characters are mixed with Latin or Greek characters within one identifier. Latin-Greek mixing (`Δt`, `σx`) is common in mathematical names and is not warned about; neither is mixing Hangul, Han characters or Kana with Latin.

## 1.4.3. Case

Identifiers are case-sensitive. `count` and `Count` are distinct tokens.

The recommended naming conventions `snake_case`, `PascalCase` and `SCREAMING_SNAKE` belong to the style guide and are not enforced by the specification.

## 1.4.4. Collision with keywords

A token that exactly matches a reserved word defined in [Keywords](105-keywords.md) is classified as a keyword, not an identifier. Contextual keywords are identifiers outside their designated positions, and after `.`, in field name declarations, and in named argument positions, even reserved words are identifiers ([1.5.5](105-keywords.md#155-keywords-in-name-positions)). Raw identifiers that would allow reserved words to be used as identifiers, such as Rust's `r#fn`, are not adopted; the cost of confusion was judged to outweigh their value.

## 1.4.5. Length limit

This specification sets no upper bound on identifier length and follows the general rule of [Source files](101-source-files.md).

## 1.4.6. Single underscore

A single underscore (`_`) is accepted as an identifier but has a special meaning in pattern matching and value discarding. That meaning is defined in the pattern specification. At the lexical stage it is tokenized as a one-character identifier.
