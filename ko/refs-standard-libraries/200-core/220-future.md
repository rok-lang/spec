# 2.20. future: `Future` / `await` 통합

`rok::core::future` 모듈에는 비동기 메커니즘의 권위 트레잇이 있습니다. 컴파일러는 `await expr`를 풀어서 이 장에서 정의하는 `Future` 트레잇의 디스패치로 낮춥니다. lang slot 48개 가운데 `future_trait`도 이 장에서 정의합니다.

## 2.20.1. `Future`: 비동기 결과

`Future`는 비동기 계산의 결과를 나타내는 중심 트레잇입니다. 결과 타입은 연관 타입 `Output`입니다.

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

impl<T> Poll<T> {
    const fn is_ready(self) -> bool;        // Ready(_) 이면 true
    const fn is_pending(self) -> bool;      // Pending 이면 true
}
```

`poll(ctx)`의 결과는 두 가지입니다. `Ready(v)`는 결과가 준비되었다는 뜻이고 `v`를 그대로 반환합니다. `Pending`은 아직 결과가 없다는 뜻입니다. `ctx`의 waker가 깨우면 future를 다시 폴링합니다.

이 모델은 Rust의 `Future`, `rok::core::task::Poll`과 동형입니다. future는 **스택 없는 코루틴**(stackless coroutine)으로 컴파일됩니다. 컴파일러가 만든 상태 기계입니다([refs-lang 7.8](../../refs-lang/007-runtime/708-coroutine.md)).

## 2.20.2. `Context` / `Waker`

`Context`와 `Waker`는 poll과 스케줄러 사이의 통신 채널입니다.

```rok
struct Context { /* ... */ }         // 필드는 비공개

impl Context {
    const fn from_waker(waker: Waker) -> Context;   // waker 를 waker 로 갖는 context
    const fn waker(self) -> Waker;                  // Pending 을 돌려주기 전에 등록할 waker
}

struct Waker {
    // 내부: scheduler 와 통신하는 callback
}

impl Waker {
    fn wake(self);              // 자신을 다시 schedule 하라 신호
    fn clone(self) -> Waker;
}
```

`poll`이 `Pending`을 반환하면 내부적으로 Waker가 등록됩니다. 나중에 이벤트(I/O나 타이머)가 일어나면 waker가 호출되고, task는 다시 스케줄됩니다. 대부분의 코드는 `Waker`를 직접 다루지 않습니다. `await`가 이 과정을 자동으로 처리하기 때문입니다.

`waker` 필드는 라이브러리 내부 전용입니다. 손으로 쓴 `poll`은 `ctx.waker()`로 waker를 읽어 복제본을 보관해 둡니다. 진행할 수 있게 되면 `wake`를 부릅니다. `Context::from_waker`는 이미 손에 있는 waker로 context를 만듭니다. 안쪽 future를 같은 waker로 직접 폴링할 때 씁니다. `Waker` 자체는 실행기(executor)만 만듭니다.

## 2.20.3. `await expr` 풀어 쓰기

가장 흔한 형태는 다음과 같습니다.

```rok
let v = await some_future;
```

개념적으로는 다음과 같이 풀립니다.

```rok
let __f = IntoFuture::into_future(some_future);
let v = loop {
    match __f.poll(ref CURRENT_CONTEXT) {
        Ready(v)  => break v,
        Pending   => yield_to_scheduler(),
    }
};
```

컴파일러는 이 코드를 상태 기계로 바꿉니다. 각 `await` 자리는 **중단 지점**(suspend point)이 됩니다. GC 힙에 있는 코루틴 상태에는 프레임 변수만 보관됩니다.

## 2.20.4. `async fn` / `async { }` / `async || { }`

세 형태 모두 **`AsyncFuture<T>`**를 반환합니다([refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#코루틴-함수의-결과-타입)).

```rok
// (a) named function
async fn fetch(url: str) -> Response {
    let raw = await get(url);
    await parse(raw)
}
await fetch("/");       // fetch("/"): AsyncFuture<Response>

