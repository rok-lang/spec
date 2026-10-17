# 0. Standard library meta

This chapter collects in one place the policies and tables that apply across the standard library specification.

## Contents

- [0.1. The 3-tier model (bare / core / std)](001-tier-model.md)
- [0.2. Import policy (`use rok::core::**;` — no automatic prelude)](002-import.md)
- [0.3. Intrinsic mechanism — the compiler ↔ standard library boundary](003-intrinsics.md)
- [0.4. `#[lang]` slot mapping table](004-lang-slot-table.md)
- [0.5. Notation](005-notation.md)

## Role of this chapter

This chapter does not directly define individual traits or types. Instead, it gathers the rules and
policies that the library **as a whole** must follow consistently.

The **3-tier model** prescribes the direction of dependencies: `std` may use `core` and `bare`, and
`core` may use `bare`, but dependencies in the opposite direction are not allowed. The **import policy**
specifies how `use rok::core::**;` is applied implicitly and automatically. There is no separate
prelude module; the recursive glob over `rok::core` plays that role directly. The **intrinsic** entry that follows covers how
standard library methods are connected to the compiler backend (PrimOp, etc.), and finally the **lang
slot** entry shows how the 48 slots the compiler looks up through the `#[lang(...)]` attribute map to
standard library items.

Only once these policies are defined can the trait definitions of [200-core/](../200-core/) follow.
