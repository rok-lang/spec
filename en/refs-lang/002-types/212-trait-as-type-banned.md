# 2.12. Trait-as-type ban

> [!IMPORTANT]
> Writing a trait name directly in type expression position is forbidden. When a value whose implementing type is decided at runtime is needed, write `any Trait`, whose cost is visible in the syntax (2.12.3).

## 2.12.1. What is forbidden

```rok
let a: Number = 32i32;                          // ERROR - Number is a trait
fn process(x: Display) -> str { ... }            // ERROR
struct Holder { item: Iterator; }                // ERROR
```

As in these examples, a trait name appearing directly as a variable type, parameter type, return type or struct field type is a compile error.

## 2.12.2. Rationale

The target of the ban is **hidden** dynamic dispatch. Writing a trait name directly in type position is indistinguishable at a glance from an ordinary static type, but actually creates runtime dispatch. That dispatch has real cost. The implementation is chosen and called at runtime, so on a branch misprediction it can be more than twice as slow as a direct call. It needs extra memory accesses to the dispatch data, hurting cache friendliness. Above all, it blocks the compiler's inlining and optimization.

So that core numeric loops in scientific and data code do not take on this cost unknowingly, rok does not allow a bare trait name as a type. Runtime polymorphism exists only where `any` is written. `any Trait` is allowed because its dynamic dispatch cost is visible in the syntax through the `any` keyword.

## 2.12.3. When polymorphism is needed: three alternatives

rok provides three alternatives when polymorphism is needed.

### 1. Sum types for a closed set of candidates

If the set of candidates is fixed in advance, use a sum type.

```rok
let a: i32 | f32 | f64 = 32i32;
fn process(x: i32 | f32) -> str { ... }
```

This fits when the candidates are fixed at compile time. Memory and dispatch costs are explicit (see [Sum types](206-sum-types.md)), and pattern matches can be checked for exhaustiveness.

### 2. Generics for open polymorphism

If the candidates are open, use generics.

```rok
fn process<T: Display>(x: T) -> str { ... }
struct Holder<T: Iterator> { item: T; }
```

This fits when candidates are not fixed at compile time or differ per caller. Monomorphization produces direct calls, so the cost is near zero (see [Generics](210-generics.md)), and methods can be called through trait bounds.

### 3. `any Trait` when the implementing type is decided at runtime

If values of different implementing types must be mixed in one slot or collection, use a constrained `any`.

```rok
let shapes: List<any Shape> = [Circle { r: 1.0 }, Square { side: 2.0 }];
fn report(e: any Error) -> str { f"{e}" }
struct ConfigError { msg: str; cause: any Error | none; }
```

An `any Trait` slot accepts only values of types implementing `Trait` (checked at compile time), and methods of `Trait` and its supertraits can be called on its value. Calls are dispatched by runtime tag, with the same cost model as [The `any` slot](202-any-slot.md). The exact rules are defined in [2.2.10](202-any-slot.md#2210-constrained-any-any-trait).

## 2.12.4. Decision criteria

Use the following criteria to choose among the alternatives.

| Situation                                                     | Choice      |
|---------------------------------------------------------------|-------------|
| Candidates are a small fixed set (3-5)                        | Sum type    |
| Candidates are extensible by users / libraries                | Generics    |
| Candidates are decided only at runtime (e.g. plugins, heterogeneous collections) | `any Trait` |
| Anything must be held, with no constraint                     | `any`       |

In the third case, where polymorphism with types decided at runtime is needed, use `any Trait`. It incurs dynamic dispatch cost, but that cost is explicitly concentrated where `any` is written.

## 2.12.5. Supertraits and bound expressions are allowed

To be clear: what is forbidden is only a trait name appearing in type position. The following are all allowed.

```rok
trait Ord: Eq { ... }                            // supertrait declaration
fn f<T: Add + Display>(x: T) { ... }              // trait bound
fn g<T>(x: T) where T: Ord, T: Clone { ... }      // where clause
impl Display for Point { ... }                    // trait position of an impl
```

In these positions a trait name expresses a constraint, not a type.

## 2.12.6. Form of the compile error

When a trait name appears in type position, the compiler emits a diagnostic similar to:

```
ERROR: 'Display' is a trait, not a type.
       To allow multiple types, use generics, a sum type, or `any Display`:
         fn f<T: Display>(x: T) { ... }
         fn f(x: SomeType | OtherType) { ... }
         fn f(x: any Display) { ... }
```

This diagnostic guides the user to one of the three alternatives above.
