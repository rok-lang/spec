# 2.20. future — `Future` / `await` 통합

`rok::core::future` 모듈은 async 메커니즘의 권위 트레잇을 제공합니다. `await expr` 의 desugar 는 이 장에서 정의하는 `Future` 트레잇 dispatch 로 lower 됩니다. 이 장은 48 개의 lang slot 가운데 `future_trait` 를 정의하는 자리이기도 합니다.

## 2.20.1. `Future` — async 결과

`Future` 는 비동기 계산의 결과를 표현하는 핵심 트레잇이며, 결과 타입은 associated type `Output` 입니다.

```rok
#[lang("future_trait")]
trait Future {
    type Output;

    fn poll(ref self, ctx: ref Context) -> Poll<Output>;
}

enum Poll<T> {
    Ready(T),
    Pending,
}
```

`poll(ctx)` 의 동작은 두 가지 결과로 갈립니다. `Ready(v)` 가 반환되면 결과가 준비된 상태이며 `v` 가 그대로 반환되고, `Pending` 이 반환되면 아직 결과가 없는 상태이므로 `ctx` 의 waker 가 깨워 줄 때 다시 poll 됩니다.

이 모델은 Rust 의 `Future` 와 `rok::core::task::Poll` 와 동형이며, **stackless coroutine** 으로 컴파일됩니다 (LLVM coro.* intrinsic 활용).

## 2.20.2. `Context` / `Waker`

`Context` 와 `Waker` 는 poll 과 scheduler 사이의 통신 채널을 형성합니다.

```rok
struct Context {
    waker: Waker;
}

struct Waker {
    // 내부: scheduler 와 통신하는 callback
}

impl Waker {
    fn wake(self);              // 자신을 다시 schedule 하라 신호
    fn clone(self) -> Waker;
}
```

`poll` 이 `Pending` 을 반환할 때 내부적으로 Waker 가 등록되며, 이후 이벤트 (I/O 또는 timer) 가 발생하면 waker 가 호출되어 task 가 다시 schedule 됩니다. 다만 대부분의 사용자는 `Waker` 를 직접 다루지 않습니다. `await` 가 이 과정을 자동으로 처리해 주기 때문입니다.

## 2.20.3. `await expr` desugar

가장 흔한 사용 형태는 다음과 같습니다.

```rok
let v = await some_future;
```

이 표현은 개념적으로 다음과 같이 desugar 됩니다.

```rok
let __f = IntoFuture::into_future(some_future);
let v = loop {
    match __f.poll(ref CURRENT_CONTEXT) {
        Ready(v)  => break v,
        Pending   => yield_to_scheduler(),
    }
};
```

실제 lowering 은 LLVM coro.* state machine 으로 변환되며, 각 `await` 자리는 **suspend point** 가 됩니다. frame 변수만 GC heap 에 자리잡은 coroutine state 에 보관됩니다.

## 2.20.4. `async fn` / `async { }` / `async || { }`

