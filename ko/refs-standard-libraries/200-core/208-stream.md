# 2.8. stream — async iterator

`rok::core::stream` 모듈은 async 단계 진행 트레잇 `Stream<T>` 를 정의합니다. [2.7 iter](207-iter.md) 의 `Iterator` 가 **sync** 인 반면, `Stream` 은 **async** 이며, 각 `next()` 호출이 **await 가능** 하다는 점에서 차이가 납니다.

또한 이 모듈은 `for await x in stream` 의 desugar 권위를 가지며, 해당 구문은 이 장에서 정의하는 트레잇으로 dispatch 됩니다.

## 2.8.1. `Stream<T>` — async 단계 진행

먼저 트레잇 정의는 다음과 같습니다.

```rok
trait Stream {
    type Item;

    fn poll_next(ref self, ctx: ref Context) -> Poll<Item | none>;
}
```

이는 `Iterator::next` 의 async 버전에 해당하며, 반환값의 의미는 다음과 같이 해석됩니다. `Poll::Ready(Some(v))` 는 다음 원소 `v` 가 준비되었음을 뜻하고, `Poll::Ready(none)` 은 stream 의 끝을 의미하며, `Poll::Pending` 은 아직 값이 준비되지 않았음을 나타내어 Waker 가 깨우면 다시 poll 하게 됩니다.

요컨대 `Iterator` 의 `next() -> T | none` 과 의미는 같지만, **await 가능** 하다는 점이 결정적인 차이입니다.

## 2.8.2. `for await` desugar

다음과 같은 구문이 있을 때를 살펴봅니다.

```rok
for await x in stream { body }
```

이는 개념적으로 다음과 같이 desugar 됩니다.

```rok
loop {
    match stream.poll_next(ref CURRENT_CONTEXT).await {
        Some(v) => { let x = v; body; }
        none => break,
    }
}
```

이 구조는 [2.7.2 for desugar](207-iter.md) 의 sync 버전과 동형이며, 단지 `await` 만 추가된 형태입니다.

## 2.8.3. `IntoStream` — async iterable

이어서 async iterable 을 표현하는 트레잇을 살펴봅니다.

```rok
trait IntoStream {
    type Item;
    type IntoStream: Stream<Item = Item>;

    fn into_stream(self) -> IntoStream;
}
```

`for await` 의 시작 자리에서는 `IntoStream::into_stream` 을 호출한 뒤 `Stream::poll_next` 를 반복하게 됩니다. 구체적인 변환 형태는 다음과 같습니다.

```rok
for await event in event_source { ... }
// = let __s = event_source.into_stream();
//   loop {
//       match __s.poll_next(ref CTX).await {
//           Some(e) => { let event = e; body; }
//           none => break,
//       }
//   }
```

## 2.8.4. `async yield` — Stream 생성

Stream 을 직접 생성하는 가장 자연스러운 방법은 `async yield` 함수를 작성하는 것입니다.

```rok
async yield fn ticks(interval: Duration) -> Stream<Instant> {
    loop {
        task::sleep(interval).await;
        yield Instant::now();
    }
}
```

`async yield` 는 Stream 을 반환하는 함수를 만들며, 컴파일러는 이를 LLVM coro.* state machine 으로 변환합니다. 이때 `yield` 자리는 **suspend point** 이며, `await` 자리 역시 **suspend point** 가 됩니다.

자세한 syntax 권위는 [refs-lang 4.13 stream (`async yield`)](../../refs-lang/004-functions/413-stream.md) 을 따릅니다.

## 2.8.5. Stream 어댑터

`Iterator` 와 비슷하게 다양한 어댑터가 제공됩니다.

```rok
impl<T> Stream<Item = T> {
    fn map<U, F: Fn(T) -> U>(self, f: F) -> impl Stream<Item = U>;
    fn filter<F: Fn(T) -> bool>(self, pred: F) -> impl Stream<Item = T>;
    fn take(self, n: usize) -> impl Stream<Item = T>;
    fn skip(self, n: usize) -> impl Stream<Item = T>;

    fn collect<C: FromStream<T>>(self) -> C;        // await 까지 포함
    fn for_each<F: Fn(T)>(self, f: F);              // await 까지 포함

    // async-specific
    fn buffer_unordered(self, n: usize) -> impl Stream<Item = T>;
    fn buffered(self, n: usize) -> impl Stream<Item = T>;
    fn throttle(self, dur: Duration) -> impl Stream<Item = T>;
}
```

