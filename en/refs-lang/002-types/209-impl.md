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

impl Add for Point {
    type Output = Point;
    fn add(self, rhs: Point) -> Point { ... }
}
```

This form is `impl Trait for Type { ... }`. The body contains the bodies of the methods required by the [`trait`](208-trait.md) and the concrete definitions of associated types. Items for which the trait provides a default method may be omitted from the body, in which case the default body is used.

The `throws` of an impl method must be the same as the `throws` of the trait method. An impl cannot add `throws` or widen the error type ([4.14.11](../004-functions/414-throws.md#41411-throws-on-methods)).

`Send` / `Sync` are automatic marker traits with two manual declarations ([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync--thread-boundary-markers)).

- Opt-in: `unsafe impl Send for X {}`. `unsafe impl` may only be used for `Send` / `Sync`, and is required for manual implementations of these two traits.
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

## 2.9.2. The `self` convention for methods

For a function in the body to be an instance method, its first parameter must be `self` (or `ref self`).

```rok
impl Point {
    fn origin() -> Point { ... }                    // static — called as Type::method()
    fn distance(self, other: Point) -> f64 { ... }  // instance — called as instance.method()
    fn replace_with(ref self, other: Point) { ... } // ref self — may reassign the caller's variable
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

However, a given trait can be `impl`emented only once for the same type.

```rok
impl Display for Point { ... }
impl Display for Point { ... }   // ERROR — duplicate implementation
```

## 2.9.4. Generic `impl`

An `impl` can also be declared generic.

```rok
impl<T> List<T> {
    fn len(self) -> usize { ... }
}

impl<T> Add for Pair<T> where T: Add<T> {
    type Output = Pair<T::Output>;
    fn add(self, rhs: Pair<T>) -> Pair<T::Output> { ... }
}
```

Type parameters are declared with `<...>` after `impl`, and may appear in the target type, the trait arguments, and associated types. Additional constraints can be added with a `where` clause, in the same way as for trait declarations.

## 2.9.5. Orphan rule

Implementing a trait defined in one package for a type defined in another package is forbidden.

```rok
// trait Display and type i32 are both defined in the standard library (another package)
impl Display for i32 { ... }     // ERROR — both from external packages
```

So at least one must hold: the trait is defined in the current package, or the target type is defined in the current package.

The unit of this rule is the package of [Source units](../000-meta/003-source-units.md). Different modules within the same package may therefore `impl` freely.

This rule prevents two packages from simultaneously creating conflicting `impl`s for the same (trait, type) pair, ensuring coherence.

### Coherence

A (trait, type) pair has only one `impl`, and there is no specialization.

- A generic `impl` is monomorphized for each type it is used with.
- A concrete `impl` overlaps a generic `impl` only when that concrete type satisfies the generic `impl`'s bounds.
- Two generic `impl`s overlap if their targets (and trait arguments) unify; this is reported as an error at declaration. Bounds on parameters that remain generic after unification do not separate them. It is an error even if no type yet satisfies both bounds (same as Rust's coherence).
- Negative reasoning is done only for known types.
  1. A parameter that becomes a concrete type through unification has its bound checked. `impl<U: Num> X for Pair<U, i32>` and `impl<U> X for Pair<str, U>` unify with `U = str`, but `str: Num` does not hold, so they do not overlap.
  2. A bound on a known type constructor (`List<T>: Iterator`) separates the two if none of that trait's impls targets that constructor or a bare parameter. So `impl<I: Iterator> IntoIterator for I` and `impl<T> IntoIterator for List<T>` can coexist.

```rok
impl<T: Display> Describe for List<T> { ... }
impl<T: Hash> Describe for List<T> { ... }   // ERROR — i32 satisfies both Display and Hash
impl<T: Shape> Area for T { ... }
impl<T: Color> Area for T { ... }            // ERROR — T and T unify (bounds irrelevant)
```

```rok
impl<T: Display> Describe for T { ... }
impl Describe for i32 { ... }        // ERROR — overlaps the impl above since i32: Display
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

Defining two methods with the same name in the same `impl` block is a compile error (there is no overloading).
