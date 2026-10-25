# 6.8. User-defined meta definitions

A **meta definition** is a user-defined code generator. All of them are one mechanism: a definition binds parts of an input and transcribes a template of rok code and template tokens ([6.9](609-template-tokens.md)), hygienically ([6.8.9](#689-no-token-stream-manipulation)) and to a fixed point ([6.8.8](#688-fixed-point-expansion)). They differ only in how they are *triggered* and what input they bind. There are four forms; the first three are triggered by an attribute on an item, the fourth by a call:

| Form                                    | Defined as                         | Triggered by           | Binds                                   |
|-----------------------------------------|------------------------------------|------------------------|-----------------------------------------|
| derive                                  | `#[impl_derive(T)] @derive {}`     | `#[derive(T)]` on a type | the type's shape (`@Self`, `@for_fields`) |
| attribute                               | `#[impl_attribute(n)] @attribute {}` | `#[n]` on an item    | the item (`@item`)                      |
| per-field derive attribute              | `#[impl_derive_attribute(T, a)] @derive {}` | `#[derive(T)]` + `#[a]` on fields | shape + field attributes      |
| **macro**                               | `#[impl_macro(n)] @macro { (m) => {t} }` | `n!(..)` call      | the call's tokens (`@x: expr`, `@rep`)  |

The first three generate code *from a type or item the author attaches them to*; the macro generates code *from what the caller writes*, the one need the others cannot serve ([6.8.12](#6812-the-macro-form)). They share the template engine, the `@` tokens, hygiene and the tooling guarantees below; this chapter and [6.9](609-template-tokens.md) note where the bound input differs. These four are rok's entire metaprogramming surface - together they take the place of Rust's derive, attribute and function-like macros, without procedural code or raw token manipulation.

```
meta_block      ::= meta_attr visibility? meta_kind block
meta_attr       ::= "#" "[" meta_kind_attr "(" meta_args ")" "]"
meta_kind_attr  ::= "impl_derive" | "impl_attribute" | "impl_derive_attribute" | "impl_macro"
meta_kind       ::= "@derive" | "@attribute" | "@macro"
meta_args       ::= identifier ("," identifier)*

# the @macro block holds rules instead of a flat template:
macro_body      ::= macro_rule+
macro_rule      ::= "(" matcher ")" "=>" block
```

## 6.8.1. The attribute-triggered forms

The three attribute-triggered forms:

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

The fourth, the call-triggered `macro`, is defined in [6.8.12](#6812-the-macro-form).

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

This form takes two arguments - the Trait being derived and the attribute that marks per-field information - and branches inside the derive body by reading the fields' attribute information.

## 6.8.5. Body (template block)

In every form the body is a template block containing both ordinary rok code and template tokens (`@Self`, `@field`, `@for_fields` for the attribute-triggered forms; `@x`, `@rep`, `@for`, `@concat` for a macro, [6.8.12](#6812-the-macro-form)).

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
| `@macro`     | macro definitions (`impl_macro`); its block holds rules ([6.8.12](#6812-the-macro-form)) |

The introducer determines which template tokens are meaningful in the body. For example, `@item` is attribute-only, and the matcher tokens (`@x: expr`, `@rep`) are `@macro`-only.

## 6.8.8. Fixed-point expansion

If the result of template expansion produces an item carrying another derive or attribute, that item is expanded recursively.

```rok
#[impl_attribute(adds_helper)] pub @attribute {
    @item                                       // the original item as is

    #[derive(MyHelper)]                          // new item - derive processed again
    struct Helper { ... }
}
```

When this attribute is applied, the `#[derive(MyHelper)]` in the body is processed in the next expansion pass.

Expansion terminates when it reaches a fixed point where no derive or attribute remains to expand.

Circular expansion (A invokes B and B invokes A again) is detected by the compiler and is an ERROR. The message shows the cycle path (`A → B → A`).

## 6.8.9. No token stream manipulation

Every meta definition - the three attribute-triggered forms and the macro - generates code using template tokens only. rok has no arbitrary token transformation facility like Rust's `syn` or `quote`, and no procedural form that runs code at compile time.

This restriction guarantees that the output of a meta definition is rok code the compiler can understand, securing both tool friendliness and debuggability.

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

A meta block carrying `#[impl_derive(TraitName)]` registers a derive named `TraitName`, `#[impl_attribute(name)]` registers the `#[name]` attribute, and `#[impl_macro(name)]` registers the macro `name!`. The compiler collects meta blocks from every file of libraries and user packages before parsing, so they can be used regardless of definition location or file order. If a `#[cfg]` on a meta block is false, that block is not registered ([6.9.9](609-template-tokens.md#relation-to-cfg)).

A `pub` meta block is visible throughout the build without `use` (6.8.6). Meta blocks are not items of the name space, so they are not targets of `use` either.

```rok
// pkg::derives
#[impl_derive(MyHash)] pub @derive {
    impl MyHash for @Self { ... }
}

// another module - used without import
#[derive(MyHash)]
struct Foo { ... }
```

The following are errors:

- Two meta blocks registering the same derive, attribute or macro name (one macro is one `@macro` block holding all its rules).
- A meta block redefining the name of a built-in derive ([6.2.1](602-built-in-attributes.md): `Clone`, `Copy`, `Default`, `Debug`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`) or a built-in attribute (`cfg`, `derive`, `test`, `inline`, `repr`, `keyword`, and the other attributes of 6.2 to 6.7).

## 6.8.12. The macro form

A **macro** is the call-triggered meta definition. Where a derive or attribute generates code from a type or item it is attached to, a macro is *invoked* like `name!(..)` and expands into code at that position - generating code from what the caller writes. It is the one need the attribute-triggered forms cannot serve: repeating or generating code driven by the call, not by a type's fields.

### Definition

Like the other forms, a macro is `#[impl_macro(name)]`, a visibility, the `@macro` introducer, and a block. The block holds one or more **rules** instead of a flat template: each is a matcher in parentheses, `=>` (the same arrow as a [`match`](../003-expressions/312-match-expression.md) arm), and a template in braces.

```rok
#[impl_macro(name)] pub @macro {
    ( <matcher> ) => { <template> }
    ( <matcher> ) => { <template> }
    ..
}
```

`#[impl_macro(name)]` registers the macro `name!`, exactly as `#[impl_derive(T)]` registers the derive `T`. On a call the rules are tried top to bottom and the first whose matcher matches is expanded; if none matches it is a compile error at the call. A macro's name is build-global and used without `use`, like a derive or attribute ([6.8.6](#686-pub--visibility)); the trailing `!` at every call marks it as a macro, not a function.

```rok
#[impl_macro(twice)] pub @macro {
    ( @e: expr ) => { { let v = @e; (v, v) } }
}

let p = twice!(compute());      // (the value, the same value)
```

### Invocation

A macro is called as `name!( .. )`, `name![ .. ]` or `name!{ .. }`; the delimiters are interchangeable and do not change meaning. The arguments are an arbitrary **token tree**: any tokens with balanced `()` `[]` `{}`, so the lexer and formatter tokenize a call even without knowing the macro. A macro may expand at these positions, in whichever its template produces:

| Position   | Example                                   | Template yields         |
|------------|-------------------------------------------|-------------------------|
| expression | `let xs = grid![1, 2; 3, 4];`             | an expression           |
| statement  | `debug!("here");`                         | zero or more statements |
| item       | `int_impl!(i8, i16, i32);` (module level) | zero or more items      |
| type       | `let f: fn_ptr!(i32, i32) = ..;`          | a type                  |
| pattern    | `match x { pair!(a, b) => .. }`           | a pattern               |

### Matchers and fragments

A matcher is a token pattern. Literal tokens must appear verbatim in the call; a **metavariable** `@name: <fragment>` captures a part of the call for the template. The fragment fixes how much is parsed and as what: `expr`, `stmt`, `block`, `ty`, `pat`, `path`, `ident`, `literal`, `item`, or `tt` (one raw token tree). A fragment is parsed as that grammar into a real syntax node, so the argument of `@e: expr` has a type, is checked, and the language server resolves names inside it ([6.8.12](#tooling-and-hygiene)); `tt` alone is unparsed, for a DSL that is not rok grammar, and even then stays balanced.

```rok
#[impl_macro(clamp)] pub @macro {
    ( @x: expr , @lo: expr , @hi: expr ) => {
        { let v = @x; if v < @lo { @lo } else if v > @hi { @hi } else { v } }
    }
}
```

### Repetition

`@rep` matches a sub-pattern zero or more times; `@for` - the same iterator as `@for_fields` ([6.9.3](609-template-tokens.md#693-field-iteration)), here over a captured sequence - transcribes once per element.

```rok
#[impl_macro(list)] pub @macro {
    ( @rep[sep = ","] { @x: expr } ) => {
        { let acc = List::new(); @for @x { acc.push(@x); } acc }
    }
}

let xs = list![a, b, c];        // acc.push(a); acc.push(b); acc.push(c);
```

`@rep[sep = ","] { <sub-pattern> }` matches the sub-pattern repeated with the `sep` token between (`sep` optional; `trailing` also allows one trailing separator). A metavariable captured inside `@rep` binds a sequence. `@for @x { <body> }` emits the body per element; several metavariables from one `@rep` iterate in lockstep and must have equal length. `@rep` and `@for` nest. The shape tokens (`@Self`, `@field`, `@apply_if`, ..) belong to the type-driven forms, not a macro: a macro has no shape to test, so using one is an error.

### Name synthesis

`@concat(..)` joins name tokens and string literals into one identifier, so a macro can generate distinct, named items:

```rok
#[impl_macro(accessors)] pub @macro {
    ( @rep[sep = ","] { @f: ident : @t: ty } ) => {
        @for @f { fn @concat(get_, @f)(self) -> @t { self.@f } }
    }
}
// accessors!{ x: i32, y: i32 } -> fn get_x(self) -> i32 { self.x }  (and get_y)
```

A name made by `@concat` is **recorded**: the compiler indexes each synthesized item under its generated name, so symbol search, go-to-definition and find-references reach it. `@concat` forms an identifier at a definition or path position only; its pieces are name tokens and string literals, and it cannot assemble arbitrary syntax.

### The library's boilerplate

The classic reason for a macro is repeating a definition across a fixed list the author writes - which no attribute-triggered form can do. It replaces the hand-copied per-type impls of [`rok::core::num`](../../refs-standard-libraries/200-core/README.md):

```rok
#[impl_macro(nonzero_div)] pub @macro {
    ( @rep[sep = ","] { @t: ty } ) => {
        @for @t {
            impl Div<NonZero<@t>> for @t {
                type Output = @t;
                const fn div(self, rhs: NonZero<@t>) -> @t { self / rhs.0 }
            }
        }
    }
}

nonzero_div!(i8, i16, i32, i64, u8, u16, u32, u64);
```

### Recursion

A template may call macros, including itself; expansion repeats until no macro call remains. It is bounded: a depth over the limit (256) is a compile error naming the call, so a non-terminating macro fails rather than hangs. Mutual recursion counts against the same depth and, as for the attribute forms, a cycle is reported with its path ([6.8.8](#688-fixed-point-expansion)).

### Built-in `!`-forms are reserved

`cfg!` ([6.3](603-cfg.md)) and `include_str!` / `include_bytes!` ([6.11](611-include.md)) end in `!` but are fixed compiler built-ins, not macros, and cannot be redefined, shadowed or taken as a macro name. They are the only `!`-forms the compiler provides; every other `name!` is a macro defined in rok.

### Tooling and hygiene

A macro keeps the guarantees of every meta definition ([6.8.9](#689-no-token-stream-manipulation)), with the specifics a call site adds:

- **Hygiene** is automatic: a name the template introduces is fresh and cannot capture or be captured by the call site, while a name passed in through a metavariable resolves at the call site.
- **Spans**: every expanded token carries the span of the call, so a diagnostic points at the call with the expansion shown as a note.
- **Resolution**: because fragments are parsed as real nodes and synthesized names are indexed, hover, go-to-definition and find-references work through a macro - into its arguments and onto the items it generates.
- **Expansion on request**: the language server can expand a call in place ([9.9.13](../009-tools/909-debug-tools.md)).
- **Formatting**: the formatter formats a call's arguments when they are rok fragments, and leaves a raw `tt` DSL as written within its balanced delimiters.

## 6.8.13. Summary

These four forms - derive, attribute, per-field derive attribute, and the call-triggered macro - make up rok's entire metaprogramming surface, over one template engine. They lack the expressiveness of procedural macros but cover the core use cases: derives, attributes, per-field attributes, and call-site generation, each hygienic and followable by the tools.

[Template tokens and expansion](609-template-tokens.md) defines the exact list of template tokens - for both the type-driven and the call-driven input - and the expansion mechanism.
