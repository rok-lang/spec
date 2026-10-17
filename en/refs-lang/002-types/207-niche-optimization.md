# 2.7. Niche optimization

A niche is a bit pattern that never occurs among a type's valid bit patterns. Storing a sum type's tag in such bit patterns removes the need for a separate tag slot, producing a sum type with zero memory overhead.

This optimization works because the memory representation of [sum types](206-sum-types.md) is left to the compiler.

## 2.7.1. Motivation

```rok
let p: Ptr<u8> | none      // 64-bit pointer + none
```

This type must be representable in 8 bytes rather than 16 (pointer 8 + tag 1 + padding 7). The pointer's `null` bit pattern (0x0000_0000_0000_0000) never occurs in a valid pointer, so that position can be used for `none`.

This saving is central to memory efficiency in scientific and data code (inspired by Julia's isbits `union`).

## 2.7.2. Automatic niches

The bits of `none` are implementation-defined. A sum `T | none` consisting of one term plus `none` has the size of `T` if `T` has a niche, and `none` is represented by that niche value. Zero-filled storage reads as `none`.

| Type                                       | Niche value                           |
|--------------------------------------------|---------------------------------------|
| `bool`                                     | `2`                                   |
| `char` (compiler primitive)                | surrogate `0xD800`                    |
| Field-less `enum`, `enum` with no GC references in payloads | first unused tag     |
| `extern "C" fn(..) -> R`                   | null                                  |
| `Ptr<T>` / `ConstPtr<T>` / `ManyPtr<T>`    | null (`#[niche(0)]`)                  |
| Multi-field `struct` / tuple               | niche of the first field that has one |

- An `enum` whose payloads hold GC references has no niche.
- When the niche value is 0 (null pointer, `extern "C" fn`, `NonZero`), the term's bits are unchanged, so such a `T | none` passes across the C ABI as-is as a nullable value ([5.9.4](../005-modules/509-extern-block.md#nullable-c-function-pointers-extern-c-fn---r--none)).
- A sum with two or more terms besides `none` has a tag.

```rok
Ptr<u8> | none           // 8 bytes, null is none
bool | none              // 1 byte
```

## 2.7.3. No automatic niche

Integers (`i8` .. `i64`, `u8` .. `u64`, `usize`, `isize`) and floating-point types (`f32`, `f64`) have no niche, since every bit pattern is valid. When a niche is needed, use a wrapper (2.7.6) or `#[niche]` (2.7.4).

## 2.7.4. Opt-in niches: `#[niche(..)]`

```
niche_attr ::= "#" "[" "niche" "(" niche_arg ("," niche_arg)* ","? ")" "]"
niche_arg  ::= int_lit | "-" int_lit | const_path | range
range      ::= niche_val "..=" niche_val | ".." niche_val | niche_val ".."
const_path ::= "MIN" | "MAX" | "NAN" | "INFINITY" | "NEG_INFINITY" | type "::" const_name
```

```rok
#[niche(0)]
struct UserId(u64);

#[niche(i32::MIN..=-1)]
struct NonNegative(i32);

#[niche(NAN, INFINITY, NEG_INFINITY)]
struct Real { value: f64; }
```

- Arguments are integers, `MIN` / `MAX` (`T::MIN`), `NAN` / `INFINITY` / `NEG_INFINITY` (`F::NAN`), ranges `a..=b` / `..b` / `a..`, and several of these. They are read against the instantiated field type.
- The listed values are a promise that they are not valid values of that type. The compiler does not check this; if such a value occurs, sum type discrimination goes wrong.

## 2.7.5. Where it may be attached

`#[niche(..)]` is attached to a single-field `struct` (a tuple newtype or one named field). Attaching it to an `enum`, a `union`, or a multi-field `struct` is an error.

## 2.7.6. Standard library wrappers

| Name          | Excluded values                       | Definition                    |
|---------------|---------------------------------------|-------------------------------|
| `NonZero<T>`  | 0 (integer T only)                    | [stdlib 2.2.2](../../refs-standard-libraries/200-core/202-non-zero.md#222-nonzerot--zero-forbidden) |
| `NonNan<F>`   | NaN                                   | [stdlib 2.2.3](../../refs-standard-libraries/200-core/202-non-zero.md#223-nonnanf--nan-forbidden) |
| `Finite<F>`   | NaN, ±∞                               | [stdlib 2.2.4](../../refs-standard-libraries/200-core/202-non-zero.md#224-finitef--nan-and--forbidden) |
| `Positive<T>` | Negatives (0 included). Unsigned `T` has no niche (`Positive<u32> \| none` is 8 bytes) | [stdlib 2.2.5](../../refs-standard-libraries/200-core/202-non-zero.md#225-positivet--negative-values-forbidden) |
