# 2.9. `impl`

An `impl` block is a container that attaches methods to a type or implements a trait for a type.

## 2.9.1. Two forms

The grammar of an `impl` block has two forms:

```
impl_block      ::= "impl" generics? type_expr "{" impl_item* "}"
                  | "unsafe"? "impl" generics? type_expr "for" type_expr where_clause? "{" impl_item* "}"
                  | "impl" generics? "!" type_expr "for" type_expr where_clause? "{" "}"
impl_item       ::= fn_decl
                  | "type" identifier "=" type_expr ";"
                  | const_decl
```

In `impl <`, the `<` opens `generics` only when a generic parameter follows (`<>`, `<T>`, `<T:`, `<T,`, `<T =`, `<const N`); otherwise it begins the target type, e.g. an inherent `impl <A as P>::Out { .. }`.

### Inherent `impl`: attaching methods

First, an inherent `impl`:

```rok
impl Point {
    fn origin() -> Point { Point { x: 0.0f64, y: 0.0f64 } }
    fn distance(self, other: Point) -> f64 { ... }
}
```

This form is `impl Type { ... }`, and the body is a collection of function definitions. The functions defined this way become methods of the type.

Fixed arrays and tuples can also have inherent `impl`s. The standard library uses this to give arrays `len()` / `is_empty()`.

```rok
impl<T, const N: usize> [T; N] {
    fn len(self) -> usize { N }
}

impl<A, B> (A, B) {
    fn swap(self) -> (B, A) { (self.1, self.0) }
}
```

### Trait `impl`: implementing a trait

Next, a trait `impl`:

```rok
impl Describe for Point {
    fn describe(self) -> str { f"({self.x}, {self.y})" }
}

impl ops::Add for Point {
    type Output = Point;
    fn add(self, rhs: Point) -> Point { ... }
}
```

This form is `impl Trait for Type { ... }`. The body contains the bodies of the methods required by the [`trait`](208-trait.md) and the concrete definitions of associated types. Items for which the trait provides a default method may be omitted from the body, in which case the default body is used. An associated type or constant the trait declares without a default must be defined; leaving one out is an error (E0072) whose help writes the line to add, with the type the impl's signatures give it where they name it (`type Item = i32;`).

