# 2.4. `enum`

`enum` defines a type whose value is one of several variants. It is similar to a sum type but differs in being a named container.

## 2.4.1. Basic form

The grammar of `enum` is:

```
enum_decl   ::= visibility? "enum" identifier generics? where_clause? "{" variant_list? "}"
variant_list ::= variant ("," variant)* ","?
variant     ::= visibility? identifier                                  // unit
              | visibility? identifier "(" type_expr ("," type_expr)* ","? ")"  // tuple
              | visibility? identifier "{" named_field_list? "}"        // struct
              | visibility? identifier "=" const_expr                   // discriminant
```

The simplest form looks like this:

```rok
enum Direction {
    North,
    East,
    South,
    West,
}
```

The body is enclosed in `{ ... }`, and variants are separated by commas. A trailing comma is allowed.

## 2.4.2. Four kinds of variants

`enum` variants fall into four kinds according to how they hold data.

### Unit variant: no data

The simplest, the unit variant, holds no data.

```rok
enum Color {
    Red,
    Green,
    Blue,
}
```

It consists only of a plain label without data.

### Tuple variant: positional fields

A tuple variant has positional fields.

```rok
enum Shape {
    Circle(f64),
    Rectangle(f64, f64),
}
```

It has positional fields like the tuple `struct` of [`struct`](203-struct.md).

### Named variant: named fields

A named variant has named fields.

```rok
enum Event {
    Click { x: i32; y: i32; },
    Scroll { dy: i32; },
    Resize { w: i32; h: i32; },
}
```

Each variant has fields shaped like a named `struct`. Fields end with `;`, consistent with [`struct`](203-struct.md).

### Value variant: explicit integer value

Finally, a value variant assigns an integer value directly to the variant.

```rok
enum Status {
    Ok = 0,
    NotFound = 404,
    InternalError = 500,
}
```

The assigned value must be a `const` integer expression.

This form is used for C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) compatibility, mapping external protocols, bit flags, and so on. A value variant cannot be mixed with other variant kinds: if any variant is a value variant, the whole `enum` must consist only of value variants.

## 2.4.3. Mixing variant kinds

Unit, tuple and named variants can be mixed freely in one `enum`. Value variants, as stated above, may only be used alone.

The following example uses all three kinds together.

```rok
enum Result {
    Ok(i32),                    // tuple
    Err { code: i32; msg: str; }, // named
    Pending,                    // unit
}
```

## 2.4.4. Visibility

Variants have no visibility of their own; they follow the visibility of the `enum`. All variants of a `pub enum` are public ([5.6.9](../005-modules/506-visibility.md#569-visibility-of-enum-variants)).

## 2.4.5. Memory layout

The default layout is left to the compiler: it may choose the alignment and packing of tag and payload. This freedom enables [niche optimization](207-niche-optimization.md).

If C ABI compatibility is needed, the `#[repr(...)]` attribute can force a representation. Its exact meaning is defined in [Built-in attributes](../006-meta-system/602-built-in-attributes.md).

## 2.4.6. No duplicate definitions in the same module

Declaring two `enum`s with the same name in the same module is a compile error. Duplicate variant names within one `enum` are also an error.

## 2.4.7. `enum` vs sum type vs `union`: preview

The three forms compare as follows.

| Form              | Naming                                         | Tag  | Variants share memory?             |
|-------------------|------------------------------------------------|------|------------------------------------|
| `enum`            | Named variants defined inside the container    | Yes  | No (separate payloads)             |
| Sum type (`A \| B`) | The variants are existing types              | Yes  | No (inline tag + payload)          |
| `union`           | Untagged; the same memory read as several types | No  | Yes (intentional overlap)          |

The detailed differences are defined in each container's document.
