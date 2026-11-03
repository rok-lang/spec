# 0.4. `#[lang]` slot mapping table

The `#[lang(slot_name)]` attribute ([refs-lang 6.4](../../refs-lang/006-meta-system/604-lang-magic.md)) registers a standard library item in a **slot** that the compiler can look up. This chapter collects in one place, in tabular form, every slot name, its meaning, and the item it is attached to.

The slot table is the authoritative location in the spec; the compiler, the standard library and users all refer to it to stay consistent. If slot names drift, the contract between the language and the library breaks down, so any change starts from this table.

## 0.4.1. Slot classification

The 67 slots fall into four groups according to their responsibility. The table below shows the role and slot count of each group at a glance, indicating which category the detailed tables in the following sections belong to.

| Group   | Responsibility                                                              | Slots   |
|---------|-----------------------------------------------------------------------------|---------|
| Type    | Standard library types Rok recognizes as **concrete types**                 | 14      |
| Trait   | Traits Rok uses for **operator / control-flow desugaring**, derive, and thread-boundary checks | 32 |
| Method  | Methods Rok **calls automatically** (drop)                                  | 1       |
| Function | Standard library functions Rok calls as **intrinsic calls**, and the hooks lowered code calls | 20 |
| Total   |                                                                             | **67**  |

## 0.4.2. Type slots (18)

Type slots indicate the concrete type the compiler lowers to when it encounters a literal or keyword. This group divides further into two kinds. First, there are the language default types exposed directly as lowercase identifiers. They need no separate keyword registration and are the primitives users encounter every day.

Language defaults (built-in primitives, no `#[keyword]` needed):

| Slot name    | Meaning                      | Standard library item    | Ref  |
|--------------|------------------------------|--------------------------|------|
| `i32_type`   | `i32` primitive              | built-in (methods: `rok::core::num`) | 2.1  |
| `i64_type`   | `i64` primitive              | built-in (methods: `rok::core::num`) | 2.1  |
| `f32_type`   | `f32` primitive              | built-in (methods: `rok::core::num`) | 2.1  |
| `f64_type`   | `f64` primitive              | built-in (methods: `rok::core::num`) | 2.1  |
| `usize_type` | `usize` primitive            | built-in (methods: `rok::core::num`) | 2.1  |
| `isize_type` | `isize` primitive            | built-in (methods: `rok::core::num`) | 2.1  |
| `bool_type`  | `bool` primitive             | built-in (methods: `rok::core::num`) | 2.1  |
| `cchar_type` | `c_char` primitive (FFI)     | built-in (impls: `rok::core::num`) | 2.15 |
| `str_type`   | `str` primitive (UTF-8 view) | built-in (methods: `rok::core::str`) | 2.15 |
| `char_type`  | `char` primitive (Unicode scalar) | built-in (methods: `rok::core::str`) | 2.15 |

Second, there are the slots tied to language syntax: the auto-promoting `int` and `float`, the `none` sentinel, and the `Range` literal. `int` and `float` are PascalCase library structs that expose a lowercase name via `#[keyword(...)]`; `none` is built in, and the library names its type `NoneSentinel` with `pub type NoneSentinel = none`.

Syntax-tied slots:

| Slot name    | Meaning                       | Standard library item                                     | Ref  |
|--------------|-------------------------------|-----------------------------------------------------------|------|
| `none_type`  | Type of the `none` literal    | built-in (alias: `rok::core::marker::NoneSentinel`) | 2.13 |
| `int_type`   | `int` auto-promote type       | `rok::core::num::bigint::Int` + `#[keyword("int")]` | 2.1  |
| `float_type` | `float` auto-promote type     | `rok::core::num::bigfloat::Float` + `#[keyword("float")]` | 2.1  |
| `range_type` | `Range<T>` (`a..b`)           | `rok::core::range::Range` + `#[literal( expr ".." expr )]` | 2.7  |
| `slice_arg_type` | a slice part `a:b:k` of a multi-part index | `rok::core::index::SliceArg` | 2.3  |
| `rest_axis_type` | `..` beside other parts of an axis index | `rok::core::index::Rest` | 2.3  |
| `new_axis_type`  | `none` in an axis index         | `rok::core::index::NewAxis` | 2.3  |
| `ndarray_type`   | a nested list literal in its slot | `rok::core::array::Array` | 2.9.14 |

