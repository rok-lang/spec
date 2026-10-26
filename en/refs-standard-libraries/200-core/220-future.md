# 2.20. future - `Future` / `await` integration

The `rok::core::future` module provides the authoritative traits of the async mechanism. The desugaring of `await expr` lowers to dispatch on the `Future` trait defined in this chapter. This chapter also defines `future_trait`, one of the 48 lang slots.

## 2.20.1. `Future` - async result

`Future` is the core trait representing the result of an asynchronous computation; the result type is the associated type `Output`.

```rok
#[lang("future_trait")]
trait Future {
    type Output;

    fn poll(ref self, ref ctx: Context) -> Poll<Output>;
}

enum Poll<T> {
    Ready(T),
    Pending,
}
```

`poll(ctx)` has two outcomes. `Ready(v)` means the result is ready and `v` is returned as is; `Pending` means there is no result yet, and the future is polled again when the waker in `ctx` wakes it.

This model is isomorphic to Rust's `Future` and `rok::core::task::Poll`, and compiles to a **stackless coroutine** (a compiler-generated state machine, [refs-lang 7.8](../../refs-lang/007-runtime/708-coroutine.md)).

## 2.20.2. `Context` / `Waker`

`Context` and `Waker` form the communication channel between poll and the scheduler.

```rok
struct Context {
    waker: Waker;
}

struct Waker {
    // internal: a callback that communicates with the scheduler
}

impl Waker {
    fn wake(self);              // signal to schedule this task again
    fn clone(self) -> Waker;
}
```

When `poll` returns `Pending`, a Waker is registered internally; when an event (I/O or timer) later occurs, the waker is called and the task is scheduled again. Most users never handle `Waker` directly, because `await` takes care of this automatically.

## 2.20.3. `await expr` desugaring

The most common form of use:

```rok
let v = await some_future;
```

This conceptually desugars to:

```rok
let __f = IntoFuture::into_future(some_future);
let v = loop {
    match __f.poll(ref CURRENT_CONTEXT) {
        Ready(v)  => break v,
        Pending   => yield_to_scheduler(),
    }
};
```

The compiler turns this into a state machine, and each `await` becomes a **suspend point**. Only frame variables are kept, in coroutine state located on the GC heap.

## 2.20.4. `async fn` / `async { }` / `async || { }`

