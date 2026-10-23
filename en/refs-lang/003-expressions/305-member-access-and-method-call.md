# 3.5. Member access and method calls

The `.` token ([punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md)) is used in two forms that access items attached to a value.

```
member_expr     ::= primary "." identifier
                  | primary "." int_lit               // tuple position
method_call     ::= primary "." identifier ("::" "<" type_expr ("," type_expr)* ">")?
                    "(" arg_list? ")"
arg_list        ::= expr ("," expr)* ","?
```

## 3.5.1. Member access: `obj.field`

```rok
let p = Point { x: 1.0f64, y: 2.0f64 };
let xv = p.x;               // 1.0
let yv = p.y;               // 2.0
```

Here `obj` is any expression and `field` follows the form of an [identifier](../001-lexical/104-identifiers.md). The result is the value of that field.

### Positional access on tuple `struct`s and tuples

```rok
let pair = Pair(1i32, 2i32);
let first = pair.0;
let second = pair.1;
```

Decimal digits after the dot are interpreted as a position index; this is the same tokenization as the `1.foo` rule of [numeric literals](../001-lexical/106-numeric-literals.md). For example, `pair.0` tokenizes into three tokens - the identifier `pair`, `.`, and the integer `0` - and the parser interprets them as positional access.

### Static and dynamic resolution

If the static type of the left-hand side is known, the field position is determined at compile time and compiled to a direct memory access. If the left-hand side is an [`any` slot](../002-types/202-any-slot.md), dynamic dispatch occurs: the tag is checked at runtime and the appropriate field position is looked up.

## 3.5.2. Method call: `obj.method(args)`

```rok
let s = "hello";
let n = s.len();
let p = Point { x: 1.0, y: 2.0 };
let d = p.distance(other);
```

The form is `expr . method_name ( arguments )`. Method resolution looks at the static type of the left-hand side and finds a method defined on that type or a method of a trait that type satisfies.

### Method resolution order

A method with the same name may be defined in several places. The compiler looks for candidates in this order:

1. Methods defined in an inherent `impl` ([`impl`](../002-types/209-impl.md)) first.
2. Then methods of trait `impl`s, limited to traits the left-hand type satisfies that are visible in the current module. Visibility comes from the automatically visible traits of `rok::**`, traits imported with `use`, or traits defined in the same module.

If several candidates exist at once, it is an ambiguity compile error. The user must then resolve it explicitly with the fully qualified form of [type paths](../002-types/213-type-paths.md).

### Automatic `ref` for the method receiver

The receiver (`obj`) at a method call site is handled automatically according to the method's `self` kind (`self` / `ref self`). `ref` need not be written at the call site.

```rok
p.distance(other);          // self
p.replace_with(other);      // ref self - caller does not write 'ref'
```

This is a special rule for method receivers, different from the explicit `ref` rule for ordinary function calls. Details are defined in the function specification.

### Desugaring of method calls

```
obj.method(a, b)
```

The expression above is semantically equivalent to:

```
<typeof(obj) as TraitContaining_method>::method(obj, a, b)
```

(or `Type::method(obj, a, b)` if defined in an inherent `impl`).

This desugaring follows the same mechanism as the trait method mapping of [operator dispatch](301-operator-dispatch.md). The `.` token does not itself carry a mapping like `#[infix(".")]`, but the shape of the resulting call is the same.

### Turbofish: explicit generic methods

The generic arguments of a method can be given explicitly with the turbofish form of [type paths](../002-types/213-type-paths.md).

```rok
let v = expr.parse::<i32>();
let m = arr.collect::<List<f64>>();
```

The form is `.method::<...>(args)`: the method name follows `.`, then the turbofish, then the argument parentheses.

## 3.5.3. Difference from static item access

`Type::item` ([type paths](../002-types/213-type-paths.md)) accesses an item attached to a type.

```
Type::item       // static method / associated const / associated type - no instance
obj.item         // member / method attached to an instance - instance required
```

These two forms are entirely different kinds of access. The tokens differ (`::` vs. `.`) and so does the meaning.

### Method path values

A method path written without parentheses (`T::method`, `Trait::method`, `<T as Tr>::method`, `i32::add`, `S::new`) is a function value.

| Method kind      | Type of the function value                   |
|------------------|----------------------------------------------|
| static method    | `fn(args) -> R`                              |
| instance method  | `fn(Self, args) -> R` (`self` is the first argument) |

```rok
let mk = Point::new;               // fn(f64, f64) -> Point
let add = i32::add;                // fn(i32, i32) -> i32
xs.map(Point::norm);               // fn(Point) -> f64
```

- A method with `ref self` or a `ref` parameter cannot be a value ([4.7.6](../004-functions/407-ref-keyword.md#476-ref-and-types)).
- A throwing method gives a throwing function type (`fn(..) throws (E) -> R`).
- For `Tr::m`, or a method of a generic type whose `Self` is not determined, the function type is determined from the expected type. Without an expected type, a later call determines it (`let f = Box::get; f(b)`, [2.10.2](../002-types/210-generics.md#type-argument-inference)); if it is still undetermined, it is a "type annotations needed" error.
- The type arguments of a generic method are determined by turbofish or by the expected function type.
- `#[intrinsic]` methods can also be values.

A form attached to an instance, such as `obj.method`, is not a function value (3.5.4).

## 3.5.4. `.method` (without a call)

When `.method` is written without parentheses, rok does not interpret it as a function value. No separate meaning such as partial application or method reference is assigned.

```
let f = obj.method;        // ERROR: not a method call; the expression is incomplete
let f = || obj.method();   // wrap in an explicit closure
```

This decision avoids the ambiguity of implicit partial application and makes the user state the intent directly.

## 3.5.5. Safe member access: `?.`

Dot access on a value that may be absent is written with optional chaining, defined in [optional chaining and null coalescing](308-optional-chaining-and-coalescing.md).
