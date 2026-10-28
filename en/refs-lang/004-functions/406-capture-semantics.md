# 4.6. Capture semantics

A [closure](405-closure.md) can capture variables from the outer environment. This chapter defines the rules that determine whether a capture copies a value or holds a reference.

## 4.6.1. Default: value copy

> [!IMPORTANT]
> The default meaning of a capture is **value copy**. When the closure is created, a snapshot of the outer variable's value is taken.

```rok
let count = 0;
let f = || { count + 1 };         // the value 0 of count is copied
count = 10;
f();                               // 1 (f holds 0)
```

This decision has three motivations. First, explicitness: the capture mode is clearly visible in the code. Second, blocking side effects: it prevents the pitfall of a closure silently modifying an outer variable. Third, friendliness to scientific and numeric code: it avoids Julia's 100x boxing pitfall, where reference captures are boxed into dynamic slots and degrade performance.

```
capture_list    ::= "[" capture_item ("," capture_item)* ","? "]"
capture_item    ::= "ref" capture_name | capture_name
capture_name    ::= identifier | "self"
```

`self` names a method's receiver. `[ref self]` is allowed only in a `ref self` method, where it shares the receiver ([4.7.4](407-ref-keyword.md#474-closure-capture-lists)); a by-value `self` is captured by value.

## 4.6.2. Reference capture: the `[ref x]` capture list

To capture a reference to an outer variable's binding, use an explicit capture list.

```rok
let count = 0;
let inc = [ref count] || { count = count + 1 };
let dec = [ref count] || { count = count - 1 };

inc(); inc(); dec();
// count = 1
```

The list is written before the closure definition, as `[ref var, ref var, ...]`. Variables in the list are held in a shared cell (boxing); the outer scope and the closure use the same cell, so reassigning inside the closure is reflected in the outer variable. The cell is a GC object, so the closure may outlive the variable's scope. Variables not in the list are captured by value as usual.

### Multiple and mixed captures

```rok
let counter = 0;
let config = load_config();

let handler = [ref counter] || {
    counter = counter + 1;        // ref: reflected in the outer variable
    print(config.version);         // value: snapshot of config at capture time
};
```

Only the variables named in the capture list are `ref` captures; the rest are automatically captured by value.

## 4.6.3. Tokenization of capture lists

`[ref x] || body` is tokenized as follows: `[`, `ref`, `x`, `]` form the bracketed capture list, and `||` or `|x|` marks the start of the closure.

Ambiguity is resolved by the longest-match rule of [punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md) and by context. If `[...]` is immediately followed by `||` or `|x|`, it is a capture list; otherwise it is ordinary indexing or an array literal.

```rok
let xs = [1, 2, 3];               // List literal
let f = [ref count] || { ... };   // capture list + closure
arr[ref count]                     // ERROR: ref cannot be used in indexing
```

## 4.6.4. Reassigning an outer variable in the body → `ref` required

Attempting to reassign a value-captured outer variable in the closure body is a compile error.

```rok
let count = 0;
let f = || { count = count + 1 };
// ERROR: 'count' is value-captured.
//        To mutate the outer variable, add `[ref count]` capture list.
```

This rule guides the user to state a `ref` capture deliberately.

## 4.6.5. `ref` shorthand (none)

```rok
ref || { ... }                    // not adopted
```

There is no shorthand that `ref`-captures everything. A capture can become `ref` only through an explicit list, so that the code clearly shows which variables are held by reference.

## 4.6.6. Automatic handling of loop variables

Because value capture is the default, each iteration's loop variable keeps its own value.

```rok
let fns: List<fn() -> i32> = [];
for i in 0i32..5i32 {
    fns.push(|| i);               // copies the value of i in each iteration
}
fns[0]();                          // 0
fns[4]();                          // 4
```

No separate rule like Go 1.22's per-iteration scope is needed; value capture produces the intended meaning automatically.

## 4.6.7. Capturing structs and reference handles

Value capture copies the value into the closure. A [`struct`](../002-types/203-struct.md) is a value type, so the closure holds its own copy; a field write inside the closure does not reach the outer variable. Use `[ref p]` for that.

Collections (`List`, `Dict`, `Set`, `StrBuf`, ...) are reference handles ([stdlib 2.9](../../refs-standard-libraries/200-core/209-collections.md#reference-handles)). Value capture copies the handle, so the object is shared.

```rok
let xs = [1, 2, 3];
let f = || { xs.push(4) };        // copies the handle (same List)
f();
// xs == [1, 2, 3, 4]

let p = Point { x: 0.0, y: 0.0 };
let g = [ref p] || { p.x = 1.0 }; // ref capture: writes the outer p
g();
// p.x == 1.0
```

For an independent copy of a collection, call `.clone()`.

## 4.6.8. `ref` capture vs object mutation

```rok
let xs = [1, 2, 3];

// value capture: mutating the object is OK, reassigning the binding is not
let f = || {
    xs.push(4);                   // OK: object mutation
    // xs = [];                    // ERROR: binding reassignment
};

// ref capture: reassigning the binding is also allowed
let g = [ref xs] || {
    xs = [];                      // OK: the outer xs becomes []
};
```

For a reference handle, value capture and object mutation go together, since the object is shared. `ref` capture is used when the binding itself must be reassigned.

This distinction is the same as the `self` and `ref self` model of [`impl`](../002-types/209-impl.md).

## 4.6.9. Additional constraints in parallel and concurrent contexts

Every capture of a `fn() + Send` closure (one passed to `task::spawn`, `par::for_each`, `thread::spawn`, etc.) must be `Send + Sync` ([4.4](404-function-type.md#send--sync-bounds)). A `[ref x]` cell is written by both the closure and the code that created it, so it is never `Send` or `Sync`, whatever the type of `x`: a closure with a `[ref]` capture (or one that captures such a closure, or a future made by an `async [ref x] ||` closure) fits no `+ Send` / `+ Sync` slot. Same-thread uses (`spawn_local`, ordinary calls) are unaffected. Collections are `!Sync` ([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers)) and cannot be value-captured this way either.

```rok
let buf: List<i32> = [];
par::for_each(xs, [ref buf] |x| { buf.push(x) });
// ERROR: a [ref buf] cell cannot cross threads
```

Instead, use a reduce (`par::reduce(xs, 0, |a, b| a + b)`), share through `Mutex` / `RwLock` / `Atomic` / channels, or return the value from the thread (`join()`).

A value shared this way belongs to its new owner: after `Mutex::new(buf)` or `tx.send(buf)`, using `buf` is a compile error ([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers)). Use it through the lock, or hand over `buf.clone()`.

```rok
let m = Mutex::new(buf);
buf.push(1);
// ERROR: `buf` was moved into the `Mutex`; use it through the lock: using g = m.lock() { g.get().push(1) }
```

The detailed parallelism and concurrency rules are defined by the standard library specification; this chapter covers only the language-level capture mechanism.

## 4.6.10. Lifetime of captured variables

Captured variables live as long as the closure value. Even after the closure leaves the outer scope, the captured values are kept inside the closure object.

```rok
fn make_counter() -> fn() -> i32 {
    let count = 0;
    [ref count] || { count = count + 1; count }
    // count is captured by ref; kept until the closure goes away
}

let c = make_counter();
c();                               // 1
c();                               // 2
c();                               // 3
```

In this example, `count` stays alive, attached to the closure object, after the outer function returns. The [GC](../000-meta/007-glossary.md#gc-garbage-collector) manages this lifetime ([7. Runtime Model](../007-runtime/README.md)).
