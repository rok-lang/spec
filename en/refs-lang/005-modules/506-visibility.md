# 5.6. Visibility

Giving each item - function, type, trait, module, `const`, `static` - a visibility controls how far it is exposed.

```
visibility      ::= "pub" ("(" reach ")")?
reach           ::= "pkg" | "sub"
```

## 5.6.1. Four levels

Visibility has four levels:

| Keyword    | Meaning                                                   |
|------------|-----------------------------------------------------------|
| (none)     | Module-private. Within the same module only.              |
| `pub(sub)` | Only within submodules of the same module tree.           |
| `pub(pkg)` | Within the same package only.                             |
| `pub`      | Also from external packages.                              |

The narrowest scope is the default, private; the widest is `pub`.

`pkg` and `sub` are contextual keywords that act as qualifiers only inside the `pub(...)` parentheses; elsewhere they are ordinary identifiers (see [1.5.4 Visibility qualifiers](../001-lexical/105-keywords.md#visibility-qualifiers-pkg--sub)).

## 5.6.2. Default: no visibility keyword

Without a visibility keyword an item is module-private and visible only within the same module.

```rok
mod m {
    fn helper() -> i32 { 42i32 }   // module-private (default)
    pub fn api() -> i32 { helper() }
}

let r = m::api();                   // OK
let h = m::helper();                // ERROR - module-private
```

A private `mod` is visible from the parent module that declared it and from that parent's submodules.

## 5.6.3. `pub`: the widest exposure

For the widest exposure, add `pub`.

```rok
pub fn calculate(x: i32) -> i32 { ... }
pub struct Point { pub x: f64; pub y: f64; }
pub trait Display { ... }
```

Items declared this way can be imported with `use` and used from other packages.

## 5.6.4. `pub(pkg)`: within the package

`pub(pkg)` means public within the package.

```rok
pub(pkg) fn internal_helper() -> i32 { ... }
```

It is visible from any module in the same package but not from external packages. Dependency packages and each tier of `rok::` are separate packages too ([5.7.14](507-paths.md#5714-paths-between-packages)).

This level is for package-internal library tooling: functions and types that are not public API but must be shared across modules.

## 5.6.5. `pub(sub)`: submodules

`pub(sub)` is used to share only within one branch of the module tree.

```rok
mod parent {
    pub(sub) fn shared_within_subtree() -> i32 { ... }

    pub mod child {
        pub fn use_it() -> i32 { sup::shared_within_subtree() }   // OK
    }
}
```

It is visible only in `parent` and all of `parent`'s descendant modules, not in parent or sibling modules.

This level is for tooling internal to a group of modules, shared only within one branch of the module tree.

## 5.6.6. Inclusion between levels

Visibility forms a monotonically increasing lattice.

```
pub  ⊇  pub(pkg)  ⊇  pub(sub)  ⊇  private
```

The inclusion for `pub(sub)` is meaningful only among the related modules, that is, the defining module and its descendants.

- `pub` is visible to every caller.
- `pub(pkg)` is visible only within the same package.
- `pub(sub)` is visible only within the same module tree.
- private is visible only within the same module.

## 5.6.7. Glob imports and visibility

A glob import (`*`, `**`, including the implicit `use rok::core::**`) imports only items visible from the importing module. An item is pulled in only if its reach includes that module, and invisible items neither hide names nor cause conflicts. There is no per-item glob-block marker; to narrow the external surface, create a separate `prelude` module and curate it with `pub use`.

## 5.6.8. Visibility of `struct` fields

Each field may be given its own visibility.

```rok
struct Account {
    pub id: i64;                    // readable externally
    pub balance: f64;               // readable externally
        secret: str;                // module-private
}
```

Field visibility works independently of the visibility of the `struct` itself.

```rok
pub struct Counter {                // the struct itself is visible externally
        count: i32 = 0i32;          // the field is module-private (default)
}

let c = Counter { count: 0i32 };    // ERROR - private field, cannot construct directly from outside
let c = Counter::new();             // OK - factory method (if any)
```

This is the standard pattern for exposing a type while preventing direct instantiation (newtype, opaque type).

### Fields of tuple structs

Each field of a tuple struct also has its own visibility, private by default.

```rok
pub struct S(pub i32, i32);         // field 0 is pub, field 1 is private
```

- Constructing with `S(a, b)` requires every field to be visible at the use site. If any field is private, outside its visibility scope a constructor function is used (`NonZero::new`).
- Reading or assigning `x.N` requires field N to be visible.
- In the pattern `S(p0, p1, ..)`, every position whose subpattern is not `_` must be visible. The same holds for struct patterns with named fields (only the touched fields are checked).
- A `(` immediately after `pub` is read as a visibility restriction (`pub(pkg)` etc.). To make a field of tuple type public, add another pair of parentheses: `struct S(pub ((i32, i32)));`.

## 5.6.9. Visibility of `enum` variants

Variants have no separate visibility; they follow the visibility of the `enum`. A visibility keyword before a variant is a compile error.

```rok
pub enum Shape {
    Circle(f64),                    // pub - follows Shape
    Square(f64),
}

pub enum Bad {
    pub A,                          // ERROR - no visibility on variants
}
```

Payload fields of variants also have no per-field visibility; they follow the `enum`. Unlike `struct` fields, hiding a single variant would make exhaustiveness impossible for external `match`es, hence this rule. By this rule the standard library's `pub enum Result<T, E> { Ok(T), Err(E) }` makes `Ok` / `Err` public.

## 5.6.10. Visibility of `impl` methods

Methods of an inherent `impl` ([`impl`](../002-types/209-impl.md)) are module-private by default; to expose them, mark them `pub` explicitly.

```rok
impl Account {
    pub fn balance(self) -> f64 { self.amount }
        fn _internal_check(self) { ... }
}
```

Methods of a trait `impl` follow the visibility of the trait and cannot be marked separately.

## 5.6.11. Visibility of `use`

Imports via [`use`](505-use.md) also have a visibility.

```rok
use foo::bar;                       // module-private (default)
pub use foo::bar;                   // re-export - visible externally too
pub(pkg) use foo::bar;              // within the package only
```

Re-exports, combined with the visibility section of [`use`](505-use.md), determine the external surface.

## 5.6.12. Position of visibility

A visibility keyword appears as the first token of an item declaration.

```rok
pub fn f() { ... }
pub(pkg) fn g() { ... }
pub struct S { ... }
pub mod m { ... }
pub const X: i32 = 0;
pub static Y: i32 = 0;
pub use foo::bar;
pub trait T { ... }
```

It may be attached to every declaration form of `fn`, `struct`, `enum`, `union`, `trait`, `impl` (not on the `impl` itself but on each method), `mod`, `const`, `static`, `use`, and `type` (alias).

## 5.6.13. Visibility and name resolution

Combined with [`use`](505-use.md), an item is visible in the current module only if both of the following hold.

1. The item's visibility reaches the current module. Every module segment of the path must also be visible (`a::b::f` requires `a`, `b`, and `f` all to be visible).
2. The item is directly visible (a child module or a name imported with `use`).

When both hold, the item can be used by its short name.

For an item with insufficient visibility, the compiler diagnostic reports why it is not visible.

```
ERROR: 'foo::secret' is not visible from this module.
       It is declared as private - only visible within 'foo'.
       To use it externally, add 'pub' or other visibility keyword.
```
