# 2.6. Sum types (`A | B`)

A sum type is a tagged union whose value is one of several existing types. It is similar to `enum`, but its variants are not separately named; existing types are used directly as variants.

## 2.6.1. Form

Examples of sum types:

```rok
i32 | f32
T | none
A | B | C
i32 | str | bool
```

Terms are separated by the `|` token of [Punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md). A term may be any type expression: a simple name, a path, a generic, or another sum type.

## 2.6.2. Meaning

A sum type value belongs to one of the terms, and which term it belongs to is tracked by a tag.

```rok
let x: i32 | f32 = 42i32;       // tag is at the i32 position
let y: i32 | f32 = 3.14f32;     // tag is at the f32 position
```

## 2.6.3. Memory representation

The default representation places the tag and payload inline.

```
+------+----------------------+
| tag  | payload (size of     |
|      |  largest variant)    |
+------+----------------------+
```

The tag identifies which term is active. The payload is sized to the largest of all terms, and the alignment is the largest alignment among all terms.

This representation is subject to [niche optimization](207-niche-optimization.md), so the tag slot may disappear.

### Value recursion and size limits

- A recursive type that contains itself by value (even through structs, enums, tuples, sums, arrays or generic instances) is an error at the declaration site. Use GC reference types (`List<T>` etc.) for recursion.
- Rok's implementation limit (implementation-defined, [0.4.4](../000-meta/004-language-vs-standard-libraries.md#044-implementation-limits)): a value type exceeding 65536 flattened scalar slots or 4 GiB is an error.

The exact placement is up to the implementation, but one thing is guaranteed: zero-filled `T | none` storage (elements of `GcBuf<T | none>` and collection buffers, fields of newly allocated GC objects) reads as `none`.

### Exact bit width

The tag's bit width is decided by the compiler and may be 1, 2, 4, 8 bits and so on, depending on the number of terms. User code has no way to access the tag bits directly.

## 2.6.4. Set semantics

> [!IMPORTANT]
> Sum types have the semantics of set operations.

### Normalization

Sum types are always handled in normal form.

- Nested sums are flattened. `(A | B) | C`, `A | (B | C)` and `A | B | C` are the same type.
- Duplicates are removed. `A | A | B` is `A | B`.
- Term order is irrelevant. `A | B` and `B | A` are the same type.
- If one term remains, it is that type itself. `A | A` is `A`.

The result of expanding a type alias is normalized again (with `type N = i32 | f64;`, `N | i32` is `i32 | f64`). Substitution of generic parameters follows "Layers of generic sums" below.

So sum types of the same shape are the same type, and the user need not care whether `i32 | f32` or `f32 | i32` was written.

### Layers of generic sums

A sum with a generic term, `T | X1 | ... | Xn` (e.g. `T | none`, `T | ParseErr`), semantically has **its own layer**. `X1 ... Xn` are the non-generic terms written in that sum. Generic terms are these four:

- type parameters (`T`)
- `Self`
- associated types (`Self::X`, `T::X`, `<T as Tr>::X`)
- parameters of generic type aliases

The layering rule applies when a generic term is replaced by an actual type. If the substituted type overlaps a co-written term, a nested layer results; if not, it is flattened. Below, `T` stands for one generic term.

**When a layer remains.** If the type substituted for `T` shares any term with `X1 ... Xn`, the substituted `T` remains a single term and becomes a **nested layer**. If there is no overlap, it is flattened. Flattening loses no information distinguishing values, so the result appears and is treated as a flat sum.

| Generic sum          | `T` substitution     | Result                                              |
|----------------------|----------------------|-----------------------------------------------------|
| `T \| none`          | `i32 \| str`         | `i32 \| str \| none` (no overlap, flattened)         |
| `T \| none`          | `i32 \| none`        | `(i32 \| none)?` (two layers)                        |
| `T \| ParseErr`      | `i32 \| ParseErr`    | `(i32 \| ParseErr) \| ParseErr`                      |
| `T \| str \| none`   | `i32 \| str`         | `(i32 \| str) \| str \| none` (`str` overlaps)       |

- Overlap counts any term, not only `none`.
- A sum of only two parameters, `T | U`, has both substituted, so it is flattened even if they overlap, not nested. If the two values must be distinguished, use an `enum`.
- Each layer is normalized separately, so `A | B == B | A` holds at every layer.
- A sum written in the source with concrete types is always flattened. Writing `(i32 | none) | none` gives `i32 | none`.
- A non-generic type alias is a written type, so it is always flattened. With `type Opt = i32 | none; type Flat = Opt | none;`, `Flat` is `i32 | none`.
- Parameters of a generic alias are generic terms and follow the rules above (see "Writing nested types" below).

