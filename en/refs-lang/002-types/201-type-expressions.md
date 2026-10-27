# 2.1. Type expressions

A type expression is the notation that denotes a type in source code. Type expressions appear in many positions: after `:` in a variable declaration, after `:` in a function parameter, after `->` for a return type, as generic arguments, after `as` in a conversion, and so on.

This document defines only the form; which name denotes which type is decided by the standard library.

## 2.1.1. Basic form

The basic grammar of type expressions is:

```
type_expr   ::= sum_type
sum_type    ::= unary_type ("|" unary_type)*
unary_type  ::= base_type "?"*                  // T?: optional (2.1.5a), `??` is two `?`
base_type   ::= path_type
              | fn_type
              | tuple_type
              | array_type
              | any_type
              | self_type
              | qualified_type
              | macro_type
              | "none"                          // the none type (2.6)
              | "!"                             // never (4.3.7)
              | "(" type_expr ")"               // grouping
self_type   ::= "Self" ("::" identifier ("::" identifier)* type_args?)?   // Self, Self::Item
qualified_type ::= "<" type_expr "as" trait_ref ">" "::" identifier       // 2.13.4
macro_type  ::= identifier "!" ( "(" token* ")" | "[" token* "]" | "{" token* "}" )   // 6.8.12
any_type    ::= "any" ( trait_ref | "(" trait_ref ("+" trait_ref)* ")" )?
trait_ref   ::= identifier ("::" identifier)* trait_args?
trait_args  ::= "<" trait_arg ("," trait_arg)* ","? ">"
trait_arg   ::= type_expr | identifier "=" type_expr | const_arg
path_type   ::= identifier ("::" identifier)* type_args?
type_args   ::= "<" trait_arg ("," trait_arg)* ","? ">"
const_arg   ::= literal                         // int, char, true / false
              | "-" literal
              | "{" expr "}"                    // 2.11.4
              | string_lit                      // e.g. an MLIR op name
fn_type     ::= "unsafe"? ("extern" string_lit)? "fn" "( (type_expr ("," type_expr)*)? ")" ("throws" ("(" type_expr ")")?)? ("->" unary_type)? fn_bound*
fn_bound    ::= "+" ("Send" | "Sync")
tuple_type  ::= "(" ")"
              | "(" type_expr "," ")"
              | "(" type_expr ("," type_expr)+ ","? ")"
array_type  ::= "[" type_expr ";" const_expr "]"
```

