# 4.7. The `ref` Keyword

`ref` is a keyword marking that "changes in this position are reflected in the caller's (or outer) variable". For parameters and call arguments it is implemented as copy-in / copy-out; in capture lists, as a shared cell. It is not a first-class reference value.

## 4.7.1. Three positions

```
1. function parameter:      fn f(ref x: i32) { x = ... }
2. closure capture list:    [ref x] before |...| { ... }
3. at the call:             f(ref x)
```

In all three positions it has the same meaning: changes are reflected in the outer variable.

## 4.7.2. Function parameters

```rok
fn increment(ref n: i32) {
    n = n + 1;
}

fn swap(ref a: i32, ref b: i32) {
    let t = a;
    a = b;
    b = t;
}
```

With `ref` before a parameter name, reassigning that parameter in the body is reflected in the caller's variable. The parameter's type itself (`i32` above) is the same as usual.

### Meaning - copy-in / copy-out

A `ref` parameter is copy-in / copy-out. At the call, the value at the caller's place (variable, field, element) is copied into the parameter; the callee works with that value; when the call ends, the parameter's final value is written back to the caller's place.

- **Write-back timing**: write-back happens not only on normal return but also when the callee throws. Changes made before a `throw` are therefore visible to the caller.
- **Exclusivity**: while the call is in progress, the same place must not be accessed through another path, and the compiler checks this. Passing the same place as `ref` twice, or passing it as `ref` in one argument and reading it in another, is a compile error. Distinct fields (`ref p.x`, `ref p.y`) do not overlap and are allowed.

```rok
fn bump_then_fail(ref n: i32) throws {
    n = n + 1;
    throw MyError {};
}
let a = 1i32;
try bump_then_fail(ref a) catch _ { };   // a = 2 - written back on throw too
swap(ref a, ref a);                       // ERROR - same place passed as ref twice
add(ref a, a);                            // ERROR - place passed as ref read by another argument
swap(ref p.x, ref p.y);                   // OK - distinct fields
```

> [!IMPORTANT]
> The caller must write `ref` at the call site; calling without it is a compile error. This makes visible which variables can change.

## 4.7.3. Explicit at the call (C# style)

```rok
let x = 5i32;
let y = 10i32;
increment(ref x);                 // x = 6
swap(ref x, ref y);
```

Where, and only where, a function parameter is declared `ref`, the caller must also write `ref`. Omitting `ref`, or writing `ref` for a non-`ref` parameter, is a compile error. The same rule applies when calling a `ref self` method by path (`Counter::increment(ref c)`, 4.7.5).

This rule makes visible at the call site which variables can change, with the same motivation as C#'s `ref` pattern.

```rok
fn read(x: i32) { ... }
fn write(ref x: i32) { ... }

read(a);                          // plain - clearly a does not change
write(ref a);                      // ref - clearly a may change
```

## 4.7.4. Closure capture lists

```rok
let count = 0;
let inc = [ref count] || { count = count + 1 };
```

Following the `ref` capture defined in [Capture Semantics](406-capture-semantics.md), `ref` variables in a capture list are held in a shared cell (boxing). The outer scope and the closure read and write the same cell, so the closure may outlive the variable's scope.

## 4.7.5. Exception for method receivers

Methods in [`impl`](../002-types/209-impl.md) may have a `ref self` parameter.

```rok
impl Counter {
    fn increment(ref self) { self.count = self.count + 1; }
}
```

In this case, `ref` is not required at the call site.

```rok
let c = Counter { count: 0 };
c.increment();                    // no 'ref' - handled automatically for method receivers
```

