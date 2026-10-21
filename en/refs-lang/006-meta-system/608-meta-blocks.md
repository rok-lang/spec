# 6.8. User-defined meta blocks

The three forms `#[impl_derive(...)]`, `#[impl_attribute(...)]`, and `#[impl_derive_attribute(...)]` are code generators that let users define their own derives or attributes; they replace Rust's proc macros.

```
meta_block      ::= meta_attr visibility? meta_kind block
meta_attr       ::= "#" "[" meta_kind_attr "(" meta_args ")" "]"
meta_kind_attr  ::= "impl_derive" | "impl_attribute" | "impl_derive_attribute"
meta_kind       ::= "@derive" | "@attribute"
meta_args       ::= identifier ("," identifier)*
```

## 6.8.1. Three kinds

The three forms:

```rok
#[impl_derive(TraitName)] pub @derive {
    impl TraitName for @Self { ... }
}

#[impl_attribute(name)] pub @attribute {
    @item                                    // reuse the original item
    fn additional_helper() { ... }            // generate an additional item
}

#[impl_derive_attribute(Trait, attr)] pub @derive {
    @for_fields {
        @apply_if(@has_derive_attr(attr)) {
            // process only fields carrying that attr
        }
    }
}
```

What each form is used for:

| Form                                    | Use                                        |
|-----------------------------------------|--------------------------------------------|
| `#[impl_derive(TraitName)]`             | User-defined derive                        |
| `#[impl_attribute(name)]`               | User-defined attribute (item wrapper)      |
| `#[impl_derive_attribute(Trait, attr)]` | Per-field derive attribute                 |

## 6.8.2. `#[impl_derive(TraitName)]` user-defined derive

An example of a user-defined derive:

```rok
#[impl_derive(MyHash)] pub @derive {
    impl MyHash for @Self {
        fn my_hash(self) -> u64 {
            let h: u64 = 0u64;
            @for_fields {
                h = combine_hash(h, self.@field.my_hash());
            }
            h
        }
    }
}

// use site
#[derive(MyHash)]
struct Point { x: f64; y: f64; }
// → generated automatically.
//   impl MyHash for Point {
//       fn my_hash(self) -> u64 {
//           let h: u64 = 0u64;
//           h = combine_hash(h, self.x.my_hash());
//           h = combine_hash(h, self.y.my_hash());
//           h
//       }
//   }
```

This form consists of `#[impl_derive(name)]` and a body (template block). When `#[derive(name)]` appears at a use site, the template is expanded for that type to generate code.

## 6.8.3. `#[impl_attribute(name)]` user-defined attribute

An example of a user-defined attribute:

```rok
#[impl_attribute(timed)] pub @attribute {
    @accept(fn)                                  // attachable only to fn items

    fn @item.name(@item.params) -> @item.return_type {
        let start = now();
        let result = {
            @item.body
        };
        log(f"@item.name took {now() - start}");
        result
    }
}

// use site
#[timed]
fn process(x: i32) -> i32 {
    expensive(x)
}
// → generated automatically.
//   fn process(x: i32) -> i32 {
//       let start = now();
//       let result = { expensive(x) };
//       log(f"process took {now() - start}");
//       result
//   }
```

This is a wrapper around an item: it reuses the contents and signature of the original item (`@item`) to generate a transformed form. It can be attached not only to functions but also to `struct`, `enum`, and `impl`; target restrictions are expressed with `@accept`.

## 6.8.4. `#[impl_derive_attribute(Trait, attr)]` per-field derive attribute

An example of a per-field derive attribute:

```rok
#[impl_derive_attribute(Serialize, rename)] pub @derive {
    impl Serialize for @Self {
        fn serialize(self) -> Json {
            let obj = JsonObj::new();
            @for_fields {
                @apply_if(@has_derive_attr(rename)) {
                    obj.set(@derive_attr(rename), self.@field.serialize());
                }
                @apply_if(!@has_derive_attr(rename)) {
                    obj.set(@field.name.as_str(), self.@field.serialize());
                }
            }
            obj.into()
        }
    }
}

// use site
#[derive(Serialize)]
struct User {
    #[rename("user_name")]
    name: str;
    age: i32;
}
// → generated automatically.
//   impl Serialize for User {
//       fn serialize(self) -> Json {
//           let obj = JsonObj::new();
//           obj.set("user_name", self.name.serialize());
//           obj.set("age", self.age.serialize());
//           obj.into()
//       }
//   }
```

This form takes two arguments — the Trait being derived and the attribute that marks per-field information — and branches inside the derive body by reading the fields' attribute information.

