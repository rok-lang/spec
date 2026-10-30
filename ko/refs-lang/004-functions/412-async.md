# 4.12. 비동기(`async`와 `await`)

`async`와 `await` 키워드([키워드](../001-lexical/105-keywords.md))로 비동기 계산을 정의하고 그 결과를 기다립니다. 비동기 계산은 나중에 나올 값을 약속하는 함수입니다.

## 4.12.1. 세 가지 정의 형태

비동기 정의의 문법 형태는 세 가지입니다.

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

세 형태 모두 결과는 라이브러리 타입 `AsyncFuture<T>`(`Future<Output = T>` 구현)의 값입니다. `throws (E)`이면 `TryAsyncFuture<T, E>`입니다([4.1.11](401-function-declaration.md#코루틴-함수의-결과-타입)). [제너레이터](411-generators.md)의 `yield`를 같은 세 형태로 쓰는 것과 같은 구조입니다.

## 4.12.2. `await` 연산자

`await`는 `Future<Output = T>`를 구현하는 값에 쓰는 전위 단항 연산자입니다([3.2.3](../003-expressions/302-precedence-and-associativity.md#323-우선순위와-결합성-표), prec=12). 결과는 `T`입니다. `await f() == x`는 `(await f()) == x`입니다.

```rok
await expr
```

그 Future가 완료될 때까지 실행을 멈추고, 완료되면 결과 값을 반환합니다. `await` 자체가 식이며, 그 결과 값이 연산자의 평가 결과입니다.

```rok
let data = await fetch_data();
let processed = await transform(data);
```

`await`는 `async` 함수, `async` 블록, `async` 클로저의 본체 안에서만 쓸 수 있습니다. 일반 함수 본체에서 `await`를 쓰면 컴파일 에러입니다.

## 4.12.3. 명명된 `async` 함수

`async fn`으로 비동기 함수를 선언합니다.

```rok
async fn fetch_user(id: i64) -> User {
    let response = await http_get(f"/users/{id}");
    await parse_user(response)
}
```

본체 안에서는 `await`로 다른 비동기 결과를 기다립니다. `-> R` 자리에는 완료될 때 나오는 값의 타입을 적습니다. 호출 결과는 `AsyncFuture<R>`입니다. `-> R`은 생략할 수 있습니다. 그러면 컴파일러가 [4.1.3](401-function-declaration.md#413-반환-타입-생략)처럼 `R`을 본체에서 추론합니다.

```rok
let f = fetch_user(42i64);        // f: `AsyncFuture<User>`
let u = await f;                   // 결과를 기다림
```

호출은 바로 반환하고 **Future** 객체 하나만 만듭니다. 실제 작업은 `await`에서 시작합니다. 스케줄러가 그보다 먼저 시작해도 됩니다.

## 4.12.4. 인라인 `async` 블록: 값

`async` 블록은 식 자리에서 바로 `AsyncFuture<T>`를 만듭니다.

```rok
let f = async {
    let a = await op1();
    let b = await op2();
    a + b
};

let r = await f;
```

본체는 [블록 표현식](../003-expressions/309-block-expressions.md)입니다. 짧은 비동기 작업을 이름 없이 그 자리에 적을 때 씁니다.

`async` 블록은 클로저처럼 바깥 값을 **만들 때 복사해** 캡처합니다([4.6](406-capture-semantics.md)). 그래서 블록을 만든 뒤 바깥 변수를 바꿔도 future 안의 값은 그대로입니다. future 안에서 재할당해도 바깥에는 보이지 않습니다.

```rok
let n = 1;
let f = async { n * 10 };
n = 2;
await f;                       //→ 10 (만들 때의 n)
```

## 4.12.5. `async` 클로저

`async ||`는 호출할 수 있는 비동기 클로저를 만듭니다.

```rok
let task = async || {
    let data = await fetch();
    process(data)
};

let f1 = task();                   // f1: `AsyncFuture<...>`
let f2 = task();                   // 매번 새 Future 생성
```

호출할 때마다 새 **Future**가 생깁니다. `async [ref x] || { .. }`처럼 `[ref x]` 캡처 리스트를 붙이면 바깥 스코프와 그 클로저가 만드는 모든 future가 `x`의 공유 셀을 함께 씁니다.

```rok
let hits = 0;
let bump = async [ref hits] || { hits += 1; };
await bump();
await bump();
hits;                          //→ 2
```

## 4.12.6. 세 형태의 대응

세 형태는 다음과 같이 대응합니다.

```
일반 함수    fn f() { ... }       ↔  || { ... }       ↔  { ... }
yield 형     yield fn f() { ... } ↔  yield || { ... } ↔  yield { ... }
async 형     async fn f() { ... } ↔  async || { ... } ↔  async { ... }
```

수식어 `yield`나 `async`는 같은 함수, 블록, 클로저 형태에 붙어 그 형태를 일관되게 제너레이터나 **Future**로 바꿉니다.

## 4.12.7. Future 트레잇

비동기 함수, 블록, 클로저의 결과는 표준 라이브러리의 `AsyncFuture<T>`(`Future<Output = T>` 구현)입니다. 정확한 트레잇 정의와 `then`, `map`, `join` 같은 메서드는 표준 라이브러리에서 정의합니다.

## 4.12.8. `await`의 의미론

다음 예에서 본체는 이렇게 평가됩니다.

```rok
async fn f() -> i32 {
    let a = await op1();
    let b = await op2();
    a + b
}
```

먼저 `op1()`을 호출해 **Future**를 받습니다. 이어서 `await`가 그 Future가 완료되기를 기다립니다. 완료될 때까지 현재 함수 `f`도 멈춥니다. Future가 완료되면 결과를 `a`에 바인딩하고 평가를 이어 갑니다. `op2()`와 두 번째 `await`도 같은 방식입니다. 모든 `await`가 끝나면 `a + b`를 평가해 함수의 결과로 반환합니다.

이렇게 멈추고 다시 이어 가는 동작은 [제너레이터(`yield`)](411-generators.md)와 같은 코루틴 메커니즘 위에서 돕니다.

## 4.12.9. 동시 실행: 라이브러리 함수

여러 Future를 동시에 진행하려면 표준 라이브러리 함수를 씁니다.

```rok
let rs = await task::join([task::spawn(op1()), task::spawn(op2())]);   // 모두: List<T>
let first = await task::race([task::spawn(op1()), task::spawn(op2())]); // 먼저 끝나는 것
let r = try await task::timeout(d, task::spawn(op1()));                // 시간 초과면 TimeoutError
```

`join` / `race`는 `List<Task<T>>`를 받습니다. 그 결과는 `await`로 기다립니다([stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)).

`join`, `race`, `timeout`, `select` 등은 언어 키워드가 아니라 라이브러리 함수입니다. [언어와 표준 라이브러리의 경계](../000-meta/004-language-vs-standard-libraries.md)의 원칙에 따른 결정입니다. 자세한 시그니처는 표준 라이브러리에서 정의합니다.

## 4.12.10. `await`가 가능한 위치

`await`를 쓸 수 있는 위치는 다음과 같습니다.

| 위치                       | `await` 사용                    |
|----------------------------|---------------------------------|
| `async fn` 본체            | OK                              |
| `async { ... }` 본체       | OK                              |
| `async \|...\| { ... }` 본체 | OK                              |
| 일반 `fn` 본체             | ERROR                           |
| 일반 `{ ... }` 블록        | ERROR                           |
| `yield` 함수나 블록 본체   | ERROR (단, `async yield`는 OK) |

## 4.12.11. 비동기 함수 호출의 즉시 반환

비동기 함수 호출은 바로 반환하므로 다음 패턴을 쓸 수 있습니다.

```rok
let f = fetch_data();             // 즉시 반환. 작업은 시작되었거나 스케줄에 있음
do_other_work();                   // 다른 작업 병행
let data = await f;                // 결과 필요 시점에 기다림
```

이것이 효율적인 동시성의 기본 패턴입니다. 시간이 걸리는 작업을 호출 직후가 아니라 결과가 필요한 시점에 기다립니다.

## 4.12.12. task와 관계

비동기 작업을 별도 실행 단위로 띄우려면 표준 라이브러리의 `task::spawn`을 씁니다.

```rok
let handle = task::spawn(async { compute() });       // 새 task 로 실행
let result = await handle;                     // 완료 대기
```

`task::spawn`은 future(여기서는 `async` 블록)를 스케줄러에 등록하고, `Future`를 구현하는 `Task<T>` 핸들을 반환합니다.

`task::spawn`은 라이브러리 함수이며 `async`와 직교합니다. 함수에 `async`를 붙이면 "이 함수는 비동기다"라고 표시할 뿐입니다. 실제로 별도 실행 단위로 띄우려면 `task::spawn` 같은 호출을 명시해야 합니다.

## 4.12.13. `async` 메서드

[메서드](409-method.md)에도 일반 메서드와 마찬가지로 `async`를 붙일 수 있습니다.

```rok
impl Database {
    async fn query(self, sql: str) -> ResultSet { ... }
}

let r = await db.query("SELECT ...");
```

## 4.12.14. 에러 처리

비동기 함수에서도 [에러 제어 흐름](../003-expressions/314-error-control-flow.md)의 `try`, `catch`, `throw`, `finally`를 그대로 쓸 수 있습니다. 에러를 던지는 `async` 함수의 에러는 `await` 지점에서 전달됩니다. 그래서 `await` 식을 `try`로 감쌉니다(`try await fetch()`). `throws` 시그니처와 함께 쓰는 규칙은 [`throws` 시그니처](414-throws.md)에서 다룹니다.

```rok
async fn safe_fetch() throws -> Data {
    try await fetch_or_throw() catch e: NetworkError {
        try await fetch_fallback()
    }
}
```
