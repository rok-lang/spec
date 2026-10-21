# 2.8. stream — async iterator

`rok::core::stream` 모듈은 async 단계 진행 트레잇 `Stream` (원소 타입은 associated type `Item`) 을 정의합니다. [2.7 iter](207-iter.md) 의 `Iterator` 가 **sync** 인 반면, `Stream` 은 **async** 이며, 각 `next()` 호출이 **await 가능** 하다는 점에서 차이가 납니다.

또한 이 모듈은 `for await x in stream` 의 desugar 권위를 가지며, 해당 구문은 이 장에서 정의하는 트레잇으로 dispatch 됩니다.

## 2.8.1. `Stream` — async 단계 진행

먼저 트레잇 정의는 다음과 같습니다.

```rok
trait Stream {
    type Item;

    fn poll_next(ref self, ref ctx: Context) -> Poll<Item | none>;
}
```

이는 `Iterator::next` 의 async 버전에 해당하며, 반환값의 의미는 다음과 같이 해석됩니다. `Poll::Ready(v)` (`v: Item`) 는 다음 원소 `v` 가 준비되었음을 뜻하고, `Poll::Ready(none)` 은 stream 의 끝을 의미하며, `Poll::Pending` 은 아직 값이 준비되지 않았음을 나타내어 Waker 가 깨우면 다시 poll 하게 됩니다.

요컨대 `Iterator` 의 `next() -> T | none` 과 의미는 같지만, **await 가능** 하다는 점이 결정적인 차이입니다.

## 2.8.2. `for await` desugar

다음과 같은 구문이 있을 때를 살펴봅니다.

```rok
for await x in stream { body }
```

이는 개념적으로 다음과 같이 desugar 됩니다.

```rok
loop {
    match await stream.poll_next(ref CURRENT_CONTEXT) {
        v: Item => { let x = v; body; }
        none    => break,
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

// 모든 stream 은 자기 자신의 IntoStream (IntoIterator 처럼, 2.7)
impl<S: Stream> IntoStream for S {
    type Item = S::Item;
    type IntoStream = S;
    fn into_stream(self) -> S { self }
}
```

`for await` 의 시작 자리에서는 `IntoStream::into_stream` 을 호출한 뒤 `Stream::poll_next` 를 반복합니다. 구체적인 변환 형태는 다음과 같습니다.

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

## 2.8.4. `async yield` — Stream 생성

Stream 을 직접 생성하는 가장 자연스러운 방법은 `async yield` 함수를 작성하는 것입니다.

```rok
async yield fn ticks(interval: Duration) -> Instant {
    loop {
        await task::sleep(interval);
        yield Instant::now();
    }
}
```

