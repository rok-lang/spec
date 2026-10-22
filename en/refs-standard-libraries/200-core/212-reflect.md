# 2.12. reflect — `Reflect` / `Inspect`

The `rok::core::reflect` module provides the authoritative signatures for runtime type information and **field introspection**. The detailed semantics are governed by [refs-lang 6.10 reflection](../../refs-lang/006-meta-system/610-reflection.md); this chapter covers only the standard library's trait shapes.

## 2.12.1. Two-level traits

Reflection is split into two levels of traits.

| Trait     | Automatic / opt-in            | Provides                          | Where used                                         |
|-----------|-------------------------------|-----------------------------------|----------------------------------------------------|
| `Reflect` | **automatic** (every type)    | type_name / size / align          | available through the implicit `rok::**` (rarely used) |
| `Inspect` | opt-in (`#[derive(Inspect)]`) | field_names / get_field / methods | trait available through the implicit `rok::**`, derive is explicit |

The split into automatic and opt-in controls binary size. If every struct carried field reflection metadata, the binary would grow; the opt-in scheme generates the metadata only for the types that need it.

## 2.12.2. `Reflect` — both static and instance

```rok
trait Reflect {
    fn type_name() -> str;        // static — T::type_name()
    fn size() -> usize;
    fn align() -> usize;

    fn type_name_of(self) -> str; // instance — x.type_name_of() (any / sum also OK)
    fn size_of(self) -> usize;
}
```

This trait is implemented automatically for every type; the compiler attaches the type info to the binary automatically.

The compiler provides `Reflect` for every type; there is no impl in source.

