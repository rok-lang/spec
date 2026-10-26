# 1.1. Source files

This section covers the physical format of Rok source files: how a file must be encoded on disk, and what the compiler assumes when it begins reading the first byte.

## 1.1.1. Extension

The extension of a Rok source file is `.rok`. Other extensions, such as `.no` or `.rokr`, are not recognized as Rok source.

## 1.1.2. Character encoding

> [!IMPORTANT]
> Source files must be encoded in **UTF-8**.

Files written in other encodings such as UTF-16, GBK or Shift_JIS are rejected by the compiler immediately. Automatic encoding detection is not attempted, because guesses are wrong more often than not. A BOM (`U+FEFF`) at the start of a file is silently ignored; a BOM appearing in the middle of the text is treated as an ordinary code point and is meaningful only inside identifiers or strings.

### Bidirectional control characters are forbidden

The bidirectional (bidi) control characters U+202A-202E and U+2066-2069 are a compile error anywhere in the source (including comments and string literals). This prevents attacks in which the displayed code differs from the actual token order. If a string needs one of these characters, write it with an escape such as `\u{202E}`.

## 1.1.3. Line terminators

All three of the following are recognized as the end of a logical line.

- LF (`U+000A`)
- CRLF (`U+000D U+000A`)
- CR (`U+000D` alone)

The compiler accepts these three forms even when mixed in one file, since it must work regardless of where a file was produced. This specification recommends LF alone.

Other line-separating code points, such as `U+2028` LINE SEPARATOR or `U+2029` PARAGRAPH SEPARATOR, are not line terminators. They appear as ordinary code points inside identifiers or strings.

## 1.1.4. Line and file size limits

This specification sets no upper bound on line length, file size or identifier length. A compiler may impose reasonable limits, but such limits are implementation decisions and not part of the specification.

## 1.1.5. Position units in diagnostics

Compiler diagnostics (errors, warnings) give positions as file, line and column.

Line numbers start at 1. The column is a 1-based index counted in code points from the start of the line - code points, not bytes. A multibyte character such as a Hangul syllable therefore occupies one column. A tab (`U+0009`) also occupies one column and does not imitate its visual width.

These rules make diagnostic positions easy for machines - editors and LSPs - to process. Visual alignment is left to the display tool.