`extern "C" fn(A, B) -> R` is a C function pointer type ([5.9.4](../005-modules/509-extern-block.md#c-function-pointer-types)). The ABI string may only be `"C"`; anything else is an error.

The forms are summarized below.

| Form              | Example                                     |
|-------------------|---------------------------------------------|
| Simple name       | `i32`, `bool`, `Point`                      |
| Path              | `rok::std::time::Duration`                  |
| Generic application | `List<i32>`, `Dict<str, f64>`, `Pair<A, B>` |
| Sum type (`\|`)   | `i32 \| f32`, `T \| none`                     |
| Optional (`?`)    | `i32?`, `(i32 \| str)?`, `List<T>?`           |
| Function type     | `fn(i32, i32) -> i32`, `fn() -> T + Send`   |
| Tuple type        | `(i32, str)`, `(i32,)`                      |
| Array type        | `[T; N]` (fixed size)                       |
| Dynamic slot      | `any`, `any Error`, `any (Display + Hash)`  |
| `none` / never    | `none`, `!`                                 |
| `Self` and its items | `Self`, `Self::Item`                     |
| Qualified path    | `<T as Iterator>::Item`                     |
| Macro call        | `name!(..)`, expands to a type ([6.8.12](../006-meta-system/608-meta-blocks.md#6812-the-macro-form)) |
| Const argument    | `Buf<16>`, `Buf<-1>`, `Buf<{ N + 1 }>`      |

The exact meaning of each form is defined in turn below.

An associated type projection of the form `T::Assoc` (`I::Item`, `F::Output`) is also a type. It is one of the path forms (`path_type`) and is valid when `T` is a type parameter constrained by a trait that has that associated type ([Type paths](213-type-paths.md)).

```rok
fn zip<I: Iterator, J: Iterator>(a: I, b: J) -> List<(I::Item, J::Item)> { ... }
```

## 2.1.2. Simple names

A simple name consists of a single identifier and follows the [identifier](../001-lexical/104-identifiers.md) form.

```rok
i32
bool
Point
변수타입
```

Which item the name refers to is decided during name resolution. Candidates include items defined in the current module and items imported with `use`.

## 2.1.3. Paths

Several identifiers joined by `::` can also be used as a type expression.

```rok
rok::std::time::Duration
rok::core::collections::List
pkg::geom::Point
```

The `::` token here is the path separator defined in [Punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md). The last token of the path is the type name; the preceding part is a module or item path saying where to find that type.

The meaning and resolution rules of paths are defined in [Type paths](213-type-paths.md); this document only states that a path may appear as a form of type expression.

## 2.1.4. Generic application

Type arguments can be given by appending `<...>` to a type name.

```rok
List<i32>
Dict<str, f64>
Pair<A, B>
Result<T, E>
Array<f64, 3>          // const generic (value argument)
```

The rules are: arguments are separated by commas. An argument may be another type expression, and for `const` generics a `const` expression is also allowed. An empty `<>` is not allowed; with no arguments, `<>` is not written at all. A trailing comma (`<A, B,>`) is allowed.

The exact meaning of value arguments (e.g. `Array<f64, 3>`) is defined in [`const` generics](211-const-generics.md).

### Turbofish in expression position

Generic arguments given inside an expression use the `::<...>` form, defined in the expression specification. This document covers only `<...>` in type expression position.

## 2.1.5. Sum types (`|`)

A sum type, whose value is one of several types, is written with `|`.

```rok
i32 | f32
T | none
A | B | C | D
```

Each term separated by `|` is a distinct type expression, and the resulting sum type holds a value of one of those terms. Its meaning and memory representation are defined in [Sum types](206-sum-types.md).

The `|` token binds more loosely than the other type combinators. So `i32 | List<f32>` groups as `i32 | (List<f32>)`, because generic application binds tighter than a sum type.

## 2.1.5a. Optional types (`T?`)

A postfix `?` makes an optional type. `T?` behaves exactly like `Maybe<T>` (`type Maybe<T> = T | none;`). That is, the operand of `?` is a generic term, and the [layering rule of sum types](206-sum-types.md#264-set-semantics) applies.

```rok
i32?                // i32 | none
(i32 | str)?        // i32 | str | none (no overlap, flattened)
(i32 | none)?       // two layers: outer none ≠ inner none
i32??               // two layers: same as (i32 | none)?
i32 | none          // written sum: always flat i32 | none
```

- Postfix `?` binds tighter than `|` and tighter than `->` of a function type (2.1.10). `i32 | str?` is `i32 | (str?)`.
- `fn() -> i32?` is a function returning an optional; `(fn() -> i32)?` is an optional function.
- In type position the `??` token is two `?` (`i32??`).
- In the expressions `x as T?` / `x is T?`, the trailing `?` belongs to the type. To apply the `?` operator ([3.14](../003-expressions/314-error-control-flow.md)) to a conversion result, parenthesize: `(x as T)?`. When the conversion in `x as T?` fails, the diagnostic hints "to propagate, write `(x as T)?`".
- However, `??` after the target of `as` / `is` is the null-coalescing operator (`x as i32 ?? 0` is `(x as i32) ?? 0`). To convert to a two-layer optional, write `x as (i32??)`.

## 2.1.6. Function types

The type denoting a function itself is written as:

```rok
fn(i32, i32) -> i32
fn() -> str
fn(T) -> T
fn(i32)                  // return type may be omitted (unit type)
```

A function type starts with the `fn` token; the parentheses contain only parameter types, not names. The return type follows `->`; omitting it means there is no return value (the exact meaning is defined in the function specification).

## 2.1.7. Tuple types

A tuple type grouping several types is written as:

```rok
(i32, str)
(f64, f64, f64)
(i32,)                  // a one-element tuple requires a trailing comma
()                      // empty tuple (unit type)
```

Rules: with two or more elements, they are separated by commas. A one-element tuple must have a trailing comma (`(T,)`); without it, it is parsed as a grouping of `T`. A tuple with no elements (`()`) denotes the unit type.

## 2.1.8. Array types

A fixed-size array type is written with brackets.

```rok
[T; N]
[i32; 16]
[f64; 1024]
[u8; SIZE]
```

An array type is written as element type, semicolon, length, inside brackets. The length is a `const` expression and must be determined at compile time. A length of 0 (`[T; 0]`, the empty array type) is allowed.

## 2.1.9. Dynamic slots

A dynamic type slot is written with the `any` keyword.

```rok
any
any Error
any (Display + Hash)
any Iterator<Item = i32>
any Future<Output = User>
```

`any` is a type-name slot keyword reserved in [Keywords](../001-lexical/105-keywords.md); its meaning is defined in [The `any` slot](202-any-slot.md). Following `any` with one trait path, or with parenthesized trait paths joined by `+`, makes a constrained dynamic slot that holds only values implementing those traits ([2.2.10](202-any-slot.md#2210-constrained-any-any-trait)). Inside the `<...>` of a trait path, besides type arguments, a binding that fixes an associated type such as `Item = T` may appear (`trait_arg`). Two or more trait paths require parentheses; `any` binds only the single trait immediately following it, so `any Error | none` is `(any Error) | none`. A trait path may appear in type position only after `any` ([Trait-as-type ban](212-trait-as-type-banned.md)).

## 2.1.10. Precedence / binding

When combinators are mixed in a type expression, their precedence (tight ↔ loose) is:

```
strong  ─→  weak

[T; N]   (...)   <...>   ::   .  →  T?  →  fn(...) -> ...   →   |
```

That is, paths (`::`), generic application (`<...>`), arrays (`[...]`) and grouping (`(...)`) bind tightest. Next is postfix optional (`?`), then function types (`fn(...) -> ...`), and sum types (`|`) bind loosest.

As a result, `i32 | List<f32>` groups as `i32 | (List<f32>)`, and `fn(A | B) -> C` as `fn((A|B)) -> C`.

Where explicit grouping is needed, use parentheses.

```rok
fn(i32) -> (i32 | str)         // return type is a sum type
(fn(i32) -> i32) | (fn(f64) -> f64)   // sum type with two function types as terms
```
