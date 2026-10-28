# 6.10. Reflection

The standard library provides two levels of reflection traits: `Reflect` and `Inspect`.

```
Reflect    automatic: shallow metadata every type carries for free
Inspect    opt-in via #[derive(Inspect)]: rich field/method description
```

The split into two levels controls binary size while still giving every type, for free, what is essential for `any` slots, sum types, and REPL auto-printing.

## 6.10.1. `Reflect` - automatically implemented trait

`Reflect` is minimal type introspection attached automatically to every type. Without the user writing `#[derive(Reflect)]`, the compiler synthesizes an impl for every type.

```rok
trait Reflect {
    // static: called directly on the type
    fn type_name() -> str;
    fn size() -> usize;
    fn align() -> usize;

    // instance: runtime dispatch on any slots / sum types
    fn type_name_of(self) -> str;
    fn size_of(self) -> usize;
}
```

```rok
i32::type_name() == "i32"
i32::size()      == 4
i32::align()     == 4

let x: any = 42i32;
x.type_name_of() == "i32"        // runtime type tag check

let v: i32 | f64 = pick();
v.type_name_of()                  // "i32" or "f64" (sum type variant check)
```

Being automatic, it requires no user code. The compiler already knows the `type_name`, `size`, and `align` of every type and synthesizes them at zero cost.

`Reflect` is provided by the compiler for every type; there is no impl in source.

