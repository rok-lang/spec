# 2.5. `union`

`union` is an untagged container whose fields share the same memory. The language does not track which field's value is meaningful at any moment; that is the user's responsibility.

```
union_decl      ::= "union" identifier generics? where_clause? "{" named_field_list? "}"
```

## 2.5.1. Basic form

The simplest `union` is declared as:

```rok
union EpollData {
    ptr: usize;
    fd: i32;
    val: u64;
}
```

The body and field syntax are the same as the named `struct` of [`struct`](203-struct.md). The differences are the keyword `union` and the semantics: all fields share the same memory location.

## 2.5.2. Semantics

The following example shows how a `union` behaves.

```rok
let x = EpollData { fd: 5i32 };
x.fd = 7i32;                    // write — safe
let y = unsafe { x.val };       // read — requires unsafe; reinterprets x's bit pattern as u64
```

Creating an instance with some field writes that field's bits to memory. Reading it later through another field reinterprets the same memory bits as that field's type. Writing a field is safe, but reading a field is allowed only inside [`unsafe`](../004-functions/415-unsafe.md#4152-unsafe-----block-expression), because the language does not guarantee that the bits read are a valid value of that type (e.g. `bool`, `char`, pointers).

## 2.5.3. Where safety lies

Bit reinterpretation by a `union` is behavior defined by the language, but the meaning of the result is the program domain's responsibility. For example, writing `1.0` to an `f32` field and reading the `i32` field gives `1065353216` (`0x3F800000`); conversely, reading `i32` bits as `f32` gives the real value given by the IEEE 754 bit representation. The language does not check whether that value is meaningful for the domain.

This model differs from C's `union`. In C, reading from a different member is undefined behavior; in rok it is defined as bit reinterpretation. Since the result is not guaranteed to be a valid value, reads are marked `unsafe`.

## 2.5.4. Default layout is left to the compiler

The default `union` layout is left to the compiler. It need only satisfy the size and alignment of the largest field; the exact field alignment and padding are decided by the compiler.

If a C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface)-compatible `union` is needed, attach `#[repr(C)]`.

```rok
#[repr(C)]
union CUnion {
    a: i32;
    b: f32;
}
```

A `union` with `#[repr(C)]` is bit-compatible with C's `union` and is used in [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) interfaces. The exact meaning is defined in [Built-in attributes](../006-meta-system/602-built-in-attributes.md).

## 2.5.5. `union` vs `#[repr(C)] union`

The two forms differ as follows.

| Form                         | Used in     | Layout            | Bit compatibility         |
|------------------------------|-------------|-------------------|---------------------------|
| `union T { ... }`            | Ordinary code | Left to compiler | Guaranteed only within rok |
| `#[repr(C)] union T { ... }` | FFI         | Same as C ABI     | Compatible with C code    |

## 2.5.6. Visibility / duplicate definitions

The same rules as [`struct`](203-struct.md) apply.

> [!WARNING]
> Reading a `union` field is a bit reinterpretation and is allowed only inside `unsafe` (writing is safe). Whether the bits read are a valid value of that type, and whether it is meaningful for the domain, is the author's responsibility.

## 2.5.7. `union` vs sum types

`union` and sum types may look similar, but they are entirely different tools.

```rok
union  Foo { a: i32; b: f32; }    // same memory, no tag
type   Bar = i32 | f32;            // separate memory, tagged
```

The differences in each aspect:

| Aspect                         | `union`                          | Sum type (`A \| B`)            |
|--------------------------------|----------------------------------|-------------------------------|
| Tracks which field is active   | No                               | Tracked by tag                |
| Conversion between fields      | Bit reinterpretation             | Variant change (value rebuilt) |
| Memory use                     | Size of the largest field        | Tag + size of the largest variant |
| Reading a field                | Requires `unsafe` (bit reinterpretation) | Safe, via patterns / `is` |

The two serve completely different purposes: `union` provides different interpretations at the bit level, and a sum type expresses different kinds at the value level. They must not be confused.