This is a special rule for method receivers, unlike ordinary `ref` parameters. Method call syntax is itself a mechanism that [dispatches](../000-meta/007-glossary.md#dispatch) the receiver automatically, so an extra marker would only be visual noise.

```rok
fn external_inc(ref c: Counter) { ... }
external_inc(ref c);              // ordinary function calls require 'ref'
Counter::increment(ref c);        // calling a ref self method by path also requires ref
```

### Meaning of `ref self` - copy-in / copy-out

`ref self` behaves as copy-in / copy-out. At the call, the receiver's value is copied into `self`; when the call ends, the final value of `self` is written back to the receiver's place.

- **Exclusivity**: while the call is in progress, the receiver variable must not be accessed through another path (another argument, a capturing closure, etc.). If the compiler can see this, it is a compile error (like Swift's exclusivity rule).
- **Temporary receivers**: if the receiver is a temporary (`make().increment()`), the write-back result is discarded.

```rok
c.increment();                     // copies c in, writes back to c at the end
c.add_from(|| c.count);            // ERROR - c accessed through a closure during the call
```

## 4.7.6. `ref` and types

`ref` is a modifier, not a type, and appears nowhere in [type expressions](../002-types/201-type-expressions.md).

```rok
let x: ref i32 = ...;             // ERROR - ref is not a type
struct S { x: ref i32; }          // ERROR - same reason
List<ref i32>                      // ERROR - same reason
```

`ref` may therefore appear only in:

- Parameter declarations of functions and methods.
- Closure capture lists.
- Call arguments of functions and methods.

Since `ref` cannot be used in a return type, rok has no functions that return references (no forms like Rust's `deref_mut`). APIs that change an inner value exchange values, as in `set` / `update(f)`.

## 4.7.7. References cannot be first-class values

```rok
let r = ref x;                    // ERROR - ref itself is not an expression
let r: SomePointerType = ref x;   // ERROR
```

`ref` is not an expression that produces a value; it is only a modifier token attached to calls, parameters and capture lists.

Coroutine functions (`yield fn`, `async fn`, `async yield fn`) cannot have `ref` parameters; doing so is a compile error. The body keeps running after the call returns, so there is no point at which to write back.

For the same reason, a function with `ref` parameters cannot be used as a function value (`let f = increment;` is an error), and closures cannot have `ref` parameters. The function type `fn(..)` has no place to express `ref`.

If a first-class reference is needed (e.g., [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface)), use the raw pointer types of [type expressions](../002-types/201-type-expressions.md) (`Ptr<T>` and `ConstPtr<T>`), which belong to the `unsafe` area of the [standard library tiers](../000-meta/005-standard-libraries-tiers.md).

## 4.7.8. Summary: pass by value vs pass by `ref`

```rok
fn by_value(x: i32) { x = 10i32; }
fn by_ref(ref x: i32) { x = 10i32; }

let a = 5i32;
by_value(a);                      // a = 5 (unchanged)
by_ref(ref a);                    // a = 10
```

| Aspect                                   | Pass by value (default) | Pass by `ref` (`ref`) |
|------------------------------------------|-------------------------|-----------------------|
| Effect of reassignment in the parameter  | local only              | caller's variable     |
| Explicit call token                      | none                    | `ref` required        |
| Visual signal                            | ordinary call           | marks possible change |

## 4.7.9. Structs and reference handles

A `struct` is a value type: pass by value copies it, so a field write in the callee is local. To change the caller's struct, take it as `ref` (or `ref self`), or return the new value. A reference handle (`List`, `Dict`, …, [stdlib 2.9](../../refs-standard-libraries/200-core/209-collections.md#reference-handles)) is copied as a handle, so changes to its elements are visible to the caller.

```rok
fn set_x(p: Point) { p.x = 10.0; }          // local copy - the caller's p is unchanged
fn set_x_ref(ref p: Point) { p.x = 10.0; }  // written back to the caller's p
fn fill(xs: List<i32>) { xs.push(1); }      // same List - the caller sees it
fn reset(ref xs: List<i32>) { xs = []; }    // the caller's variable holds a new List
```

| Scenario                                          | Keyword needed                  |
|---------------------------------------------------|---------------------------------|
| Change a field of the caller's struct             | `ref`                           |
| Change elements of a collection                   | ordinary pass by value - enough |
| Make the caller's variable hold another value     | `ref`                           |

## 4.7.10. `ref` on two bindings at once

```rok
fn swap(ref a: i32, ref b: i32) {
    let t = a;
    a = b;
    b = t;
}

let x = 1i32;
let y = 2i32;
swap(ref x, ref y);               // x = 2, y = 1
```

As shown, a single call may have multiple `ref` parameters.

Passing the same place as two `ref`s at the call site is a compile error under the exclusivity rule of 4.7.2.

```rok
swap(ref x, ref x);               // ERROR - same place passed as ref twice
```