**Writing nested types.** An optional is written with postfix `?` ([2.1.5a](201-type-expressions.md#215a-optional-types-t)). `T?` is the same as `Maybe<T>`, so its operand is a generic term and the same layering rule applies. Other nestings (`(i32 | E) | E` etc.) are written with a generic alias or `Self::X`.

```rok
type Maybe<T> = T | none;
let r: (i32 | none)? = xs.get(0);       // two layers
let q: Maybe<i32 | none> = xs.get(0);   // same type
let p: i32? = 1;                         // i32 | none (no overlap)
let f: (i32 | str)? = none;              // i32 | str | none
```

**Storing values.** When a value is stored into a nested sum, a value of one of `T`'s term types goes into the inner layer. A value of an outer-layer term type (the `none` literal, a `ParseErr` value, etc.) is a value of the outer layer. To put an outer-term-type value into the inner layer, first place it in a slot of type `T`, then store that.

**Conversion.** A nested type is not implicitly converted to a flat type; it is a type error.

```rok
let v: i32 | none = xs.get(0);    // ERROR: xs: List<i32 | none>
// mismatched types: `let v` expects `i32 | none`, found `(i32 | none)?`
```

**Operations.** Patterns, `is`, `?.`, `??` and `?` act on the **outermost layer**. The other side of the outer layer is bound to the inner `T`. A binding pattern after arms that cover every outer-layer term binds the inner layer.

**Diagnostic notation.** Diagnostics show nested layers in parentheses. If the outer layer is only `none`, it is written `(i32 | none)?`; otherwise as `(i32 | E) | E`, `(i32 | none) | str | none`. `(i32 | none)?` is also valid source with the same meaning.

```rok
struct ParseErr { msg: str; }

fn check<T>(x: T) -> T | ParseErr {
    if bad(x) { return ParseErr { msg: "bad" }; }   // ParseErr of the outer layer
    x                                                // a value of T: inner layer
}

let a = check(1i32);                  // T = i32, no overlap: i32 | ParseErr
let v: i32 | ParseErr = ParseErr { msg: "earlier" };
let r = check(v);                     // T = i32 | ParseErr, overlap: (i32 | ParseErr) | ParseErr
match r {
    e: ParseErr => print(f"check failed: {e.msg}"),        // outer layer: error made by check
    inner => match inner {                                // inner: i32 | ParseErr
        e: ParseErr => print(f"original value was an error: {e.msg}"), // inner layer: error that was in v
        n: i32 => print(n),
    },
}
```

**Associated types.** Inside a trait and all its impls, both the bare name `X` and `Self::X` denote the associated type ([2.8.3](208-trait.md#283-associated-types)). Associated types are generic terms, so the same rule applies. `fn next(ref self) -> Self::Item?` nests when `Item` contains `none`, so a user iterator yielding `i32 | none` elements also iterates over stored `none`s as elements.

For the `none` case (nested optional), see the example in 2.6.5.

### Canonical order

Terms of the normal form have a defined order.

- `none` is always last.
- The remaining terms are ordered by a structural key: a primitive by its name, a struct or enum by its full path, a generic instance by `path<arg keys>`. Nested sums are flattened (Normalization above), so a sum is never a term of a sum; a sum inside a generic argument is keyed by its terms in canonical order (`List<A | B>` has the key `List<key(A)|key(B)>`). A layer kept by "Layers of generic sums" is a single term whose key is `(` + its terms' keys in canonical order joined by `|` + `)`. The order is independent of the written order.
- The order, and so the tags, never depend on a build's internal type numbering, so a prebuilt library and its users agree ([9.4.8](../009-tools/904-rokhir-bytecode.md#948-rules-for-separate-compilation)).

`PartialOrd` / `Ord` and `min` / `max` / `clamp` on sum type values follow this order ([stdlib 2.5.7](../../refs-standard-libraries/200-core/205-cmp.md#257-implementation-matrix)). Values of different terms are compared by canonical order; values of the same term by that term type's `Ord`. So in `T | none`, `none` is greater than any `T` value.

### Subsets

- A sum type value fits as-is into a larger sum type that contains all its terms. An `i32 | none` value can be placed in an `i32 | str | none` position. The same holds for a value of a single term (`i32`).
- A type pattern `v: A | B` can match a subset of the terms of a larger sum. In that arm, `v` is `A | B`.

```rok
let v: i32 | str | f64 | none = ...;
match v {
    n: i32 | f64 => use_num(n),     // n: i32 | f64
    s: str => use_str(s),
    none => {},
}
```

### Generic inference

Generic inference treats sums as sets. Matching `T | none` against `i32 | str | none` collects the remaining terms: `T = i32 | str`.

```rok
fn unwrap_or<T>(x: T | none, d: T) -> T { ... }
let v: i32 | str | none = ...;
unwrap_or(v, 0i32);       // T = i32 | str
```

## 2.6.5. Combining with `none`

The `none` of [Boolean and sentinel literals](../001-lexical/109-other-literals.md) most often appears as a term of a sum type.

```rok
T | none
```

This form expresses that a value may or may not be present, corresponding to `Option<T>`, `T?` or `Maybe T` in other languages.

Combined with niche optimization (e.g. for a pointer-like T), null can represent `none` with no memory overhead. The exact rules are defined in [Niche optimization](207-niche-optimization.md).

### Nested optional

This is the "Layers of generic sums" rule of 2.6.4 applied to `none`. If the `T` of a generic `T | none` already contains `none`, the outer `none` layer remains and it becomes a **nested optional** (like Swift's `T??`). `Maybe<i32 | none>`, i.e. `Maybe<T> = T | none` with `T = i32 | none`, which is `(i32 | none)?`, has two layers. The `none` pattern, `is none`, `?.`, `??` and `?` act on the outer layer, and the non-none side is bound to the inner `T` (`i32 | none`).

So `List<i32 | none>` iterates over stored `none`s as elements, and `get` / `pop` / `Dict` lookups distinguish "absent" from "stored none".

```rok
let xs: List<i32 | none> = [1, none, 3];

for x in xs { print(f"{x:?}"); }  // 1, none, 3 (stored none is also an element)

match xs.get(1) {                // get: (i32 | none) | none (two layers)
    none => print("out of range"),     // outer none: no element
    v => match v {               // v: i32 | none (stored value)
        none => print("stored none"),
        n: i32 => print(n),
    },
}

let d: Dict<str, i32 | none> = {"a": none};
d.get("a") ?? 0i32;              // none: key present, value is none (?? strips only the outer layer)
d.get("b") ?? 0i32;              // 0: key absent
```

## 2.6.6. Difference from `any`

The `any` of [The `any` slot](202-any-slot.md) is an open slot: any type is possible and variants can be added freely.

A sum type (`A | B`), by contrast, is a closed slot: only the listed variants are possible. In return, the compiler knows every possible term and can check pattern matches for exhaustiveness.

| Aspect          | `any`                                  | `A \| B`                                       |
|-----------------|----------------------------------------|-----------------------------------------------|
| Possible types  | All types                              | Only the listed terms                         |
| Dispatch        | Runtime tag check                      | Runtime tag check (fewer terms, fewer branches) |
| Exhaustiveness  | Not possible                           | Possible                                      |
| Memory          | Tag + payload (general)                | Tag + largest term (tighter)                  |
| Gradualness     | Gradual (new types OK without code changes) | Static (adding a variant changes the type) |

## 2.6.7. Difference from `enum`

In an `enum`, named variants are defined inside the container, and the variants are part of the `enum`.

A sum type uses existing types directly as variants, so there are no separate variant names.

```rok
enum Result { Ok(i32), Err(str) }       // variant names Ok / Err are defined inside the enum
type Either = i32 | str;                  // no variant names: just i32 or str
```

Choose an `enum` when the variants carry distinct meanings and need names. Choose a sum type when combining existing types as they are is the natural fit.

## 2.6.8. Narrowing

A sum type value can be narrowed to a term by pattern matching or an `is` check, in the same way as [The `any` slot](202-any-slot.md).

```rok
let v: i32 | str = ...;
match v {
    n: i32 => ...,           // inside this arm v is i32
    s: str => ...,           // inside this arm v is str
}
```

The exact narrowing syntax is defined in the pattern and expression specifications.

## 2.6.9. Combining with type aliases

A long sum type can be given an alias with `type` (`type Num = i32 | f64;`). A cyclic type alias (even through generic arguments) is an error. The `where` of a type alias goes after the aliased type (`type A<T> = B<T> where T: X;`). An alias's `where` is checked at every use of the alias. When a use's arguments name type parameters, the bounds in scope at the use decide it, as in a generic body ([2.10.9](210-generics.md#2109-body-checking)): the enclosing function's, `impl`'s or trait's parameter bounds and `where` clauses, a trait's supertraits and associated-type bounds. With `type W<T> = S<T> where T: Num;`, `fn get<T: Num>(x: W<T>)` and `impl<T: Num> W<T> { .. }` are valid; `fn get<T>(x: W<T>)` is an error even if never called. The same holds in declaration types: a `struct`, `enum` or `union` field, a trait `impl`'s trait arguments and associated types, and a trait method's signature are rejected when the alias's `where` is not met, even if nothing uses them; the help names the generics to add the bound to (``add `T: Num` to `F`'s generics``). `type` is a contextual keyword, a keyword only at the start of an item when followed by an identifier and `=` or `<` ([1.5.4](../001-lexical/105-keywords.md#154-contextual-keywords)). The exact alias syntax is defined in the type alias specification.

```rok
type Number = i32 | i64 | f32 | f64;
type Maybe<T> = T | none;
```
