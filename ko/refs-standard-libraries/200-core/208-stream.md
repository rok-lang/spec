# 2.8. stream: 비동기 반복자

`rok::core::stream` 모듈은 비동기로 한 단계씩 나아가는 트레잇 `Stream`을 정의합니다. 원소 타입은 연관 타입(associated type) `Item`입니다. [2.7 iter](207-iter.md)의 `Iterator`는 **동기**이고 `Stream`은 **비동기**입니다. 각 `next()` 호출을 **기다릴(await) 수 있다**는 점이 다릅니다.

`for await x in stream`을 어떻게 풀어 쓰는지도 이 모듈이 정합니다. 컴파일러는 이 구문을 이 장에서 정의하는 트레잇으로 디스패치합니다.

## 2.8.1. `Stream`: 비동기 단계 진행

트레잇 정의는 다음과 같습니다.

```rok
trait Stream {
    type Item;

    fn poll_next(ref self, ref ctx: Context) -> Poll<Item | none>;
}
```

이 메서드는 `Iterator::next`의 비동기 버전입니다. 반환값의 뜻은 다음과 같습니다. `Poll::Ready(v)`(`v: Item`)는 다음 원소 `v`가 준비되었다는 뜻입니다. `Poll::Ready(none)`은 스트림의 끝입니다. `Poll::Pending`은 아직 값이 준비되지 않았다는 뜻입니다. Waker가 깨우면 스트림을 다시 폴링합니다.

정리하면 의미는 `Iterator`의 `next() -> T | none`과 같습니다. 결정적인 차이는 **기다릴 수 있다**는 점입니다.

## 2.8.2. `for await` 풀어 쓰기

다음 구문을 예로 듭니다.

```rok
for await x in stream { body }
```

이 구문은 개념상 다음 코드로 풀립니다.

```rok
loop {
    match await stream.poll_next(ref CURRENT_CONTEXT) {
        v: Item => { let x = v; body; }
        none    => break,
    }
}
```

[2.7.2 for desugar](207-iter.md)의 동기 버전과 구조가 같고 `await`만 더했습니다.

## 2.8.3. `IntoStream`: 비동기 iterable

다음은 비동기로 순회할 수 있는 값(async iterable)을 나타내는 트레잇입니다.

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

`for await`는 시작할 때 `IntoStream::into_stream`을 호출하고, 그다음 `Stream::poll_next`를 반복합니다. 구체적으로는 다음과 같이 바뀝니다.

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

## 2.8.4. `async yield`: Stream 생성

스트림을 만드는 가장 직접적인 방법은 `async yield` 함수를 작성하는 것입니다.

```rok
async yield fn ticks(interval: Duration) -> Instant {
    loop {
        await task::sleep(interval);
        yield Instant::now();
    }
}
```

