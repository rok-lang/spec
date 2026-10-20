# 2.13. Type paths

A sequence of identifiers joined by the `::` token ([Punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md)) is a path denoting an item. This document defines only paths in type position and associated item access; the exact meaning of module imports is covered separately in [`use` declarations](../005-modules/505-use.md).

```
type_path       ::= path_root ("::" path_segment)*
path_root       ::= identifier | "pkg" | "sup"
path_segment    ::= identifier type_args?
                  | identifier "::" "<" type_expr ">" "::" identifier  // assoc item
```

## 2.13.1. Module paths

If a type name is defined in another module, write the module path to that location together with the type name.

```rok
rok::std::time::Duration
rok::core::collections::List
pkg::geom::Point
```

Each identifier names a module or item. The last identifier is the type name; the preceding part is the path saying where that type can be found.

## 2.13.2. Starting point of a path

The first identifier of a path must be a name visible in the current module: the name of one of its own child modules, a name imported with `use`, or a context keyword. The context keywords are `pkg`, denoting the package root, and `sup`, denoting the parent module; their exact meaning is defined in [Paths](../005-modules/507-paths.md).

## 2.13.3. Associated item access

`::` is also used to access items attached to a type.

```rok
i32::MAX                  // const item of an integer type
Point::origin()           // static method
List::<i32>::new()        // generic given with turbofish
T::Output                 // associated type of a trait parameter
```

Associated item access has several forms. A static method is called as `Type::method(args)`. An associated `const` is referenced as `Type::CONSTANT`, and an associated type of a trait parameter as `T::AssocType`.

## 2.13.4. Fully qualified path

When the same method name exists in several traits and an inherent `impl` and is ambiguous, or when it must be stated which trait an associated type is referenced through, use the fully qualified form.

```rok
<i32 as Display>::display(42i32)
<T as Add<U>>::Output
<Self as Iterator>::next(ref self)
```

The fully qualified form is `<Type as Trait>::item`. `Type as Trait` is written inside angle brackets, followed by `::` and an item name or method call. Here `as` is a keyword use stating which trait the type is viewed through. The same `as` token has a different meaning in expression position (binary use), defined in the expression specification.

The fully qualified form is valid both in type position (`<T as Tr>::X` as an associated type) and in call position (`<T as Tr>::m(x)`). For a generic trait, the impl matching the arguments is selected. `<T as Tr<A>>::X` and `T::X` under the bound `T: Tr<A>` are the `X` of the `Tr<A>` impl, which may differ from the `X` of the same type's `Tr<B>` impl. A projection such as `<Sq as Shape>::Out` is valid in function signatures, `static` types, struct fields, enum payloads and type aliases (generic or not), regardless of the order in which the type, trait and impl are declared.

The fully qualified form rarely appears in ordinary use. It is used explicitly only when disambiguation is needed or automatic inference fails.

## 2.13.5. Turbofish: generics in expression position

To give generic arguments in value position, use the `::<...>` form.

```rok
parse::<i32>("42")
collect::<List<f64>>(iter)
```

That is, `::<...>` is appended to a function or method name to give the generic arguments at that position. In type position (after `:`, after `->`, after `as`, etc.), `<...>` is written directly without `::`.

The difference exists because in value position `<` would be ambiguous with the comparison operator.

## 2.13.6. Absolute type paths

An absolute path starting from the root is written with `pkg::` ([Paths](../005-modules/507-paths.md)). The leading-`::` form does not exist in rok.

## 2.13.7. Combining paths with sum types

A path itself is the simple-name or path form of a [type expression](201-type-expressions.md), and mixes freely with other combinators such as sum types (`|`) and generic application (`<...>`).

```rok
rok::std::io::Result<T>
pkg::Foo | rok::std::Bar
List<rok::core::Range<i32>>
```

Precedence applies as defined in [Type expressions](201-type-expressions.md): paths (`::`) and generic application (`<...>`) bind tighter than sum types (`|`).

## 2.13.8. Empty paths and a lone `::`

An empty path (only `::` with no identifier) is always a compile error. Likewise, consecutive `::` between identifiers (`A::::B`) is an error.
