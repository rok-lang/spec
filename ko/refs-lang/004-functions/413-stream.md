# 4.13. 스트림 (`async yield`)

`async`와 `yield` 두 수식어를 함께 쓰면 비동기 제너레이터, 곧 여러 값을 시간 차를 두고 내보내는 함수를 정의합니다. 결과는 라이브러리 타입 `AsyncStream<T>`(`Stream<Item = T>` 구현)의 값이며, `throws (E)`이면 `TryAsyncStream<T, E>`입니다 ([4.1.11](401-function-declaration.md#코루틴-함수의-결과-타입)).

## 4.13.1. 세 가지 정의 형태

스트림을 정의하는 문법은 다음 세 가지입니다.

```
stream_fn       ::= "async" "yield" fn_decl
stream_block    ::= "async" "yield" block
stream_closure  ::= "async" "yield" closure_expr
```

```rok
async yield fn name(args) -> R { ... await ...; ... yield ...; }   // 명명된
async yield { ... await ...; yield ...; }                           // 인라인 블록
async yield || { ... await ...; yield ...; }                        // 클로저 수식어
```

[제너레이터](411-generators.md)와 [비동기](412-async.md)의 세 형태를 합친 모습입니다.

## 4.13.2. 동기, 비동기, 제너레이터, 스트림의 관계

네 형태의 관계는 다음과 같습니다.

| 형태             | 호출 결과                           | 의미               |
|------------------|-------------------------------------|--------------------|
| `fn`             | `R`                                 | 즉시 한 값         |
| `async fn`       | `AsyncFuture<R>` | 미래의 한 값       |
| `yield fn`       | `GenIter<R>` | 동기적으로 여러 값 |
| `async yield fn` | `AsyncStream<R>`   | 시간 차로 여러 값  |

직교하는 두 차원, 곧 지연 여부(sync/async)와 값의 수(단일/복수)의 모든 조합입니다.

## 4.13.3. 본체에서 사용하는 두 키워드

`async yield` 함수의 본체에서는 `await`와 `yield`를 둘 다 쓸 수 있습니다.

```rok
async yield fn read_lines(file: str) -> str {
    let f = await open(file);
    loop {
        let line = await f.read_line();
        match line {
            l: str => yield l,
            none   => break,
        }
    }
}
```

`await expr;`는 Future가 완료될 때까지 함수를 일시 중단하고, `yield expr;`는 값을 산출한 뒤 Stream의 다음 단계로 진행하기 전까지 일시 중단합니다. 한 함수 안에 일시 중단 메커니즘 두 가지가 함께 있으며, 어느 시점에 어떤 종류로 중단되었는지는 코루틴 프레임이 추적합니다.

## 4.13.4. 호출과 소비

스트림을 호출하고 소비하는 코드는 다음과 같습니다.

```rok
async fn run() {
    let s = read_lines("data.txt");   // s: `AsyncStream<str>`

    for await line in s {              // 비동기 for
        process(line);
    }
}
```

호출 자체는 바로 **Stream** 객체를 반환하며, 실제 소비는 비동기 `for` 루프, 곧 `for await` 형태로 합니다.

`for await`는 [루프](../003-expressions/313-loop-expressions.md)의 `for`의 비동기 변형입니다. 반복마다 `await`가 들어 있어 스트림의 다음 값을 비동기로 기다립니다.

## 4.13.5. `for await` 형태

`for await`의 기본 형태는 다음과 같습니다.

```rok
for await pat in stream { body }
```

`for await`는 비동기 반복을 위한 키워드 조합입니다. 반복마다 `await stream.next()`와 같은 효과가 있고, 본체는 일반 `for` 본체와 똑같이 씁니다.

이 형태는 `async fn`, `async {}`, `async ||` 본체 안에서만 쓸 수 있습니다.

## 4.13.6. `yield` 와 `await` 의 평가 흐름

두 키워드가 함께 동작하는 흐름은 다음 예와 같습니다.

```rok
async yield fn paginate() -> Page {
    let next = "/start";
    loop {
        let page = await fetch(next);
        yield page;
        match page.next_url {
            url: str => next = url,
            none     => break,
        }
    }
}
```

먼저 `await fetch(...)`가 페이지를 받을 때까지 기다립니다. 이어서 `yield page`가 그 페이지를 산출하고, 소비자가 다음 값을 요청할 때까지 일시 중단됩니다. 소비자가 다음 값을 요청하면 함수가 재개되어 다음 페이지를 가져옵니다.

## 4.13.7. Stream 트레잇

결과 객체는 표준 라이브러리의 `AsyncStream<T>`(`Stream<Item = T>` 구현)입니다. 의미로 보면 Iterator와 Future를 합친 형태이며, 다음 값을 기다리는 메커니즘입니다.

자세한 시그니처와 메서드(`map`, `filter`, `take`, `merge`, `zip` 등)는 표준 라이브러리에서 정의합니다.

## 4.13.8. 라이브러리 함수와의 결합

스트림의 변환과 결합은 표준 라이브러리 함수가 제공합니다.

```rok
stream::merge(s1, s2)             // 두 스트림 합침
stream::zip(s1, s2)               // 짝지어 동시 진행
stream::throttle(s, delay)        // 속도 제한
```

자세한 함수와 메서드는 표준 라이브러리 사양에서 정의합니다.

## 4.13.9. 무한 스트림

스트림은 다음처럼 끝없이 이어질 수도 있습니다.

```rok
async yield fn ticks() -> Time {
    loop {
        yield now();
        await sleep(1.second());
    }
}

async fn run() {
    for await t in ticks() {
        if should_stop() { break; }
        print(t);
    }
}
```

이런 무한 스트림은 소비자가 `break`로 빠져나오거나 라이브러리의 `take(n)` 같은 변환으로 끝낼 수 있습니다.

## 4.13.10. 채널이나 파이프와의 비교

표준 라이브러리의 `Channel<T>`도 시간 차를 두고 여러 값을 옮기는 메커니즘이지만, 스트림과 다음과 같은 차이가 있습니다.

| 측면            | `Stream<Item = T>`        | `Channel<T>`                |
|-----------------|---------------------------|-----------------------------|
| 정의            | `async yield` 함수의 결과 | 라이브러리 타입             |
| 생산자와 소비자 | 한 함수 안에 통합         | 분리된 두 endpoint          |
| 다대다          | 별도 결합 (merge 등)      | 자연스러움 (sender 여러 개) |

`Stream`은 비동기 계산 하나가 시간 차를 두고 값을 만들어 내는 모델이고, `Channel`은 여러 동시 작업이 값을 주고받는 모델입니다. 두 추상화는 서로를 보완합니다.

## 4.13.11. 메서드로서의 `async` `yield`

`async yield`는 메서드로도 정의할 수 있습니다.

```rok
impl Database {
    async yield fn iter_rows(self, sql: str) -> Row {
        let cursor = await self.query(sql);
        loop {
            let row = await cursor.next();
            match row {
                r: Row => yield r,
                none   => break,
            }
        }
    }
}

async fn run(db: Db) {
    for await row in db.iter_rows("SELECT ...") {
        process(row);
    }
}
```

일반 메서드와 마찬가지로 `async yield` 메서드도 정의해서 쓸 수 있습니다.