// (b) inline block (값)
let f = async { await compute() };       // AsyncFuture<T>
await f;

// (c) closure
let g = async || { await compute() };    // 호출하면 Future<Output = T> 구현 값을 돌려주는 closure
await g();
```

이 **세 형태**의 권위는 [refs-lang 4.12 async](../../refs-lang/004-functions/412-async.md)에 있습니다.

## 2.20.5. `IntoFuture`: await 가능한 값

`IntoFuture`는 await의 통일된 진입점입니다. `Future`뿐 아니라 `Thread`처럼 그 자체는 `Future`가 아닌 핸들도 이 트레잇으로 `await`할 수 있습니다.

```rok
trait IntoFuture {
    type Output;
    type IntoFuture: Future<Output = Output>;

    fn into_future(self) -> IntoFuture;
}

impl<F: Future> IntoFuture for F {               // Task<T>: Future<Output = T> 도 여기에 포함
    type Output = F::Output;
    type IntoFuture = F;
    fn into_future(self) -> F { self }
}
impl<T: Send> IntoFuture for JoinHandle<T> { type Output = T; type IntoFuture = Blocking<T>; ... }
```

`JoinHandle<T>`의 impl은 블로킹 풀(`task::spawn_blocking`, [2.18](218-task.md))에서 스레드를 기다립니다. 그래서 기다리는 task만 기다립니다. 이 impl은 그 타입이 `Future`를 구현하지 않는 동안에만 블랭킷 impl과 겹치지 않습니다. 그래서 그동안에만 합법입니다([refs-lang 2.9.5](../../refs-lang/002-types/209-impl.md#일관성coherence)).

`await expr`는 먼저 `IntoFuture::into_future(expr)`를 호출하고, 그 결과에 `Future::poll`을 반복해서 부릅니다(2.20.3). [2.8 stream](208-stream.md)의 `for await`가 `IntoStream`을 다루는 방식과 같습니다.

lang slot으로 등록하는 것은 `future_trait`뿐이고, `IntoFuture`는 일반 트레잇입니다. 컴파일러가 `await`를 풀 때 `IntoFuture::into_future`를 직접 호출합니다. 그래서 `IntoStream`과 마찬가지로 별도 lang slot이 필요하지 않습니다.

## 2.20.6. async / await의 "color of functions"

JavaScript와 Python에는 잘 알려진 **함수 색깔**(function color) 문제가 있습니다. async 함수와 sync 함수를 자유롭게 섞을 수 없다는 문제입니다. await는 async 안에서만 쓸 수 있고, sync 함수에서는 async 호출이 막힙니다.

Rok의 정책은 다음과 같습니다.

- `async fn`과 `await`가 색깔을 드러냅니다.
- `for`와 `for await`가 색깔을 드러냅니다.
- 색깔을 통일하려 하지 않습니다. JS, Python, C#, Swift가 모두 같은 모델을 따릅니다.

색깔을 나누는 이유는 **스케줄러 협력**을 보장하기 위해서입니다. "이 자리에서 중단할 수 있다"는 정보가 코드에 드러나야 컴파일러가 코루틴 상태 기계를 만들 수 있습니다.

```rok
// sync fn 안에서 await 호출: 컴파일 에러
fn sync_caller() {
    let v = await some_future;       // ERROR: async 함수 안에서만 await 가능
}

async fn async_caller() {
    let v = await some_future;       // OK
}
```

### 동기 문맥에서 future 결과 받기

sync 환경에서 future의 결과를 받으려면 `block_on`을 씁니다.

```rok
let v = task::block_on(some_future);    // 현재 thread 가 executor 에 붙어 끝까지 실행
```

`block_on`은 최상위(main이나 sync 호출자)에서만 써야 합니다. 중첩된 async 코드 안에서 `block_on`을 쓰면 교착 상태가 생길 수 있습니다.

## 2.20.7. `for await`: Stream 소비

Stream은 `for await` 형태로 소비합니다.

```rok
for await event in stream {            // Stream<Item = T> 소비 (async)
    handle(event);
}