A compiler that provides the primitive types built in may leave `i32_type` ... `str_type`, `none_type` and `char_type` unattached. The reference compiler does: these slot names are accepted, but the library attaches no item to them (`str` and `c_char` are not library structs, and `NoneSentinel` is an alias, not a `#[lang]` item). Where 0.4.6 looks up one of them, the compiler uses its built-in type.

## 0.4.3. Trait slots (33)

Trait slots indicate which trait the compiler goes to when user-written syntax is desugared into a standard library trait call. Core syntax such as `for` loops, indexing, slicing, formatting and `await` all resolves through the traits registered here, so this table is the thickest bridge between the language syntax and the library.

| Slot name             | Meaning                         | Standard library trait          | Ref  |
|-----------------------|---------------------------------|---------------------------------|------|
| `iterator_trait`      | Stepping in `for x in xs`       | `rok::core::iter::Iterator`     | 2.7  |
| `into_iterator_trait` | Start of `for x in xs`          | `rok::core::iter::IntoIterator` | 2.7  |
| `try_iterator_trait`  | Stepping in `try for x in xs`   | `rok::core::iter::TryIterator`  | 2.7.13 |
| `index_trait`         | `arr[i]` read                   | `rok::core::index::Index` | 2.3  |
| `index_set_trait`     | `arr[i] = v` write              | `rok::core::index::IndexSet` | 2.3  |
| `slice_trait`         | `s[1:3]` slicing                | `rok::core::index::Sliceable` | 2.3  |
| `display_trait`       | `f"{x}"` default format         | `rok::core::fmt::Display`       | 2.10 |
| `debug_trait`         | `f"{x:?}"` debug format         | `rok::core::fmt::Debug`         | 2.10 |
| `future_trait`        | Stepping in `await expr`        | `rok::core::future::Future`     | 2.20 |
| `axis_indexed_trait`  | index parts as axis arguments   | `rok::core::index::AxisIndexed` | 2.3  |
| `broadcastable_trait` | An operand of a fused dot expression | `rok::core::ops::Broadcastable` | 2.6.6 |
| `materialize_trait`   | The result of a fused dot expression | `rok::core::ops::Materialize`   | 2.6.6 |
| `close_trait`         | A close that can fail: the owner closes it | `rok::core::io::Close`     | 2.23 |

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
| `eq_trait`          | `==` / `!=`, literal patterns            | `rok::core::ops::PartialEq` | 2.5  |
| `cmp_trait`         | `<` / `>` / `<=` / `>=`, range patterns  | `rok::core::ops::PartialOrd` | 2.5  |
| `ord_trait`         | Total order (`#[derive(Ord)]`)           | `rok::core::cmp::Ord`        | 2.5  |
| `hash_trait`        | Hash (`#[derive(Hash)]`)                 | `rok::core::cmp::Hash`       | 2.5  |
| `clone_trait`       | Clone (`#[derive(Clone)]`)               | `rok::core::marker::Clone`   | 2.13 |
| `send_trait`        | Auto marker: safe to move across threads | `rok::core::marker::Send`   | 2.13 |
| `sync_trait`        | Auto marker: safe to share across threads | `rok::core::marker::Sync`  | 2.13 |

> [!NOTE]
> Comparison slot names and trait names do not correspond one-to-one. `eq_trait` is attached to `PartialEq`, which defines the equality operators, and `cmp_trait` to `PartialOrd`, which defines the ordering operators. The marker traits `Eq`, `Copy` and `Default` have no slots: sum types get them structurally (`T | none` is `Eq` when `T` is; its `Default` is `none`), so the compiler finds them by their fixed paths in `rok::core`.

## 0.4.4. Function slots (23)

Function slots are for standard library functions that the compiler calls directly, like intrinsics. They mainly cover abnormal termination paths (`throw`, OOM, unreachable) and are used where a uniform entry point is needed outside normal flow.

