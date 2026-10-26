# 0.4. `#[lang]` slot mapping table

The `#[lang(slot_name)]` attribute ([refs-lang 6.4](../../refs-lang/006-meta-system/604-lang-magic.md)) registers a standard library item in a **slot** that the compiler can look up. This chapter collects in one place, in tabular form, every slot name, its meaning, and the item it is attached to.

The slot table is the authoritative location in the spec; the compiler, the standard library and users all refer to it to stay consistent. If slot names drift, the contract between the language and the library breaks down, so any change starts from this table.

## 0.4.1. Slot classification

The 48 slots fall into four groups according to their responsibility. The table below shows the role and slot count of each group at a glance, indicating which category the detailed tables in the following sections belong to.

| Group   | Responsibility                                                              | Slots   |
|---------|-----------------------------------------------------------------------------|---------|
| Type    | Standard library types Rok recognizes as **concrete types**                 | 14      |
| Trait   | Traits Rok uses for **operator / control-flow desugaring**, derive, and thread-boundary checks | 30 |
| Method  | Methods Rok **calls automatically** (drop)                                  | 1       |
| Function | Standard library functions Rok calls as **intrinsic calls**                | 3       |
| Total   |                                                                             | **48**  |

## 0.4.2. Type slots (14)

Type slots indicate the concrete type the compiler lowers to when it encounters a literal or keyword. This group divides further into two kinds. First, there are the language default types exposed directly as lowercase identifiers. They need no separate keyword registration and are the primitives users encounter every day.

Language defaults (lowercase struct, no `#[keyword]` needed):

| Slot name    | Meaning                      | Standard library item    | Ref  |
|--------------|------------------------------|--------------------------|------|
| `i32_type`   | `i32` primitive              | `rok::core::num::i32`    | 2.1  |
| `i64_type`   | `i64` primitive              | `rok::core::num::i64`    | 2.1  |
| `f32_type`   | `f32` primitive              | `rok::core::num::f32`    | 2.1  |
| `f64_type`   | `f64` primitive              | `rok::core::num::f64`    | 2.1  |
| `usize_type` | `usize` primitive            | `rok::core::num::usize`  | 2.1  |
| `isize_type` | `isize` primitive            | `rok::core::num::isize`  | 2.1  |
| `bool_type`  | `bool` primitive             | `rok::core::num::bool`   | 2.1  |
| `cchar_type` | `c_char` primitive (FFI)     | `rok::core::str::c_char` | 2.15 |
| `str_type`   | `str` primitive (UTF-8 view) | `rok::core::str::str`    | 2.15 |
| `char_type`  | `char` primitive (Unicode scalar) | `char` (methods: `rok::core::str`) | 2.15 |

Second, there are the PascalCase types additionally provided by the standard library. These expose a familiar lowercase alias to users separately via `#[keyword(...)]`; items directly entangled with language syntax gather here, such as the auto-promoting `int` and `float`, the `none` sentinel, and the `Range` literal.

Standard library additions (PascalCase struct + `#[keyword(...)]`):

| Slot name    | Meaning                       | Standard library item                                     | Ref  |
|--------------|-------------------------------|-----------------------------------------------------------|------|
| `none_type`  | Type of the `none` literal    | `rok::core::marker::NoneSentinel`                         | 2.13 |
| `int_type`   | `int` auto-promote type       | `rok::core::num::Int` + `#[keyword("int")]`               | 2.1  |
| `float_type` | `float` auto-promote type     | `rok::core::num::Float` + `#[keyword("float")]`           | 2.1  |
| `range_type` | `Range<T>` (`a..b`)           | `rok::core::range::Range` + `#[literal( expr ".." expr )]` | 2.7  |

A compiler that provides the primitive types built in may leave `i32_type` ... `str_type`, `none_type` and `char_type` unattached.

## 0.4.3. Trait slots (30)

Trait slots indicate which trait the compiler goes to when user-written syntax is desugared into a standard library trait call. Core syntax such as `for` loops, indexing, slicing, formatting and `await` all resolves through the traits registered here, so this table is the thickest bridge between the language syntax and the library.

| Slot name             | Meaning                         | Standard library trait          | Ref  |
|-----------------------|---------------------------------|---------------------------------|------|
| `iterator_trait`      | Stepping in `for x in xs`       | `rok::core::iter::Iterator`     | 2.7  |
| `into_iterator_trait` | Start of `for x in xs`          | `rok::core::iter::IntoIterator` | 2.7  |
| `index_trait`         | `arr[i]` read                   | `rok::core::ops::Index`         | 2.3  |
| `index_set_trait`     | `arr[i] = v` write              | `rok::core::ops::IndexSet`      | 2.3  |
| `slice_trait`         | `s[1:3]` slicing                | `rok::core::ops::Sliceable`     | 2.3  |
| `display_trait`       | `f"{x}"` default format         | `rok::core::fmt::Display`       | 2.10 |
| `debug_trait`         | `f"{x:?}"` debug format         | `rok::core::fmt::Debug`         | 2.10 |
| `future_trait`        | Stepping in `await expr`        | `rok::core::future::Future`     | 2.20 |

