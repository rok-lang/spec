# 1.9. Boolean and sentinel literals

rok has three literals that take the form of keywords but actually denote values. Grammatically they are defined as follows.

```
sentinel_lit    ::= "true" | "false" | "none"
```

## 1.9.1. Boolean literals

Boolean literals are written as follows.

```rok
true
false
```

Boolean literals consist of exactly two tokens, `true` and `false`. Both names are in the reserved list of [Keywords](105-keywords.md) and cannot be reused as identifiers. They evaluate to the standard library's boolean type (`bool`); its exact definition is covered by the type system.

## 1.9.2. Sentinel literal: `none`

The sentinel literal is written as follows.

```rok
none
```

`none` is the single literal representing the absence of a value. It occupies the place of `null`, `nil`, `None` and `undefined` in other languages, and is reserved as a keyword, so it cannot be used as an identifier. Which sum type's empty slot it fills is determined by context; its semantics are defined in the type system specification.

## 1.9.3. Lexical classification

The three literals belong to the boolean / sentinel literal class of [Lexical structure](102-lexical-structure.md). They are classified separately from the keyword token kind because they are tokens that appear in value position, and the parser accepts them where a literal may appear.

At the lexical stage only the following is guaranteed: `true`, `false` and `none` are always separated as the same token kind and are never reinterpreted as identifiers or other keywords.

## 1.9.4. Literals not adopted

The following literals common in other languages are not introduced as separate tokens at the lexical stage. Floating-point infinity and NaN are provided as standard library constants, and the other forms of "emptiness", `undefined`, `null` and `nil`, are all unified into `none`. The **unit** value (`()`) appears not at the lexical stage but at the expression stage, as empty tuple notation.

Fewer literal kinds mean a lower learning cost and a smaller tooling surface.