`-> Instant`는 원소 타입입니다. 호출 결과는 `Stream<Item = Instant>`를 구현하는 `AsyncStream<Instant>`입니다([refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#코루틴-함수의-결과-타입)). 컴파일러는 이 함수를 상태 기계로 바꿉니다([refs-lang 7.8](../../refs-lang/007-runtime/708-coroutine.md)). 이때 `yield` 자리와 `await` 자리는 모두 **중단 지점(suspend point)**입니다.

문법은 [refs-lang 4.13 stream (`async yield`)](../../refs-lang/004-functions/413-stream.md)이 정합니다.

## 2.8.5. Stream 어댑터

`Iterator`처럼 `Stream` 트레잇에도 여러 어댑터가 기본 메서드로 들어 있습니다. 그래서 `Stream`을 구현한 모든 타입(`AsyncStream<T>`, `Receiver<T>`, 사용자 타입)에 같은 어댑터가 있습니다.

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

반환 타입은 [2.7 iter](207-iter.md)의 어댑터처럼 이름 있는 구체 struct입니다. 모두 `rok::core::stream` 모듈에 있습니다(`stream::Map<S, U>`, `stream::Filter<S>` 등). `Map<Self, U>`는 `Stream<Item = U>`를 구현하고, 나머지는 `Stream<Item = Item>`을 구현합니다.

> [!NOTE]
> `stream::Map` / `Filter` / `Take` / `Skip` / `Zip`은 `iter::Map` 등과 이름이 같습니다. 묵시 `use rok::core::**;` 아래에서는 맨 이름 `Map`이 모호합니다([refs-lang 5.5.6](../../refs-lang/005-modules/505-use.md)). 타입을 적어야 할 때는 `stream::Map`처럼 경로를 쓰거나 명시 `use`로 가져옵니다.

이 가운데 `buffer_unordered(n)`과 `buffered(n)`은 동시성을 제어합니다. 둘 다 future N개를 동시에 처리합니다.

## 2.8.6. `FromStream<T>`: `collect`의 디스패치

`collect`의 디스패치는 `FromStream<T>` 트레잇이 맡습니다. [2.7 iter](207-iter.md)의 `FromIterator<T>`에 대응하는 비동기 버전입니다. 시그니처 형태도 같고, `from_stream`을 기다릴 수 있다는 점만 다릅니다.

```rok
trait FromStream<T> {
    async fn from_stream<S: IntoStream<Item = T>>(stream: S) -> Self;
}

impl<T> FromStream<T> for List<T> { ... }
impl<T: Hash> FromStream<T> for Set<T> { ... }
impl<K: Hash, V> FromStream<(K, V)> for Dict<K, V> { ... }
```

`collect`의 대상은 받는 슬롯이 정합니다. 슬롯이 정하지 않으면 `List<Item>`입니다.

`Stream::collect`를 호출하면 대상 컬렉션의 `from_stream`이 불립니다. 이 메서드는 스트림의 원소를 모두 기다려 모으고 컬렉션을 만듭니다.

```rok
let v: List<i32> = await (0..).take(5).map(|x| async { x * 2 }).buffered(3).collect();
//   = await List::from_stream(...)
```

`List`, `Dict`, `Set`은 `FromIterator`와 `FromStream`을 모두 구현합니다. 그래서 동기 코드와 비동기 코드에서 같은 패턴을 씁니다.

## 2.8.7. `Channel` ↔ `Stream`

채널의 `Receiver<T>`는 `Stream<Item = T>`를 구현합니다. 그래서 채널 수신을 스트림처럼 다룰 수 있습니다.

```rok
let (tx, rx) = channel::<Event>();
task::spawn(async { tx.send(event1); tx.send(event2); });

for await event in rx {
    handle(event);
}
```

[2.17 channel](217-channel.md)의 수신 쪽은 **저절로 스트림**이 되므로 `for await`에 바로 쓸 수 있습니다.

## 2.8.8. `merge` / `zip` 등

여러 스트림을 하나로 엮는 결합 함수(combinator)도 있습니다.

```rok
fn merge<S1: Stream, S2: Stream>(a: S1, b: S2) -> Merge<S1, S2>;    // Stream<Item = S1::Item | S2::Item>
fn zip<S1: Stream, S2: Stream>(a: S1, b: S2) -> Zip<S1, S2>;        // Stream<Item = (S1::Item, S2::Item)>
fn select<S1: Stream, S2: Stream>(a: S1, b: S2) -> Select<S1, S2>;  // Stream<Item = S1::Item | S2::Item>, 먼저 준비된 쪽
```

`merge`는 두 스트림을 하나로 합칩니다. `zip`은 두 스트림의 원소를 쌍으로 묶습니다. `select`는 두 스트림을 경쟁시켜 먼저 준비된 쪽을 가져옵니다.

> [!NOTE]
> `select`는 `task::select`와 이름이 같습니다. 묵시 `use rok::core::**;` 아래에서는 맨 이름이 모호합니다([refs-lang 5.5.6](../../refs-lang/005-modules/505-use.md#556-glob--재귀-glob의-충돌)). `stream::select`로 적거나 가져와서(import) 씁니다.

## 2.8.8a. `AsyncStream<T>` / `TryAsyncStream<T, E>`: 비동기 스트림의 값

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

`AsyncStream<T>`의 어댑터(`map`, `filter`, `collect` 등)는 `Stream`의 기본 메서드입니다(2.8.5). `TryAsyncStream`은 `poll_next`가 에러를 던지므로 `Stream`을 구현하지 않습니다. 이 값은 `try for await`로 소비합니다. 비공개 필드는 사양이 정하지 않습니다. 복사본은 같은 코루틴 프레임을 공유하므로 진행 위치도 공유합니다.

## 2.8.9. 모듈 매핑

각 항목이 있는 모듈은 다음 표와 같습니다.

| 항목                          | 모듈                            | 형태                         |
|-------------------------------|---------------------------------|------------------------------|
| `Stream`                      | `rok::core::stream::stream`      | 폴링 기반                    |
| `IntoStream`                  | `rok::core::stream::into_stream` |                              |
| `Receiver<T>`                 | `rok::core::channel::receiver`   | `Stream<Item = T>` 구현      |
| `Stream::map`/`filter`/`take` | `rok::core::stream::adapter`     | 어댑터 struct(`Stream` 구현)   |
| `merge`/`zip`/`select`        | `rok::core::stream::combinator`  | `Merge`/`Zip`/`Select` struct |

## 2.8.10. `Stream` lang slot 여부

이 장은 `Stream::poll_next`에 `#[lang(...)]`을 붙이지 않습니다. `Stream`은 일반 트레잇입니다.

이유는 두 가지입니다. 첫째, 컴파일러는 `for await`를 풀 때 `Stream::poll_next`를 **직접** 호출하도록 고정되어 있습니다. 그래서 lang slot이 없어도 인식합니다. 둘째, lang slot은 디스패치 진입점이 여럿일 때 쓰는 장치입니다. Stream의 진입점은 하나뿐이므로 필요하지 않습니다.
