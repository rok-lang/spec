# 6. Meta-system overview

This part defines rok's meta-system: attributes, magic, and meta [blocks](../000-meta/007-glossary.md#block). The meta-system is the core mechanism that enables the [boundary between language and standard library](../000-meta/004-language-vs-standard-libraries.md) model: the language provides a small core, and the standard library fills in its meaning.

## Contents

- [6.1. Attribute syntax](601-attribute-syntax.md)
- [6.2. Built-in attributes](602-built-in-attributes.md)
- [6.3. `#[cfg]` conditional compilation](603-cfg.md)
- [6.4. `#[lang(...)]` magic](604-lang-magic.md)
- [6.5. `#[literal(...)]` magic](605-literal-magic.md)
- [6.6. `#[keyword(...)]` magic](606-keyword-magic.md)
- [6.7. Operator magic](607-operator-magic.md)
- [6.8. User-defined meta blocks](608-meta-blocks.md)
- [6.9. Template tokens and expansion](609-template-tokens.md)
- [6.10. Reflection](610-reflection.md)
- [6.11. Compile-time file embedding](611-include.md)

## Three-level structure

The meta-system has three levels:

```
attribute
  ├── plain attribute - hints / metadata for compiler / tools
  └── magic - channel where the library binds to language meaning

meta block - user-defined derive / attribute definitions
  └── template token (@Self, @field, ...)
```

### Attributes

Attributes look like this:

```rok
#[derive(Debug, Clone)]
struct Point { x: f64; y: f64; }

#[inline(always)]
fn fast() { ... }

#[cfg(target_os = "linux")]
fn linux_only() { ... }
```

Attributes are attached to items in the form `#[...]` or `#![...]`. Built-in attributes recognized by the compiler (for example `#[derive]`, `#[repr]`, `#[cfg]`) and user-defined attributes share the same form. The detailed forms are defined in [attribute syntax](601-attribute-syntax.md) and [built-in attributes](602-built-in-attributes.md).

### Magic

Magic refers to special attributes that connect the language and the library.

```rok
#[lang(slot_name)]              // connects a standard library item ↔ a compiler slot
#[literal("...")]                // registers literal syntax
#[keyword("name")]               // registers a new keyword
```

In addition there is the operator magic family.

```rok
#[infix("+")]
#[prefix("-")]
#[infix("⊕", like = "+")]
```

These magics connect language tokens such as `1 + 2` to library trait method calls. They are defined from [`#[literal]` magic](605-literal-magic.md) through [operator magic](607-operator-magic.md).

### Meta blocks

A meta block is a code generator that lets users define their own derives or attributes; it takes the place of Rust's proc macros.

```rok
#[impl_derive(MyTrait)] pub @derive {
    impl MyTrait for @Self {
        @for_fields {
            // per-field code generation
        }
    }
}
```

There are three kinds of meta blocks: `#[impl_derive]`, `#[impl_attribute]`, and `#[impl_derive_attribute]`. Their bodies write code generation rules with template tokens such as `@Self`, `@field`, and `@for_fields`. proc-macro-style token stream manipulation is not provided; template tokens alone give sufficient expressiveness. The full definition is in [meta blocks](608-meta-blocks.md).

## Core principles

Four core principles underpin the meta-system.

### 1. No token stream manipulation

rok has no arbitrary token-manipulating macros like Rust's `syn` or `quote`; user code generation is possible only through template blocks and meta tokens.

The reason is to keep the code produced by user-defined derives in a state the compiler can understand. Allowing arbitrary token transformations forces trade-offs: unpredictability, difficult debugging, and an explosion of the tooling surface.

### 2. Magic is used by the standard library

`#[lang]`, `#[literal]`, `#[keyword]`, and the operator magics are used mainly by the standard library; user code rarely uses them directly.

These magics are the mechanism by which the standard library binds to the language to produce meaning; the [boundary between language and standard library](../000-meta/004-language-vs-standard-libraries.md) is realized through this meta-system.

### 3. Macro hygiene

Code generation in meta blocks is hygienic. Identifiers in the generated code see only the name space of the generation context and do not pollute the name space of the use site.

This prevents unpredictable collisions between user code and meta blocks.

### 4. Compile time only

All meta processing completes at compile time. Registering new derives or injecting new magic at runtime is not supported.