Next come the operator trait slots. The dispatch target of an operator token is determined by the `#[infix]` / `#[prefix]` ([refs-lang 6.7](../../refs-lang/006-meta-system/607-operator-magic.md)) attached to each trait method, and precedence is fixed by the table in [refs-lang 3.2](../../refs-lang/003-expressions/302-precedence-and-associativity.md); the slots below are fixed locations that let the compiler find "the addition trait", "the comparison trait", etc. by name during primitive lowering and diagnostics. Compound assignment (`AddAssign`, etc., 2.4), broadcast (`DotAdd`, etc., 2.6) and floored modulo (`Modulo`, `%%`) traits need only `#[infix]` registration and have no slots.

| Slot name           | Meaning                                  | Standard library trait       | Ref  |
|---------------------|------------------------------------------|------------------------------|------|
| `add_trait`         | `a + b`                                  | `rok::core::ops::Add`        | 2.3  |
| `sub_trait`         | `a - b`                                  | `rok::core::ops::Sub`        | 2.3  |
| `mul_trait`         | `a * b`                                  | `rok::core::ops::Mul`        | 2.3  |
| `div_trait`         | `a / b`                                  | `rok::core::ops::Div`        | 2.3  |
| `rem_trait`         | `a % b`                                  | `rok::core::ops::Rem`        | 2.3  |
| `neg_trait`         | `-a` (prefix)                            | `rok::core::ops::Neg`        | 2.3  |
| `not_trait`         | `!a`                                     | `rok::core::ops::Not`        | 2.3  |
| `bitnot_trait`      | `~a`                                     | `rok::core::ops::BitNot`     | 2.3  |
| `bitand_trait`      | `a & b`                                  | `rok::core::ops::BitAnd`     | 2.3  |
| `bitor_trait`       | `a \| b`                                 | `rok::core::ops::BitOr`      | 2.3  |
| `bitxor_trait`      | `a ^ b`                                  | `rok::core::ops::BitXor`     | 2.3  |
| `shl_trait`         | `a << b`                                 | `rok::core::ops::Shl`        | 2.3  |
| `shr_trait`         | `a >> b`                                 | `rok::core::ops::Shr`        | 2.3  |
| `logical_and_trait` | `a && b`                                 | `rok::core::ops::LogicalAnd` | 2.3  |
| `logical_or_trait`  | `a \|\| b`                               | `rok::core::ops::LogicalOr`  | 2.3  |
| `eq_trait`          | `==` / `!=`, literal patterns            | `rok::core::cmp::PartialEq`  | 2.5  |
| `cmp_trait`         | `<` / `>` / `<=` / `>=`, range patterns  | `rok::core::cmp::PartialOrd` | 2.5  |
| `ord_trait`         | Total order (`#[derive(Ord)]`)           | `rok::core::cmp::Ord`        | 2.5  |
| `hash_trait`        | Hash (`#[derive(Hash)]`)                 | `rok::core::cmp::Hash`       | 2.5  |
| `clone_trait`       | Clone (`#[derive(Clone)]`)               | `rok::core::marker::Clone`   | 2.13 |
| `send_trait`        | Auto marker - safe to move across threads | `rok::core::marker::Send`   | 2.13 |
| `sync_trait`        | Auto marker - safe to share across threads | `rok::core::marker::Sync`  | 2.13 |

> [!NOTE]
> Comparison slot names and trait names do not correspond one-to-one. `eq_trait` is attached to `PartialEq`, which defines the equality operators, and `cmp_trait` to `PartialOrd`, which defines the ordering operators. The marker traits `Eq`, `Copy` and `Default` have no slots: sum types get them structurally (`T | none` is `Eq` when `T` is; its `Default` is `none`), so the compiler finds them by their fixed paths in `rok::core`.

## 0.4.4. Function slots (3)

Function slots are for standard library functions that the compiler calls directly, like intrinsics. They mainly cover abnormal termination paths - `throw`, OOM, unreachable - and are used where a uniform entry point is needed outside normal flow.

| Slot name        | Meaning                                  | Standard library function       | Ref  |
|------------------|------------------------------------------|---------------------------------|------|
| `panic_fn`       | Abort function called on panic           | `rok::core::panic::panic`       | 2.14 |
| `oom_fn`         | OOM abort (runtime allocation failure calls `rok_oom` directly) | `rok::core::panic::oom` | 2.14 |
| `unreachable_fn` | Called by the compiler at unreachable points | `rok::core::panic::unreachable` | 2.14 |

