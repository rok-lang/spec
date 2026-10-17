# 4.2. Parameters

This section defines the form of function parameters and the rules for matching arguments at the call site.

```
param           ::= self_param | regular_param
self_param      ::= "ref"? "self" (":" type_expr)?
regular_param   ::= "ref"? identifier (":" type_expr)? ("=" const_expr)?
```

## 4.2.1. Basic form

```rok
fn f(x: i32, y: i32) -> i32 { x + y }
```

Each parameter is written as ` name: type`, separated by commas. Parameter names follow the form of [identifiers](../001-lexical/104-identifiers.md), and types follow the rules of [type expressions](../002-types/201-type-expressions.md).

## 4.2.2. Omitting the type

A parameter without a type is treated as generic. See the monomorphization rules of [generics](../002-types/210-generics.md) for details.

```rok
fn id(x) { x }                                   // x is generic — monomorphized at call
fn pair(x, y) { ... }                             // both parameters generic
fn mixed(a: i32, b) { ... }                       // a is i32, b is generic
```

`id` above is equivalent to the form with explicit [generics](../002-types/210-generics.md):

```rok
fn id<T>(x: T) -> T { x }
```

Traits used in the body are not automatically synthesized into a `where` clause in the signature. Each call site is monomorphized, and the body's trait uses are checked there.

```rok
fn compute(x, y) { x + y }                       // signature: <T, U>(x: T, y: U)
                                                 //   no where clause synthesized

compute(1i32, 2i32);                             // OK — i32 + i32 dispatch
compute(1.0f64, 2.0f64);                         // OK — separate monomorphized instance
compute(MyType, 1i32);                           // error at the call site:
                                                 //   "MyType has no Add<i32>"
                                                 //   (reported clearly where it is called,
                                                 //    not at the signature)
```

There are two reasons for this design. First, it preserves module boundaries: changing one line of the body to use more traits does not change the signature, so external callers do not break. Second, it avoids bloated signatures: an automatically synthesized `where` clause accumulating every trait in the body would bloat the signature, and the types inferred from it would surface in tooling in huge forms.

If a parameter is meant to be a dynamic slot, write `any` explicitly (see [`any` slot](../002-types/202-any-slot.md)).

```rok
fn handle(x: any) { ... }                         // x is an any slot
```

## 4.2.3. Default values

Parameters may have default values.

```rok
fn retry(n: i32 = 3i32, delay: f64 = 0.1f64) { ... }

retry();                          // n=3, delay=0.1
retry(5i32);                       // n=5, delay=0.1
retry(5i32, 1.0f64);               // n=5, delay=1.0
```

The following rules apply to default values. Only `const` expressions are allowed: literals, `const` variables and `const` function calls. This is the same constraint as for field defaults of [`struct`](../002-types/203-struct.md). Only parameters omitted at the call site are evaluated, and they are evaluated anew on each call. Default values are deterministic: the same `const` expression always produces the same value. Runtime calls such as the current time or random numbers cannot be used; when needed, use a factory function pattern instead. Names in a default are resolved in the scope of the module that declares the function, not the caller's.

## 4.2.4. Named argument calls

At the call site, arguments may be given by parameter name.

```rok
fn make_user(name: str, age: i32 = 0i32, email: str = "") -> User { ... }

make_user(name: "Alice", age: 30i32);
make_user(name: "Bob", email: "bob@example.com");      // age uses its default
make_user("Charlie", age: 25i32);                       // only the first argument positional, the rest named
```

The following rules apply to named argument calls. Positional arguments must come first and named arguments after them. Giving the same parameter both positionally and by name in one call is an error. After all positional arguments are matched, the remaining parameters must be filled by named arguments or default values.

## 4.2.5. Variadic: `...T`

A parameter that accepts a variable number of arguments.

