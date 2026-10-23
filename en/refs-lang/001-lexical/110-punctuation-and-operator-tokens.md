# 1.10. Punctuation and operator tokens

This document covers the tokenization rules for operator and punctuation character sequences, limited to their lexical form. The meaning of each token - precedence, associativity, which trait it dispatches to, and so on - is defined in [3. Expressions](../003-expressions/README.md).

## 1.10.1. Tokenization principles

Tokenization of operators and punctuation works at two levels. First, there is a fixed token table: the character sequences listed in this document are always separated into fixed tokens. Second, there is an extension token table: character sequences additionally registered by the standard libraries through the meta-system also become tokens. The registration mechanism is defined in [`#[infix]` / `#[prefix]` / `#[postfix]` magic](../006-meta-system/607-operator-magic.md).

A conflict between the two tables is rejected at registration time; fixed tokens cannot be overridden.

## 1.10.2. Longest match

> [!IMPORTANT]
> When an input admits several token candidates, the longest match is chosen.

| Input | Tokenization                                |
|-------|---------------------------------------------|
| `==`  | One token                                   |
| `=`   | One token                                   |
| `= =` | Two tokens (separated by whitespace)        |
| `..=` | One token (`..=` is longer than `..`)       |
| `..`  | One token                                   |
| `<<=` | One token (`<<=` is longer than `<<`)       |

## 1.10.3. Fixed token table

The following is the complete list of operator and punctuation character sequences in the ASCII range reserved by the language. The standard libraries only give these tokens meaning, i.e. a trait method mapping; the token form itself is determined by this table.

### Arithmetic / comparison / logic

```
+       -       *       /       %       %%
==      !=      <       >       <=      >=
&&      ||      !
&       |       ^       ~       <<      >>
```