| Slot name        | Meaning                                  | Standard library function       | Ref  |
|------------------|------------------------------------------|---------------------------------|------|
| `panic_fn`       | Abort function called on panic           | `rok::core::panic::panic`       | 2.14 |
| `oom_fn`         | OOM abort (runtime allocation failure calls `rok_oom` directly) | `rok::core::panic::oom` | 2.14 |
| `unreachable_fn` | Called by the compiler at unreachable points | `rok::core::panic::unreachable` | 2.14 |

The other twenty are **hooks**: functions that code the compiler writes calls, for a negative index, a slice bound, a writable view, a spread, an f-string hole, a cancellation point and a broadcast over collections. Lowering finds each through its slot, not by its path, so a hook need not be visible to the code it is lowered into. A hook is not API: each is `pub(pkg)` in `rok::core`, and a call written by hand in another package is an error.

| Slot name            | Called for                                             | Standard library function                 | Ref  |
|----------------------|--------------------------------------------------------|-------------------------------------------|------|
| `spread_len_fn`      | A `List` / `Slice` spread `f(..xs)` checks its length  | `rok::core::index::spread_len`            | 2.3  |
| `spread_names_fn`    | A `Dict` spread `f(..d)` checks its keys               | `rok::core::index::spread_names`          | 2.3  |
| `spread_misfit_fn`   | A `Dict` spread's key for a parameter it cannot fill   | `rok::core::index::spread_misfit`         | 2.3  |
| `index_from_end_fn`  | A signed index `xs[i]`, negative from the end          | `rok::core::index::index_from_end`        | 2.3  |
| `slice_bound_fn`     | A signed slice bound `xs[i:]`                          | `rok::core::index::slice_bound`           | 2.3  |
| `slice_range_fn`     | A stepped slice `xs[a:b:k]`                            | `rok::core::index::slice_range`           | 2.3  |
| `slice_mut_fn`       | `xs` or `xs[a:b]` of a list where a `SliceMut` is expected | `rok::core::index::slice_mut` (`pub(pkg)`) | 2.9  |
| `fstr_display_fn`    | An f-string hole `{v}`                                 | `rok::core::fmt::fstr_display`            | 2.10 |
| `fstr_debug_fn`      | An f-string hole `{v:?}`                               | `rok::core::fmt::fstr_debug`              | 2.10 |
| `fstr_num_fn`        | An `int` or `float` hole with a spec                   | `rok::core::fmt::fstr_num`                | 2.10 |
| `fstr_pack_fn`       | A hole whose spec is computed, `{v:{spec}}`            | `rok::core::fmt::fstr_pack`               | 2.10 |
| `fstr_text_any_fn`   | A computed spec on a value that is not a number        | `rok::core::fmt::fstr_text_any`           | 2.10 |
| `cancel_point_fn`    | An `await` that polls `Pending` (cancellation point)   | `rok::core::task::cancel_point`           | 2.18 |
| `cancel_left_fn`     | An `await` whose poll was `Ready`                      | `rok::core::task::cancel_left`            | 2.18 |
| `cleanup_running_fn` | A frame left by a cancel enters its cleanups           | `rok::core::task::cleanup_running`        | 2.18 |
| `cleanup_done_fn`    | That frame finished its cleanups                       | `rok::core::task::cleanup_done`           | 2.18 |
| `broadcast_len_fn`   | A broadcast `xs .+ ys` checks the lengths agree        | `rok::core::collections::broadcast::check_len` | 2.6 |
| `axis_from_u64_fn`   | An unsigned position in an axis index                  | `rok::core::index::axis_from_u64`         | 2.3  |
| `axis_from_i64_fn`   | A wide signed position in an axis index                | `rok::core::index::axis_from_i64`         | 2.3  |
| `array_literal_fn`   | A nested list literal in an `Array` slot               | `rok::core::array::array_literal`         | 2.9.14 |
| `fused_shape_fn` | A fused dot expression: the shape operands stretch to | `rok::core::array::fused_shape` | 2.6.6 |
| `fused_view_fn` | A fused dot expression: an operand viewed at that shape | `rok::core::array::fused_view` | 2.6.6 |
| `fused_alloc_fn` | A fused dot expression: the fresh result array | `rok::core::array::fused_alloc` | 2.6.6 |
| `fused_flat_fn` | A fused dot expression: a view is contiguous | `rok::core::array::fused_flat` | 2.6.6 |
| `fused_inner_fn` | A fused dot expression: the length of a row (the last axis) | `rok::core::array::fused_inner` | 2.6.6 |
| `fused_rows_fn` | A fused dot expression: the number of rows | `rok::core::array::fused_rows` | 2.6.6 |
| `fused_row_fn` | A fused dot expression: the buffer position of a row | `rok::core::array::fused_row` | 2.6.6 |
| `fused_step_fn` | A fused dot expression: the step along the last axis | `rok::core::array::fused_step` | 2.6.6 |
| `fused_at_fn` | A fused dot expression: an element of a contiguous run | `rok::core::array::fused_at` | 2.6.6 |
| `fused_put_fn` | A fused dot expression: a store into a contiguous run | `rok::core::array::fused_put` | 2.6.6 |
| `fused_get_fn` | A fused dot expression: an element of a strided row | `rok::core::array::fused_get` | 2.6.6 |
| `fused_set_fn` | A fused dot expression: a store into a strided row | `rok::core::array::fused_set` | 2.6.6 |
| `fused_dest_fn` | A fused dot expression: an in-place result fits its target | `rok::core::array::fused_dest` | 2.6.6 |
| `fused_source_fn` | A fused dot expression: an in-place source that overlaps the target is copied | `rok::core::array::fused_source` | 2.6.6 |

