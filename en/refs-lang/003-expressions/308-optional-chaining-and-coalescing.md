# 3.8. Optional chaining and null coalescing

To express chaining and default handling concisely for `T | none` values of [sum types](../002-types/206-sum-types.md), two notations are defined.

```
optional_chain  ::= primary "?." identifier                       // optional field
                  | primary "?." identifier "(" arg_list? ")"     // optional method
coalesce_expr   ::= expr "??" expr
```

## 3.8.1. Optional chaining: `?.`

```rok
x?.field
x?.method()
```

The `?.` token is defined in [punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md). When the left-hand side `x` has type `T | none` and a value is present, the dot access or method call is performed; when there is no value (`none`), the whole result is `none`.

### Desugaring

```
x?.field            →   match x { v: T => v.field, none => none }
x?.method(args)     →   match x { v: T => v.method(args), none => none }
```

Here `T` denotes the variants of the left-hand sum type other than `none`. A type-test pattern (`v: T`) takes and narrows the non-none branch of the sum type, then the dot access or method call proceeds.

### Non-Optional receivers

If the left-hand type does not include `none`, `?.` is a compile error. Use `.` instead.

```rok
let p: Point = ...;
p?.x;          // ERROR - p is not optional; use .
```

## 3.8.2. Chaining

`?.` can be chained.

```rok
let name = user?.profile?.name;
```

If any step yields `none`, the whole expression finishes as `none` at that point. If `user` is `none` the whole result is `none`; if `user.profile` is `none` the whole result is also `none`; only when every value is present is the value of `name` produced.

## 3.8.3. Result type

The result type of `?.` propagates to the end of the chain. Even if the final expression is `T`, the result type of the chain is always `T | none`.

```rok
let user: User | none = ...;
let name = user?.profile?.name;
// name: str | none
```

Note that the flat type of the last step (`str`) is not automatically promoted to a sum type (`str | none`); the chain itself produces the result type `T | none`.

## 3.8.4. Null coalescing: `??`

```rok
x ?? y
```

When the left-hand side `x` is `T | none`, `??` returns its value if present and `y` otherwise.

### Desugaring

```
x ?? y          →   match x { v: T => v, none => y }
```

### Result type

The result type of `??` follows the branch result rules of the [expressions overview](README.md#branch-result-types). If the two branches (`T` and `typeof(y)`) have the same type, that is the result type. If they differ, the receiving side must have a type; there is no automatic widening to a sum.

```rok
let a = foo() ?? 0;                           // i32 (foo(): i32 | none)
let r = foo() ?? "default";                   // ERROR - i32 and str, no receiving type
let r2: i32 | str = foo() ?? "default";       // OK
```

If `y` is also `T | none`, the result is `T | none` (the union of `T | (T | none)`). So when `??` is chained, `none` drops out only once the last alternative is not `none`.

```rok
let a: i32 | none = ...;
let b: i32 | none = ...;
let r = a ?? b;              // r: i32 | none
let s = a ?? b ?? 0i32;      // s: i32
```

## 3.8.5. Precedence

In the levels of [precedence and associativity](302-precedence-and-associativity.md), `?.` has the same precedence as member access (`.`) and binds very tightly. `??` (prec=1) sits below logical OR (`||`) and above the range operators (`..`, prec=0), and is right-associative.

```
a ?? b ?? c     →   a ?? (b ?? c)              // right-associative
foo() ?? bar    →   (foo()) ?? (bar)           // a call binds tighter than ??
a + b ?? c      →   (a + b) ?? c               // + binds tighter than ??
a ?? 0..3       →   (a ?? 0)..3                // ?? binds tighter than ..
```

## 3.8.6. Combining chaining and coalescing

```rok
let name = user?.profile?.name ?? "anonymous";
```

Here `user?.profile?.name` yields `str | none`, and the following `?? "anonymous"` fills the `none` case, so the final result is `str`. This is the standard form for safely extracting a possibly absent value while applying a default.

## 3.8.7. Distinguishing `?.` and `?`

`?.` is optional chaining; postfix `?` on a `T | none` value is defined in [3.14.10](314-error-control-flow.md).

That `?.` and `?` `.` are distinct tokens is guaranteed by the longest-match rule of [punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md).

## 3.8.8. Short-Circuit evaluation of optional chaining

```
x?.foo().bar()
```

In this expression, if `x` is `none`, neither `foo()` nor `bar()` is called. Short-circuit evaluation is guaranteed.

This is because the desugared `match` returns `none` immediately in the `none` branch.

Consequently, when a method with side effects is placed in a chain, it is not called when the receiver is `none`.

The chain is the run of postfix operations after `?.`: field access, method calls and indexing, including `?.[i]`. Parentheses end it: in `(x?.a).b`, `.b` applies to the result of `x?.a`, which may be `none`.

## 3.8.9. Alternatives to deep dot access

When chaining becomes deep and complex, combining it with `??` or using explicit matching ([`match` expression](312-match-expression.md)) may be clearer.

```rok
// chaining
let x = a?.b?.c?.d ?? default;

// explicit (for a: A | none, b: B | none, c: C | none)
let x = match a {
    av: A => match av.b {
        bv: B => match bv.c {
            cv: C => cv.d,
            none => default,
        },
        none => default,
    },
    none => default,
};
```

The two forms are semantically identical; choose by readability.
