# 3.17. `using` expression - deterministic release of resources

The `using` keyword ([keywords](../001-lexical/105-keywords.md)) is an expression form that ties a resource to the lifetime of a block. However the block ends (normal completion, early `return`, `none` return by `?`, error propagation), `Drop::drop` of the bound resource is always called.

The GC reclaims memory automatically, but external resources (files, sockets, mutexes, FFI handles, GC pins, etc.) must be released deterministically. `using` guarantees this determinism syntactically and is the only path by which `Drop::drop` is called. An ordinary binding is not `drop`ped when it leaves scope ([stdlib 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop---cleanup-on-using-release)).

## 3.17.1. Form

```
using_expr     ::= "using" using_binding ("," using_binding)* block
using_binding  ::= pattern "=" expr
```

The components mean:

- `pattern` may be any irrefutable pattern defined in [patterns](311-patterns.md) (identifier, tuple, struct destructuring, etc.). In most cases a single identifier is used. What is released is the whole value of `expr`.
- `expr` is an expression that creates or obtains the resource. Its result type must implement the [`Drop`](../../refs-standard-libraries/200-core/213-marker.md) trait.
- If acquiring the resource can fail, mark `expr` with `try` (`using f = try fs::open(path) { ... }`). If acquisition raises an error, control leaves with no resource bound, so no `Drop` call occurs. The detailed semantics follow [error control flow](314-error-control-flow.md).
- Several bindings joined with `,` form multiple resources; the detailed behavior is defined in [3.17.4 multiple resources](#3174-multiple-resources).
- `block` follows the rules of [block expressions](309-block-expressions.md), and the value of its last expression is the value of the whole `using` expression.

## 3.17.2. Simplest example

```rok
using f = fs::open(path) {
    f.read_to_end()                   // the last expression is the value of using
}                                  // ← f.drop() is called automatically here
```

The return type of `fs::open(path)` is `File`, and `File: Drop`, so it can be the target of `using`. When the block ends, `File::drop` is called and the OS handle is closed.

## 3.17.3. Combining with `try`

When acquiring the resource can itself fail, mark the acquisition expression with `try`.

```rok
fn read_first_line(path: str) throws (IoError) -> str {
    using f = try fs::open(path) {     // on failure, IoError propagates
        try f.read_line()              // try can propagate inside the body too
    }                                  // if f was bound, it is dropped on both success and error
}
```

This form is semantically equivalent to the following expanded code.

```rok
let f = try fs::open(path);
try {
    f.read_line()
} finally {
    f.drop();          // a call inserted by the compiler; writing it directly is an error
}
```

Even if an error propagates inside the body, `drop` is called at the position corresponding to `finally`.

## 3.17.4. Multiple resources

To bind several resources to one block, nest `using` or join the bindings with commas. This specification allows both forms, and their meaning is equivalent.

```rok
using a = AcquireA() {
    using b = AcquireB(a) {
        body
    }
}

// or the shorthand form
using a = AcquireA(), b = AcquireB(a) {
    body
}
```

Both forms follow **reverse-order release**: `b` is dropped first, then `a`. This is the same rule as Java try-with-resources. Releasing in reverse order safely unwinds dependencies (`b` referring to `a`).

## 3.17.5. `using` as an expression

`using` is an expression, not a statement, and its value is the value of the body block.

```rok
let content = using f = try fs::open(path) {
    try f.read_to_end()
};
```

This is the biggest structural difference from function-scoped `defer` (Go). The `using` expression follows the relevant rules of [block expressions](309-block-expressions.md) and can be used in any expression position.

## 3.17.6. When drop is called

`using` guarantees `drop` of the bound resource on all five of the following paths:

- normal completion of the block (last expression evaluated)
- `return` inside the body
- `break` / `continue` inside the body
- `none` return by `?` inside the body
- `throw` inside the body, and error propagation from calls marked with `try`

Panics and fatal errors terminate the process immediately, so `drop` is not called ([3.14.8](314-error-control-flow.md#3148-panics-and-fatal-errors)). This is the general result of a panic, not behavior specific to `using`.

## 3.17.7. Types that can be bound

The type of the right-hand expression of `using` must implement [`Drop`](../../refs-standard-libraries/200-core/213-marker.md) or contain fields that have `Drop` (drop glue, [stdlib 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop---cleanup-on-using-release)). Otherwise it is a compile error.

```rok
using x = 42 { ... }   // ERROR: using requires a type implementing Drop; i32 does not implement Drop
```

This restriction ensures only meaningful resources are bound with `using`, ruling out `using` on plain values at the specification level.

## 3.17.8. Comparison with other languages

| Language | Form                          | Scope unit   | Notes                                   |
|----------|-------------------------------|--------------|-----------------------------------------|
| Rok      | `using x = try e { ... }`     | block        | expression, multiple resources, `try` integration |
| C#       | `using (var x = e) { ... }`   | block        | requires `IDisposable`                  |
| Java     | `try (var x = e) { ... }`     | try block    | requires `AutoCloseable`, multiple resources |
| Python   | `with x as f: ...`            | block        | `__enter__` / `__exit__`                |
| Swift    | `defer { x.close() }`         | function     | must be written explicitly              |
| Go       | `defer x.Close()`             | **function** | not block-scoped                        |
| Zig      | `defer x.close();`            | block        | `errdefer` variant                      |
| Rust     | (automatic RAII, no keyword)  | block        | no GC; drop point determined automatically  |

Rok's `using` combines the explicit block pattern of GC languages (C#, Java, Python) with composability as an expression, `try` integration, and reverse-order release of multiple resources.
