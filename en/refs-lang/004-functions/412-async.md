# 4.12. Async (`async` and `await`)

The `async` and `await` keywords ([keywords](../001-lexical/105-keywords.md)) are the mechanism for defining asynchronous computations, functions that promise a future value, and for waiting on their results.

## 4.12.1. Three definition forms

Asynchronous definitions have three grammatical forms.

```
async_fn        ::= "async" fn_decl
async_block     ::= "async" block
async_closure   ::= "async" closure_expr
await_expr      ::= "await" expr
```

```rok
async fn name(args) -> R { ... await expr; ... }    // named async function
async { ... await expr; ... }                        // inline block (value)
async || { ... await expr; ... }                     // closure modifier
```

In all three forms, the result is a value of the library type `AsyncFuture<T>` (implementing `Future<Output = T>`), or `TryAsyncFuture<T, E>` with `throws (E)` ([4.1.11](401-function-declaration.md#result-types-of-coroutine-functions)). This structure is consistent with the pattern in which `yield` of [generators](411-generators.md) is used in the same three forms.

## 4.12.2. The `await` operator

`await` is a prefix operator applied to a value implementing `Future<Output = T>`; its result is `T`.

```rok
await expr
```

It suspends until the Future completes and then returns the result value. `await` itself is an expression, and the result value is the evaluation result of the operator.

```rok
let data = await fetch_data();
let processed = await transform(data);
```

`await` is allowed only inside the bodies of `async` functions, `async` blocks and `async` closures. Using `await` in an ordinary function body is a compile error.

## 4.12.3. Named `async` functions

An asynchronous function is declared with `async fn`.

```rok
async fn fetch_user(id: i64) -> User {
    let response = await http_get(f"/users/{id}");
    await parse_user(response)
}
```

Inside the body, `await` waits on other asynchronous results; the `-> R` position gives the type of the value produced on completion. The call result is `AsyncFuture<R>`. `-> R` may be omitted; `R` is then inferred from the body as in [4.1.3](401-function-declaration.md#413-omitting-the-return-type).

```rok
let f = fetch_user(42i64);        // f: `AsyncFuture<User>`
let u = await f;                   // waits for the result
```

The call itself returns immediately; all that is created at that point is a **Future** object. The actual work starts at `await`, or the scheduler may start it earlier.

## 4.12.4. Inline `async` blocks: values

An `async` block immediately creates an `AsyncFuture<T>` in expression position.

```rok
let f = async {
    let a = await op1();
    let b = await op2();
    a + b
};

let r = await f;
```

The body is a [block expression](../003-expressions/309-block-expressions.md). This form is for writing short asynchronous work anonymously and inline.

Like a closure, an `async` block captures outer values **by copying them when it is created** ([4.6](406-capture-semantics.md)). Changing an outer variable after the block is created does not change the value inside the future, and reassignments inside the future are not visible outside.

```rok
let n = 1;
let f = async { n * 10 };
n = 2;
await f;                       //→ 10 — n at creation time
```

## 4.12.5. `async` closures

`async ||` creates a callable asynchronous closure.

```rok
let task = async || {
    let data = await fetch();
    process(data)
};

let f1 = task();                   // f1: `AsyncFuture<...>`
let f2 = task();                   // a new Future each time
```

Each call creates a new **Future**. With a `[ref x]` capture list, `async [ref x] || { .. }`, the shared cell of `x` is used together by the outer scope and every future the closure creates.

```rok
let hits = 0;
let bump = async [ref hits] || { hits += 1; };
await bump();
await bump();
hits;                          //→ 2
```

## 4.12.6. Correspondence of the three forms

The three forms correspond as follows.

```
plain        fn f() { ... }       ↔  || { ... }       ↔  { ... }
yield form   yield fn f() { ... } ↔  yield || { ... } ↔  yield { ... }
async form   async fn f() { ... } ↔  async || { ... } ↔  async { ... }
```

The modifiers `yield` and `async` attach to the same function, block and closure forms and consistently turn them into generators or **Futures**.

## 4.12.7. The Future trait

The result of an async function, block or closure is the standard library's `AsyncFuture<T>` (implementing `Future<Output = T>`). The exact trait definition and methods such as `then`, `map` and `join` are defined in the standard library.

## 4.12.8. Semantics of `await`

The following example shows how the body is evaluated.

```rok
async fn f() -> i32 {
    let a = await op1();
    let b = await op2();
    a + b
}
```

First, `op1()` is called and returns a **Future**. Then `await` waits for that Future to complete; until it does, the current function `f` is also suspended. When the Future completes, its result is bound to `a` and evaluation resumes. `op2()` and the second `await` follow the same pattern, and after all `await`s finish, `a + b` is evaluated and returned as the function's result.

This suspension and resumption runs on the same coroutine mechanism as [generators (`yield`)](411-generators.md).

## 4.12.9. Concurrent execution: library functions

To advance multiple Futures concurrently, use standard library functions.

```rok
let rs = await task::join([task::spawn(op1()), task::spawn(op2())]);   // all — List<T>
let first = await task::race([task::spawn(op1()), task::spawn(op2())]); // whichever finishes first
let r = try await task::timeout(d, task::spawn(op1()));                // TimeoutError on timeout
```

`join` / `race` take a `List<Task<T>>`, and their result is `await`ed ([stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)).

`join`, `race`, `timeout`, `select` and the like are library functions, not language keywords, following the principle of [the boundary between language and standard library](../000-meta/004-language-vs-standard-libraries.md). Their exact signatures are defined in the standard library.

## 4.12.10. Where `await` is allowed

The positions where `await` is allowed are:

| Position                     | `await` allowed?                     |
|------------------------------|--------------------------------------|
| `async fn` body              | OK                                   |
| `async { ... }` body         | OK                                   |
| `async \|...\| { ... }` body | OK                                   |
| ordinary `fn` body           | ERROR                                |
| ordinary `{ ... }` block     | ERROR                                |
| `yield` function or block body | ERROR (but `async yield` is OK)    |

## 4.12.11. Immediate return of async function calls

Because an async function call returns immediately, the following pattern is possible.

```rok
let f = fetch_data();             // returns immediately — work has started or is scheduled
do_other_work();                   // do other work in parallel
let data = await f;                // wait when the result is needed
```

This is the basic pattern of efficient concurrency: time-consuming work is waited on when its result is needed, not right after the call.

## 4.12.12. Relationship to tasks

To launch asynchronous work as a separate unit of execution, use the standard library's `task::spawn`.

```rok
let handle = task::spawn(async { compute() });       // runs as a new task
let result = await handle;                     // waits for completion
```

`task::spawn` registers a future (here, an `async` block) with the scheduler and returns a `Task<T>` handle that implements `Future`.

`task::spawn` is a library function, orthogonal to `async`. Marking a function `async` only states that "this function is asynchronous"; to actually launch it as a separate unit of execution, an explicit call such as `task::spawn` is required.

## 4.12.13. `async` on methods

Like ordinary [methods](409-method.md), methods may be marked `async`.

```rok
impl Database {
    async fn query(self, sql: str) -> ResultSet { ... }
}

let r = await db.query("SELECT ...");
```

## 4.12.14. Error handling

`try`, `catch`, `throw` and `finally` from [error control flow](../003-expressions/314-error-control-flow.md) work as usual in async functions. Errors of a throwing `async` function are delivered at the `await` point, so the `await` expression is covered with `try` (`try await fetch()`). Interaction with `throws` signatures is covered in [`throws` signatures](414-throws.md).

```rok
async fn safe_fetch() throws -> Data {
    try await fetch_or_throw() catch e: NetworkError {
        try await fetch_fallback()
    }
}
```
