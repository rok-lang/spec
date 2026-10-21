# 6.11. Compile-time file embedding

## 6.11.1. `include_str!` / `include_bytes!`

```rok
const TEMPLATE: str = include_str!("templates/page.html");
static ICON: [u8; 1406] = include_bytes!("res/icon.png");
```

`include_str!("path")` evaluates to the contents of the file `path` as a `str`, and `include_bytes!("path")` to its bytes as a `[u8; N]` (`N` the file's length; a byte string, [1.7.5](../001-lexical/107-string-literals.md#175-byte-strings), including its `Slice<u8>` rule). The argument is exactly one string literal. The path is relative to the directory of the source file that contains the call. The file is read at compile time; the result is a constant and may appear anywhere a literal may, including `const` and `static` initializers.

It is a compile error if the file cannot be read, or, for `include_str!`, if its contents are not valid UTF-8. The included file is an input of the build: changing it rebuilds what uses it ([9.2.16](../009-tools/902-aot-pipeline.md#9216-incremental-build)).

Like `cfg!` ([6.3.5](603-cfg.md#635-cfg-compile-time-bool-value)), these are compiler built-in forms, not user-defined macros.
