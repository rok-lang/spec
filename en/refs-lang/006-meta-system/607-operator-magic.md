# 6.7. Operator magic

The three attributes `#[infix]`, `#[prefix]`, and `#[postfix]` only connect operator tokens to trait methods. Precedence and associativity are fixed at the language level by the table in [3.2.3](../003-expressions/302-precedence-and-associativity.md#323-precedence-and-associativity-table) and cannot be set or changed by these attributes. They are core magics required for the desugaring mechanism of [operator dispatch](../003-expressions/301-operator-dispatch.md) to work.

```
infix_attr      ::= "#" "[" "infix" "(" string_lit ("," like_arg)? ("," "swap")? ")" "]"
prefix_attr     ::= "#" "[" "prefix" "(" string_lit ("," like_arg)? ")" "]"
postfix_attr    ::= "#" "[" "postfix" "(" string_lit ("," like_arg)? ")" "]"
like_arg        ::= "like" "=" string_lit
```

## 6.7.1. The three attributes

First, an example of how the three attributes are used.

```rok
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}

trait Neg {
    type Output;
    #[prefix("-")]
    fn neg(self) -> Output;
}

trait Degrees {
    #[postfix("°", like = "?")]    // extension token — same level as postfix ?
    fn degrees(self) -> f64;
}
```

Where each attribute is attached and which form of operation it expresses:

| Attribute             | Position      | Form                     |
|-----------------------|---------------|--------------------------|
| `#[infix(sym)]`       | trait method  | binary (`a SYM b`)       |
| `#[prefix(sym)]`      | trait method  | prefix unary (`SYM a`)   |
| `#[postfix(sym)]`     | trait method  | postfix unary (`a SYM`)  |

## 6.7.2. Arguments

The arguments each attribute takes:

| Argument | Meaning                                                            | Required?                                    |
|----------|--------------------------------------------------------------------|----------------------------------------------|
| `sym`    | Operator token (string literal)                                    | Required                                     |
| `like`   | Existing token the new token follows (its level and associativity) | Required, only for tokens not in the 3.2 table |
| `swap`   | Connects with operands swapped (infix only)                        | Optional                                     |

`swap` connects a binary operation with its two operands swapped. `contains` with `#[infix("∈", swap)]` desugars `a ∈ b` into `b.contains(a)`. It exists for operators whose container is not on the left (`∈ ∉`).

`like` cannot be written for tokens in the 3.2 table. There are no arguments specifying precedence or associativity (`prec`, `left` / `right` / `none`, etc.); writing one is a compile error.

```rok
#[infix("+")]
#[prefix("-")]
#[infix("⊕", like = "+")]      // new token — same level and associativity as +
```

## 6.7.3. Connection mechanism

How the compiler behaves when `#[infix("+")]` is attached to the trait method `add`:

First, the `+` token is bound to that trait as its connection point. The compiler then desugars the expression `a + b` into `<typeof(a) as the_trait<typeof(b)>>::add(a, b)`. Precedence and associativity are the values fixed for `+` in the table of [precedence and associativity](../003-expressions/302-precedence-and-associativity.md) (7, left-associative).

All three attributes follow the same pattern; what is bound is only the token, the position, and the trait method.

## 6.7.4. Unicode and multiple aliases

Attaching several `#[infix]` to the same method maps several tokens to that method.

```rok
trait Cmp<Rhs = Self> {
    #[infix("<=")]
    #[infix("≤")]
    fn le(self, rhs: Rhs) -> bool;

    #[infix(">=")]
    #[infix("≥")]
    fn ge(self, rhs: Rhs) -> bool;

    #[infix("==")]
    #[infix("≡")]
    fn eq(self, rhs: Rhs) -> bool;
}
```

This ties ASCII and Unicode tokens together with the same meaning. The 3.2 table places `≤`, `≥`, `≠`, `≡` at the same level as their corresponding ASCII tokens.

## 6.7.5. Conflict checking

If different traits are attached to the same (token, position) combination, it is an ERROR at registration.

```rok
trait MyAdd { #[infix("+")] fn add(...) ... }
// another library / module.
trait Add { #[infix("+")] fn add(...) ... }
// → two traits for the binary '+' position. ERROR.
```

Each (token, position) combination admits exactly one registration.

## 6.7.6. Multiple positions of the same token

A token usable in both prefix and binary positions, such as `-`, is treated as separate registrations.

```rok
trait Sub<Rhs = Self> {
    type Output;
    #[infix("-")]
    fn sub(self, rhs: Rhs) -> Output;
}

trait Neg {
    type Output;
    #[prefix("-")]
    fn neg(self) -> Output;
}
```

For example, in `-a + b` the prefix `-` binds tighter than the binary `+`, so it is parsed as `(-a) + b`.

Binary `-` and prefix `-` are separate mappings, so attaching them to different traits is no problem.

## 6.7.7. Dot-prefixed (broadcast) mappings

The dot-prefixed tokens of [broadcast operators](../003-expressions/303-broadcast-operators.md) are registered through the same magic.

```rok
trait DotAdd<Rhs = Self> {
    type Output;
    #[infix(".+")]
    fn dot_add(self, rhs: Rhs) -> Output;
}

trait DotMul<Rhs = Self> {
    type Output;
    #[infix(".*")]
    fn dot_mul(self, rhs: Rhs) -> Output;
}
```

Only the token differs; the registration mechanism is exactly the same.

## 6.7.8. Visibility of mappings

A registration through attributes such as `#[infix]` takes effect only where the trait is visible. Visible means accessible under [5.6](../005-modules/506-visibility.md): a `pub` trait reachable by path suffices; no `use` is needed. Where the trait is not visible, the operator does not work either.

This rule fits library isolation: a user-defined operator trait defined in a package works only in code that imports that package.

The standard library's operator traits (`Add`, `Sub`, `Mul`, ...) live in `rok::core::ops`. Since the compiler implicitly applies `use rok::core::**;` to every source ([standard library tiers](../000-meta/005-standard-libraries-tiers.md)), arithmetic and comparison expressions work without a separate import.

## 6.7.9. User-defined operators

Users can also define operator mappings by attaching attributes such as `#[infix]` to their own traits.

```rok
trait CustomCombine<Rhs = Self> {
    type Output;
    #[infix("⊕")]
    fn combine(self, rhs: Rhs) -> Output;
}

impl CustomCombine for MyType { ... }

let r = a ⊕ b;            // user token
```

A token not in the 3.2 table, such as `⊕`, names one existing token with `like = "+"` and follows that token's level and associativity. New levels cannot be created. The token must satisfy the extended-token conditions of [punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md#1104-extension-tokens): every character must be Unicode `Pattern_Syntax`, and it is an error if it equals an existing fixed token.

## 6.7.10. Precedence and associativity levels

The single authoritative location for operator levels is the [3.2.3 precedence table](../003-expressions/302-precedence-and-associativity.md#323-precedence-and-associativity-table). Neither the standard library nor user code can set or change levels; a new token attaches to one of the existing levels via `like`.

## 6.7.11. When it is evaluated

Registrations such as `#[infix]` are processed in the meta-expansion phase right after parsing. All subsequent expression parsing consults the registration table.

The order of registration and use within the same compilation unit therefore does not matter, because the scanner pre-pass collects them in advance. The precedence of tokens in the 3.2 table is fixed from the start, independent of registration.

## 6.7.12. No aliases at different levels

All aliases attached to the same trait method must be tokens of the same level.

```rok
trait Cmp {
    #[infix("<")]
    #[infix("⊲", like = "+")]    // ERROR — < is level 4, + is level 7
    fn lt(self, rhs: Self) -> bool;
}
```

This restriction stems from the assumption that aliases share the same meaning.

## 6.7.13. Attachment only to trait methods

`#[infix]`, `#[prefix]`, and `#[postfix]` can be attached only to method signatures of a trait. They cannot be attached to free functions, `impl` methods, or ordinary methods outside a trait.

```rok
fn my_add(a: i32, b: i32) -> i32 { a + b }
#[infix("+++")]                              // ERROR — cannot attach to a free function
fn my_concat(a: str, b: str) -> str { ... }
```

The reason is to bind operator mappings to the trait system and guarantee consistent dispatch.

## 6.7.14. Summary

Operator magic, together with `#[lang]`, is the core mechanism that implements the binding of language tokens to library traits in the [boundary between language and standard library](../000-meta/004-language-vs-standard-libraries.md).

The standard library's operator specification determines the meaning of every token through this magic.
