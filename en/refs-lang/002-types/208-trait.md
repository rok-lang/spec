# 2.8. `trait`

`trait` is a container that declares a set of common behavior. A type that implements that behavior satisfies the trait.

This document defines only the declaration form of traits. The `impl` block that implements a trait is defined in [`impl`](209-impl.md).

## 2.8.1. Basic form

The grammar of a trait declaration is:

```
trait_decl  ::= visibility? "trait" identifier generics? supertraits? where_clause? "{" trait_item* "}"
supertraits ::= ":" type_expr ("+" type_expr)*
trait_item  ::= visibility? method_sig
              | visibility? assoc_type_decl
              | visibility? method_default
              | visibility? assoc_const_decl
method_sig  ::= "fn" identifier "(" param_list? ")" throws_clause? return_type? ";"
method_default ::= "fn" identifier "(" param_list? ")" throws_clause? return_type? block
assoc_type_decl ::= "type" identifier (":" type_expr ("+" type_expr)*)? where_clause? ("=" type_expr)? ";"
assoc_const_decl ::= "const" identifier ":" type_expr ("=" const_expr)? ";"
```

Examples of trait declarations:

```rok
trait Describe {
    fn describe(self) -> str;
}

trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}

trait Iterator {
    type Item;
    fn next(ref self) -> Item | none;
}
```

The trait body is enclosed in `{ ... }`. The body may contain method signatures, associated types, and default method bodies.

## 2.8.2. Method signatures

A method signature is written as:

```rok
fn name(params) -> ReturnType;
```

A signature without a body is a method the trait requires; the implementation must provide the body. With a body, it is a default method, and that body is used unless the implementation overrides it. The meaning of the `self` and `ref self` parameters is defined in the function specification.

An example with both a required method and a default method:

```rok
trait Greeter {
    fn greet(self) -> str;                          // required
    fn greet_loudly(self) -> str { self.greet() }    // default — may be overridden
}
```

## 2.8.3. Associated types

An associated type lets a trait have a different type per implementation.

```rok
trait Iterator {
    type Item;
    fn next(ref self) -> Item | none;
}
```

An associated type is declared as `type Name;`, and the implementation fills in a concrete type. Within one implementation, an associated type is fixed to exactly one concrete type; an implementation cannot have several possibilities at once.

An associated type can be used in signatures within the same trait, as in `Item | none` above. It can also be used in a throws clause, as in `throws (Self::X)`, in both trait and impl signatures.

### Defaults

An associated type can have a default with `type X: Bounds = T;`. An impl either omits the associated type to use the default or overrides it with `type X = U;`. The default is resolved at the trait's definition site, with the trait's parameters substituted.

```rok
trait TryFrom<U> {
    type Error = ConversionError;
    fn try_from(value: U) throws (Self::Error) -> Self;
}
impl TryFrom<RawPath> for Path { type Error = NonUtf8Error; ... }   // overridden
impl TryFrom<Celsius> for Kelvin { ... }                           // Error = ConversionError
```

Inside a trait and all impls (trait impls and inherent impls), both the bare name `Item` and `Self::Item` denote the associated type, usable in both method signatures and bodies. In an impl it is the concrete type that impl chose. If an associated type shadows an outer type of the same name (a module's `struct Item`, a primitive type, etc.), a warning is issued. A trait of the same name is not a type, so no warning is issued (`trait IntoFuture { type IntoFuture; }` is fine).

If a type implements two traits that have an associated type of the same name, a bare `X` / `Self::X` in that type's impl bodies is ambiguous and is an error. In that case, name the trait with `<Self as Tr>::X`.

```rok
impl Iterator for Stream { type Item = i32; ... }
impl Source for Stream { type Item = str; ... }

impl Stream {
    fn first(self) -> Self::Item { ... }                   // ERROR — Iterator::Item? Source::Item?
    fn first(self) -> <Self as Iterator>::Item { ... }     // OK
}
```

### Referring to associated types from outside: `T::AssocType`

To refer to the associated type of a trait-bounded parameter from outside, use the `T::AssocType` notation.

```rok
fn first_item<T: Iterator>(ref it: T) -> T::Item | none { it.next() }
```

If the bound is a generic trait `T: Tr<A>`, `T::X` is the `X` of the `Tr<A>` impl, because one type can implement `Tr<A>` and `Tr<B>` with different `X` (`<T as Tr<A>>::X` is the same, [2.13.4](213-type-paths.md#2134-fully-qualified-path)).

This notation is associated item access using the `::` path separator of [Punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md). The general rules of paths (module paths, turbofish, fully qualified form) are defined in [Type paths](213-type-paths.md).

## 2.8.3a. Associated constants

A trait can declare an associated constant with `const N: T;` and give it a default with `= value`. Each `impl` sets the value (may be omitted if there is a default), and on a type parameter bounded by that trait it is read as `T::N`.

```rok
trait Shape {
    const SIDES: u32;
    const NAME: str = "shape";
}

impl Shape for Square { const SIDES: u32 = 4; }

fn sides<T: Shape>() -> u32 { T::SIDES }      // same as <T as Shape>::SIDES
```

In an `impl` for a generic type, an associated constant can have a different value per instantiation (`impl<const N: usize> Shape for Poly<N> { const SIDES: u32 = N as u32; }`). When ambiguous, use the fully qualified form `<T as Tr>::N` ([2.13.4](213-type-paths.md#2134-fully-qualified-path)).

## 2.8.4. Supertraits

A trait can require that it be implemented only for types that satisfy another trait.

```rok
trait Eq {
    fn eq(self, other: Self) -> bool;
}

trait Ord: Eq {
    fn cmp(self, other: Self) -> Ordering;
}
```

Supertraits are written after `:`. So implementing `Ord` requires also implementing `Eq`. Several supertraits are joined with `+`, as in `trait Foo: Bar + Baz`.

## 2.8.5. Generic traits

Traits can also take type parameters.

```rok
trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}

trait Convert<Target> {
    fn convert(self) -> Target;
}
```

Type parameters are declared inside `<...>` after the trait name, and a parameter can have a default, as in `Rhs = Self`. `Self` is the keyword denoting the type the trait is implemented for ([Keywords](../001-lexical/105-keywords.md)).

The exact monomorphization rules for generic traits are defined in [Generics](210-generics.md).

## 2.8.6. where clauses: additional constraints on parameters

Additional trait bounds can be written on a trait declaration or a method signature.

```rok
trait Sortable<T> where T: Ord {
    fn sort(ref self);
}

trait Container {
    type Item where Item: Clone;
    fn first(self) -> Item;
}
```

Constraints follow the `where` keyword in the form `Type: Trait`, separated by commas. Attached to the trait itself, they constrain the whole trait; attached to a method, they constrain only that method.

## 2.8.7. Dispatch of trait methods

When a trait method is called, the compiler looks at the receiver type at the call site and statically calls the method implemented for that type.

```rok
let p = Point { x: 1.0, y: 2.0 };
let s = p.describe();       // Point's Describe implementation is called statically
```

This dispatch is always static, because trait-as-type — writing a trait name as a variable, parameter or return type — is forbidden by the [Type system overview](README.md).

So when polymorphism is needed, use generics ([Generics](210-generics.md)) or sum types ([Sum types](206-sum-types.md)).

## 2.8.8. Visibility

The visibility of a trait itself and of its methods can be controlled separately. The exact rules are defined in [Visibility](../005-modules/506-visibility.md).

## 2.8.9. No duplicate definitions in the same module

Declaring two traits with the same name in the same module is a compile error.
