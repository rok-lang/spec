# 5.2. `const`

The `const` keyword (a contextual keyword that is a keyword only at the start of an item, [1.5.4](../001-lexical/105-keywords.md#154-contextual-keywords)) declares a value determined at compile time. The grammar is:

```
const_decl      ::= "const" identifier ":" type_expr "=" const_expr ";"
const_fn        ::= "const" fn_decl
```

## 5.2.1. Basic form

```rok
const PI: f64 = 3.14159f64;
const MAX_RETRY: i32 = 3i32;
const NAME: str = "rok";
```

A declaration consists of the `const` keyword, a name, `:`, a type, `=`, and a `const` expression, in that order. The type must be written explicitly; unlike `let`, it is not inferred. The value position accepts only a `const` expression, that is, an expression that can be evaluated at compile time.

## 5.2.2. Permitted positions

`const` may be declared in all of the following positions.

```rok
// module level
const PI: f64 = 3.14159f64;

// inside a function body
fn area(r: f64) -> f64 {
    const FACTOR: f64 = 3.14159f64;
    FACTOR * r * r
}

// inside an impl block
impl Circle {
    const UNIT_AREA: f64 = 3.14159f64;
    fn unit() -> Circle { Circle { r: 1.0f64 } }
}

// inside a trait - associated const
trait Numbers {
    const ZERO: Self;
    const ONE: Self;
}
```

A `const` in a function body is an item of its block ([3.9.1](../003-expressions/309-block-expressions.md#391-form)): visible in the whole block and the items declared in it, it reads no local variable, parameter or generic parameter of the code around it.

## 5.2.3. Meaning of `const`: inlined at each use

A `const` is a value, not an address. Its value is inlined at every place it is used.

```rok
const X: i32 = 42i32;

fn f() {
    let a = X;                    // a = 42i32 (X is inlined)
    let b = X + 1i32;              // b = 43i32 (X is inlined, then evaluated)
}
```

The result of a `const` does not reside at any memory location, so its address cannot be taken.

```rok
let p = &X;                       // ERROR - a const has no address (conceptually)
```

Whether Rok actually has a direct address operator such as `&` is defined by the pointer types in [type expressions](../002-types/201-type-expressions.md) and by the [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) specification.

## 5.2.4. Constant expressions and the evaluator

`const` items, array lengths, const generic arguments, `struct` field defaults, `static` initializers, and `const fn` literal handlers receiving constant arguments ([6.5.8](../006-meta-system/605-literal-magic.md#658-const-compatibility-of-literal-handlers)) all use the same compile-time evaluator.

What can be evaluated:

- Values: integers of every width, floating-point, `bool`, `char`, `str`, tuples, `struct`, `enum`, sum types, fixed-size arrays.
- Expressions: literals, references to other `const`s, const generic parameters, operators, `const fn` calls.
- Control flow: `if`, `match`, `loop` / `while` / `for`, labeled `break`, `return`, `throw`. Recursion and generic `const fn` are allowed.

What cannot be evaluated: calls to functions that are not `const fn` (I/O, time, random numbers, etc.), reading a `static`, references to outer `let` variables.

The following are compile errors:

- Integer overflow (including the result of folding an expression made only of literals and constants, `let m: u8 = 2 * 200`), division by zero, `MIN / -1`, shifts by the bit width or more, and out-of-range floating-point → integer conversions. Wrapping arithmetic is only via `wrapping_*`.
- A panic or an uncaught `throw` during evaluation.
- Exceeding the limit on evaluation steps or call depth. The limits are implementation-defined (rok: 4,000,000 steps, depth 128).
- Circular references between `const`s.

## 5.2.5. `const fn`: functions evaluable at compile time

```rok
const fn double(x: i32) -> i32 {
    x * 2i32
}

const fn gcd(a: i64, b: i64) -> i64 {
    if b == 0i64 { a } else { gcd(b, a % b) }
}

const TWO: i32 = double(1i32);                  // compile time
const TWELVE: i64 = gcd(24i64, 36i64);          // compile time

let y = double(user_input);                      // runtime call
```

- A `const fn` may call only `const fn`s. Calls through `fn` values or closures inside a `const fn` are checked only when evaluated, since a static check cannot follow them. The standard library marks pure functions as `const fn`.
- The body must be pure. I/O and access to mutable globals are errors.
- A result successfully evaluated at compile time equals the runtime result for the same arguments. An operation that would overflow at runtime is an error at compile time (5.2.4).
- A `const fn` cannot take a type as an argument (`const fn sizeof(T: type)` is an error). Type introspection belongs to meta blocks.
- A `const fn` can also be called at runtime like an ordinary function.

### Generic `const fn`

```rok
const fn id<T>(x: T) -> T { x }

const A: i32 = id(42i32);                        // evaluated after monomorphization with T = i32
```

### Results that use the heap

If the result contains a heap allocation (`List`, buffers, closures, etc.), it is not embedded in the binary. Such a `const` is evaluated and checked at compile time and constructed at runtime at each use.

## 5.2.6. Orthogonality of `const` and other mechanisms

```
generic <T>     = type parameterization (call-site specialize)
meta block      = code generation (impl, struct, etc. at AST level)
const fn        = value computation (compile-time pure function)
```

The three mechanisms are orthogonal. Their domains do not overlap, so they may be combined freely.

## 5.2.7. When `const` is evaluated

Module-level, function-body and `impl`-level `const`s are evaluated once at package build. A `const` that depends on generic parameters (an associated `const` of a generic `impl`) is evaluated per monomorphization. Results that do not use the heap are embedded in the binary and have no runtime cost.

## 5.2.8. Relation to `struct` field defaults

[`struct`](../002-types/203-struct.md) field defaults are constant expressions (5.2.4).

## 5.2.9. Relation to `const` generics

Arguments of [const generics](../002-types/211-const-generics.md) are constants that become part of the type, and are evaluated by the same evaluator (5.2.4).