for x in iterator {                    // Iterator<Item = T> 소비 (sync)
    use(x);
}
```

`for await`의 권위는 [2.8 stream](208-stream.md)의 `Stream` 트레잇에 있습니다.

## 2.20.8. `async yield`: Stream 생성

반대로 Stream은 `async yield`로 만듭니다.

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

`async yield`는 async와 yield를 합친 것이고, Stream을 만듭니다. 구문의 권위는 [refs-lang 4.13 stream](../../refs-lang/004-functions/413-stream.md)에 있습니다.

## 2.20.8a. `AsyncFuture<T>` / `TryAsyncFuture<T, E>`: 비동기 함수의 값

`async fn` 호출과 `async { .. }` 블록의 값입니다([refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#코루틴-함수의-결과-타입)).

```rok
struct AsyncFuture<T> { /* 비공개 */ }
impl<T> Future for AsyncFuture<T> {
    type Output = T;
    fn poll(ref self, ref ctx: Context) -> Poll<T>;   // 본체를 다음 대기 지점까지 진행
}

struct TryAsyncFuture<T, E> { /* 비공개 */ }    // async fn .. throws (E)
impl<T, E> TryAsyncFuture<T, E> {
    fn poll(ref self, ref ctx: Context) throws (E) -> Poll<T>;
}
```

`TryAsyncFuture`는 `poll`이 던지므로 `Future`를 구현하지 않습니다. `try await`로 기다립니다. 비공개 필드는 사양이 정하지 않습니다. 복사본은 같은 코루틴 프레임을 공유하므로 진행 위치도 공유합니다.

## 2.20.9. 모듈 매핑

각 항목의 모듈 위치와 lang slot / 역할은 다음과 같습니다.

| 항목             | 모듈                          | 역할                      |
|------------------|-------------------------------|---------------------------|
| `Future`         | `rok::core::future::Future`    | `#[lang("future_trait")]` |
| `Poll<T>`        | `rok::core::future::poll`      | enum Ready/Pending        |
| `Context`        | `rok::core::future::context`   | waker holder              |
| `Waker`          | `rok::core::future::waker`     | scheduler callback        |
| `IntoFuture`     | `rok::core::future::into_future` | 일반 트레잇(lang slot 없음) |
| `task::block_on` | `rok::core::task`              | sync entry point          |

## 2.20.10. 다른 언어와 비교

언어별 비동기 모델을 비교하면 다음과 같습니다.

| 측면           | Rust                    | C#                 | Swift                  | Rok                               |
|----------------|-------------------------|--------------------|------------------------|-----------------------------------|
| 모델           | poll-based              | callback (state)   | continuation           | poll-based (Rust 동형)            |
| `async fn`     | ✓                       | ✓                  | ✓                      | ✓                                 |
| `await`        | postfix `.await`        | prefix `await`     | prefix `await`         | prefix `await`                    |
| 코루틴 구현    | 컴파일러 생성           | 컴파일러 생성      | 컴파일러 생성          | 컴파일러 상태 기계                |
| Stackless      | ✓                       | ✓                  | ✓                      | ✓                                 |
| 색깔 통일      | ✗ (sync/async 분리)     | ✗                  | ✗                      | ✗ (업계 표준을 따름)              |
| await 일반화    | `Future` + 외부 라이브러리 | `Task<T>` 통일  | `Task` + AsyncSequence | `IntoFuture` 일반화              |

Rok이 다른 점은 크게 두 가지입니다.

1. **prefix `await`**: C#, Swift, JS와 같은 방식입니다. 권위는 [refs-lang 4.12](../../refs-lang/004-functions/412-async.md)에 있습니다. 결과에 이어서 메서드를 부르려면 괄호로 묶습니다(`(await x).method()`).
2. 일반 트레잇 `IntoFuture` 덕분에 Task와 Thread도 똑같이 await로 기다릴 수 있습니다. Swift의 `Task` 통일과 비슷한 발상입니다. 다만 고유 타입을 그대로 두고 트레잇으로 일반화한다는 점이 다릅니다.
