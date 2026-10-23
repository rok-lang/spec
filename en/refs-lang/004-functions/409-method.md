# 4.9. Methods

This document defines in detail the parameters and calling conventions of methods defined in [`impl`](../002-types/209-impl.md). It is part of the function specification and focuses on the differences between methods and free functions.

## 4.9.1. Where methods are defined

Methods are defined inside an [`impl`](../002-types/209-impl.md) block.

```rok
impl Point {
    fn origin() -> Point { Point { x: 0.0f64, y: 0.0f64 } }
    fn distance(self, other: Point) -> f64 { ... }
    fn translate(ref self, dx: f64, dy: f64) {
        self.x = self.x + dx;
        self.y = self.y + dy;
    }
}
```

## 4.9.2. Kinds of methods by the form of the first parameter

### Static methods: no `self`

```rok
impl Point {
    fn origin() -> Point { ... }              // static
    fn from_polar(r: f64, theta: f64) -> Point { ... }
}

let p = Point::origin();                       // Type::method() call
let q = Point::from_polar(1.0f64, 0.0f64);
```

A static method is a function whose first parameter is neither `self` nor `ref self`. It is called as `Type::method(args)`, which is static item access through [type paths](../002-types/213-type-paths.md).

### Instance methods: `self`

```rok
impl Point {
    fn distance(self, other: Point) -> f64 { ... }
    fn norm(self) -> f64 { ... }
}

let d = p.distance(q);                         // instance.method() call
let n = p.norm();
```

In this form the first parameter is `self`. `self` denotes the instance's value (a copy for a `struct`, [2.3.8](../002-types/203-struct.md#238-value-semantics)), and the body accesses fields with `self.field`. It is called as `instance.method(args)`.

### Instance methods: `ref self`

```rok
impl Point {
    fn translate(ref self, dx: f64, dy: f64) {
        self.x = self.x + dx;
        self.y = self.y + dy;
    }

    fn replace_with(ref self, other: Point) {
        self = other;                          // the caller's variable becomes other
    }
}

let p = Point { x: 1.0, y: 2.0 };
p.translate(5.0f64, 0.0f64);                   // called without explicit ref
p.replace_with(other);                         // likewise
```

In this form the first parameter is `ref self`. Inside the body, `self` and its fields may be assigned, and the result is written back to the caller's variable (see [the `ref` keyword](407-ref-keyword.md)). The caller does not write `ref`, because method call receivers are handled automatically.

## 4.9.3. Two uses of the `self` Keyword

```rok
impl Point {
    fn distance(self, other: Point) -> f64 {
        let dx = self.x - other.x;             // uses the instance self
        let dy = self.y - other.y;
        (dx * dx + dy * dy).sqrt()
    }
}
```

`self` ([keywords](../001-lexical/105-keywords.md)) is used in two positions. In parameter position it is the parameter that receives the instance; inside the body it is the identifier referring to that parameter.

`self` in parameter position is written without a type. Its type is automatically `Self`, the target type of the current `impl`.

## 4.9.4. The `Self` keyword: type position

`Self` ([keywords](../001-lexical/105-keywords.md), capitalized) is a type name referring to the target type of the current `impl`.

```rok
impl Point {
    fn new(x: f64, y: f64) -> Self {           // Self == Point
        Self { x, y }                          // Self can also construct instances
    }
}
```

The two must be clearly distinguished: lowercase `self` is used as a parameter and instance identifier; uppercase `Self` is used as a type name.

## 4.9.5. Automatic `ref` for method call receivers

For a `ref self` method, `ref` is applied automatically even though the caller does not write it at the call.

```rok
impl Counter {
    fn increment(ref self) { self.count = self.count + 1; }
}

let c = Counter { count: 0 };
c.increment();                                 // no explicit 'ref'
```

This is an exception to the ordinary function call rule of [the `ref` keyword](407-ref-keyword.md) (explicit `ref` required). The reasons: method call syntax `x.method()` is itself the receiver [dispatch](../000-meta/007-glossary.md#dispatch) mechanism, and the call form is the same whether the receiver is `self` or `ref self`. An additional explicit `ref` would be visual noise.

This is consistent with the method call conventions of Rust, C# and Swift. However, when calling the same method by path, the receiver is an ordinary argument, so `ref` is written (`Counter::increment(ref c)`, [4.7.3](407-ref-keyword.md#473-explicit-at-the-call-c-style)).

## 4.9.6. Call forms of free functions and methods

| Form            | Call                    |
|-----------------|-------------------------|
| Free function   | `function_name(args)`   |
| Static method   | `Type::method(args)`    |
| Instance method | `instance.method(args)` |

The call form clearly distinguishes what a function means.

### Name conflicts between traits

When `x.method()` has multiple candidates, one is chosen in this order:

1. Methods of an inherent `impl` take precedence over trait methods.
2. Among trait methods, a trait defined by the current package or explicitly `use`d takes precedence over a trait visible only through the implicit `use rok::core::**`. The current package is the actual package in which the call is written. A dependency package's trait does not take precedence over core's traits unless it is `use`d.
3. If there is still more than one candidate, it is an ambiguity error, and the diagnostic suggests writing it as `Trait::method(x)`.

```rok
use pkg::geom::Union;              // user trait - union method
a.union(b);                        // pkg::geom::Union takes precedence over core's Union
rok::core::ops::Union::union(a, b) // explicit to use the core one
```

## 4.9.7. Calling other methods inside a method

```rok
impl Point {
    fn distance(self, other: Point) -> f64 { ... }

    fn distance_squared(self, other: Point) -> f64 {
        let d = self.distance(other);          // another method of the same instance
        d * d
    }
}
```

Other methods of the same instance may be called as `self.method()`. The same rules as ordinary instance calls apply.

## 4.9.8. Method visibility

Method visibility follows the module-private default defined in [`impl`](../002-types/209-impl.md).

```rok
impl Account {
    pub fn balance(self) -> f64 { ... }        // public
        fn _internal(self) { ... }              // module-private
}
```

The exact rules for visibility levels are defined in [visibility](../005-modules/506-visibility.md).

## 4.9.9. Freedom in method signatures

Method parameters, returns and bodies follow the ordinary function rules of [Parameters](402-parameters.md) and [Return](403-return.md). Specifically, default values, named arguments, variadics and spreads may be used freely in methods. Divergent methods, whose body always exits through `throw` or an infinite loop, are also possible, and generic methods may use both the generic parameters of the `impl` block and the method's own generics.

```rok
impl<T> List<T> {
    fn map<U>(self, f: fn(T) -> U) -> List<U> { ... }
}
```

A method's own generics follow the function name (`fn map<U>(...)`). In a trait `impl`, a method parameter without a type takes the type the trait declares at that position (trait parameters and associated types substituted), not an implicit type parameter.

## 4.9.10. Summary of `self` in methods

| First parameter | Meaning                   | At the call                          | Can `self` be reassigned in the body? |
|-----------------|---------------------------|--------------------------------------|---------------------------------------|
| (none)          | static method             | `Type::method()`                     | -                                     |
| `self`          | instance method (value)   | `instance.method()`                  | no (local only)                       |
| `ref self`      | instance method (`ref`)   | `instance.method()` (`ref` automatic) | yes (reflected in the caller)        |

These three forms are the only kinds of methods.
