# 4.12. 비동기 (`async` 와 `await`)

`async` 와 `await` 키워드 ([키워드](../001-lexical/105-keywords.md)) 는 비동기 계산, 즉 미래의 값을 약속하는 함수를 정의하고 그 결과를 기다리기 위한 메커니즘입니다.

## 4.12.1. 세 가지 정의 형태

비동기 정의는 다음과 같은 세 가지 문법 형태로 표현됩니다.

```
async_fn        ::= "async" fn_decl
async_block     ::= "async" block
async_closure   ::= "async" closure_expr
await_expr      ::= "await" unary_expr
```

```rok
async fn name(args) -> R { ... await expr; ... }    // 명명된 비동기 함수
async { ... await expr; ... }                        // 인라인 블록 (값)
async || { ... await expr; ... }                     // 클로저 수식어
```

세 형태 모두 결과는 라이브러리 타입 `AsyncFuture<T>` (`Future<Output = T>` 구현) 의 값이며, `throws (E)` 면 `TryAsyncFuture<T, E>` 입니다 ([4.1.11](401-function-declaration.md#코루틴-함수의-결과-타입)). 이는 [제너레이터](411-generators.md) 의 `yield` 가 같은 세 형태로 쓰이는 패턴과 일관된 구조입니다.

## 4.12.2. `await` 연산자

`await` 는 `Future<Output = T>` 를 구현하는 값 위에 적용되는 전위 단항 연산자 ([3.2.3](../003-expressions/302-precedence-and-associativity.md#323-우선순위와-결합성-표), prec=12) 이며, 결과는 `T` 입니다. `await f() == x` 는 `(await f()) == x` 입니다.

```rok
await expr
```

해당 Future 가 완료될 때까지 일시 중단하고, 완료되면 결과 값을 반환합니다. `await` 자체는 expression 이며, 결과 값이 곧 그 연산자의 평가 결과가 됩니다.

```rok
let data = await fetch_data();
let processed = await transform(data);
```

`await` 가 가능한 위치는 `async` 함수, `async` 블록, `async` 클로저의 본체 안으로 한정됩니다. 일반 함수 본체에서 `await` 를 쓰면 컴파일 에러가 발생합니다.

## 4.12.3. 명명된 `async` 함수

`async fn` 으로 비동기 함수를 선언할 수 있습니다.

```rok
async fn fetch_user(id: i64) -> User {
    let response = await http_get(f"/users/{id}");
    await parse_user(response)
}
```

본체 안에서는 `await` 로 다른 비동기 결과를 기다릴 수 있으며, `-> R` 자리는 완료 시 산출되는 값의 타입을 가리킵니다. 호출 결과는 `AsyncFuture<R>`입니다. `-> R` 은 생략할 수 있으며, 그러면 `R` 은 [4.1.3](401-function-declaration.md#413-반환-타입-생략) 처럼 본체에서 추론합니다.

```rok
let f = fetch_user(42i64);        // f: `AsyncFuture<User>`
let u = await f;                   // 결과를 기다림
```

호출 자체는 즉시 반환되며, 이 시점에 만들어지는 것은 **Future** 객체뿐입니다. 실제 작업은 `await` 시점에 시작되거나, 스케줄러가 미리 시작할 수도 있습니다.

## 4.12.4. 인라인 `async` 블록: 값

`async` 블록은 식 자리에서 즉시 `AsyncFuture<T>`을 만들어 줍니다.

```rok
let f = async {
    let a = await op1();
    let b = await op2();
    a + b
};

let r = await f;
```

본체는 [블록 표현식](../003-expressions/309-block-expressions.md) 입니다. 이 형태는 짧은 비동기 작업을 익명으로, 그리고 인라인으로 적기 위한 수단입니다.

`async` 블록은 클로저와 같이 바깥 값을 **만들 때 복사해** 캡처합니다 ([4.6](406-capture-semantics.md)). 그래서 블록을 만든 뒤 바깥 변수를 바꿔도 future 안의 값은 그대로이고, future 안에서의 재할당도 바깥에 보이지 않습니다.

```rok
let n = 1;
let f = async { n * 10 };
n = 2;
await f;                       //→ 10 — 만들 때의 n
```

## 4.12.5. `async` 클로저

`async ||` 는 호출 가능한 비동기 클로저를 만듭니다.

```rok
let task = async || {
    let data = await fetch();
    process(data)
};

let f1 = task();                   // f1: `AsyncFuture<...>`
let f2 = task();                   // 매번 새 Future 생성
```

호출할 때마다 새로운 **Future** 가 생성된다는 점이 특징입니다. `[ref x]` 캡처 리스트를 붙인 `async [ref x] || { .. }` 는 `x` 의 공유 셀을 그 클로저가 만드는 모든 future 와 바깥이 함께 씁니다.

```rok
let hits = 0;
let bump = async [ref hits] || { hits += 1; };
await bump();
await bump();
hits;                          //→ 2
```

## 4.12.6. 세 형태의 대응

세 가지 형태는 다음과 같이 서로 대응됩니다.

```
일반 함수    fn f() { ... }       ↔  || { ... }       ↔  { ... }
yield 형     yield fn f() { ... } ↔  yield || { ... } ↔  yield { ... }
async 형     async fn f() { ... } ↔  async || { ... } ↔  async { ... }
```

수식어인 `yield` 나 `async` 가 같은 함수, 블록, 클로저 형태에 붙어 제너레이터나 **Future** 로 일관되게 변환되는 구조입니다.

## 4.12.7. Future 트레잇

비동기 함수, 블록, 클로저의 결과는 표준 라이브러리의 `AsyncFuture<T>` (`Future<Output = T>` 구현) 입니다. 정확한 트레잇 정의와 `then`, `map`, `join` 같은 메서드는 표준 라이브러리에서 정의됩니다.

## 4.12.8. `await` 의 의미론

다음 예시를 통해 본체가 어떻게 평가되는지 살펴봅니다.

```rok
async fn f() -> i32 {
    let a = await op1();
    let b = await op2();
    a + b
}
```

먼저 `op1()` 을 호출하여 **Future** 를 받습니다. 이어서 `await` 가 그 Future 의 완료를 기다리며, 완료될 때까지는 현재 함수 `f` 도 함께 일시 중단됩니다. Future 가 완료되면 결과를 `a` 에 바인딩하고 평가를 재개합니다. `op2()` 와 두 번째 `await` 도 같은 패턴을 따르며, 모든 `await` 가 끝난 뒤에는 `a + b` 를 평가하여 함수의 결과로 반환합니다.

이러한 일시 중단과 재개는 [제너레이터 (`yield`)](411-generators.md) 와 동일한 코루틴 메커니즘 위에서 작동합니다.

## 4.12.9. 동시 실행: 라이브러리 함수

여러 Future 를 동시에 진행하려면 표준 라이브러리 함수를 사용합니다.

```rok
let rs = await task::join([task::spawn(op1()), task::spawn(op2())]);   // 모두 — List<T>
let first = await task::race([task::spawn(op1()), task::spawn(op2())]); // 먼저 끝나는 것
let r = try await task::timeout(d, task::spawn(op1()));                // 시간 초과면 TimeoutError
```

`join` / `race` 는 `List<Task<T>>` 를 받으며 그 결과를 `await` 합니다 ([stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)).

`join`, `race`, `timeout`, `select` 등은 언어 키워드가 아니라 라이브러리 함수이며, [언어와 표준 라이브러리의 경계](../000-meta/004-language-vs-standard-libraries.md) 의 원칙을 따릅니다. 자세한 시그니처는 표준 라이브러리에서 정의됩니다.

## 4.12.10. `await` 가 가능한 위치

`await` 가 허용되는 위치는 다음 표와 같습니다.

| 위치                       | `await` 가능?                   |
|----------------------------|---------------------------------|
| `async fn` 본체            | OK                              |
| `async { ... }` 본체       | OK                              |
| `async \|...\| { ... }` 본체 | OK                              |
| 일반 `fn` 본체             | ERROR                           |
| 일반 `{ ... }` 블록        | ERROR                           |
| `yield` 함수나 블록 본체   | ERROR (단, `async yield` 는 OK) |

## 4.12.11. 비동기 함수 호출의 즉시 반환

비동기 함수 호출은 즉시 반환되므로 다음과 같은 패턴이 가능합니다.

```rok
let f = fetch_data();             // 즉시 반환 — 작업은 시작되었거나 스케줄에 있음
do_other_work();                   // 다른 작업 병행
let data = await f;                // 결과 필요 시점에 기다림
```

이 패턴은 시간이 걸리는 작업을 호출 직후가 아니라 결과가 필요한 시점에 기다리도록 하는, 효율적인 동시성의 기본 패턴입니다.

## 4.12.12. task 와의 관계

비동기 작업을 별도 실행 단위로 띄우려면 표준 라이브러리의 `task::spawn` 을 사용합니다.

```rok
let handle = task::spawn(async { compute() });       // 새 task 로 실행
let result = await handle;                     // 완료 대기
```

`task::spawn` 은 future (여기서는 `async` 블록) 를 스케줄러에 등록하고, `Future` 를 구현하는 `Task<T>` handle 을 반환합니다.

`task::spawn` 은 라이브러리 함수이며 `async` 와 직교하는 개념입니다. 함수에 `async` 를 붙이는 것 자체는 "이 함수가 비동기다" 를 표시할 뿐이며, 실제로 별도의 실행 단위로 띄우려면 `task::spawn` 같은 명시적 호출이 필요합니다.

## 4.12.13. 메서드로서의 `async`

[메서드](409-method.md) 의 일반 메서드와 마찬가지로, `async` 를 메서드에도 붙일 수 있습니다.

```rok
impl Database {
    async fn query(self, sql: str) -> ResultSet { ... }
}

let r = await db.query("SELECT ...");
```

## 4.12.14. 에러 처리

비동기 함수에서도 [에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 `try`, `catch`, `throw`, `finally` 를 그대로 사용할 수 있습니다. throwing `async` 함수의 에러는 `await` 지점에서 전달되므로, `await` 식을 `try` 로 덮습니다 (`try await fetch()`). `throws` 시그니처와의 결합은 [`throws` 시그니처](414-throws.md) 에서 다룹니다.

```rok
async fn safe_fetch() throws -> Data {
    try await fetch_or_throw() catch e: NetworkError {
        try await fetch_fallback()
    }
}
```