세 가지 형태가 모두 **`Future<Output = T>` 를 구현하는 익명 타입의 값**을 반환합니다 ([refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#코루틴-함수의-결과-타입)).

```rok
// (a) named function
async fn fetch(url: str) -> Response {
    let raw = await get(url);
    await parse(raw)
}
await fetch("/");       // fetch("/"): Future<Output = Response> 구현 익명 타입

// (b) inline block (값)
let f = async { await compute() };       // Future<Output = T> 구현 익명 타입의 값
await f;

// (c) closure
let g = async || { await compute() };    // 호출하면 Future<Output = T> 구현 값을 돌려주는 closure
await g();
```

이 **3 형태** 의 권위는 [refs-lang 4.12 async](../../refs-lang/004-functions/412-async.md) 에 있습니다.

## 2.20.5. `IntoFuture` — await 가능한 값

`IntoFuture` 는 await 의 통일된 진입점입니다. `Future` 뿐 아니라 `Thread` 처럼 스스로는 `Future` 가 아닌 핸들도 이 트레잇을 통해 `await` 할 수 있습니다.

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
impl<T> IntoFuture for Thread<T>   { type Output = T; type IntoFuture = ThreadFuture<T>; ... }
```

`Thread<T>` 의 impl 은 이 타입이 `Future` 를 구현하지 않는 동안에만 blanket impl 과 겹치지 않아 합법입니다 ([refs-lang 2.9.5](../../refs-lang/002-types/209-impl.md#일관성-coherence)).

`await expr` 는 먼저 `IntoFuture::into_future(expr)` 를 호출하고, 그 결과에 `Future::poll` 을 반복합니다 (2.20.3). [2.8 stream](208-stream.md) 의 `for await` 가 `IntoStream` 을 다루는 방식과 같습니다.

`future_trait` 만 lang slot 으로 등록되며, `IntoFuture` 는 일반 트레잇으로 둡니다. 컴파일러가 `await` desugar 에서 `IntoFuture::into_future` 를 직접 호출하므로, `IntoStream` 과 마찬가지로 별도 lang slot 이 필요하지 않습니다.

## 2.20.6. async / await 의 "color of functions"

JavaScript 와 Python 에는 잘 알려진 **function color** 문제가 있습니다. async 함수와 sync 함수가 자유롭게 섞이지 못한다는 것입니다 (async 안에서만 await 가 가능하고, sync 함수에서는 async 호출이 차단됩니다).

rok 의 정책은 다음과 같습니다.

- `async fn` 과 `await` 가 색깔을 명시합니다
- `for` 와 `for await` 가 색깔을 명시합니다
- 색깔 통일은 시도하지 않습니다. JS, Python, C#, Swift 가 모두 같은 모델을 따릅니다

이렇게 색깔을 분리하는 이유는 **scheduler 협력** 을 보장하기 위함입니다. "이 자리는 suspend 가 가능하다" 는 정보가 명시되어야 컴파일러가 coroutine state machine 을 만들 수 있기 때문입니다.

```rok
// sync fn 안에서 await 호출 — 컴파일 에러
fn sync_caller() {
    let v = await some_future;       // ERROR — async 함수 안에서만 await 가능
}

async fn async_caller() {
    let v = await some_future;       // OK
}
```

### sync 환경에서 future 결과 받기

sync 환경에서 future 의 결과를 받아야 할 때는 `block_on` 을 사용합니다.

```rok
let v = task::block_on(some_future);    // task::block_on — 현재 thread block
```

`block_on` 은 최상위 (main 또는 sync caller) 에서만 사용해야 합니다. nested async 안에서 `block_on` 을 사용하면 deadlock 의 위험이 있습니다.

## 2.20.7. `for await` — Stream 소비

Stream 을 소비할 때는 `for await` 형태를 사용합니다.

```rok
for await event in stream {            // Stream<Item = T> 소비 (async)
    handle(event);
}

for x in iterator {                    // Iterator<Item = T> 소비 (sync)
    use(x);
}
```

`for await` 의 권위는 [2.8 stream](208-stream.md) 의 `Stream` 트레잇에 있습니다.

## 2.20.8. `async yield` — Stream 생성

반대로 Stream 을 생성할 때는 `async yield` 를 사용합니다.

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

`async yield` 는 async 와 yield 의 결합이며, Stream 을 생성합니다. syntax 권위는 [refs-lang 4.13 stream](../../refs-lang/004-functions/413-stream.md) 에 있습니다.

## 2.20.9. 모듈 매핑

각 항목의 모듈 위치와 lang slot / 역할은 아래와 같습니다.

| 항목             | 모듈                          | 역할                      |
|------------------|-------------------------------|---------------------------|
| `Future`         | `rok::core::future::Future`    | `#[lang("future_trait")]` |
| `Poll<T>`        | `rok::core::future::poll`      | enum Ready/Pending        |
| `Context`        | `rok::core::future::context`   | waker holder              |
| `Waker`          | `rok::core::future::waker`     | scheduler callback        |
| `IntoFuture`     | `rok::core::future::into_future` | 일반 트레잇 (lang slot X) |
| `task::block_on` | `rok::core::task`              | sync entry point          |

## 2.20.10. 다른 언어와의 비교

각 언어의 비동기 모델을 비교하면 아래 표와 같습니다.

| 측면           | Rust                    | C#                 | Swift                  | rok                               |
|----------------|-------------------------|--------------------|------------------------|-----------------------------------|
| 모델           | poll-based              | callback (state)   | continuation           | poll-based (Rust 동형)            |
| `async fn`     | ✓                       | ✓                  | ✓                      | ✓                                 |
| `await`        | postfix `.await`        | prefix `await`     | prefix `await`         | prefix `await`                    |
| coroutine 구현 | LLVM coro / 사용자 자체 | 컴파일러 generated | 컴파일러 generated     | LLVM coro                         |
| Stackless      | ✓                       | ✓                  | ✓                      | ✓                                 |
| color 통일     | ✗ (sync/async 분리)     | ✗                  | ✗                      | ✗ (산업 표준 채택)                |
| await 일반화    | `Future` + 외부 lib     | `Task<T>` 통일     | `Task` + AsyncSequence | `IntoFuture` 일반화              |

rok 의 가장 큰 차이점은 다음 두 가지입니다.

1. **prefix `await`**: C#, Swift, JS 와 같은 방식이며, 권위는 [refs-lang 4.12](../../refs-lang/004-functions/412-async.md) 에 있습니다. 결과에 체이닝하려면 괄호로 묶습니다 (`(await x).method()`).
2. `IntoFuture` 일반 트레잇을 도입하여 Task, Thread 도 동일하게 await 될 수 있습니다. Swift 의 `Task` 통일과 비슷한 발상이지만, 고유 타입을 보존한 후 trait 으로 일반화한다는 점이 다릅니다.
