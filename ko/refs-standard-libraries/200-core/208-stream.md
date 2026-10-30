# 2.8. stream - async iterator

`rok::core::stream` 모듈은 async 단계 진행 트레잇 `Stream` (원소 타입은 associated type `Item`)을 정의합니다. [2.7 iter](207-iter.md)의 `Iterator`는 **sync**이고 `Stream`은 **async**입니다. 각 `next()` 호출을 **await할 수 있다**는 점이 다릅니다.

이 모듈은 `for await x in stream`의 desugar 권위이기도 합니다. 이 구문은 이 장에서 정의하는 트레잇으로 dispatch됩니다.

## 2.8.1. `Stream` - async 단계 진행

트레잇 정의는 다음과 같습니다.

```rok
trait Stream {
    type Item;

    fn poll_next(ref self, ref ctx: Context) -> Poll<Item | none>;
}
```

이 메서드는 `Iterator::next`의 async 버전입니다. 반환값의 의미는 다음과 같습니다. `Poll::Ready(v)` (`v: Item`)는 다음 원소 `v`가 준비되었다는 뜻이고, `Poll::Ready(none)`은 stream의 끝입니다. `Poll::Pending`은 아직 값이 준비되지 않았다는 뜻이며, Waker가 깨우면 stream을 다시 poll합니다.

정리하면 `Iterator`의 `next() -> T | none`과 의미는 같고, **await할 수 있다**는 점이 결정적인 차이입니다.

## 2.8.2. `for await` desugar

다음 구문을 예로 듭니다.

```rok
for await x in stream { body }
```

이 구문은 개념적으로 다음과 같이 desugar됩니다.

```rok
loop {
    match await stream.poll_next(ref CURRENT_CONTEXT) {
        v: Item => { let x = v; body; }
        none    => break,
    }
}
```

이 구조는 [2.7.2 for desugar](207-iter.md)의 sync 버전과 같은 구조에 `await`만 더한 형태입니다.

## 2.8.3. `IntoStream` - async iterable

다음은 async iterable을 나타내는 트레잇입니다.

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

`for await`의 시작 자리에서는 `IntoStream::into_stream`을 호출한 뒤 `Stream::poll_next`를 반복합니다. 구체적인 변환 형태는 다음과 같습니다.

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

## 2.8.4. `async yield` - Stream 생성

Stream을 직접 만드는 가장 자연스러운 방법은 `async yield` 함수를 작성하는 것입니다.

```rok
async yield fn ticks(interval: Duration) -> Instant {
    loop {
        await task::sleep(interval);
        yield Instant::now();
    }
}
```