All three forms return **`AsyncFuture<T>`** ([refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#result-types-of-coroutine-functions)).

```rok
// (a) named function
async fn fetch(url: str) -> Response {
    let raw = await get(url);
    await parse(raw)
}
await fetch("/");       // fetch("/"): AsyncFuture<Response>

// (b) inline block (a value)
let f = async { await compute() };       // AsyncFuture<T>
await f;

// (c) closure
let g = async || { await compute() };    // a closure that, when called, returns a value implementing Future<Output = T>
await g();
```

The authority for these **three forms** is [refs-lang 4.12 async](../../refs-lang/004-functions/412-async.md).

## 2.20.5. `IntoFuture` - awaitable values

`IntoFuture` is the unified entry point of await. Not only `Future`s but also handles such as `Thread` that are not themselves `Future` can be `await`ed through this trait.

```rok
trait IntoFuture {
    type Output;
    type IntoFuture: Future<Output = Output>;

    fn into_future(self) -> IntoFuture;
}

impl<F: Future> IntoFuture for F {               // Task<T>: Future<Output = T> is included here too
    type Output = F::Output;
    type IntoFuture = F;
    fn into_future(self) -> F { self }
}
impl<T: Send> IntoFuture for JoinHandle<T> { type Output = T; type IntoFuture = Blocking<T>; ... }
```

The impl for `JoinHandle<T>` joins the thread in the blocking pool (`task::spawn_blocking`, [2.18](218-task.md)), so only the awaiting task waits. It is legal only as long as that type does not implement `Future`, so that it does not overlap with the blanket impl ([refs-lang 2.9.5](../../refs-lang/002-types/209-impl.md#coherence)).

`await expr` first calls `IntoFuture::into_future(expr)` and then repeatedly calls `Future::poll` on the result (2.20.3). This is the same way `for await` in [2.8 stream](208-stream.md) handles `IntoStream`.

Only `future_trait` is registered as a lang slot; `IntoFuture` is an ordinary trait. The compiler calls `IntoFuture::into_future` directly in the `await` desugaring, so, as with `IntoStream`, no separate lang slot is needed.

## 2.20.6. The "color of functions" of async / await

JavaScript and Python have the well-known **function color** problem: async and sync functions cannot mix freely (await is possible only inside async, and async calls are blocked in sync functions).

Rok's policy:

- `async fn` and `await` make the color explicit
- `for` and `for await` make the color explicit
- No attempt is made to unify colors. JS, Python, C# and Swift all follow the same model

The colors are separated to guarantee **scheduler cooperation**. The compiler can build a coroutine state machine only if "this position can suspend" is stated explicitly.

```rok
// await inside a sync fn - compile error
fn sync_caller() {
    let v = await some_future;       // ERROR - await is allowed only inside an async function
}

async fn async_caller() {
    let v = await some_future;       // OK
}
```

### Getting a future's result in a sync context

To get a future's result in a sync context, use `block_on`.

```rok
let v = task::block_on(some_future);    // the current thread attaches to the executor and runs it to completion
```

`block_on` must be used only at the top level (main or a sync caller). Using `block_on` inside nested async code risks deadlock.

## 2.20.7. `for await` - consuming a Stream

A Stream is consumed with the `for await` form.

```rok
for await event in stream {            // consumes Stream<Item = T> (async)
    handle(event);
}

for x in iterator {                    // consumes Iterator<Item = T> (sync)
    use(x);
}
```

The authority for `for await` is the `Stream` trait in [2.8 stream](208-stream.md).

## 2.20.8. `async yield` - producing a Stream

Conversely, a Stream is produced with `async yield`.

```rok
async yield fn ticks(interval: Duration) -> Instant {
    loop {
        await task::sleep(interval);
        yield Instant::now();
    }
}

for await t in ticks(Duration::ms(100)) {
    log("tick at {t}");
}
```

`async yield` combines async and yield and produces a Stream. The syntax authority is [refs-lang 4.13 stream](../../refs-lang/004-functions/413-stream.md).

## 2.20.8a. `AsyncFuture<T>` / `TryAsyncFuture<T, E>` - values of async functions

These are the values of `async fn` calls and `async { .. }` blocks ([refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#result-types-of-coroutine-functions)).

```rok
struct AsyncFuture<T> { /* private */ }
impl<T> Future for AsyncFuture<T> {
    type Output = T;
    fn poll(ref self, ref ctx: Context) -> Poll<T>;   // advances the body to the next wait point
}

struct TryAsyncFuture<T, E> { /* private */ }    // async fn .. throws (E)
impl<T, E> TryAsyncFuture<T, E> {
    fn poll(ref self, ref ctx: Context) throws (E) -> Poll<T>;
}
```

`TryAsyncFuture` does not implement `Future`, because its `poll` throws; it is awaited with `try await`. The private fields are not specified. Copies share the same coroutine frame and therefore share the progress position.

## 2.20.9. Module mapping

The module location and lang slot / role of each item:

| Item             | Module                           | Role                        |
|------------------|----------------------------------|-----------------------------|
| `Future`         | `rok::core::future::Future`      | `#[lang("future_trait")]`   |
| `Poll<T>`        | `rok::core::future::poll`        | enum Ready/Pending          |
| `Context`        | `rok::core::future::context`     | waker holder                |
| `Waker`          | `rok::core::future::waker`       | scheduler callback          |
| `IntoFuture`     | `rok::core::future::into_future` | ordinary trait (no lang slot) |
| `task::block_on` | `rok::core::task`                | sync entry point            |

## 2.20.10. Comparison with other languages

The asynchronous models of each language compare as follows.

| Aspect               | Rust                    | C#                 | Swift                  | Rok                               |
|----------------------|-------------------------|--------------------|------------------------|-----------------------------------|
| Model                | poll-based              | callback (state)   | continuation           | poll-based (isomorphic to Rust)   |
| `async fn`           | ✓                       | ✓                  | ✓                      | ✓                                 |
| `await`              | postfix `.await`        | prefix `await`     | prefix `await`         | prefix `await`                    |
| Coroutine implementation | compiler-generated  | compiler-generated | compiler-generated     | compiler state machine            |
| Stackless            | ✓                       | ✓                  | ✓                      | ✓                                 |
| Color unification    | ✗ (sync/async split)    | ✗                  | ✗                      | ✗ (adopts the industry standard)  |
| await generalization | `Future` + external libs | unified `Task<T>` | `Task` + AsyncSequence | generalized via `IntoFuture`      |

Rok differs in two main ways.

1. **Prefix `await`**: the same as C#, Swift and JS; the authority is [refs-lang 4.12](../../refs-lang/004-functions/412-async.md). To chain on the result, wrap it in parentheses (`(await x).method()`).
2. The ordinary trait `IntoFuture` lets Task and Thread be awaited uniformly. The idea is similar to Swift's `Task` unification, but differs in preserving the distinct types and generalizing through a trait.
