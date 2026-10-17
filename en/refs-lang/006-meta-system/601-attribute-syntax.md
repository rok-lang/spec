# 6.1. Attribute syntax

This section defines the lexical and syntactic form of attributes and their attachment rules. Attributes follow this grammar:

```
attribute       ::= outer_attr | inner_attr
outer_attr      ::= "#" "[" attr_body "]"
inner_attr      ::= "#" "!" "[" attr_body "]"
attr_body       ::= path attr_args?
attr_args       ::= "(" attr_arg ("," attr_arg)* ","? ")"
                  | "=" literal
attr_arg        ::= literal | identifier | attr_body | identifier "=" literal
```

## 6.1.1. Introducers

Attributes are introduced by two tokens defined in [punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md).

### `#[...]` outer attributes

An outer attribute is attached to the item that immediately follows it. For example:

```rok
#[derive(Debug)]
struct Point { x: f64; y: f64; }

#[inline(always)]
fn fast() { ... }
```

`#[...]` attaches to the item that follows.

### `#![...]` inner attributes

An inner attribute is attached to the item or module that encloses it:

```rok
#![cfg(target_os = "linux")]      // applies to this whole module / file

mod m {
    #![allow(unused)]              // applies to module m itself
    fn helper() { ... }
}
```

Inner attributes appear at the start of a module, file, or function body. The outer/inner distinction follows the same pattern as `///` (outer doc comment) and `//!` (inner doc comment) in [lexical structure](../001-lexical/102-lexical-structure.md).

## 6.1.2. Body forms

An attribute body has three forms: `#[name]`, `#[name(args)]`, and `#[name = value]`.

### Bare name

The simplest form uses only a name:

```rok
#[test]
#[inline]
#[derive_attr(default)]
```

The bare-name form is used as a boolean flag or marker.

### With arguments

The form with arguments is also common:

```rok
#[derive(Debug, Clone, PartialEq)]
#[repr(C)]
#[link("m")]
#[inline(always)]
#[cfg(target_os = "linux")]
#[infix("+")]
```

This is the ` name(args)` form, with arguments separated by commas. Each argument may be an identifier, a literal, a key-value (`like = "+"`), or a nested attribute form (`Display` inside `derive(...)`).

### Key-value

Finally there is the key-value form:

```rok
#[doc = "This function ..."]
#[link_name = "external_symbol"]
```

This is the ` name = value` form, where the value is a literal. It looks the same as the key-value notation within the argument form, except that the whole body is a single key-value.

## 6.1.3. Items that may carry attributes

Attributes may be attached to the following:

| Item                                | Example                                 |
|-------------------------------------|-----------------------------------------|
| Function / method                   | `#[inline] fn f() { ... }`              |
| `struct` / `enum` / `union`         | `#[derive(PartialEq, Eq)] struct S { ... }` (derive on a `union` is an error) |
| `trait` / `impl`                    | `#[deprecated] trait T { ... }`         |
| `const` / `static`                  | `#[deprecated] const X: i32 = 0;`       |
| Module                              | `#[cfg(test)] mod tests { ... }`        |
| Fields / variants within an item    | `struct S { #[deprecated] x: i32; }`    |
| Trait methods / associated items    | `#[infix("+")] fn add(...);`            |
| `use` / `extern`                    | `#[link("m")] extern "C" { ... }`       |
| Expressions (certain positions)     | `#[cfg(test)] my_test_call();` (limited) |

Which attributes are meaningful in each position depends on each attribute's definition; for example, `#[derive]` applies only to types and `#[inline]` only to functions.

## 6.1.4. Multiple attachment

Several attributes may be attached to the same item.

```rok
#[derive(Debug, Clone)]
#[repr(C)]
#[deprecated]
struct OldPoint { x: f64; y: f64; }
```

Each `#[...]` may be on its own line, or several may share a line for readability.

```rok
#[inline] #[cold] fn rarely_called() { ... }
```

## 6.1.5. When attributes take effect

When an attribute takes effect varies by attribute.

- Compiler-recognized attributes (e.g. `#[inline]`, `#[derive]`) are processed directly by the compiler at compile time.
- Magic attributes (e.g. `#[lang]`, `#[literal]`, `#[infix]`) are processed by the compiler in combination with language meaning.
- User-defined attributes (from meta blocks) generate code in the template expansion pass.

All such processing completes in the meta-expansion phase right after parsing. The detailed flow is defined in [template tokens and expansion](609-template-tokens.md).

## 6.1.6. Unknown attributes

The compiler may encounter an attribute it does not recognize:

```rok
#[mystery_attribute]
fn f() { ... }
```

The compiler then looks for a meta block definition. If none is found, it is a compile error (or a warning; the exact policy is up to the compiler implementation). This policy prevents typos from being silently ignored.

## 6.1.7. Attributes and hygiene

When an attribute attached to an item leads to code generation, identifiers in the generated code follow the hygiene rules of [6. Meta-system](README.md), so they do not collide with the user code's name space. The detailed hygiene rules are defined in [template tokens and expansion](609-template-tokens.md).

## 6.1.8. Multiple mappings of the same token

Several attributes may be attached to the same method:

```rok
#[infix("≤")]
#[infix("<=")]
fn le(self, rhs: Self) -> bool;
```

Both tokens (`≤`, `<=`) then dispatch to the same method. This pattern is often used to tie together Unicode and ASCII tokens with the same meaning.

## 6.1.9. Interaction of attributes with visibility and other keywords

Attributes may be combined in any order with visibility keywords (`pub`, `pub(pkg)`, etc.) and function qualifiers (`async`, `yield`, etc.).

```rok
#[inline]
pub fn f() { ... }

pub
#[inline]
fn g() { ... }                     // same meaning

pub async fn h() { ... }
#[inline] pub async fn h() { ... }
pub #[inline] async fn h() { ... }
```

Idiomatically the order is attribute → visibility → qualifier (`#[inline] pub async fn ...`), but the specification does not enforce an order.
