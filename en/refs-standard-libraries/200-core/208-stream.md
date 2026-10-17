# 2.8. stream — async iterator

The `rok::core::stream` module defines `Stream`, the async stepping trait (its element type is the associated type `Item`). Whereas `Iterator` of [2.7 iter](207-iter.md) is **sync**, `Stream` is **async**: each `next()` call is **awaitable**.

This module is also the desugaring authority for `for await x in stream`, which dispatches to the traits defined in this chapter.

## 2.8.1. `Stream` — async stepping

The trait is defined as follows.

```rok
trait Stream {
    type Item;

    fn poll_next(ref self, ref ctx: Context) -> Poll<Item | none>;
}
```

This is the async version of `Iterator::next`. The return value means: `Poll::Ready(v)` (`v: Item`) — the next element `v` is ready; `Poll::Ready(none)` — the end of the stream; `Poll::Pending` — no value is ready yet, and the stream is polled again when the Waker wakes it.

In short, it has the same meaning as `Iterator`'s `next() -> T | none`, the decisive difference being that it is **awaitable**.

## 2.8.2. `for await` desugaring

Consider the following.

```rok
for await x in stream { body }
```

Conceptually it desugars to:

```rok
loop {
    match await stream.poll_next(ref CURRENT_CONTEXT) {
        v: Item => { let x = v; body; }
        none    => break,
    }
}
```

This has the same shape as the sync version in [2.7.2 for desugar](207-iter.md), with only `await` added.

## 2.8.3. `IntoStream` — async iterable

Next is the trait representing an async iterable.

```rok
trait IntoStream {
    type Item;
    type IntoStream: Stream<Item = Item>;

    fn into_stream(self) -> IntoStream;
}
```

At the start of a `for await`, `IntoStream::into_stream` is called, then `Stream::poll_next` is repeated. The concrete transformation is:

```rok
for await event in event_source { ... }
// = let __s = event_source.into_stream();
//   loop {
//       match await __s.poll_next(ref CTX) {
//           e: Item => { let event = e; body; }
//           none    => break,
//       }
//   }
```

## 2.8.4. `async yield` — creating a Stream

The most direct way to create a Stream is to write an `async yield` function.

```rok
async yield fn ticks(interval: Duration) -> Instant {
    loop {
        await task::sleep(interval);
        yield Instant::now();
    }
}
```

