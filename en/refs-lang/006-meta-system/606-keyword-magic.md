# 6.6. `#[keyword(...)]` magic

The `#[keyword("name")]` attribute is the mechanism by which the standard library registers new keywords.

## 6.6.1. Motivation

The reserved keyword list in [keywords](../001-lexical/105-keywords.md) is a fixed set defined by the language itself. Must the language specification be changed every time another keyword is to be added?

The standard library sometimes wants to introduce tokens that look like keywords, such as domain keywords of a library DSL. The `#[keyword]` magic is the sanctioned channel through which a library can register language-level keywords in such situations.

## 6.6.2. Form

```
keyword_attr    ::= "#" "[" "keyword" "(" string_lit ")" "]"
```

```rok
#[keyword("yield_priority")]
fn yield_priority_handler(...) { ... }

#[keyword("with")]
fn with_handler(...) { ... }
```

The basic form is `#[keyword("name")]`, taking the keyword name (a string literal in identifier shape) as its argument. The handler is a **free function**; it cannot be attached to methods or trait items.

`#[keyword("name")]` on a `struct` registers a **type keyword**: that name designates the struct in type position (`int` → `rok::core::num::bigint::Int`, `float` → `rok::core::num::bigfloat::Float`, [stdlib 2.1.2](../../refs-standard-libraries/200-core/201-numbers.md#212-int--arbitrary-precision-integer)). Only the standard library may register type keywords. Attaching it to an item that is neither a function nor a struct is an error.

## 6.6.3. Effect of registration

When `#[keyword("name")]` is registered somewhere, the following takes effect.

1. In that compilation unit, the identifier `name` is tokenized as a keyword.
2. Uses of the keyword desugar into handler calls. `name(args)` is a call through the handler's absolute path, and a following block becomes a [trailing closure](../004-functions/408-trailing-closure.md) argument.

```rok
#[keyword("with")]
fn with_handler(m: Mutex<i32>, body: fn()) { ... }

with(m) { critical(); }
// ⇒ pkg::with_handler(m) { critical(); }     — handler + trailing closure
```

Before registration `name` could be used as an ordinary identifier; after registration it is a keyword token. Using `name` as the name of a declaration or member is therefore a compile error. Those positions are `let name`, `fn name`, `struct name`, `enum name`, `trait name`, `mod name`, `for name in`, `.name`, `::name`, `name:` (field / argument name), and `name::`. The same rule applies inside `{..}` of f-strings.

## 6.6.4. Conflict checking

```rok
#[keyword("if")]                  // ERROR — 'if' is a reserved keyword
fn ... { ... }
```

Reserved keywords from [keywords](../001-lexical/105-keywords.md), names that are keywords only in fixed positions (`type`, `any`, `in`, `where`, `throws`, `rethrows`, `catch`, `finally`, `const`, `static`, `extern`, `mut`, `union`), and path roots (`pkg`, `sup`, `rok`) cannot be registered. They are rejected immediately at registration.

Registering the same keyword twice within one build is an error, whether within the same package or across different libraries.

```
package_a registers #[keyword("with")]
package_b also registers #[keyword("with")]    → ERROR if both packages are used in one build
```

## 6.6.5. Scope of application

A `#[keyword]` takes effect only within the compilation unit (usually a package) that contains the registration. In other packages the same name works as an ordinary identifier. As an exception, the standard library's type keywords (`int`, `float`) apply to every package.

This locality reduces conflicts between libraries and lets users decide consciously which package introduces which keyword.

## 6.6.6. Usage patterns

`#[keyword]` is used rarely, mainly in these cases:

- Domain-specific languages (DSLs): when it is natural for domain words — tabular data, grids, circuits, etc. — to behave like keywords in identifier positions.
- New syntax in block or body form that is hard to express with operator magic.

In most cases, a combination of a function and a trailing closure suffices to build a keyword-like pattern ([trailing closures](../004-functions/408-trailing-closure.md)). Where that form is possible, prefer it.

## 6.6.7. Combination with the scanner pre-pass

When `#[keyword]` registrations are recognized is handled by the compiler's scanner pre-pass.

1. Before parsing, the compiler quickly scans the source files and collects `#[keyword(...)]` registrations.
2. The registration table is passed to the lexer.
3. During full lexical analysis, the registered names are split off as keyword tokens.
4. The parser desugars them into the registered handlers.

Thanks to this pre-pass, consistency within the compilation unit is maintained regardless of where a registration appears (which file, which line). Users need not care about registration order.

## 6.6.8. Signature of keyword handlers

The compiler does not check the handler's signature separately. After a keyword use becomes a handler call (6.6.3), the call is type-checked as an ordinary function call. The handler's parameters therefore determine the usage form of the keyword (arguments, trailing closure).

## 6.6.9. Absence of contextual keywords (restated)

The principle in [keywords](../001-lexical/105-keywords.md) of not introducing contextual keywords applies equally to registered keywords.

Once `#[keyword("with")]` is registered, `with` is tokenized as a keyword everywhere. Forms such as "a keyword only in certain positions" are not introduced.

This consistency helps keep the lexer and parser simple.

## 6.6.10. Visibility of registration

The visibility ([visibility](../005-modules/506-visibility.md)) of the item carrying `#[keyword]` applies only to accessing the item and is separate from the effect of keyword registration.

```rok
pub(pkg) fn keyword_handler(...) { ... }

#[keyword("with")]
fn another_handler(...) { ... }     // keyword registration — applies to the whole package
```

Whether the handler can be called from outside is determined by visibility, but the keyword registration itself affects the whole compilation unit.

## 6.6.11. Difference between `#[keyword]` and user-defined macros

Compared with macro systems of other languages:

- Rust proc macro: performs arbitrary token transformations; users generate code freely. Hygiene is a separate matter.
- Lisp macro: dynamic macros that can also be registered at runtime.
- rok `#[keyword]`: compile-time keyword registration; a handler function determines the desugaring. Free token transformation is impossible.

rok deliberately has a restricted macro system, favoring tool friendliness in the trade-off between expressiveness and tool friendliness.

## 6.6.12. Registered keywords vs function calls

```rok
// keyword form
with(mutex) { critical(); }

// function call form (no registration)
with_lock(mutex) { critical(); }     // function + trailing closure
```

Most DSL-like patterns are served by combining a function with a trailing closure. Restrict `#[keyword]` registration to cases where that form would be unnatural.

## 6.6.13. When it is evaluated

Keyword registration is processed at the scanner pre-pass, before other meta processing (cfg, meta block expansion, etc.).

Because of this order, cfg can be applied conditionally to keyword handlers themselves, and keywords can be used inside meta block bodies. A meta block's or keyword handler's `#[cfg]` is evaluated with the cfg of the package that defines it (its own features, [9.10.4c](../009-tools/910-rok-frontend.md#9104c-features)).

## 6.6.14. Use sparingly

Use `#[keyword]` rarely and carefully. Every registration enlarges the language surface and burdens other users with guessing the meaning of a new keyword.

For a language keyword that would be adopted universally, adding it directly to the reserved list in [keywords](../001-lexical/105-keywords.md) (that is, as a language specification change) may be more appropriate.