`%` is the arithmetic remainder (Rem - the sign follows the dividend), and `%%` is floored modulo (the sign of the result follows the divisor - always in `[0, divisor)` for a positive divisor; Python's `%`). The two tokens map to distinct traits and have different meanings.

The mapping of each token to the trait method it calls is defined in the standard library operator specification. This document guarantees only lexical tokenization.

### Broadcast (`.` prefix)

These are dot-prefixed forms for element-wise (broadcast) operations. Each arithmetic, comparison, logical, bitwise and shift operator above, prefixed with a single dot, is separated as a distinct token.

```
.+      .-      .*      ./      .%      .%%
.==     .!=     .<      .>      .<=     .>=
.&&     .||
.&      .|      .^      .<<     .>>
```

Each of these forms maps to a different trait method from its dotless form. For example, `+` maps to `Add::add` and `.+` to `DotAdd::dot_add`; these mappings are defined in the standard libraries. Unlike their dotless forms, the dot-prefixed logical operators (`.&&`, `.||`) do **not** short-circuit - because broadcast is lowered to a fused loop, both operands are evaluated (see [2.6 broadcast](../../refs-standard-libraries/200-core/206-broadcast.md) for details).

#### Distinguishing from floating-point literals

Because floating-point literals such as `1.` or `1.5` contain a dot, the tokenization of the following two inputs depends on position.

| Input  | Tokenization   | Reason                                                       |
|--------|----------------|--------------------------------------------------------------|
| `1.+2` | `1.`, `+`, `2` | `.` immediately follows a digit, so it is absorbed into the floating-point literal |
| `a.+b` | `a`, `.+`, `b` | `.` does not immediately follow a digit, so it starts a broadcast token |

In summary, a `.` in the middle of a numeric literal is always part of the floating-point literal, and a `.` elsewhere gives priority to other token forms (see the `.` handling rules in [Numeric literals](106-numeric-literals.md)).

### Unicode mathematical operators

The following Unicode operators are also fixed tokens. Their precedence is given by the table in [3.2.3](../003-expressions/302-precedence-and-associativity.md#323-precedence-and-associativity-table), and the traits they connect to are given by the standard library [2.3 ops](../../refs-standard-libraries/200-core/203-ops.md).

```
×       ⋅       ÷       ∘       √
≡       ≠       ≤       ≥
∈       ∉       ⊂       ⊆       ∪       ∩
```

| Token | Code point | Meaning (ASCII-named method)          |
|------|------------|----------------------------------------|
| `×`  | U+00D7     | Cross product (`cross`)                |
| `⋅`  | U+22C5     | Dot product (`dot`)                    |
| `÷`  | U+00F7     | Division (`div`, synonym for `/`)      |
| `∘`  | U+2218     | Function composition (`compose`)       |
| `√`  | U+221A     | Square root, prefix (`sqrt`)           |
| `≡ ≠ ≤ ≥` | U+2261, U+2260, U+2264, U+2265 | Synonyms for `== != <= >=` |
| `∈ ∉` | U+2208, U+2209 | Membership (`contains`)            |
| `⊂ ⊆` | U+2282, U+2286 | Subset (`is_proper_subset`, `is_subset`) |
| `∪ ∩` | U+222A, U+2229 | Union, intersection (`union`, `intersection`) |

Operator characters are all chosen from code points with the Unicode `Pattern_Syntax` property. `∑ ∏ ∫` are `Pattern_Syntax` but have been designated identifier characters ([1.4.1](104-identifiers.md#mathematical-notation-profile)), so they are not operators.

### Assignment / compound assignment

```
=       +=      -=      *=      /=      %=      %%=
&=      |=      ^=      <<=     >>=
```

The meaning of compound assignment - how many times the left-hand side is evaluated, the trait mapping, etc. - is defined in the expression specification. `%%=` is the compound assignment form of `%%` (floored modulo); by the longest-match rule, `%%=` is matched before `%=` or `%`.

### Range / slice

```
..      ..=     :       ::
```

`..` and `..=` are used for range notation. `:` is used as the position separator in slices, the introducer of format specifiers, and in various other contexts. `::` is the path separator, used to access static items of modules or types.

### Dot / question mark

```
.       ?.      ??      ?
```

`.` is used for member or method access. `?.` is dot access that passes through a sum type. `??` is the binary operation that fills an empty slot (`none`) with a fallback value. Finally, `?` alone is the postfix unary operator that, when a `T | none` value is `none`, returns `none` early from the enclosing function. Its detailed meaning and desugaring rules are defined in the `?` operator section of [Error control flow](../003-expressions/314-error-control-flow.md#31410-the--operator---early-return-of-none).

`try?` and `try!` are not single tokens. Lexical analysis produces the keyword `try` and the punctuation `?` or `!` as separate tokens, and parsing interprets a `?` / `!` attached to `try` with no whitespace as the [`try?` / `try!`](../003-expressions/314-error-control-flow.md#3143-try-and-try) prefix operator. With whitespace, as in `try !ready()`, it is interpreted as `try` and logical negation `!`.

### Single-quote token: character literal vs label

The `'` token branches into two forms. The first is the form `'X'` or `'\X'`, a single character (or escape) followed by a closing `'`, which is a [character literal](108-character-literals.md). The second is the form `'identifier`, an identifier with no closing `'`, which is a label token (see [Labels](../003-expressions/313-loop-expressions.md#3134-labels-controlling-nested-loops) and [Labeled blocks](../003-expressions/309-block-expressions.md#3910-labeled-blocks-label---)).

Lexical analysis decides by looking at the two bytes after `'`. If an identifier start is followed by a closing `'`, it is a character; if an identifier start is followed by a token other than a closing `'`, such as an identifier character, a colon or whitespace, it is a label; an escape (`'\...`) is always a character literal.

### Lambda / arrows

```
|       ||      ->      =>
```

`|` is used, depending on context, as bitwise OR, the sum type separator, and the lambda parameter delimiter. `||` is used for logical OR or to start a parameterless lambda. `->` is the function return type arrow, and `=>` is the `match` arm arrow.

### Punctuation

```
(   )   [   ]   {   }
,   ;   @
```

The uses of each punctuation mark are as follows. `( )` is used for grouping, tuples and calls. `[ ]` is used for indexing, slices, capture lists and array literals. `{ }` is used for blocks, compound literals and struct literals. `,` is the argument or item separator, and `;` is the statement terminator (see [Statement terminator](103-statement-terminator.md)). Finally, `@` is a meta-system token; its exact meaning is covered in Chapter 6.

### Attribute introducers

```
#[      #![
```

`#[` introduces an attribute attached to the following item, i.e. an outer attribute. `#![` introduces an attribute attached to the enclosing item or module, i.e. an inner attribute.

Each of these two introducers is separated as a single fixed token. Whitespace must not appear between `#` and `[`; a separate token `#` followed by `[` is undefined.

The outer/inner pair follows the same pattern as the doc comments `///` and `//!` in [Lexical structure](102-lexical-structure.md). Only the direction of attachment differs; the meaning is grouped in the same way.

## 1.10.4. Extension tokens

Beyond the fixed table above, a character sequence that the standard libraries or users can register must satisfy the following conditions: every character must be a `Pattern_Syntax` code point; it must contain no identifier characters ([1.4.1](104-identifiers.md#141-format)); it must contain no whitespace or line terminators; and it must not be identical to a fixed token.

Examples include `⊕`, `⊗`, `⋆` and `°`. The precedence of an extension token follows the existing token specified with `like` at registration ([6.7.2](../006-meta-system/607-operator-magic.md#672-arguments)).

Once such a sequence is registered, the lexer cuts it out as a single token. The registration time, syntax and meaning are defined in [`#[infix]` / `#[prefix]` / `#[postfix]` magic](../006-meta-system/607-operator-magic.md).

## 1.10.5. What is not tokenized

The following elements are not tokens but parts of larger units. The `f` of `f"..."` is the format string prefix of [String literals](107-string-literals.md). The `r` of `r"..."` and the `r#` of `r#"..."#` are raw string prefixes. The `c` of `c'...'` is the C-compatible character prefix of [Character literals](108-character-literals.md). Type suffixes attached to numeric literals also belong here; see [Numeric literals](106-numeric-literals.md) for details.

These do not appear in the token table of this document, and in their positions they are not tokenized as identifiers or ordinary tokens either.
