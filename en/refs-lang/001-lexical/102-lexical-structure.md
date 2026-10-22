# 1.2. Lexical structure

This chapter gives the overall picture of how a stream of characters becomes a stream of tokens. The detailed format of each token kind is defined in subsequent documents; here only the outline is given.

```
token           ::= identifier
                  | keyword
                  | numeric_lit
                  | string_lit
                  | byte_string_lit
                  | char_lit
                  | sentinel_lit                  // true | false | none
                  | punctuation
whitespace      ::= space_char+
                  | line_comment
                  | block_comment
space_char      ::= U+0009 | U+000A | U+000B | U+000C | U+000D | U+0020
line_comment    ::= "//" <any char except line_terminator>* line_terminator
block_comment   ::= "/*" <any char>* "*/"          // recursive
shebang_line    ::= "#!" <any char except line_terminator>*   // first line only, see 1.2.3
line_terminator ::= U+000A | U+000D U+000A? | U+000D
```

## 1.2.1. Tokens

The compiler's lexical analyzer (the **lexer**) reads a source file and produces a stream of tokens. The tokens fall into the following kinds.

| Kind                         | Example                             |
|------------------------------|-------------------------------------|
| Identifier                   | `count`, `α`, `Δt`                  |
| Keyword                      | `let`, `if`, `fn`                   |
| Numeric literal              | `42`, `3.14`, `0xFF`, `1_000`       |
| String literal               | `"hello"`, `r"raw\n"`, `f"x = {x}"`, `b"bytes"` |
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

rok supports three kinds of comments: line, block and doc comments. A shebang line on a file's first line is ignored like a comment.

### Shebang line

If a file's first line (after a BOM) starts with `#!`, that line is a shebang line and is ignored like a comment, so a script with `#!/usr/bin/env rok` runs as `./script.rok` ([9.10.1](../009-tools/910-rok-frontend.md#9101-subcommands-and-global-options)). The exception is when the first token after `#!`, past whitespace and comments other than doc comments, is `[`: then `#!` begins an inner attribute (`#![allow(unused)]`). `#!` on any later line is not a shebang line.

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

There are four forms of doc comments, as in Rust.

```rok
/// outer doc comment (the following item)
//! inner doc comment (the enclosing file, module or function)
/** outer block doc comment */
/*! inner block doc comment */
```

- A line doc comment extends to the line terminator; `/// text` is `#[doc = "text"]` (one space after the marker is dropped) and `//! text` is `#![doc = "text"]` ([6.2.8](../006-meta-system/602-built-in-attributes.md#628-doc---doc-comment-form)). A line starting with `////` is an ordinary comment.
- A block doc comment extends to its matching `*/` and nests like a block comment. `/**/`, `/***/` and a comment starting with `/***` are ordinary comments. Its text is one `#[doc]` / `#![doc]` attribute. A blank first line and a blank last line are dropped. When the comment spans lines and every line after the first starts with optional whitespace and `*`, that prefix is removed. Then the indentation common to the non-blank lines and the whitespace before `*/` are removed. Line breaks are LF.
- An inner doc comment appears only at the start of a file, a module body or a function body. Anywhere else it is an error.
- An outer doc comment documents the item that follows. One that documents nothing (before `}`, `)`, `]` or the end of the file) is an error, and so is one on a function parameter. One on a statement, match arm, struct literal field, tuple field or expression is kept by no tool and triggers the `unused_doc_comments` lint ([6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control)).
- The ROKHIR interface text of a prebuilt package ([9.4](../009-tools/904-rokhir-bytecode.md)) preserves doc comments.

## 1.2.4. Compiler output

The lexer's output consists of the token stream and position information. The position information follows the diagnostic unit rules of [Source files](101-source-files.md).

This output is the sole input to the next stage, the parser; the parser does not look at the original source again.