`-> Instant`는 원소 타입이며, 호출 결과는 `AsyncStream<Instant>` (`Stream<Item = Instant>` 구현)입니다 ([refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#코루틴-함수의-결과-타입)). 컴파일러는 이 함수를 상태 기계로 바꿉니다 ([refs-lang 7.8](../../refs-lang/007-runtime/708-coroutine.md)). 이때 `yield` 자리와 `await` 자리는 모두 **suspend point**입니다.

자세한 syntax 권위는 [refs-lang 4.13 stream (`async yield`)](../../refs-lang/004-functions/413-stream.md)을 따릅니다.

## 2.8.5. Stream 어댑터

`Iterator`처럼 여러 어댑터가 `Stream` 트레잇의 기본 메서드로 들어 있습니다. 그래서 `Stream`을 구현한 모든 타입 (`AsyncStream<T>`, `Receiver<T>`, 사용자 타입)이 같은 어댑터를 가집니다.

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
    fn buffer_unordered(self, n: usize) -> BufferUnordered<Self> where Item: Future;
    fn buffered(self, n: usize) -> Buffered<Self> where Item: Future;
    fn throttle(self, dur: Duration) -> Throttle<Self>;
}
```

반환 타입은 [2.7 iter](207-iter.md)의 어댑터처럼 이름 있는 구체 struct이며, `rok::core::stream` 모듈에 있습니다 (`stream::Map<S, U>`, `stream::Filter<S>`, ...). `Map<Self, U>`는 `Stream<Item = U>`를, `Buffered<Self>` / `BufferUnordered<Self>`는 `Stream<Item = Item::Output>`을 구현하며 (future 자체가 아니라 future의 결과를 내보냄), 나머지는 `Stream<Item = Item>`을 구현합니다.

> [!NOTE]
> `stream::Map` / `Filter` / `Take` / `Skip` / `Zip`은 `iter::Map` 등과 이름이 같습니다. 묵시 `use rok::core::**;` 아래에서 맨 이름 `Map`은 모호하므로 ([refs-lang 5.5.6](../../refs-lang/005-modules/505-use.md)) 타입을 적어야 할 때는 `stream::Map`처럼 경로를 쓰거나 명시 `use`로 가져옵니다.

이 가운데 `buffer_unordered(n)`와 `buffered(n)`는 N개의 future를 동시에 처리하는 concurrency control 수단입니다.

## 2.8.6. `FromStream<T>` - `collect` 의 dispatch

`collect`의 dispatch는 `FromStream<T>` 트레잇이 맡습니다. [2.7 iter](207-iter.md)의 `FromIterator<T>`에 대응하는 async 버전이며, 시그니처도 같은 형태이고 `from_stream`만 await할 수 있습니다.

```rok
trait FromStream<T> {
    async fn from_stream<S: IntoStream<Item = T>>(stream: S) -> Self;
}

impl<T> FromStream<T> for List<T> { ... }
impl<T: Hash> FromStream<T> for Set<T> { ... }
impl<K: Hash, V> FromStream<(K, V)> for Dict<K, V> { ... }
```

`collect`의 대상은 받는 슬롯이 정하며, 정하지 않으면 `List<Item>`입니다.

`Stream::collect`를 호출하면 대상 컬렉션의 `from_stream`이 호출되고, stream의 모든 원소를 await해 모아 컬렉션을 만듭니다.

```rok
let v: List<i32> = await (0..).take(5).map(|x| async { x * 2 }).buffered(3).collect();
//   = await List::from_stream(...)
```

`List`, `Dict`, `Set`은 `FromIterator`와 `FromStream`을 함께 구현하므로 sync와 async에서 같은 패턴을 씁니다.

## 2.8.7. `Channel` ↔ `Stream`

`Receiver<T>` (channel)는 `Stream<Item = T>`를 구현하므로, 채널 수신을 그대로 stream처럼 다룰 수 있습니다.

```rok
let (tx, rx) = channel::<Event>();
task::spawn(async { tx.send(event1); tx.send(event2); });

for await event in rx {
    handle(event);
}
```

[2.17 channel](217-channel.md)의 receiver는 **자동 stream**으로 취급되므로 `for await`에 바로 쓸 수 있습니다.

## 2.8.8. `merge` / `zip` 등

여러 stream을 결합하는 combinator도 있습니다.

```rok
fn merge<S1: Stream, S2: Stream>(a: S1, b: S2) -> Merge<S1, S2>;    // Stream<Item = S1::Item | S2::Item>
fn zip<S1: Stream, S2: Stream>(a: S1, b: S2) -> Zip<S1, S2>;        // Stream<Item = (S1::Item, S2::Item)>
fn select<S1: Stream, S2: Stream>(a: S1, b: S2) -> Select<S1, S2>;  // Stream<Item = S1::Item | S2::Item>, 먼저 준비된 쪽
```

`merge`는 두 stream의 union을 만들고, `zip`은 두 stream을 pair로 묶으며, `select`는 두 stream을 경쟁시켜 먼저 준비되는 쪽을 취합니다.

> [!NOTE]
> `select`는 `task::select`와 이름이 같습니다. 묵시 `use rok::core::**;` 아래에서 맨 이름은 모호하므로 ([refs-lang 5.5.6](../../refs-lang/005-modules/505-use.md#556-glob--재귀-glob-의-충돌)), `stream::select`로 적거나 import합니다.

## 2.8.8a. `AsyncStream<T>` / `TryAsyncStream<T, E>` - 비동기 스트림의 값

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

`AsyncStream<T>`의 어댑터 (`map`, `filter`, `collect` 등)는 `Stream`의 기본 메서드입니다 (2.8.5). `TryAsyncStream`은 `poll_next`가 던지므로 `Stream`을 구현하지 않으며 `try for await`로 소비합니다. 비공개 필드는 사양으로 정하지 않습니다. 복사본은 같은 코루틴 frame을 공유하므로 진행 위치도 공유합니다.

## 2.8.9. 모듈 매핑

모든 항목은 `rok::core::stream` (`Receiver<T>`는 `rok::core::channel`)에 정의되며, 아래 하위 모듈은 그것을 더 좁은 경로로 다시 내보냅니다.

| 항목                          | 모듈                            | 형태                         |
|-------------------------------|---------------------------------|------------------------------|
| `Stream`                      | `rok::core::stream::stream`      | poll-based                   |
| `IntoStream`                  | `rok::core::stream::into_stream` |                              |
| `Receiver<T>`                 | `rok::core::channel::receiver`   | `Stream<Item = T>` 구현      |
| `Map`/`Filter`/`Take`/`Skip`/`Throttle`/`Buffered`/`BufferUnordered` | `rok::core::stream::adapter` | adapter struct (`Stream` 구현) |
| `merge`/`zip`/`select`        | `rok::core::stream::combinator`  | `Merge`/`Zip`/`Select` struct |

## 2.8.10. `Stream` lang slot 여부

이 장에서는 `Stream::poll_next`에 `#[lang(...)]`을 붙이지 않으며, `Stream`은 일반 트레잇으로 둡니다.

이유는 두 가지입니다. 첫째, `for await` desugar는 `Stream::poll_next`를 **직접** 호출하도록 컴파일러에 hard-coded되어 있어 lang slot 없이도 인식됩니다. 둘째, lang slot은 dispatch 진입점이 여럿일 때 필요한 메커니즘인데, Stream의 진입점은 하나뿐이므로 필요하지 않습니다.