- A handwritten `impl Reflect for T` is an error. The bound `T: Reflect` always holds.
- If a type has its own method of the same name, such as `size()` / `type_name()`, that method hides the reflection method.
- Names (`type_name`) carry no module path (`List<i64>`, `Config`). The static name of a sum type is written in canonical order ([2.6.4](../002-types/206-sum-types.md#264-set-semantics)) (`f64 | i32`, with `none` last). Instance forms such as `type_name_of(self)` report the member held at runtime (for a sum) or the contents of an `any`.
- `rok::core::reflect::cast<T>(value: any) -> T | none` extracts an `any` as `T` by the same rules as `set_field`. `rok::core::reflect::methods_of<T>() -> List<MethodInfo>` is the list of `T`'s inherent methods.

## 6.10.2. `Inspect` - opt-in trait

`Inspect` is rich reflection opted into with `#[derive(Inspect)]`. It exposes field names, dynamic field access and modification, and method descriptions.

```rok
trait Inspect: Reflect {
    fn field_names() -> List<str>;
    fn get_field(self, name: str) -> any | none;            // private fields too
    fn set_field(ref self, name: str, value: any) -> bool;  // whether it succeeded
    fn methods() -> List<MethodInfo>;

    // Default bodies; the derive overrides them for an enum.
    fn fields() -> List<FieldInfo>;          // fields_of::<Self>()
    fn variant_names() -> List<str>;         // empty for a struct
    fn variant_name(self) -> str;            // a struct: self.type_name_of()
    fn field_names_of(self) -> List<str>;    // the fields self holds
}

struct FieldInfo {
    name: str;                 // "0", "1", .. for a tuple field
    type_name: str;            // instantiated: f64 for `v: T` in Box<f64>
    is_pub: bool;              // true only for a plain `pub` field
}

struct MethodInfo {
    name: str;
    arity: usize;              // number of arguments excluding self
    is_static: bool;           // a method that does not take self
    arg_types: List<str>;      // argument type names excluding self
    return_type: str;
}
```

- `get_field` returns the field's value as `any`, or `none` for a nonexistent name. Private fields can be read too.
- `set_field` assigns and returns `true` when the value has **exactly the same** type as the field, or when an integer of a different integer type **fits without loss** (`u.set_field("id", 42)` into a `u64` field). Otherwise it does not assign and returns `false`. There are no other conversions (e.g. `i32` → `f64`).
- `methods()` returns **only the inherent methods** of the type (excluding trait methods), sorted by name. Methods of a generic `impl` show types as written (`T`).
- `fields()` lists every field in declaration order with its type name and whether it is `pub`. **Private fields are listed too**: deriving `Inspect` is the type author's consent, the same as for `get_field`. The type name is the instantiated one (`Box<f64>` reports `v: f64`, `items: List<f64>`), with no module path, as `type_name` writes it. `rok::core::reflect::fields_of<T: Inspect>() -> List<FieldInfo>` is the same list as a function.
- An `enum` has no fields at the type level (`field_names()` and `fields()` are empty lists). `variant_names()` lists its variants in declaration order. On a value, `variant_name()` is the variant it holds (`"Circle"`), `field_names_of()` that variant's fields (`"0"`, `"1"` for a tuple variant, the names for a named one), and `get_field` reads them. A variant's payload is read-only through `Inspect`: `set_field` on an enum is `false`.
- On a struct, `variant_names()` is an empty list, `variant_name()` is its type name (`type_name_of()`), and `field_names_of()` is its fields.
- `fields()`, `variant_names()`, `variant_name()` and `field_names_of()` have default bodies, so a hand-written `impl Inspect` needs only the first four methods.
- Calling an `Inspect` method on a type without the derive is an error whose help says what to write: ``add `#[derive(Inspect)]` to `Point` ``.

Usage example:

```rok
#[derive(Inspect)]
struct Config { host: str; port: i32; }

let c = Config { host: "localhost", port: 8080 };
Config::field_names()              // ["host", "port"]
c.get_field("port")                // 8080 inside an any
c.set_field("port", 9090)          // true (success)
c.set_field("xxx", 1)              // false (no such field)
Config::fields()[1].type_name      // "i32"
```

`Inspect` is opt-in because its metadata must be stored in the binary. Attaching it to every type automatically would impose too large a binary size cost.

`#[derive(Inspect)]` is implemented by the [meta block](608-meta-blocks.md) `#[impl_derive(Inspect)]` in `rok::core::reflect`, a pattern in which the meta-system itself implements a standard library trait ([stdlib 2.12.3](../../refs-standard-libraries/200-core/212-reflect.md#2123-inspect---opt-in-fields--methods)).

## 6.10.3. REPL / Jupyter auto-printing

When the REPL prints a bare last expression it uses `Display`; a trailing `?` uses `Debug`; `:type` prints the expression's static type, found by type checking without running it ([9.7.7](../009-tools/907-jit-cells.md#977-visualization--output)).

```
> let x = 42i32;
> x
42

> [1, 2, 3]
List<i32>

> Config { host: "h", port: 80 }
Config

> Config { host: "h", port: 80 }?
Config { host: "h", port: 80 }
```

A bare expression uses `Display` and a trailing `?` uses `Debug`. A value without the requested trait (`Display` for a bare expression, `Debug` for a trailing `?`) prints only its type name. This behavior is used by REPL and Jupyter cell sessions ([9.7](../009-tools/907-jit-cells.md)).

## 6.10.4. No global `typeof` / `sizeof` functions

Introspection that other languages expose through keywords such as `typeof` or `sizeof` is provided in Rok only as trait methods.

```rok
// rok: method calls
T::size()
x.type_name_of()

// other languages: keywords
sizeof(T)        // C / C++
typeof(x)        // JS / TS
```

The reason is that this adds no language syntax. The single `Reflect` trait covers the entire introspection surface.

## 6.10.5. Dispatch on `any` slots and sum types

`any` slots and sum types of the form `T1 | T2` determine the actual type at runtime, so the `type_name_of` call is important.

```rok
fn handle(x: any) {
    if x.type_name_of() == "i32" { handle_int(x); }
    else if x.type_name_of() == "str" { handle_str(x); }
}

fn classify(v: i32 | f64) -> str {
    match v {
        n: i32 => "integer",
        f: f64 => "float",
    }
}
```

The compiler desugars type-test patterns (`v: T => ...`) into `Reflect` tag checks.

## 6.10.6. Relation between the levels

As the notation `Inspect: Reflect` shows, `Inspect` has `Reflect` as a supertrait. A type implementing `Inspect` therefore always implements `Reflect` too, trivially, since `Reflect` is automatic anyway.

```
every type           Reflect          (automatic, zero cost)
+ #[derive(Inspect)] Inspect+Reflect  (opt-in, metadata embedded)
```
