# 0.4. Boundary between the language and the standard libraries

This document declares the boundary between what the language defines and what the standard libraries define. This boundary is a core decision of the rok specification, and all subsequent specification documents are written on the premise of it.

## 0.4.1. What the language defines

The scope the language specification covers directly falls into the following five categories.

### 1. Logic flow

The first category is the syntax that moves values along and its meaning. It includes branches, loops, pattern matching, exception handling and labeled jumps; the syntax of async and generators, i.e. their declarator and operator keywords, also belongs to this category.

### 2. Container syntax

The second category is the forms for defining user types. A product type (record) groups several fields into one; a sum type (tagged) holds one of several variants. An untagged `union` is a device for interpreting the same memory as several types; a trait is a means of declaring a bundle of behavior; and an `impl` block is where concrete implementations of traits and methods go. Finally, generics enable forms that take types as parameters.

### 3. Meta-system

The third category is the attribute system, which attaches to the previous two categories and binds behavior. It consists of the standard attributes, a fixed set of names the language recognizes, and the template system for user-defined attributes and derive.

### 4. Variables / visibility / modules

The fourth category is the conventions for names and exposure. It includes the variable declaration keywords, reference modifiers, and global and constant keywords, together with visibility levels and the name-exposure conventions at module and package level.

### 5. Operator syntax

The last category is the shape of notations such as `a + b`, `s[1:3]`, `x.f` and `f(args)`.

The language does not, however, define what behavior these notations mean. A notation obtains its meaning by desugaring into trait methods registered in the standard libraries.

## 0.4.2. What the standard libraries define

The standard libraries are responsible for almost all meaning outside the five categories above. Specifically: numeric and character types and their operations; operator semantics such as arithmetic, comparison, logic, bitwise, indexing and ranges, all defined as trait `impl`s in the standard libraries; the conversion matrix containing all rules for conversion between types; and collections, typified by lists, maps, sets, tuples and strings. In addition, abstractions such as iterators, streams and futures; concurrency primitives typified by atomic, mutex and channel; task, thread and parallel execution APIs; I/O, time, randomness and networking; and all error types are defined by the standard libraries.

The language does not know the names of these library elements. The compiler recognizes a specific library type only through attribute bindings of the meta-system, and those bindings are themselves written inside the standard libraries.

## 0.4.3. Consequences of the boundary

### Syntax is the language, behavior is the library

The first consequence of this split is a division of labor between notation and meaning.

> [!IMPORTANT]
> The language defines every notation, but the meaning of an operation applied to values is
> determined by trait methods registered in the standard libraries.

| Notation    | What the language decides  | What the library decides                          |
|-------------|----------------------------|---------------------------------------------------|
| `a + b`     | Token sequence, precedence | Which trait method `+` is, result type            |
| `s[1:3]`    | Shape of slice notation    | Indexing, bounds checking, result type            |
| `x.f()`     | Shape of method call       | Method dispatch rules                             |
| `let x = 1` | Shape of declaration       | Type of the literal `1`                           |

### Scope of language-level questions

The second consequence of this split is a criterion for classifying questions. When facing any design or implementation decision, first determine whether it is a language-level question or a library-level question.

Questions that remain at the language level are limited to the following areas. First, compiler semantics: name resolution, closure capture, visibility and error diagnostics. Second, syntax decisions: escapes, precedence, literal suffixes, the shape of slice notation, etc. Third, run-time calling conventions and the memory model, including object layout and calling convention. Fourth, resource management: lifetimes and the exception propagation mechanism. Fifth, type system rules: the trait-as-type ban, gradual typing, niche optimization, etc.

### What is delegated to the library level

Operations on almost all values outside the list above are the library's decision. This specification therefore does not prescribe the semantics of such operations directly.

Instead, the specification states the following: which trait method call the operation desugars into; the location convention for which library tier defines that trait; and the form of the compile error that occurs when it is not defined.

This split is how rok's design slogan, "a small language, rich libraries", is realized at the specification level.

## 0.4.4. Implementation limits

The language sets no fixed limits on nesting, chain length, value size or compile-time evaluation. An implementation may set its own; they are implementation-defined, not language requirements. rok's are:

| Limit | rok |
|-------|-----|
| Parser nesting depth | 256 |
| Operator / postfix chain links ([3.2.6a](../003-expressions/302-precedence-and-associativity.md#326a-implementation-limits)) | 4096 |
| Value type size ([2.6.3](../002-types/206-sum-types.md)) | 65536 flattened scalar slots or 4 GiB |
| `const` evaluation ([5.2.4](../005-modules/502-const.md)) | 4,000,000 steps, call depth 128 |

Exceeding one is a compile error.
