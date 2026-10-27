# 6.4. `#[lang(...)]` magic

The `#[lang(slot_name)]` attribute connects a standard library item to a slot the compiler recognizes directly. It is the channel through which the compiler and runtime learn which item carries which meaning.

## 6.4.1. Motivation

As described in the [boundary between language and standard library](../000-meta/004-language-vs-standard-libraries.md), the language provides only syntax such as `+` and `-`, and the standard library decides its meaning. For this model to work, the compiler must know which library trait or type is connected to which language meaning.

```
compiler slot           ← language-known anchor
                          (e.g. "add trait", "option slot", "iterator trait")

library item            ← trait / type / fn defined by standard libraries

  ←── #[lang(slot)] connects the two sides ──→
```

## 6.4.2. Form

```
lang_attr       ::= "#" "[" "lang" "(" string_lit ")" "]"
```

```rok
#[lang("add_trait")]
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}

#[lang("iterator_trait")]
trait Iterator { ... }

#[lang("display_trait")]
trait Display { ... }              // the default f"{x}" format
```

It is written `#[lang("slot_name")]`, taking the slot name as a string literal argument. It can be attached to items such as traits, types, functions, `const`, and `static`.

## 6.4.3. Classification of slot names

Slot names form a fixed set known to the compiler in advance; users cannot create new slots.

Representative slots:

| Slot                    | Kind of item connected | Compiler use                      |
|-------------------------|------------------------|-----------------------------------|
| `"add_trait"`           | trait                  | `+` operator desugar              |
| `"sub_trait"`           | trait                  | `-` operator desugar              |
| `"mul_trait"`           | trait                  | `*` operator desugar              |
| ...                     | ...                    | ...                               |
| `"none_type"`           | type                   | Type of the `none` literal        |
| `"iterator_trait"`      | trait                  | `for` loop desugar                |
| `"into_iterator_trait"` | trait                  | `for` loop desugar                |
| `"future_trait"`        | trait                  | `await` desugar                   |
| `"index_trait"`         | trait                  | `[i]` indexing desugar            |
| `"slice_trait"`         | trait                  | `[1:3]` slicing desugar           |
| `"display_trait"`       | trait                  | `f"{x}"` formatting               |
| `"debug_trait"`         | trait                  | `f"{x:?}"` debug format           |
| `"clone_trait"`         | trait                  | `#[derive(Clone)]`                |
| `"eq_trait"`            | trait                  | `==` / `!=` operators (`PartialEq`) |
| ...                     | ...                    | ...                               |

The full slot list and the item attached to each slot are given authoritatively by the [standard library 0.4 `#[lang]` slot mapping table](../../refs-standard-libraries/000-meta/004-lang-slot-table.md); the table above is an excerpt. Changes to the slot set start from the 0.4 table.

## 6.4.4. Effect of the mapping

When a trait carries `#[lang("add_trait")]`, the compiler connects that trait to the meaning of addition. A subsequent `+` operator desugars as:

```
a + b    →    <typeof(a) as the_trait<typeof(b)>>::add(a, b)
```

Here `the_trait` is the trait carrying `#[lang("add_trait")]`.

## 6.4.5. Use in user code (rare)

`#[lang]` is used almost exclusively by the standard library. User code attaches it directly only when building its own library in a _no_std_ environment, or in environments such as embedded systems or operating systems where the language core must be redefined. Ordinary users never need to attach `#[lang]` directly.

## 6.4.6. One item per slot

Attaching the same slot name to several items is a compile error.

```rok
#[lang("add_trait")] trait MyAdd { ... }
#[lang("add_trait")] trait OtherAdd { ... }    // ERROR: duplicate slot
```

Each slot maps to exactly one item. This uniqueness holds program-wide (across all linked packages). A user package therefore cannot reattach and replace a slot defined by `rok::core` (`panic_fn`, `oom_fn`, `unreachable_fn`, etc.); doing so is a compile error.

## 6.4.7. Signature checking of slots

Each slot predetermines the signature of the item it accepts.

```rok
#[lang("add_trait")]
trait Add<Rhs = Self> {              // the form the slot requires
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}
```

If the signature does not match the form the slot requires, it is a compile error.

```rok
#[lang("add_trait")]
trait BadAdd { fn run() }            // ERROR: does not match the form of the 'add_trait' slot
```

## 6.4.8. Unknown slot names

```rok
#[lang("nonexistent_slot")]
trait T { ... }
```

Using an unknown slot name is likewise a compile error.

## 6.4.9. Stability of slots

The set of slot names is itself part of the language specification, so adding, changing, or removing a slot changes the language version (SemVer).

## 6.4.10. Difference between magic and ordinary attributes

| Aspect        | `#[derive]`, `#[inline]`, etc.          | `#[lang]`                       |
|---------------|-----------------------------------------|---------------------------------|
| Attached by   | Users                                   | Standard library (mostly)       |
| Effect        | Code generation or compiler hint        | Binds to language meaning       |
| Slot or key   | Free (identified by attribute name)     | Fixed slot set                  |

`#[lang]` is the most privileged attribute in the meta-system. The other magics (`#[literal]`, `#[keyword]` and the operator magics) can all be viewed as refined forms of `#[lang]`.

## 6.4.11. Relation between `#[lang]` and other magic

Operator magics (`#[infix]` etc.) are usually attached to methods of a trait carrying `#[lang(...)]`.

```rok
#[lang("add_trait")]
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}
```

Here `#[lang("add_trait")]` declares that this trait is where the meaning of addition lives, and `#[infix("+", ...)]` marks the method as the dispatch target of the `+` token. Working together, the two magics connect the language and the library in both directions.

`#[literal]` and `#[keyword]` can likewise be understood as tools connected to the literal position and the keyword registration slot, respectively.