- A hand-written `impl Reflect for T` is an error. The bound `T: Reflect` always holds.
- If a type has its own method with the same name, such as `size()` / `type_name()`, that method shadows the reflection method.
- The name (`type_name`) carries no module path (`List<i64>`, `Config`). The static name of a sum type is written in canonical order ([2.6.4](../../refs-lang/002-types/206-sum-types.md#264-set-semantics)) (`f64 | i32`, with `none` last). Instance forms such as `type_name_of(self)` report the member present at runtime (for a sum) or the content of an `any`.
- `rok::core::reflect::cast<T>(value: any) -> T | none` extracts a `T` from an `any` using the same rules as `set_field`. `rok::core::reflect::methods_of<T>() -> List<MethodInfo>` is the list of `T`'s inherent methods.

### Static vs instance

```rok
i32::type_name();          //→ "i32"           (static — decided at compile time)
i32::size();               //→ 4
i32::align();              //→ 4

let x: i32 = 42;
x.type_name_of();          //→ "i32"           (instance — usually the same as static)

let y: i32 | f64 = 42i32;
y.type_name_of();          //→ "i32"           (runtime dispatch — checks the sum tag)

let z: any = "hello";
z.type_name_of();          //→ "str"           (runtime — any slot dispatch)
```

`type_name_of` is a runtime dispatch, so for a sum type or `any` it returns the name of the variant currently held.

### No global functions

```rok
typeof(x);                 // ERROR — no global typeof
sizeof(T);                 // ERROR — no global sizeof
```

Instead, everything is uniformly a method.

```rok
x.type_name_of()           // instance
T::type_name()             // static
T::size()
T::align()
```

This is part of [refs-lang 3.5 Member access and method calls](../../refs-lang/003-expressions/305-member-access-and-method-call.md): all introspection is unified in method form.

## 2.12.3. `Inspect` — opt-in fields / methods

```rok
trait Inspect: Reflect {
    fn field_names() -> List<str>;
    fn get_field(self, name: str) -> any | none;            // private fields too
    fn set_field(ref self, name: str, value: any) -> bool;  // whether it succeeded
    fn methods() -> List<MethodInfo>;
}

struct MethodInfo {
    name: str;
    arity: usize;              // number of arguments excluding self
    is_static: bool;           // method that does not take self
    arg_types: List<str>;      // argument type names excluding self
    return_type: str;
}
```

- `get_field` returns the field's value as `any`, or `none` for an unknown name. Private fields can be read too.
- `set_field` assigns and returns `true` when the value's type is **exactly the same** as the field's type, or when an integer of a different integer type **fits without loss** (`u.set_field("id", 42)` into a `u64` field). Otherwise it does not assign and returns `false`. There are no other conversions (e.g. `i32` → `f64`).
- `methods()` returns **only the type's inherent methods** (excluding trait methods), sorted by name. Methods of a generic `impl` show the types as written (`T`).
- Derived on an `enum`, there are no fields (`field_names()` is an empty list, `get_field` is `none`, `set_field` is `false`); only `methods()` is meaningful.

### Derive example

```rok
#[derive(Inspect)]
struct User {
    id: u64;
    name: str;
    email: str;
}

let u = User { id: 1, name: "Alice", email: "a@b" };

User::field_names();       //→ ["id", "name", "email"]
u.get_field("name");       //→ any("Alice")
u.set_field("id", 42);     //→ true
u.id;                      //→ 42
```

### Using meta blocks

`#[derive(Inspect)]` is implemented as a meta block in `rok::core::reflect` ([refs-lang 6.8](../../refs-lang/006-meta-system/608-meta-blocks.md), [6.9](../../refs-lang/006-meta-system/609-template-tokens.md)). In summary:

```rok
#[impl_derive(Inspect)]
pub @derive {
    impl rok::core::reflect::Inspect for @Self {
        fn field_names() -> List<str> {
            let names: List<str> = List::new();
            @if_struct {
                @for_fields { names.push(@field.name.as_str()); }
            }
            return names;
        }

        fn get_field(self, name: str) -> any | none {
            @if_struct {
                @for_fields {
                    if name == @field.name.as_str() { return self.@field; }
                }
            }
            return none;
        }

        fn set_field(ref self, name: str, value: any) -> bool {
            @if_struct {
                @for_fields {
                    if name == @field.name.as_str() {
                        match rok::core::reflect::cast::<@field.type>(value) {
                            v: @field.type => { self.@field = v; return true; },
                            none => { return false; },
                        }
                    }
                }
            }
            return false;
        }

        fn methods() -> List<rok::core::reflect::MethodInfo> {
            return rok::core::reflect::methods_of::<@Self>();
        }
    }
}
```

For meta block details see [refs-lang 6.8 Meta blocks](../../refs-lang/006-meta-system/608-meta-blocks.md).

### Where dynamic dispatch is needed

`Inspect` is used in domains that need access by field name, such as config objects, serializers and debuggers. In ordinary code `Reflect` alone is enough.

```rok
fn dump_struct<T: Inspect>(x: T) {
    for name in T::field_names() {
        if let v: any = x.get_field(name) {
            println(f"{name} = {v.type_name_of()}: {v:?}");
        }
    }
}
```

## 2.12.4. REPL output

The REPL's automatic printing of the last expression combines this chapter's `Reflect::type_name_of` with `Debug` from [2.10 fmt](210-fmt.md) ([refs-lang 6.10.3](../../refs-lang/006-meta-system/610-reflection.md#6103-repl--jupyter-auto-printing)).

```
> let x = 42i32;
> x
i32: 42

> [1, 2, 3]
List<i32>: [1, 2, 3]

> User { id: 1, name: "Alice" }
User: User { id: 1, name: "Alice" }
```

The display form is `{type_name_of}: {Debug}`. For a type without `Debug`, only the type name is printed.

## 2.12.5. Module mapping

| Item               | Location                                          | Mechanism                         |
|--------------------|---------------------------------------------------|-----------------------------------|
| `Reflect` (automatic) | compiler + `rok::core::reflect`                | type info table — attached to the binary |
| `Inspect` derive   | `rok::core::reflect` + meta block `#[impl_derive]` | code generation from template tokens |
| `MethodInfo`       | `rok::core::reflect`                              | plain struct                      |
| `any` type         | compiler built-in                                 | type tag + payload                |

## 2.12.6. Comparison with other languages

| Aspect              | Rust                         | Java               | Python              | rok                                         |
|---------------------|------------------------------|--------------------|---------------------|---------------------------------------------|
| Type name           | `std::any::type_name::<T>()` | `getClass()`       | `type(x).__name__`  | `x.type_name_of()` method                   |
| Field introspection | **none** (libraries such as `bincode`) | Reflection API | `vars(x)` `getattr` | `x.get_field(name)` (opt-in)          |
| Runtime dispatch    | `dyn Trait` (vtable)         | automatic          | automatic           | sum type tag + `any` / `any Trait` (opt-in) |
| Automatic impl      | **none**                     | every object       | every object        | `Reflect` automatic, `Inspect` opt-in       |
| Memory cost         | 0                            | metadata for every class | dict for every object | only `Reflect` automatic (opt-in split) |

rok differs in two main ways. First, the two-level split provides the lightweight `Reflect` automatically and the heavyweight `Inspect` as opt-in. This is more memory-efficient than the all-automatic model of Java or Python and more ergonomic than Rust's all-manual model. Second, for method consistency there are **no** global functions such as `typeof(x)`; all introspection is provided in method form.

The detailed reflection semantics are governed by [refs-lang 6.10](../../refs-lang/006-meta-system/610-reflection.md).
