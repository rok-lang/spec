# 2.10. Generics

Generics are the mechanism for giving type parameters to functions, types, traits and `impl` blocks. A separate compilation result is produced for each tuple of concrete types at a call or instantiation site; this process is called monomorphization.

```
generics        ::= "<" generic_param ("," generic_param)* ","? ">"
generic_param   ::= type_param | const_param
type_param      ::= identifier (":" type_bounds)? ("=" type_expr)?
const_param     ::= "const" identifier ":" type_expr ("=" const_expr)?
type_bounds     ::= type_expr ("+" type_expr)*
where_clause    ::= "where" where_pred ("," where_pred)* ","?
where_pred      ::= type_expr ":" type_bounds
```

> [!NOTE]
> `where` is a contextual keyword, a keyword only after a function signature, an `impl` header, or a `struct` / `enum` / `union` / type alias header ([1.5.4](../001-lexical/105-keywords.md#154-contextual-keywords)).

## 2.10.1. Where parameters are declared

Generic parameters appear in these positions:

```rok
fn id<T>(x: T) -> T { x }                       // function
struct Pair<A, B> { first: A; second: B; }      // struct
enum Result<T, E> { Ok(T), Err(E) }              // enum
trait Convert<Target> { ... }                    // trait
impl<T> List<T> { ... }                          // impl
type Maybe<T> = T | none;                        // type alias
```

The parameter declaration form is the same in every position: parameter names listed in `<...>` after the name, separated by commas.

## 2.10.2. Kinds of parameters

### Type parameters

```rok
fn first<T>(pair: (T, T)) -> T { pair.0 }
```

Parameter names follow the [identifier](../001-lexical/104-identifiers.md) form. By convention, single uppercase letters such as `T`, `U`, `K`, `V` or short PascalCase names are used, but this is not enforced.

### `const` parameters

These take a value as a parameter. The exact rules are defined separately in [`const` generics](211-const-generics.md).

### Type argument inference

If type arguments are not written at the call site, the compiler infers them. Besides argument types, the expected type (the type annotation of a `let`, the parameter type at an argument position, the return type at a return position) is also used for inference.

```rok
fn empty<T>() -> List<T> { [] }

let xs: List<i32> = empty();        // T = i32 — from the let's type annotation
takes_strs(empty());                // T = str — from the parameter type List<str>
```

A `let` without an expected type is inferred from later uses. In an unannotated `let x = f();`, if the generic return type of `f` is not determined by the call alone, later uses determine it. Signatures are always explicit.

- **Scope**: the variable's scope, i.e. the rest of the block, including closures and async bodies within it.
- **Uses that fix the type**: uses that pass the value to a position whose type is fixed: function / method arguments, assignment, annotated `let`, struct fields, `return`. Operators, f-string holes, patterns and field reads do not fix the type.
- **Mutual fixing**: several open `let`s may fix each other (`let a = List::new(); let b = List::new(); a.push(b); b.push(1);` → `b: List<i32>`, `a: List<List<i32>>`).
- **Default types of literals**: `i32` / `f64` yield to a later concrete use of the same kind (integer / real). In `ys.push(1); sum_i64(ys)`, `ys` is `List<i64>`.
- **Conflicts**: two different uses do not form a sum type; it is an error (`xs.push(none); xs.push(1)`, same as the branch rule of [5.1.7](../005-modules/501-variable-declaration.md#517-type-annotations-and-type-inference)). `let v = none;` alone is still an error.
- Closure parameters ([4.5.2](../004-functions/405-closure.md)) and method path values ([3.5.3](../003-expressions/305-member-access-and-method-call.md#method-path-values)) without context are likewise fixed by later calls (`let f = |x| x * 2; f(4)`, `let f = Box::get; f(b)`).

Diagnostics:

| Case                 | Location                                                  | Example message                                                            |
|----------------------|-----------------------------------------------------------|----------------------------------------------------------------------------|
| Never determined     | That `let`                                                | `` type annotations needed: cannot infer the type of `d` (`Dict<_, _>`) `` |
| Conflict             | The use site, with the site that fixed the type as a secondary label | `` conflicting uses of `c` (`List<_>`): `str` here, but `i32` fixed at 8:5 `` |

```rok
let xs = empty();                   // still List<?>
takes_ints(xs);                     // T = i32 here — parameter List<i32>
let zs = List::new();
zs.push(1);                         // zs: List<i32> — from a later method argument
let ys = empty();                   // ERROR — type annotations needed (no concrete use)
```

Static trait methods with no arguments determine `Self` the same way. A call on a type parameter, `T::zero()`, and a trait-path call, `Tr::m()`, take `Self` from the expected type.

```rok
let z: i32 = Zero::zero();          // Self = i32 — rok::core::num::Zero
fn sum<T: Zero + Add>(xs: List<T>) -> T {
    let acc: T = T::zero();          // Self = T
    ...
}
```

A variant constructor of a generic `enum` determines its instance from its arguments first, then from the type parameters' defaults, then from the `let` inference variable it is bound to; otherwise it is an error, ``type annotations needed``.

```rok
enum Pair<A, B> { One(A), Two(A, B) }
let p = Pair::Two(1, "a");          // Pair<i32, str> — from the arguments
```

## 2.10.3. Trait bounds

Bounds state the traits a parameter must satisfy.

```rok
fn add_all<T: Add<T>>(items: List<T>) -> T { ... }
fn print_all<T: Display>(items: List<T>) { ... }
fn sort_all<T: Ord + Clone>(ref items: List<T>) { ... }
```

A single bound is written as `Param: Trait`; multiple bounds are joined with `+`. Without bounds, no operation can be applied to the parameter; values can only be moved.

## 2.10.4. where clauses

When bounds get long, it is better to move them into a `where` clause.

```rok
fn complex<T, U, V>(a: T, b: U) -> V
where
    T: Add<U, Output = V>,
    V: Display + Clone,
{
    ...
}
```

The `where` keyword goes between the parameter declarations and the body, followed by comma-separated bounds. Constraints on associated types can also be written here (e.g. `T::Output: Foo`).

## 2.10.5. Defaults

Type parameters can have default values.

```rok
trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}
```

If the parameter is omitted at a call or instantiation, the default is used. A parameter without a default cannot follow one with a default; defaults must be trailing.

## 2.10.6. Monomorphization

Generic calls and instantiations produce a separate compilation result for each tuple of concrete types.

```rok
fn id<T>(x: T) -> T { x }

let a = id(42i32);          // id<i32> is generated — an i32 → i32 function
let b = id(3.14f64);        // id<f64> is generated — an f64 → f64 function
let c = id("hello");         // id<str> is generated
```

Calls with the same type tuple share the same result, so it is compiled only once. The result performs the same as a direct call, with no indirect call cost. The cost is compile time and binary size, mostly offset by MLIR and LLVM dead-code elimination.

### Monomorphization cache model

An [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) build collects call sites and [specializes](../000-meta/007-glossary.md#specialize) them all at once. In [JIT](../000-meta/007-glossary.md#jit-just-in-time) mode, specialization happens for a type tuple on its first call, and the result is cached. Both modes share the same specialization machinery, which is part of the common pipeline of [Execution engines and the user surface](../000-meta/001-execution-modes.md).

## 2.10.7. Combination with gradual typing

The gradualness of function parameters described in [The `any` slot](202-any-slot.md) is handled as follows.

```rok
fn compute(x, y) { ... }
```

This code is turned into an implicit generic, `fn compute<T, U>(x: T, y: U)`. No bounds are synthesized ([4.2.2](../004-functions/402-parameters.md#422-omitting-the-type)): each instance is monomorphized at its call site and its body checked there, and an error also points at the call.

If an explicit `any` is written on a parameter, it is not monomorphized but becomes a runtime dynamic slot. This difference is the core of gradual mode.

## 2.10.8. Consistency of generic parameters

A generic parameter denotes only one type within a call. It may denote different types in different calls, but its type cannot change within the same call.

```rok
fn pair<T>(a: T, b: T) -> (T, T) { (a, b) }

let p = pair(1i32, 2i32);     // OK — T = i32 for both arguments
let q = pair(1i32, 3.14f64);  // ERROR — ambiguous whether T is i32 or f64
```

Multiple parameters (`pair<A, B>`) are each inferred independently.

## 2.10.9. Body checking

The body of a generic function may only use behavior known through its bounds.

```rok
fn double<T>(x: T) -> T {
    x + x       // ERROR — T has no Add bound
}

fn double<T: Add<T, Output = T>>(x: T) -> T {
    x + x       // OK
}
```

Because of this rule, generic bodies are checked independently of call sites, so the body is guaranteed to compile whatever concrete type a call passes. Implicit parameters ([4.2.2](../004-functions/402-parameters.md#422-omitting-the-type)) are exempt.

## 2.10.10. Generics vs trait-as-type

Generics are compile-time polymorphism, called directly through monomorphization.

Trait-as-type is runtime polymorphism using vtable dispatch, which rok forbids in the [Type system overview](README.md).

So whenever polymorphism is needed, use generics (this document) or sum types ([Sum types](206-sum-types.md)).