이 가운데 `buffer_unordered(n)` 와 `buffered(n)` 는 N개의 future 를 동시에 처리하는 concurrency control 수단으로 동작합니다.

## 2.8.6. `FromStream<T>` — `collect` 의 dispatch

`collect` 의 dispatch 를 담당하는 트레잇은 `FromStream<T>` 입니다. [2.7 iter](207-iter.md) 의 `FromIterator<T>` 에 대응하는 async 버전이며, 트레잇 시그니처도 같은 형태를 따르되 `from_stream` 이 await 가능한 형태로 동작합니다.

```rok
trait FromStream<T> {
    async fn from_stream<S: IntoStream<Item = T>>(stream: S) -> Self;
}
```

`Stream::collect` 가 호출되면 대상 컬렉션의 `from_stream` 이 호출되며, stream 의 모든 원소가 await 후 누적되어 컬렉션을 만듭니다.

```rok
let v: List<i32> = (0..).take(5).map(|x| async { x * 2 }).buffered(3).collect().await;
//   = List::from_stream(...).await
```

`List`, `Dict`, `Set`, `String` 등 모든 collectable 컨테이너가 `FromIterator` 와 `FromStream` 을 함께 impl 하여 sync / async 양쪽에서 같은 패턴을 따릅니다.

## 2.8.7. `Channel` ↔ `Stream`

`Receiver<T>` (channel) 는 `Stream<Item = T>` 를 구현하므로, 채널 수신을 그대로 stream 처럼 다룰 수 있습니다.

```rok
let (tx, rx) = channel::<Event>();
task::spawn { tx.send(event1); tx.send(event2); };

for await event in rx {
    handle(event);
}
```

즉, [2.17 channel](217-channel.md) 의 receiver 는 **자동 stream** 으로 취급되어 `for await` 와 호환됩니다.

## 2.8.8. `merge` / `zip` 등

여러 stream 을 결합하는 combinator 도 제공됩니다.

```rok
fn merge<S1: Stream, S2: Stream>(a: S1, b: S2) -> impl Stream<Item = S1::Item | S2::Item>;
fn zip<S1: Stream, S2: Stream>(a: S1, b: S2) -> impl Stream<Item = (S1::Item, S2::Item)>;
fn select<S1: Stream, S2: Stream>(a: S1, b: S2) -> impl Stream<Item = S1::Item>;  // 첫 ready
```

각 함수의 의미를 정리하면, `merge` 는 두 stream 의 union 을 만들고, `zip` 은 두 stream 을 pair 로 묶으며, `select` 는 race 식으로 먼저 준비되는 쪽을 취합니다.

## 2.8.9. 모듈 매핑

각 항목이 자리 잡는 모듈은 다음 표에 정리되어 있습니다.

| 항목                          | 모듈                            | 형태                         |
|-------------------------------|---------------------------------|------------------------------|
| `Stream`                      | `rok::core::stream::stream`      | poll-based                   |
| `IntoStream`                  | `rok::core::stream::into_stream` |                              |
| `Receiver<T>`                 | `rok::core::channel::receiver`   | impl Stream                  |
| `Stream::map`/`filter`/`take` | `rok::core::stream::adapter`     | adapter struct + impl Stream |
| `merge`/`zip`/`select`        | `rok::core::stream::combinator`  |                              |

## 2.8.10. `Stream` lang slot 여부

이 장에서는 `Stream::poll_next` 에 `#[lang(...)]` 을 부착하지 않으며, `Stream` 은 일반 트레잇으로 둡니다.

그 이유는 다음과 같습니다. 우선 `for await` desugar 가 **직접** `Stream::poll_next` 를 호출하는 형태로 컴파일러에 hard-coded 되어 있어, lang slot 을 따로 두지 않고도 인식이 가능합니다. 또한 lang slot 은 다중 dispatch 진입점에 필요한 메커니즘인데, Stream 의 자리는 단일하므로 굳이 필요하지 않습니다.