## 0.4.5. Method slots (1)

Method slots are for methods the compiler calls automatically without the user writing the call. There is only one: disposal when a `using` binding is released. The `?` operator is expanded directly by the compiler for `T | none` ([refs-lang 3.14.10](../../refs-lang/003-expressions/314-error-control-flow.md#31410-the--operator---early-return-of-none)), so it has no method slot. Cloning a value (`Clone::clone`) is never called automatically ([2.13.2](../200-core/213-marker.md#2132-clone---deep-copy)), so it is registered as the trait slot `clone_trait`, not a method slot.

| Slot name     | Meaning                              | Standard library method         | Ref  |
|---------------|--------------------------------------|---------------------------------|------|
| `drop_method` | Called when a `using` block releases | `rok::core::marker::Drop::drop` | 2.13 |

## 0.4.6. Slot consumers: where the compiler looks up slots

Where the previous sections listed "which slots exist", this section summarizes the reverse: "when the compiler looks up which slot". When tracing which standard library item a line of user syntax resolves to, the following table is the starting point.

| Compiler action                                    | Slot looked up                           |
|----------------------------------------------------|------------------------------------------|
| Lowering the `none` literal                        | `none_type`                              |
| Literal type of `42` (no expected type)            | `i32_type`                               |
| Literal type of `3.14` (no expected type)          | `f64_type`                               |
| Literal type of `"hello"`                          | `str_type`                               |
| Desugaring `for x in xs`                           | `into_iterator_trait`, `iterator_trait`  |
| Desugaring `arr[i]`                                | `index_trait`                            |
| Desugaring `arr[i] = v`                            | `index_set_trait`                        |
| Desugaring `s[1:3]`                                | `slice_trait`                            |
| Desugaring a default `f"{x}"` hole                 | `display_trait`                          |
| Desugaring `f"{x:?}"`                              | `debug_trait`                            |
| Desugaring `await expr`                            | `future_trait`                           |
| Primitive lowering of arithmetic / bitwise / logical operators | `add_trait` ... `logical_or_trait` |
| `==` / `!=`, literal pattern matching              | `eq_trait`                               |
| `<` / `>` / `<=` / `>=`, range pattern matching    | `cmp_trait`                              |
| Expanding `#[derive(Clone / Ord / Hash)]`          | `clone_trait`, `ord_trait`, `hash_trait` |
| Lowering panic (including `try!` failure)          | `panic_fn`                               |
| On OOM                                             | `oom_fn`                                 |
| Unreachable after `match` exhaustiveness check of all arms | `unreachable_fn`                 |
| Releasing a `using` block                          | `drop_method`                            |
| Automatic `Send` / `Sync` implementation, `+ Send` checks | `send_trait`, `sync_trait`        |

## 0.4.7. Duplicate / missing detection

If two standard library items are attached to the same slot, the compiler rejects it (the duplicate detection rule of refs-lang 6.4.6). Conversely, if a required slot such as `iterator_trait` is missing, compilation of the features that depend on it - here, `for` loops - is rejected. Both are devices that force the slot contract to have a single source of truth.

The actual attached items are stated in each chapter of [200-core/](../200-core/). Every slot in this table has its `#[lang(...)]` attachment site in the chapter indicated by the "Ref" column. A slot name not in this table is a compile error per [refs-lang 6.4.8](../../refs-lang/006-meta-system/604-lang-magic.md) - for example, `Stream` has no slot ([2.8.10](../200-core/208-stream.md)).

## 0.4.8. Slot naming policy

Finally, the naming rules to follow when adding a new slot or renaming an existing one. Consistent names let the kind of a slot be guessed just by scanning the table, so the policy is not taken lightly.

Names are written in snake_case: type slots take the `_type` suffix, trait slots the `_trait` suffix, function slots the `_fn` suffix, and method slots the `_method` suffix (`drop_method`). A slot designating a whole trait is unified under a single `_trait` even if the compiler handles several methods or derives of that trait; method slots are limited to cases where the compiler **automatically calls one specific method**. By this criterion, `Clone` is registered as `clone_trait`, not `clone_method`. The suffix also determines the attachment site: a `_trait` slot is attached to **the trait declaration itself**, a `_method` slot to **the corresponding method inside the trait**, a `_type` slot to a type declaration, and a `_fn` slot to a function declaration; attaching it elsewhere is rejected by the signature check of [refs-lang 6.4.7](../../refs-lang/006-meta-system/604-lang-magic.md).
