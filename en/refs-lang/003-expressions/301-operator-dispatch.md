# 3.1. Operator dispatch

This chapter defines the general principle by which the operator tokens defined in [punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md) are converted into trait method calls. The concrete matrix of which operator maps to which method of which trait is defined by the standard library.

## 3.1.1. Every operator is a trait method

As established in [the boundary between the language and the standard libraries](../000-meta/004-language-vs-standard-libraries.md), the language provides operator syntax, but the behavior is determined by standard library traits.

```
a + b         →   <typeof(a) as Add<typeof(b)>>::add(a, b)
a - b         →   <typeof(a) as Sub<typeof(b)>>::sub(a, b)
a * b         →   <typeof(a) as Mul<typeof(b)>>::mul(a, b)
a < b         →   <typeof(a) as Cmp<typeof(b)>>::lt(a, b)
-a            →   <typeof(a) as Neg>::neg(a)
!a            →   <typeof(a) as Not>::not(a)
```

Every operator follows the same pattern: the language recognizes the token and converts it into a trait method call.

## 3.1.2. Operator → trait mapping lives in the standard library

The exact table of which trait method each operator token maps to is defined by the standard library. This document is responsible only for the existence of the mapping and the dispatch mechanism.

The standard library attaches mappings to trait methods through the meta-system attributes `#[infix]` / `#[prefix]` / `#[postfix]`. The exact mechanism is defined in [`#[infix]` / `#[prefix]` / `#[postfix]` magic](../006-meta-system/607-operator-magic.md).

```
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
```

The example shows the shape of the traits and method signatures; their meaning arises from the trait specification in Chapter 2 combined with the magic specification in Chapter 6. Chapter 3 defines only that the language converts the token into a call like the above.

## 3.1.3. Dispatch without a vtable

Following the [trait-as-type ban](../002-types/212-trait-as-type-banned.md), operator call dispatch is designed to resolve without vtable indirection.

- For a static slot, the trait method is determined at compile time and compiled to a direct function call.
- For an `any` slot, the trait method is determined by the runtime tag check of the [`any` slot](../002-types/202-any-slot.md) and then called directly. The cost is one tag check plus a direct call.

In neither case is there an extra indirect call from a vtable lookup. Keeping the hot loops of scientific and numeric code free of that cost is the motivation for the [trait-as-type ban](../002-types/212-trait-as-type-banned.md), and this dispatch rule is its consequence.

### Selecting an impl

A binary operation `a op b` selects from the impls of that operator's trait for the left operand's type.

- If a type has several impls of the same operator trait (`Mul<i64>`, `Mul<f64>`, etc.), the impl whose `Rhs` matches the right operand's type is selected.
- If the left side is a primitive type and the right side is a struct, the impl declared for the primitive type is used (`2.0 * d` is `impl Mul<Duration> for f64`).

The same rule is not limited to operators. It applies wherever an impl is selected by the type of the value filling a trait's type parameter: an operator's `Rhs`, the argument of `T::from(x)` / `T::try_from(x)` / `Tr::f(x)`, and the receiver of `.into()` / `.try_into()` in a position with a known type. If that value is an unsuffixed literal, selection proceeds in this order:

1. The impl for the literal's default type (`i32` / `f64`), if it exists and the value fits that type.
2. Otherwise, the unique impl whose parameter type can hold the value (for integers, an in-range integer type or `int`; for reals, `f32` / `f64` where the value is finitely representable, or `float`). `d * 2` is `Duration`'s `Mul<i64>`.
3. If several such impls exist, a compile error requiring a suffix.

This rule applies only to bare literals and signed literals (`-1`), not to expressions such as `1 + 1`. The `Self` of a static trait method called as `Tr::f(..)` is determined from the expected type. When no argument fills the trait's parameter, the call's expected type selects the impl (or bound) whose return type it is; otherwise the call is ambiguous ([4.9.6](../004-functions/409-method.md#496-call-forms-of-free-functions-and-methods)).

## 3.1.4. Operations without a mapping

If no trait implementation exists for a given pair of types, the operator call is a compile error.

```
let a: i32 = 1i32;
let b: str = "hi";
let c = a + b;          // ERROR: no Add impl for (i32, str)
```

The diagnostic states which trait is missing.

```
ERROR: cannot apply '+' to 'i32' and 'str'
       (no impl Add<str> for i32)
```

## 3.1.5. Unary, binary, postfix

Operators are classified by position into one of three kinds.

| Position              | Form     | Mapping mechanism  |
|-----------------------|----------|--------------------|
| binary (infix)        | `a OP b` | `#[infix("OP")]`   |
| prefix unary          | `OP a`   | `#[prefix("OP")]`  |
| postfix unary         | `a OP`   | `#[postfix("OP")]` |

The same token may appear in several positions. For example, `-` is used both as binary (`a - b`) and as prefix unary (`-a`). Position is determined by the context in which the token appears: with an operand only before it, it is postfix; with operands on both sides, it is binary.

## 3.1.6. Precedence and associativity

Besides positional dispatch, operators have precedence and associativity. These are defined in [precedence and associativity](302-precedence-and-associativity.md).

Precedence and associativity are not part of the token → trait mapping. The table in 3.2 fixes them at the language level; a registration such as `#[infix("+")]` only connects a token to a trait method.

## 3.1.7. User-Defined operators

The standard library gives meaning to the basic operators through `#[infix]` / `#[prefix]` / `#[postfix]`. User code may use the same mechanism to attach new mappings to additional traits.

```
#[infix("⊕", like = "+")]          // the new token follows the level of an existing token
fn combine(self, rhs: Rhs) -> Output;
```

A mapping for the same (token, position, type pair) must be unique; a conflict is a compile error.

To register a character sequence not in the fixed token table of [punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md) as a new operator, the token itself must be registered through the same mechanism. The exact procedure is defined in [`#[infix]` / `#[prefix]` / `#[postfix]` magic](../006-meta-system/607-operator-magic.md).

## 3.1.8. Unicode operators

For operators outside the ASCII range, such as `≤`, `≥`, `≠`, `⊕`, `∈`, the standard library may register them as multiple aliases of the same trait.

```
trait Cmp<Rhs = Self> {
    #[infix("<=")]
    #[infix("≤")]
    fn le(self, rhs: Rhs) -> bool;
}
```

Multiple tokens map to the same trait method and behave identically, so either spelling gives the same result.
