# 1.2. Lexical structure

This chapter gives the overall picture of how a stream of characters becomes a stream of tokens. The detailed format of each token kind is defined in subsequent documents; here only the outline is given.

```
token           ::= identifier
                  | keyword
                  | numeric_lit
                  | string_lit
                  | char_lit
                  | sentinel_lit                  // true | false | none
                  | punctuation
whitespace      ::= space_char+
                  | line_comment
                  | block_comment
space_char      ::= U+0009 | U+000A | U+000B | U+000C | U+000D | U+0020
line_comment    ::= "//" <any char except line_terminator>* line_terminator
block_comment   ::= "/*" <any char>* "*/"          // recursive
line_terminator ::= U+000A | U+000D U+000A? | U+000D
```

## 1.2.1. Tokens

The compiler's lexical analyzer (the **lexer**) reads a source file and produces a stream of tokens. The tokens fall into the following kinds.

| Kind                         | Example                             |
|------------------------------|-------------------------------------|
| Identifier                   | `count`, `α`, `Δt`                  |
| Keyword                      | `let`, `if`, `fn`                   |
| Numeric literal              | `42`, `3.14`, `0xFF`, `1_000`       |
| String literal               | `"hello"`, `r"raw\n"`, `f"x = {x}"` |
| Character literal            | `'A'`, `c'A'`                       |
| Boolean / sentinel literal   | `true`, `false`, `none`             |
| Operator / punctuation       | `+`, `==`, `..`, `[`, `;`           |

The exact format rules for each kind are defined in turn in the following documents.

## 1.2.2. Whitespace

Whitespace characters are meaningful only as separators between tokens and do not form tokens themselves.

The following code points are recognized as whitespace.

| Code point | Name                 |
|------------|----------------------|
| `U+0009`   | Tab (TAB)            |
| `U+000A`   | Line feed (LF)       |
| `U+000B`   | Vertical tab (VT)    |
| `U+000C`   | Form feed (FF)       |
| `U+000D`   | Carriage return (CR) |
| `U+0020`   | Space                |

Unicode space-like characters not in the table above (`U+00A0` NO-BREAK SPACE, `U+200B` ZERO WIDTH SPACE, etc.) are not whitespace. They appear only as ordinary code points inside identifiers or strings.

Indentation is not significant. Python-style significant indentation is not adopted.

## 1.2.3. Comments

rok supports three kinds of comments.

### Line comments

```rok
// comment to the end of this line
```

From `//` to the line terminator of that line is a comment. The line break itself is not part of the comment.

### Block comments

```rok
/* spanning
   multiple lines */
```

The text between `/*` and `*/` is a comment, and nesting is allowed. The outer comment closes only after inner `/*`/`*/` pairs are properly closed (the same rule as Rust), which is useful for temporarily disabling large regions of code.

```rok
/* outer
   /* inner */
   still a comment */
```

### Doc comments

There are two forms of doc comments.

```rok
/// This item ...      (outer doc comment — attached to the following item)
//! This item ...      (inner doc comment — attached to the enclosing item / module)
```

- The extent is from the starting token to the line terminator of that line.
- `/// text` is the attribute `#[doc = "text"]`, and `//! text` is `#![doc = "text"]` ([6.2.8](../006-meta-system/602-built-in-attributes.md#628-doc---doc-comment-form)).
- A line starting with `////` is an ordinary comment.
- A doc comment in a position where an attribute cannot appear (inside an expression, between statements, etc.) is ignored.
- The RHIR interface text of a prebuilt package ([9.4](../009-tools/904-rhir-bytecode.md)) preserves doc comments.

Block-form doc comments (`/** ... */`, `/*! ... */`) are not adopted; the single-line form is used uniformly for tool friendliness and consistency.

## 1.2.4. Compiler output

The lexer's output consists of the token stream and position information. The position information follows the diagnostic unit rules of [Source files](101-source-files.md).

This output is the sole input to the next stage, the parser; the parser does not look at the original source again.
