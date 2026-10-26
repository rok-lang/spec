# 2. Type system overview

This chapter sets out what the Rok type system is responsible for and what it delegates to the standard library. The concrete kinds of types and their syntax are defined in the documents that follow.

## Contents

- [2.1. Type expressions](201-type-expressions.md)
- [2.2. The `any` slot](202-any-slot.md)
- [2.3. `struct`](203-struct.md)
- [2.4. `enum`](204-enum.md)
- [2.5. `union`](205-union.md)
- [2.6. Sum types (`A | B`)](206-sum-types.md)
- [2.7. Niche optimization](207-niche-optimization.md)
- [2.8. `trait`](208-trait.md)
- [2.9. `impl`](209-impl.md)
- [2.10. Generics](210-generics.md)
- [2.11. Const generics](211-const-generics.md)
- [2.12. Trait-as-type ban](212-trait-as-type-banned.md)
- [2.13. Type paths](213-type-paths.md)
- [2.14. Conversions](214-cast-and-conversion.md)

## Positional definition

A type is the information by which the compiler decides what shape and behavior a value is allowed to have. Every Rok value belongs to some type; there is no untyped value.

Variables are slightly different. A variable is either fixed to one type (static) or a slot that can hold several types (dynamic), and both modes coexist in the same language.

## Two modes: static and gradual

```
static (explicit)   let x: i32 = 42i32;     x is always i32
static (inferred)   let z = 42i32;          z is fixed to the initializer's type i32
gradual (any)       let y: any = 42i32;     y is a slot that can hold any type
```

A static variable is one whose type is written at the declaration or omitted and inferred from the initializer; it always holds only values of that type. A gradual variable is one with `any` written in the type position; such a variable is a dynamic slot. Static is the default; dynamic exists only where `any` is written.

This chapter only establishes that both modes exist in the same language.

The motivation is in [Execution engines and the user surface](../000-meta/001-execution-modes.md): REPL/Jupyter exploration and [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) static compilation must share the same code. As a result, exploratory code tends to use the inferred style with types omitted, library code uses the explicit style, and dynamic slots are written as `any` only where they are really needed.

## What the language defines

The language specification directly defines the following about types.

The syntax of type expressions, i.e. how type names are written. The container syntax that creates types: `struct`, `enum`, `union`, `trait`, `impl`, generics, and sum types (`|`). The meaning of the dynamic slot (`any`), and the monomorphization rule for generics, which [specializes](../000-meta/007-glossary.md#specialize) per type at the call site. Finally, the type system invariants the compiler must recognize, such as the trait-as-type ban and the niche optimization rules.

## What the standard library defines

The standard library is responsible for the following about types.

The names and meanings of all primitive types: `i32`, `f64`, `bool`, `str`, and so on. All collection types: `List`, `Dict`, `Set`, `Tuple`, and so on. All conversions between types such as cast, from, and into. The contents of traits - which traits have which methods and which types implement them. Finally, which type a literal evaluates to is also decided by the standard library, through handlers.

> [!IMPORTANT]
> This chapter defines only the shape. The contents are defined in the library specification.

## Boundary of gradual typing

Gradual typing applies only where `any` is written. Omitting a type annotation never produces a dynamic slot.

A variable without a type annotation becomes a static slot inferred from its initializer's type. A function parameter without a type annotation makes the function generic, monomorphized at the call site; it does not become a dynamic slot, so a parameter that should behave gradually must be written as `any` explicitly. Struct fields require a type annotation. A function whose return type is omitted has it inferred from the body; if it cannot be inferred, it is a compile error.

This boundary confines runtime [dispatch](../000-meta/007-glossary.md#dispatch) cost to positions written as `any`. Variables, parameters and fields are all either static or explicitly `any`.

## Trait-as-type ban

A trait name cannot be written directly as a variable, parameter or return type. Where polymorphism is needed, use generics, sum types, or the constrained `any Trait`.

Runtime dispatch is confined to positions written as `any` (`any`, `any Trait`).

## Monomorphization

Generic functions and types produce a separate compilation result for each tuple of concrete types at a call or instantiation site.

For example, if the same generic function is used at two call sites with `i32` and `f64`, two distinct function bodies are generated. As a result, indirect call cost is zero, with the same performance as a direct call. The cost shows up as compile time and binary size, but is mostly offset by MLIR + LLVM dead-code elimination.