`-> Instant` 는 원소 타입이며, 호출 결과는 `AsyncStream<Instant>` (`Stream<Item = Instant>` 구현) 입니다 ([refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#코루틴-함수의-결과-타입)). 컴파일러는 이 함수를 상태 기계로 바꿉니다 ([refs-lang 7.8](../../refs-lang/007-runtime/708-coroutine.md)). 이때 `yield` 자리는 **suspend point** 이며, `await` 자리 역시 **suspend point** 가 됩니다.

자세한 syntax 권위는 [refs-lang 4.13 stream (`async yield`)](../../refs-lang/004-functions/413-stream.md) 을 따릅니다.

## 2.8.5. Stream 어댑터

`Iterator` 와 비슷하게 다양한 어댑터가 `Stream` 트레잇의 기본 메서드로 제공됩니다. 그래서 `Stream` 을 구현한 모든 타입 (`AsyncStream<T>`, `Receiver<T>`, 사용자 타입) 이 같은 어댑터를 가집니다.

```rok
trait Stream {
    type Item;
    fn poll_next(ref self, ref ctx: Context) -> Poll<Item | none>;

    // 기본 메서드
    fn map<U>(self, f: fn(Item) -> U) -> Map<Self, U>;
    fn filter(self, pred: fn(Item) -> bool) -> Filter<Self>;
    fn take(self, n: usize) -> Take<Self>;
    fn skip(self, n: usize) -> Skip<Self>;

    async fn collect<C: FromStream<Item> = List<Item>>(self) -> C;   // 슬롯이 정하지 않으면 List<Item>
    async fn for_each(self, f: fn(Item));

    // async-specific
    fn buffer_unordered(self, n: usize) -> BufferUnordered<Self>;
    fn buffered(self, n: usize) -> Buffered<Self>;
    fn throttle(self, dur: Duration) -> Throttle<Self>;
}
```

반환 타입은 [2.7 iter](207-iter.md) 의 어댑터처럼 이름 있는 구체 struct 이며, `rok::core::stream` 모듈에 있습니다 (`stream::Map<S, U>`, `stream::Filter<S>`, ...). `Map<Self, U>` 는 `Stream<Item = U>` 를, 나머지는 `Stream<Item = Item>` 을 구현합니다.

> [!NOTE]
> `stream::Map` / `Filter` / `Take` / `Skip` / `Zip` 은 `iter::Map` 등과 이름이 같습니다. 묵시 `use rok::core::**;` 아래에서 맨 이름 `Map` 은 모호하므로 ([refs-lang 5.5.6](../../refs-lang/005-modules/505-use.md)) 타입을 적어야 할 때는 `stream::Map` 처럼 경로를 쓰거나 명시 `use` 로 가져옵니다.

이 가운데 `buffer_unordered(n)` 와 `buffered(n)` 는 N개의 future 를 동시에 처리하는 concurrency control 수단으로 동작합니다.

## 2.8.6. `FromStream<T>` — `collect` 의 dispatch

`collect` 의 dispatch 를 담당하는 트레잇은 `FromStream<T>` 입니다. [2.7 iter](207-iter.md) 의 `FromIterator<T>` 에 대응하는 async 버전이며, 트레잇 시그니처도 같은 형태를 따르되 `from_stream` 이 await 가능한 형태로 동작합니다.

```rok
trait FromStream<T> {
    async fn from_stream<S: IntoStream<Item = T>>(stream: S) -> Self;
}

impl<T> FromStream<T> for List<T> { ... }
impl<T: Hash> FromStream<T> for Set<T> { ... }
impl<K: Hash, V> FromStream<(K, V)> for Dict<K, V> { ... }
```

`collect` 의 대상은 받는 슬롯이 정하며, 정하지 않으면 `List<Item>` 입니다.

`Stream::collect` 가 호출되면 대상 컬렉션의 `from_stream` 이 호출되며, stream 의 모든 원소가 await 후 누적되어 컬렉션을 만듭니다.

```rok
let v: List<i32> = await (0..).take(5).map(|x| async { x * 2 }).buffered(3).collect();
//   = await List::from_stream(...)
```

`List`, `Dict`, `Set` 은 `FromIterator` 와 `FromStream` 을 함께 구현하여 sync / async 양쪽에서 같은 패턴을 따릅니다.

## 2.8.7. `Channel` ↔ `Stream`

`Receiver<T>` (channel) 는 `Stream<Item = T>` 를 구현하므로, 채널 수신을 그대로 stream 처럼 다룰 수 있습니다.

```rok
let (tx, rx) = channel::<Event>();
task::spawn(async { tx.send(event1); tx.send(event2); });

for await event in rx {
    handle(event);
}
```

즉, [2.17 channel](217-channel.md) 의 receiver 는 **자동 stream** 으로 취급되어 `for await` 와 호환됩니다.

## 2.8.8. `merge` / `zip` 등

여러 stream 을 결합하는 combinator 도 제공됩니다.

```rok
fn merge<S1: Stream, S2: Stream>(a: S1, b: S2) -> Merge<S1, S2>;    // Stream<Item = S1::Item | S2::Item>
fn zip<S1: Stream, S2: Stream>(a: S1, b: S2) -> Zip<S1, S2>;        // Stream<Item = (S1::Item, S2::Item)>
fn select<S1: Stream, S2: Stream>(a: S1, b: S2) -> Select<S1, S2>;  // Stream<Item = S1::Item | S2::Item>, 먼저 준비된 쪽
```

각 함수의 의미를 정리하면, `merge` 는 두 stream 의 union 을 만들고, `zip` 은 두 stream 을 pair 로 묶으며, `select` 는 race 식으로 먼저 준비되는 쪽을 취합니다.

> [!NOTE]
> `select` 는 `task::select` 와 이름이 같습니다. 묵시 `use rok::core::**;` 아래에서 맨 이름은 모호하므로 ([refs-lang 5.5.6](../../refs-lang/005-modules/505-use.md#556-glob--재귀-glob-의-충돌)), `stream::select` 로 적거나 import 합니다.

## 2.8.8a. `AsyncStream<T>` / `TryAsyncStream<T, E>` — 비동기 스트림의 값

`async yield fn` 호출과 `async yield { .. }` 블록의 값입니다 ([refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#코루틴-함수의-결과-타입)).

```rok
struct AsyncStream<T> { /* 비공개 */ }
impl<T> Stream for AsyncStream<T> {
    type Item = T;
    fn poll_next(ref self, ref ctx: Context) -> Poll<T | none>;
}

struct TryAsyncStream<T, E> { /* 비공개 */ }    // async yield fn .. throws (E)
impl<T, E> TryAsyncStream<T, E> {
    fn poll_next(ref self, ref ctx: Context) throws (E) -> Poll<T | none>;
}
```

`AsyncStream<T>` 의 어댑터 (`map`, `filter`, `collect` 등) 는 `Stream` 의 기본 메서드입니다 (2.8.5). `TryAsyncStream` 은 `poll_next` 가 던지므로 `Stream` 을 구현하지 않으며 `try for await` 로 소비합니다. 비공개 필드는 사양으로 정하지 않습니다. 복사본은 같은 코루틴 frame 을 공유하므로 진행 위치도 공유합니다.

## 2.8.9. 모듈 매핑

각 항목이 자리 잡는 모듈은 다음 표에 정리됩니다.

| 항목                          | 모듈                            | 형태                         |
|-------------------------------|---------------------------------|------------------------------|
| `Stream`                      | `rok::core::stream::stream`      | poll-based                   |
| `IntoStream`                  | `rok::core::stream::into_stream` |                              |
| `Receiver<T>`                 | `rok::core::channel::receiver`   | `Stream<Item = T>` 구현      |
| `Stream::map`/`filter`/`take` | `rok::core::stream::adapter`     | adapter struct (`Stream` 구현) |
| `merge`/`zip`/`select`        | `rok::core::stream::combinator`  | `Merge`/`Zip`/`Select` struct |

## 2.8.10. `Stream` lang slot 여부

이 장에서는 `Stream::poll_next` 에 `#[lang(...)]` 을 부착하지 않으며, `Stream` 은 일반 트레잇으로 둡니다.

그 이유는 다음과 같습니다. 우선 `for await` desugar 가 **직접** `Stream::poll_next` 를 호출하는 형태로 컴파일러에 hard-coded 되어 있어, lang slot 을 따로 두지 않고도 인식이 가능합니다. 또한 lang slot 은 다중 dispatch 진입점에 필요한 메커니즘인데, Stream 의 자리는 단일하므로 굳이 필요하지 않습니다.