### Items the tiers share: `#[internal]`

A few `pub` items are called by another tier of the toolchain, not by the compiler: `rok::std` uses them from `rok::core`, `rok::core` from `rok::bare`. They are marked `#[internal]` ([refs-lang 6.2.16](../../refs-lang/006-meta-system/602-built-in-attributes.md#6216-internal-shared-by-the-toolchains-packages)): `rok::bare`, `rok::core` and `rok::std` may use them, any other package gets the error "internal to the Rok toolchain", and the docs and the language server leave them out.

| Item                                   | Used by                                                   | Ref  |
|----------------------------------------|-----------------------------------------------------------|------|
| `rok::core::marker::hand_over`         | `rok::std::thread` (a thread's result is handed over)      | 2.13 |
| `rok::core::task::wake_io`             | `rok::std::net` (waking waiters after a close or shutdown) | 2.18 |
| `rok::core::task::host_enter`          | `rok::std::wit` (a component export enters the host loop)  | 2.18 |
| `rok::core::task::host_run`            | `rok::std::wit` (one turn of the host loop)                | 2.18 |
| `rok::core::task::host_leave`          | `rok::std::wit` (ends `host_enter`)                        | 2.18 |
| `rok::core::task::readable` / `writable` | `rok::std::net` (waiting until a socket is ready)          | 2.18 |
| `rok::core::io::IoError::retold`       | `rok::std::fs` (an error told again for a path)            | 2.23 |
| `rok::core::io::IoError::resolver`     | `rok::std::net` (a failed name lookup)                     | 2.23 |
| `rok::core::io::Read::copy_handle` / `Write::copy_handle` | `rok::std` fs, net, process and `os::unix::net` (the OS handle `io::copy` gives the kernel) | 2.23 |
| `rok::core::num::bigint::Int::from_literal` / `bigfloat::Float::from_literal` | `rok::std::rokon` (a number literal's digits) | 2.1 |
| `rok::bare::gc::Pinnable::gc_object`   | `rok::core::marker` (a hand-off copying a `GcPtr`)         | 1.6  |

## 0.4.5. Method slots (1)

Method slots are for methods the compiler calls automatically without the user writing the call. There is only one: disposal when a `using` binding is released. The `?` operator is expanded directly by the compiler for `T | none` ([refs-lang 3.14.10](../../refs-lang/003-expressions/314-error-control-flow.md#31410-the--operator---early-return-of-none)), so it has no method slot. Cloning a value (`Clone::clone`) is never called automatically ([2.13.2](../200-core/213-marker.md#2132-clone---deep-copy)), so it is registered as the trait slot `clone_trait`, not a method slot.

| Slot name     | Meaning                              | Standard library method         | Ref  |
|---------------|--------------------------------------|---------------------------------|------|
| `drop_method` | Called where a value is released (`using`, or its owner leaving scope) | `rok::core::marker::Drop::drop` | 2.13 |

## 0.4.6. Slot consumers: where the compiler looks up slots

Where the previous sections listed "which slots exist", this section summarizes the reverse: "when the compiler looks up which slot". When tracing which standard library item a line of user syntax resolves to, the following table is the starting point.

| Compiler action                                    | Slot looked up                           |
|----------------------------------------------------|------------------------------------------|
| Lowering the `none` literal                        | `none_type`                              |
| Literal type of `42` (no expected type)            | `i64_type`                               |
| Literal type of `3.14` (no expected type)          | `f64_type`                               |
| Literal type of `"hello"`                          | `str_type`                               |
| Desugaring `for x in xs`                           | `into_iterator_trait`, `iterator_trait`, `try_iterator_trait` |
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
| A negative index or slice bound, a stepped slice   | `index_from_end_fn`, `slice_bound_fn`, `slice_range_fn` |
| A list or its range where a `SliceMut` is expected | `slice_mut_fn`                          |
| A spread `f(..xs)` / `f(..d)`                      | `spread_len_fn`, `spread_names_fn`, `spread_misfit_fn` |
| An f-string hole                                   | `fstr_display_fn` ... `fstr_text_any_fn` |
| An `await`, and the cleanups a cancel runs         | `cancel_point_fn`, `cancel_left_fn`, `cleanup_running_fn`, `cleanup_done_fn` |
| A broadcast over two or more collections           | `broadcast_len_fn`                       |
| Releasing a value (a `using` block ends, or its last owner leaves scope) | `drop_method`              |
| Automatic `Send` / `Sync` implementation, `+ Send` checks | `send_trait`, `sync_trait`        |

## 0.4.7. Duplicate / missing detection

If two standard library items are attached to the same slot, the compiler rejects it (the duplicate detection rule of refs-lang 6.4.6). Conversely, if a required slot such as `iterator_trait` is missing, compilation of the features that depend on it (here, `for` loops) is rejected. Both are devices that force the slot contract to have a single source of truth.

The actual attached items are stated in each chapter of [200-core/](../200-core/). Every trait, method, type and abort-function slot that is attached has its `#[lang(...)]` attachment site in the chapter indicated by the "Ref" column. The hook `_fn` slots (`index_from_end_fn` ... `broadcast_len_fn`) are compiler-facing and named only in this table, and the built-in type slots that 0.4.2 leaves unattached have no attachment site. A slot name not in this table is a compile error per [refs-lang 6.4.8](../../refs-lang/006-meta-system/604-lang-magic.md). For example, `Stream` has no slot ([2.8.10](../200-core/208-stream.md)).

## 0.4.8. Slot naming policy

Finally, the naming rules to follow when adding a new slot or renaming an existing one. Consistent names let the kind of a slot be guessed just by scanning the table, so the policy is not taken lightly.

Names are written in snake_case: type slots take the `_type` suffix, trait slots the `_trait` suffix, function slots the `_fn` suffix, and method slots the `_method` suffix (`drop_method`). A slot designating a whole trait is unified under a single `_trait` even if the compiler handles several methods or derives of that trait; method slots are limited to cases where the compiler **automatically calls one specific method**. By this criterion, `Clone` is registered as `clone_trait`, not `clone_method`. The suffix also determines the attachment site: a `_trait` slot is attached to **the trait declaration itself**, a `_method` slot to **the corresponding method inside the trait**, a `_type` slot to a type declaration, and a `_fn` slot to a function declaration; attaching it elsewhere is rejected by the signature check of [refs-lang 6.4.7](../../refs-lang/006-meta-system/604-lang-magic.md).