```rok
fn sum(xs: ...i32) -> i32 {
    let total = 0i32;
    for x in xs { total += x; }
    total
}

sum();                            // xs: empty slice
sum(1i32);                         // xs = [1]
sum(1i32, 2i32, 3i32);             // xs = [1, 2, 3]
let ys: List<i32> = [4, 5];
sum(..ys);                         // xs = [4, 5] — list spread
```

The following rules apply to variadic parameters:

- Only the last parameter may be variadic.
- In the body, `xs: ...T` is a read-only slice view of element type `T` (`Slice<T>`, [stdlib 2.9.7](../../refs-standard-libraries/200-core/209-collections.md#297-slicet--view)). The remaining arguments of the call are bundled into one, and the length may differ per call. The function is therefore not monomorphized separately per argument count.
- Individual arguments, tuple spreads and list spreads (`..xs`) may be mixed (4.2.6).
- The body may read, index and iterate it. If modification is needed, build a new collection with library methods.

### Heterogeneous variadic: `...any`

```rok
fn log_all(xs: ...any) { ... }
log_all(1i32, "hi", 3.14f64);     // various types allowed
```

With parameter type `any`, the variadic accepts heterogeneous arguments.

## 4.2.6. Spread: `..args`

A form that expands an existing collection into arguments at the call site.

```rok
let args = (1i32, 2i32, 3i32);
foo(..args);                      // foo(1, 2, 3)

let xs: List<i32> = [1i32, 2i32, 3i32];
sum(..xs);                         // sum(1, 2, 3) — received as variadic
```

The following rules apply to spread:

- A tuple spread has its length and types checked at compile time.
- A list spread uses the runtime length; it is OK for a variadic or when the positions match.
- A Dict spread is converted to named arguments.
- In a variadic position, tuple, `List`, `Slice` and array spreads may be mixed with ordinary arguments (`f(a, ..xs, b)`). A variadic parameter may be passed as-is to another variadic (`fn f(xs: ...i32) { g(..xs) }`).
- Spreading a non-collection value is an error ("needs a tuple, a List, a Slice or an array").
- Calling a variadic whose element type `T` is an open generic with no arguments cannot determine `T`, so a type annotation is required.

```rok
let opts = {"lr": 0.01, "epochs": 100};
train(..opts);                  // train(lr: 0.01, epochs: 100)
```

Spreads and ordinary arguments may also be freely mixed within one call.

```rok
foo(0, ..middle, 10);
foo(..a, ..b, ..c);
```

## 4.2.7. Discard: `_`

The notation for explicitly discarding a call result.

```rok
_ = compute_with_side_effect();
```

Here `_` is the single underscore of [identifiers](../001-lexical/104-identifiers.md). In left-hand position it receives a value but binds it nowhere, signaling that the user intentionally ignores the result. `let _ = e;` is the same and may be used any number of times in the same scope (`_` is not a binding, so it is not a redeclaration).

## 4.2.8. Parameter evaluation order

On a call, argument expressions are evaluated in written order (left to right). A method call evaluates the receiver first.

```rok
foo(a(), b(), c());               // a() first, then b(), c() last
```

Named arguments are also evaluated in written order.

```rok
make_user(name: f(), age: g());   // f() first, then g()
```

This order guarantees that arguments with side effects run in a predictable order.

## 4.2.9. Parameter scope

Parameters are introduced into the top-level scope of the function body and are used like ordinary variables inside the body.

Default value expressions are evaluated in an environment without the parameters. They therefore cannot refer to other parameters of the same function.

```rok
fn f(a: i32, b: i32 = a + 1) { ... }   // ERROR — default cannot refer to a
```

This constraint has the same motivation as the field default constraint of [`struct`](../002-types/203-struct.md): determinism and predictable cost.

## 4.2.10. Static vs gradual parameters

| Written form | Meaning                                              |
|--------------|------------------------------------------------------|
| `x: i32`     | Static. Accepts only i32 at the call site            |
| `x`          | Generic. Monomorphized by the call-site type         |
| `x: any`     | Dynamic slot. Accepts any type with a runtime tag    |

These three forms may be freely mixed within one function.