An impl method must have the trait declaration's parameter count, the same `ref` on each parameter other than the receiver, the same `unsafe` ([4.15.4](../004-functions/415-unsafe.md#4154-in-traits)), the declaration's type, with the replacements below, on each parameter it writes a type for (an untyped parameter takes that type, [4.1](../004-functions/401-function-declaration.md)), and return a value exactly when the declaration does, of the declaration's type with `Self`, the trait's parameters and its associated types replaced by the impl's (there are no narrower or covariant returns); otherwise it is an error at the impl, and the error shows the declaration. The `throws` of an impl method must be the same as the `throws` of the trait method; an impl method without `throws` takes the trait method's. An impl cannot add `throws` or widen the error type ([4.14.11](../004-functions/414-throws.md#41411-throws-on-methods)).

`Send` / `Sync` are automatic marker traits with two manual declarations ([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers)).

- Opt-in: `unsafe impl Send for X {}`. `unsafe impl` may only be used for `Send` / `Sync` and for an `unsafe trait` ([4.15.4](../004-functions/415-unsafe.md#4154-in-traits)), and is required for manual implementations of these.
- Opt-out: `impl !Send for X {}`, `impl<T> !Send for Cell<T> {}`. It prevents the automatic implementation, and the body must be empty. `!` may only be used with `Send` / `Sync`.

### Constants in an `impl`

`const N: T = v;` inside an `impl` block is an associated constant of that type. It is read as `Type::N` from outside and `Self::N` inside the `impl`. Associated constants declared by a trait ([2.8.3a](208-trait.md#283a-associated-constants)) get their values in the trait `impl`.

```rok
impl Circle {
    const UNIT: f64 = 1.0;
    fn unit() -> Circle { Circle { r: Self::UNIT } }
}
let r = Circle::UNIT;
```

An `impl` target or trait argument may be a projection (`<T as Tr>::X`) or name an item declared later; it is resolved after all items are registered.

An `impl` target may name a type alias, generic or not. The alias is expanded, and the `impl` is the one written with the aliased type: with `type Al<T> = S<T>;`, `impl<T> Al<T> { .. }` is `impl<T> S<T> { .. }` and `impl<T> Tr for Al<T>` is `impl<T> Tr for S<T>`. An alias that fixes some arguments (`type Names<V> = Dict<str, V>;`) gives an `impl` that covers only those instances.

## 2.9.2. The `self` convention for methods

For a function in the body to be an instance method, its first parameter must be `self` (or `ref self`).

```rok
impl Point {
    fn origin() -> Point { ... }                    // static: called as Type::method()
    fn distance(self, other: Point) -> f64 { ... }  // instance: called as instance.method()
    fn replace_with(ref self, other: Point) { ... } // ref self: may reassign the caller's variable
}
```

The exact meaning of the `self` parameter and the behavior of `ref self` are defined in the function specification. This document only notes that the presence of `self` distinguishes static methods from instance methods.

## 2.9.3. Multiple `impl` blocks for one type

Inherent `impl`s for the same type may be split into several blocks.

```rok
impl Point {
    fn origin() -> Point { ... }
}

impl Point {
    fn distance(self, other: Point) -> f64 { ... }
}
```

This is only a visual and organizational grouping; the result is the same as one combined body.

However, a given trait can be `impl`emented only once for the same type; a second impl, empty or not, is an error (E0075) that shows the first.

```rok
impl Display for Point { ... }
impl Display for Point { ... }   // ERROR: duplicate implementation (E0075)
```

## 2.9.4. Generic `impl`

An `impl` can also be declared generic.

```rok
impl<T> List<T> {
    fn len(self) -> usize { ... }
}

impl<T> ops::Add for Pair<T> where T: ops::Add<T> {
    type Output = Pair<T::Output>;
    fn add(self, rhs: Pair<T>) -> Pair<T::Output> { ... }
}
```

Type parameters are declared with `<...>` after `impl`, and may appear in the target type, the trait arguments, and associated types. Additional constraints can be added with a `where` clause, in the same way as for trait declarations.

A generic `impl` covers the instances of its target that its parameters can produce: `impl<V> P<str, V>` gives methods to `P<str, i32>` but not to `P<i32, i32>`.

## 2.9.5. Orphan rule

Implementing a trait defined in one package for a type defined in another package is forbidden.

```rok
// trait Display and type i32 are both defined in the standard library (another package)
impl Display for i32 { ... }     // ERROR: both from external packages (E0076)
```

So at least one must hold:

1. the trait is defined in the current package;
2. the target type is defined in the current package;
3. the trait is generic and, among the target type and the trait's type arguments taken in that order (`impl Tr<T1, .., Tn> for T0`: `T0`, `T1`, .., `Tn`), a type defined in the current package comes before any bare type parameter of the `impl` (Rust's rule, RFC 2451).

Rule 3 is what lets a package put a scalar on the left of its own type's operators:

```rok
// in a package defining `Matrix`
impl Mul<Matrix> for f64 { type Output = Matrix; ... }      // OK: `Matrix` is local (rule 3)
impl DotAdd<Matrix> for f64 { type Output = Matrix; ... }   // OK
impl<T> Mul<Matrix> for T { ... }        // ERROR: the bare `T` comes first (E0076)
impl Add<List<Matrix>> for i32 { ... }   // ERROR: `List<Matrix>` is `rok::core`'s type (E0076)
```

A bare parameter before the local type is refused because another package could write the same `impl` for its own type in that position; a local type only nested in a foreign one (`List<Matrix>`) does not count. Built-in types (primitives, tuples, arrays, function types) count as defined in `rok::core`.

The unit of this rule is the package of [Source units](../000-meta/003-source-units.md). Different modules within the same package may therefore `impl` freely.

This rule prevents two packages from simultaneously creating conflicting `impl`s for the same (trait, type) pair, ensuring coherence.

### Coherence

A (trait, type) pair has only one `impl`, and there is no specialization.

- A generic `impl` is monomorphized for each type it is used with.
- A concrete `impl` overlaps a generic `impl` only when that concrete type satisfies the generic `impl`'s bounds.
- Two generic `impl`s overlap if their targets (and trait arguments) unify; this is reported as an error at declaration. Bounds on parameters that remain generic after unification do not separate them. It is an error even if no type yet satisfies both bounds (same as Rust's coherence).
- For overlap, `T?` and `T | none` are the same type.
- `const` arguments ([2.11](211-const-generics.md)) are compared by value: `Buf<3>` and `Buf<{1 + 2}>` are one type. A sum of `const` parameters times integers plus a constant (`Buf<{N + 1}>`, `Buf<{2 * N}>`) overlaps another argument only when the two can be equal for some integer values, a parameter of an unsigned type taking no negative value. So `impl<const N: usize> X for Buf<{N + 1}>` and `impl X for Buf<0>` do not overlap, nor do `Buf<{2 * N}>` and `Buf<{2 * M + 1}>`. Any other form (`{N * M}`) is assumed to overlap.
- Negative reasoning is done only for known types.
  1. A parameter that becomes a concrete type through unification has its bound checked. `impl<U: Num> X for Pair<U, i32>` and `impl<U> X for Pair<str, U>` unify with `U = str`, but `str: Num` does not hold, so they do not overlap.
  2. A bound on a known type constructor (`List<T>: Iterator`) separates the two if none of that trait's impls targets that constructor or a bare parameter. So `impl<I: Iterator> IntoIterator for I` and `impl<T> IntoIterator for List<T>` can coexist.

```rok
impl<T: Display> Describe for List<T> { ... }
impl<T: Hash> Describe for List<T> { ... }   // ERROR: i32 satisfies both Display and Hash
impl<T: Shape> Area for T { ... }
impl<T: Color> Area for T { ... }            // ERROR: T and T unify (bounds irrelevant)
```

```rok
impl<T: Display> Describe for T { ... }
impl Describe for i32 { ... }        // ERROR: overlaps the impl above since i32: Display
```

## 2.9.6. Method visibility

Methods of an inherent `impl` are module-private by default. They can be called freely within the same module; exposing them externally requires an explicit `pub`.

```rok
impl Account {
    pub fn balance(self) -> f64 { self.amount }      // public
        fn _internal_check(self) { ... }              // module-internal only
}
```

This matches the visibility model of [`struct`](203-struct.md) and [`enum`](204-enum.md). An `impl` block itself is only a visual grouping, not a unit of visibility; the unit of visibility is always the module.

Methods of a trait `impl` follow the trait's visibility and cannot be marked separately.

## 2.9.7. Method conflicts between inherent and trait `impl`s

If the same method name exists in both an inherent `impl` and a trait `impl`, the call site must choose unambiguously. The exact resolution rules are defined in [Methods](../004-functions/409-method.md#name-conflicts-between-traits).

## 2.9.8. No defining the same method twice

Defining two methods with the same name for the same type is a compile error, whether in one `impl` block or in several inherent `impl` blocks ([2.9.3](#293-multiple-impl-blocks-for-one-type)), and within one trait `impl` (there is no overloading). Methods of the same name from different traits are allowed ([2.9.7](#297-method-conflicts-between-inherent-and-trait-impls)).
