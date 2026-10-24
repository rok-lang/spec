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
- [6.8. User-defined meta definitions (meta blocks and macros)](608-meta-blocks.md)
- [6.9. Template tokens and expansion](609-template-tokens.md)
- [6.10. Reflection](610-reflection.md)
- [6.11. Compile-time file embedding](611-include.md)

## Three-level structure

The meta-system has three levels:

```
attribute
  ├── plain attribute - hints / metadata for compiler / tools
  └── magic - channel where the library binds to language meaning

meta definition - user-defined code generator, one template engine (6.8)
  ├── derive / attribute / per-field  - triggered by #[..] on a type or item
  └── macro                           - triggered by a call, name!(..)
      └── template token (@Self, @field, ..) + matcher (typed fragments)
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

A meta definition is a code generator. Three forms let users define their own derives or attributes (taking the place of Rust's derive and attribute proc macros); the fourth, a [macro](608-meta-blocks.md#6812-the-macro-form), is triggered by a call `name!(..)` and generates code from what the caller writes (the function-like case). All four share one template engine.

```rok
#[impl_derive(MyTrait)] pub @derive {
    impl MyTrait for @Self {
        @for_fields {
            // per-field code generation
        }
    }
}
```

There are four forms, all written the same way - `#[impl_<kind>(name)]`, a visibility, an `@` introducer, and a block: the attribute-triggered `#[impl_derive]`, `#[impl_attribute]`, and `#[impl_derive_attribute]`, and the call-triggered `#[impl_macro(name)] @macro { .. }` (invoked as `name!(..)`). Their bodies write code generation rules with template tokens such as `@Self`, `@field`, and `@for_fields`, and a macro adds a matcher of typed fragments (`@x: expr`). proc-macro-style token stream manipulation is not provided; template tokens alone give sufficient expressiveness. The full definition is in [meta definitions](608-meta-blocks.md).

## Core principles

Four core principles underpin the meta-system.

### 1. No token stream manipulation

rok has no procedural, token-manipulating macros like Rust's `syn` or `quote`. All user code generation - the attribute-triggered meta blocks and the call-triggered macro ([6.8.12](608-meta-blocks.md#6812-the-macro-form)) alike - is by pattern and template only: a matcher binds parts of the input, a template transcribes them. No step runs arbitrary code or rewrites a raw token stream.

The reason is to keep generated code in a state the compiler, formatter and language server can follow. Allowing arbitrary token transformations forces trade-offs: unpredictability, difficult debugging, and an explosion of the tooling surface. The macro adds call-site code generation (what the other forms cannot do) while staying inside this rule.

### 2. Magic is used by the standard library

`#[lang]`, `#[literal]`, `#[keyword]`, and the operator magics are used mainly by the standard library; user code rarely uses them directly.

These magics are the mechanism by which the standard library binds to the language to produce meaning; the [boundary between language and standard library](../000-meta/004-language-vs-standard-libraries.md) is realized through this meta-system.

### 3. Macro hygiene

Code generation in meta blocks is hygienic. Identifiers in the generated code see only the name space of the generation context and do not pollute the name space of the use site.

This prevents unpredictable collisions between user code and meta blocks.

### 4. Compile time only

All meta processing completes at compile time. Registering new derives or injecting new magic at runtime is not supported.
