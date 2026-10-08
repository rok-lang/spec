# 4.13. 스트림 (`async yield`)

`async` 와 `yield` 두 수식어를 함께 쓴 형태는 비동기 제너레이터 (여러 값을 시간 차로 내보내는 함수) 를 정의한다. 결과는 `Stream<T>` 타입.

## 4.13.1. 세 가지 정의 형태

```
stream_fn       ::= "async" "yield" fn_decl
stream_block    ::= "async" "yield" block
stream_closure  ::= "async" "yield" closure_expr
```

```nok
async yield fn name(args) -> R { ... await ...; ... yield ...; }   // 명명된
async yield { ... await ...; yield ...; }                           // 인라인 블록
async yield || { ... await ...; yield ...; }                        // 클로저 수식어
```

[제너레이터](411-generators.md) 와 [비동기](412-async.md) 의 세 형태가 합쳐져 작동한다.

## 4.13.2. 동기, 비동기, 제너레이터, 스트림의 관계

| 형태             | 결과 타입     | 의미               |
| ---------------- | ------------- | ------------------ |
| `fn`             | `R`           | 즉시 한 값         |
| `async fn`       | `Future<R>`   | 미래의 한 값       |
| `yield fn`       | `Iterator<R>` | 동기적으로 여러 값 |
| `async yield fn` | `Stream<R>`   | 시간 차로 여러 값  |

직교 두 차원 (지연 sync/async × 값 수 단일/복수) 의 모든 조합.

## 4.13.3. 본체에서 사용하는 두 키워드

`async yield` 함수 본체에서는 둘 다 쓸 수 있다.

```nok
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

- `await expr;`. Future 완료까지 일시 중단.
- `yield expr;`. 값을 산출하고 Stream 의 다음 단계로 일시 중단.

두 일시 중단 메커니즘이 한 함수 안에 공존한다. 어떤 시점에 어떤 종류로 중단되었는지는 코루틴 프레임이 추적한다.

## 4.13.4. 호출과 소비

```nok
let s = read_lines("data.txt");   // s: Stream<str>

for await line in s {              // 비동기 for
    process(line);
}
```

- 호출 자체는 즉시 _Stream_ 객체를 반환한다.
- 소비는 비동기 `for` 루프, 즉 `for await` 형태.

`for await` 는 [루프](../003-expressions/313-loop-expressions.md) 의 `for` 의 비동기 변형. 각 반복이 `await` 를 포함하며, 스트림에서 다음 값을 비동기적으로 기다린다.

## 4.13.5. `for await` 형태

```nok
for await pat in stream { body }
```

- `for await`. 비동기 반복 키워드 결합.
- 매 반복마다 `await stream.next()` 효과.
- 본체는 일반 `for` 본체와 같다.

이 형태는 `async fn`, `async {}`, `async ||` 본체 안에서만 쓸 수 있다.

## 4.13.6. `yield` 와 `await` 의 평가 흐름

```nok
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

- `await fetch(...)`. 페이지를 받기까지 기다린다.
- `yield page`. 그 페이지를 산출하고 소비자가 다음을 요청할 때까지 일시 중단한다.
- 소비자가 다음 값을 요청하면 함수가 재개되어 다음 페이지를 가져온다.

## 4.13.7. Stream 트레잇

표준 라이브러리의 `Stream<T>` 트레잇을 충족하는 객체다. Iterator 와 Future 가 결합된 의미로, 다음 값을 기다리는 메커니즘.

자세한 시그니처와 메서드 (`map`, `filter`, `take`, `merge`, `zip` 등) 는 표준 라이브러리에서 정의한다.

## 4.13.8. 라이브러리 함수와의 결합

스트림 변환과 결합은 표준 라이브러리 함수로 제공된다.

```nok
stream::merge(s1, s2)             // 두 스트림 합침
stream::zip(s1, s2)               // 짝지어 동시 진행
stream::throttle(s, delay)        // 속도 제한
```

자세한 함수와 메서드는 표준 라이브러리 사양에서 정의한다.

## 4.13.9. 무한 스트림

```nok
async yield fn ticks() -> Time {
    loop {
        yield now();
        await sleep(1.second());
    }
}

for await t in ticks() {
    if should_stop() { break; }
    print(t);
}
```

스트림은 자연스럽게 무한일 수 있다. 소비자가 `break` 하거나 라이브러리의 `take(n)` 같은 변환으로 종료한다.

## 4.13.10. 채널이나 파이프와의 비교

표준 라이브러리의 `Channel<T>` 도 시간 차로 여러 값을 옮기는 메커니즘이지만, 스트림과 다른 점이 있다.

| 측면            | `Stream<T>`               | `Channel<T>`                |
| --------------- | ------------------------- | --------------------------- |
| 정의            | `async yield` 함수의 결과 | 라이브러리 타입             |
| 생산자와 소비자 | 한 함수 안에 통합         | 분리된 두 endpoint          |
| 다대다          | 별도 결합 (merge 등)      | 자연스러움 (sender 여러 개) |

`Stream` 은 한 비동기 계산이 시간 차로 값을 만드는 모델, `Channel` 은 여러 동시 작업이 값을 주고받는 모델이다. 두 추상화는 보완적이다.

## 4.13.11. 메서드로서의 `async` `yield`

```nok
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

for await row in db.iter_rows("SELECT ...") {
    process(row);
}
```

일반 메서드와 같이 `async yield` 메서드도 자유롭게 정의할 수 있다.
