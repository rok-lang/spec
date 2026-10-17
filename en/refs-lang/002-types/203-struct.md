# 2.3. `struct`

`struct` is the container syntax that groups several fields into one, defining a product type. This chapter covers the forms a `struct` can take and the rules for structs: instance creation, default values, spread, visibility and memory layout.

## 2.3.1. Three forms

A `struct` declaration follows this grammar.

```
struct_decl     ::= "struct" identifier generics? where_clause? "{" named_field_list? "}"
                  | "struct" identifier generics? "(" tuple_field_list? ")" where_clause? ";"   // tuple struct: where comes after the field list
                  | ";"
named_field_list ::= named_field (";" named_field)* ";"?
named_field     ::= identifier ":" type_expr
tuple_field_list ::= type_expr ("," type_expr)* ","?
```

The shape of the body determines one of three forms, covered one by one below.

### Named `struct`: named fields

The most common form is the named `struct`, where each field has a name.

```rok
struct Point {
    x: f64;
    y: f64;
}

struct User {
    id: i64;
    name: str;
    age: i32;
}
```

Rules: the body is enclosed in `{ ... }`. Each field is written as `name: Type;`, and the `;` at the end of a field is the same terminator as the [statement terminator](../001-lexical/103-statement-terminator.md). Field types are required, because, as established in [The `any` slot](202-any-slot.md), struct fields do not allow gradual typing.

### Tuple `struct`: positional fields

Next, the tuple `struct` identifies fields by position only, without names.

```rok
struct Pair(i32, i32);
struct Wrapper(str);
struct Color(u8, u8, u8);
```

In this form the body is enclosed in `( ... )`, and each element has only a type, no name. Fields are accessed by positional index such as `.0`, `.1`; the exact syntax is covered in the expression specification. The single-field form `struct W(T);` is also allowed and can be used as the newtype pattern.

### Unit `struct`: field-less marker

Finally, the unit `struct` has no fields at all.

```rok
struct Marker;
struct Sentinel;
```

This form has no body and ends with `;` immediately after the declaration. It is mainly used for marker types where only a single instance is meaningful.

## 2.3.2. Instance creation: preview

The instance creation syntax for each form is formally defined in the expression specification. This document only previews what each form looks like.

```rok
let p = Point { x: 1.0f64, y: 2.0f64 };    // named
let q = Pair(1i32, 2i32);                   // tuple
let m = Marker;                              // unit
```

## 2.3.3. Field default values

Fields of a named `struct` can have default values.

```rok
struct Config {
    retry: i32 = 3i32;
    timeout: f64 = 30.0f64;
    verbose: bool = false;
}
```

Default values are constrained as follows. Only `const` expressions are allowed: literals, `const` variables, and `const` function calls. `const` expressions are defined in [`const`](../005-modules/502-const.md). Expressions that require a runtime call, such as the current time or a random number, cannot be used; such code is a compile error. Referring to another field of the same struct is also forbidden. A default is evaluated only when the field is omitted at instance creation, and it is evaluated for each instance. Because evaluation is deterministic, the compiler may apply caching optimizations.

These constraints ensure determinism, predictable cost, and SemVer safety. If runtime initialization is needed, use a separate factory function, i.e. the `fn new() -> T` pattern.

## 2.3.4. Field spread

Sometimes a new instance should copy the fields of an existing instance, changing only some of them.

```rok
let base = Point { x: 0.0f64, y: 0.0f64 };
let p = Point { x: 5.0f64, ..base };
```

Here `..base` means take all fields of base as they are, overriding the explicitly written fields with new values.

Syntactically: an instance value follows the `..` token ([Punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md)). `..` may appear only once in an instance creation expression; two or more spreads are not allowed. Placing the spread after the explicit fields is the idiomatic style.

## 2.3.5. Visibility

Each field can control its external exposure with the visibility keywords defined in [Keywords](../001-lexical/105-keywords.md).

```rok
struct Account {
    pub id: i64;        // readable from outside
    pub balance: f64;
        secret: str;    // only within the same module (default = module-private)
}
```

Fields of a tuple struct also each have their own visibility (`struct S(pub i32, i32);`, private by default). The rules for construction, `x.N` access and patterns, and the meaning of the visibility levels, are given in [5.6.8](../005-modules/506-visibility.md#568-visibility-of-struct-fields).

## 2.3.6. Memory layout

Fields are laid out in declaration order with natural alignment, the same as `#[repr(C)]` ([6.2.2](../006-meta-system/602-built-in-attributes.md#622-repr-explicit-memory-representation)). `packed` and `align(N)` change this.

## 2.3.7. No duplicate definitions in the same module

Declaring two structs with the same name in the same module is a compile error. Duplicate field names within one struct are also an error. This rule applies regardless of visibility level.