`-> Instant` is the element type, and the result of a call is `AsyncStream<Instant>` (implementing `Stream<Item = Instant>`) ([refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#result-types-of-coroutine-functions)). The compiler turns this function into a state machine ([refs-lang 7.8](../../refs-lang/007-runtime/708-coroutine.md)). Each `yield` is a **suspend point**, and so is each `await`.

The syntax authority is [refs-lang 4.13 stream (`async yield`)](../../refs-lang/004-functions/413-stream.md).

## 2.8.5. Stream adapters

As with `Iterator`, a range of adapters are provided as default methods of the `Stream` trait, so every type implementing `Stream` (`AsyncStream<T>`, `Receiver<T>`, user types) has the same adapters.

```rok
trait Stream {
    type Item;
    fn poll_next(ref self, ref ctx: Context) -> Poll<Item | none>;

    // default methods
    fn map<U>(self, f: fn(Item) -> U) -> Map<Self, U>;
    fn filter(self, pred: fn(Item) -> bool) -> Filter<Self>;
    fn take(self, n: usize) -> Take<Self>;
    fn skip(self, n: usize) -> Skip<Self>;

    async fn collect<C: FromStream<Item> = List<Item>>(self) -> C;   // List<Item> unless the slot decides
    async fn for_each(self, f: fn(Item));

    // async-specific
    fn buffer_unordered(self, n: usize) -> BufferUnordered<Self>;
    fn buffered(self, n: usize) -> Buffered<Self>;
    fn throttle(self, dur: Duration) -> Throttle<Self>;
}
```

As with the adapters of [2.7 iter](207-iter.md), the return types are named concrete structs, located in the `rok::core::stream` module (`stream::Map<S, U>`, `stream::Filter<S>`, ...). `Map<Self, U>` implements `Stream<Item = U>`; the rest implement `Stream<Item = Item>`.

> [!NOTE]
> `stream::Map` / `Filter` / `Take` / `Skip` / `Zip` share their names with `iter::Map` and the like. Under the implicit `use rok::core::**;` the bare name `Map` is ambiguous ([refs-lang 5.5.6](../../refs-lang/005-modules/505-use.md)), so when the type must be written, use a path such as `stream::Map` or import it with an explicit `use`.

Of these, `buffer_unordered(n)` and `buffered(n)` serve as concurrency control, processing N futures concurrently.

## 2.8.6. `FromStream<T>` — dispatch of `collect`

The trait that handles dispatch of `collect` is `FromStream<T>`. It is the async counterpart of `FromIterator<T>` in [2.7 iter](207-iter.md); its signature has the same form, except that `from_stream` is awaitable.

```rok
trait FromStream<T> {
    async fn from_stream<S: IntoStream<Item = T>>(stream: S) -> Self;
}

impl<T> FromStream<T> for List<T> { ... }
impl<T: Hash> FromStream<T> for Set<T> { ... }
impl<K: Hash, V> FromStream<(K, V)> for Dict<K, V> { ... }
```

The target of `collect` is determined by the receiving slot; if the slot does not determine it, it is `List<Item>`.

When `Stream::collect` is called, the target collection's `from_stream` is called, and every element of the stream is awaited and accumulated into the collection.

```rok
let v: List<i32> = await (0..).take(5).map(|x| async { x * 2 }).buffered(3).collect();
//   = await List::from_stream(...)
```

`List`, `Dict` and `Set` implement both `FromIterator` and `FromStream`, following the same pattern in sync and async code.

## 2.8.7. `Channel` ↔ `Stream`

`Receiver<T>` (channel) implements `Stream<Item = T>`, so channel reception can be treated directly as a stream.

```rok
let (tx, rx) = channel::<Event>();
task::spawn(async { tx.send(event1); tx.send(event2); });

for await event in rx {
    handle(event);
}
```

That is, the receiver of [2.17 channel](217-channel.md) is treated as an **automatic stream** and is compatible with `for await`.

## 2.8.8. `merge` / `zip`, etc.

Combinators that combine several streams are also provided.

```rok
fn merge<S1: Stream, S2: Stream>(a: S1, b: S2) -> Merge<S1, S2>;    // Stream<Item = S1::Item | S2::Item>
fn zip<S1: Stream, S2: Stream>(a: S1, b: S2) -> Zip<S1, S2>;        // Stream<Item = (S1::Item, S2::Item)>
fn select<S1: Stream, S2: Stream>(a: S1, b: S2) -> Select<S1, S2>;  // Stream<Item = S1::Item | S2::Item>, whichever is ready first
```

`merge` produces the union of two streams, `zip` pairs two streams, and `select` races them, taking whichever is ready first.

## 2.8.8a. `AsyncStream<T>` / `TryAsyncStream<T, E>` — the value of an async stream

These are the values of an `async yield fn` call and of an `async yield { .. }` block ([refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#result-types-of-coroutine-functions)).

```rok
struct AsyncStream<T> { /* private */ }
impl<T> Stream for AsyncStream<T> {
    type Item = T;
    fn poll_next(ref self, ref ctx: Context) -> Poll<T | none>;
}

struct TryAsyncStream<T, E> { /* private */ }    // async yield fn .. throws (E)
impl<T, E> TryAsyncStream<T, E> {
    fn poll_next(ref self, ref ctx: Context) throws (E) -> Poll<T | none>;
}
```

The adapters of `AsyncStream<T>` (`map`, `filter`, `collect`, etc.) are the default methods of `Stream` (2.8.5). `TryAsyncStream` does not implement `Stream`, because its `poll_next` throws; it is consumed with `try for await`. The private fields are not specified. Copies share the same coroutine frame and therefore share the progress position.

## 2.8.9. Module mapping

The module of each item:

| Item                          | Module                           | Form                         |
|-------------------------------|----------------------------------|------------------------------|
| `Stream`                      | `rok::core::stream::stream`      | poll-based                   |
| `IntoStream`                  | `rok::core::stream::into_stream` |                              |
| `Receiver<T>`                 | `rok::core::channel::receiver`   | implements `Stream<Item = T>` |
| `Stream::map`/`filter`/`take` | `rok::core::stream::adapter`     | adapter structs (implementing `Stream`) |
| `merge`/`zip`/`select`        | `rok::core::stream::combinator`  | `Merge`/`Zip`/`Select` structs |

## 2.8.10. Whether `Stream` has a lang slot

This chapter attaches no `#[lang(...)]` to `Stream::poll_next`; `Stream` is an ordinary trait.

The reasons: the `for await` desugaring is hard-coded in the compiler to call `Stream::poll_next` **directly**, so it is recognized without a lang slot. Moreover, lang slots are a mechanism for multiple dispatch entry points, and Stream has a single entry point, so none is needed.
