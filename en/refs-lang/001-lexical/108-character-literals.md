# 1.8. Character literals

A character literal is a token representing a single code point or a single byte.

## 1.8.1. Two kinds

```
char_lit    ::= "'" char_item "'"
              | "c'" byte_item "'"
char_item   ::= <any unicode codepoint except "'" or "\\"> | escape_seq
byte_item   ::= <any ASCII byte except "'" or "\\"> | byte_escape
byte_escape ::= "\\x" hex_digit hex_digit | escape_seq
```

### Ordinary characters: `'A'`

```rok
'A'
'세'
'\n'
'\u{1F600}'
```

A pair of single quotes (`'`) must contain exactly one code point. Escape sequences follow the table in [String literals](107-string-literals.md) as is: `\\`, `\'`, `\"`, `\n`, `\r`, `\t`, `\0`, `\x`, `\u{...}`, etc. A one-byte form such as `'\x41'` is also valid, and its result is a single code point in the ASCII range. Conversely, containing two or more characters, as in `'AB'`, or writing an empty `''` is an error at the lexical stage.

A literal of this form is a `char` (a Unicode scalar value, a compiler primitive). Its methods are defined by [stdlib 2.15](../../refs-standard-libraries/200-core/215-string.md).

### C-compatible characters: `c'A'`

```rok
c'A'
c'\n'
c'\x7F'
```

This form consists of the `c` prefix and a pair of single quotes; escape sequences follow the table in [String literals](107-string-literals.md) as is. The value of the contents must lie within `U+0000`-`U+00FF`; exceeding this range is a lexical error. For example, `c'세'` is an error, because `'세'` is `U+C138` and cannot fit in one byte. When filling the one-byte range by writing characters directly, only ASCII characters are possible because the source is UTF-8; non-ASCII one-byte values (`U+0080`-`U+00FF`) must be written with a `\xHH` or `\u{H...}` escape.

This form evaluates to a library type compatible with the 8-bit character type of the foreign function interface ([FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface)). The exact type name is defined in the type system specification.

## 1.8.2. Why the two kinds are separate

Rather than assigning two meanings to the same token form (`'X'`), the two are separated explicitly by a prefix. A character in the Unicode sense and a single byte of the C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) have different semantics; lumping them under one notation would leave no way to state intent when calling compatible functions. The `c` prefix also leaves a visible signal that the code "touches FFI".

## 1.8.3. Mandatory length check

A character literal must contain exactly one code point (ordinary) or exactly one byte (C-compatible). This length check is performed at the lexical stage, and a mismatch is an error immediately.

This decision rules out ambiguities such as the following. Can a combining sequence (`e\u{0301}` = `é`) go in one character literal? No: it is two code points. Such text belongs in a string literal. Likewise, what is "one character to the human eye" at the grapheme cluster level is treated as a string-level concept, not a character literal.

This simplification keeps the definition character = one code point consistent from the lexical stage onward.