## 6.8.5. Body (template block)

In all three forms the body is a template block containing both ordinary rok code and template tokens (`@Self`, `@field`, `@for_fields`, etc.).

The full list of template tokens is defined in [template tokens and expansion](609-template-tokens.md).

## 6.8.6. `pub` / visibility

Visibility is written as:

```rok
pub @derive { ... }
pub(pkg) @attribute { ... }
@derive { ... }                  // private
```

The visibility keyword is attached to the template block itself. Its effect:

| Visibility                    | Where it can be used                                       |
|-------------------------------|------------------------------------------------------------|
| `pub` (including `pub(pkg)`)  | The whole build. Used as `#[derive(X)]` / `#[x]` without `use` |
| private                       | Within the file that defines the meta block                |

Derive and attribute names are build-global names without a module path (6.8.11). A meta block's or keyword handler's `#[cfg]` is evaluated with the cfg of the package that defines it (its own features, [9.10.4c](../009-tools/910-rok-frontend.md#9104c-features)).

## 6.8.7. Body introducers (`@derive` / `@attribute`)

The body introducer indicates the kind of template.

| Introducer   | Use                                                       |
|--------------|-----------------------------------------------------------|
| `@derive`    | derive definitions (`impl_derive`, `impl_derive_attribute`) |
| `@attribute` | attribute definitions (`impl_attribute`)                  |

The introducer determines which template tokens are meaningful in the body. For example, `@item` is attribute-only.

## 6.8.8. Fixed-point expansion

If the result of template expansion produces an item carrying another derive or attribute, that item is expanded recursively.

```rok
#[impl_attribute(adds_helper)] pub @attribute {
    @item                                       // the original item as is

    #[derive(MyHelper)]                          // new item — derive processed again
    struct Helper { ... }
}
```

When this attribute is applied, the `#[derive(MyHelper)]` in the body is processed in the next expansion pass.

Expansion terminates when it reaches a fixed point where no derive or attribute remains to expand.

Circular expansion (A invokes B and B invokes A again) is detected by the compiler and is an ERROR. The message shows the cycle path (`A → B → A`).

## 6.8.9. No token stream manipulation

The three meta blocks above generate code using template tokens only. rok has no arbitrary token transformation facility like Rust's `syn` or `quote`.

This restriction guarantees that the output of a meta block is rok code the compiler can understand, securing both tool friendliness and debuggability.

## 6.8.10. Comparison with proc macros

Comparing Rust proc macros with rok meta blocks:

| Aspect          | Rust proc macro                | rok meta block                                  |
|-----------------|--------------------------------|-------------------------------------------------|
| Input           | Arbitrary token stream         | Structural information of the item (fields, variants, etc.) |
| Output          | Arbitrary token stream         | rok code and template tokens                    |
| Hygiene         | Up to the proc macro           | Automatic (hygienic)                            |
| Debugging       | Inspect output tokens          | Inspect the expanded AST                        |
| Expressiveness  | Very high                      | Limited                                         |

rok's choice trades expressiveness for predictability and ease of debugging.

## 6.8.11. Combination with registration/visibility

A meta block carrying `#[impl_derive(TraitName)]` registers a derive named `TraitName`, and `#[impl_attribute(name)]` registers the `#[name]` attribute. The compiler collects meta blocks from every file of libraries and user packages before parsing, so they can be used regardless of definition location or file order. If a `#[cfg]` on a meta block is false, that block is not registered ([6.9.8](609-template-tokens.md#relation-to-cfg)).

A `pub` meta block is visible throughout the build without `use` (6.8.6). Meta blocks are not items of the name space, so they are not targets of `use` either.

```rok
// pkg::derives
#[impl_derive(MyHash)] pub @derive {
    impl MyHash for @Self { ... }
}

// another module — used without import
#[derive(MyHash)]
struct Foo { ... }
```

The following are errors:

- Two meta blocks registering the same derive or attribute name.
- A meta block redefining the name of a built-in derive ([6.2.1](602-built-in-attributes.md): `Clone`, `Copy`, `Default`, `Debug`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`) or a built-in attribute (`cfg`, `derive`, `test`, `inline`, `repr`, `keyword`, and the other attributes of 6.2 to 6.7).

## 6.8.12. Summary

These three meta block forms make up rok's entire metaprogramming surface. They lack the expressiveness of proc macros but cover the core use cases: derives, attributes, and per-field attributes.

[Template tokens and expansion](609-template-tokens.md) defines the exact list of template tokens and the expansion mechanism.
