# 1. Lexical structure

This chapter covers the rules for breaking rok source code into tokens. It examines every element encountered during tokenization, starting with files and lines and continuing through identifiers, keywords, literals and operator tokens.

Higher layers such as types, expressions and statements receive input that has already been tokenized, so all rules that apply during tokenization are gathered in this one chapter.

## Contents

- [1.1. Source files](101-source-files.md). Packages, directories, `.rok` files.
- [1.2. Lexical structure](102-lexical-structure.md). Lines, whitespace, comments.
- [1.3. Statement terminator](103-statement-terminator.md). `;` and insertion rules.
- [1.4. Identifiers](104-identifiers.md). Name format.
- [1.5. Keywords](105-keywords.md). List of reserved words.
- [1.6. Numeric literals](106-numeric-literals.md).
- [1.7. String literals](107-string-literals.md).
- [1.8. Character literals](108-character-literals.md).
- [1.9. Boolean and sentinel literals](109-other-literals.md). `true`, `false`, `none`.
- [1.10. Punctuation and operator tokens](110-punctuation-and-operator-tokens.md).
