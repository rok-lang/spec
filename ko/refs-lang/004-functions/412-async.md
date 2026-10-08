# 4.12. 비동기 (`async` 와 `await`)

`async` 와 `await` 키워드 ([키워드](../001-lexical/105-keywords.md)) 는 비동기 계산 (미래의 값을 약속하는 함수) 을 정의하고 결과를 기다리는 메커니즘이다.

## 4.12.1. 세 가지 정의 형태

```
async_fn        ::= "async" fn_decl
async_block     ::= "async" block
async_closure   ::= "async" closure_expr
await_expr      ::= "await" expr
```

```nok
async fn name(args) -> R { ... await expr; ... }    // 명명된 비동기 함수
async { ... await expr; ... }                        // 인라인 블록 (값)
async || { ... await expr; ... }                     // 클로저 수식어
```

세 형태 모두 결과는 _Future<T>_ 라이브러리 타입의 인스턴스.

[제너레이터](411-generators.md) 의 `yield` 가 같은 세 형태로 쓰이는 패턴과 일관된다.

## 4.12.2. `await` 연산자

```nok
await expr
```

- _Future<T>_ 값 위에 적용되는 prefix 연산자.
- 그 Future 가 완료될 때까지 일시 중단하고, 완료되면 결과 값을 반환한다.
- `await` 자체는 expression. 결과 값이 연산자의 평가 결과다.

```nok
let data = await fetch_data();
let processed = await transform(data);
```

`await` 가 가능한 위치는 `async` 함수, `async` 블록, `async` 클로저의 본체 안. 일반 함수 본체에서 `await` 를 쓰면 컴파일 에러.

## 4.12.3. 명명된 `async` 함수

```nok
async fn fetch_user(id: i64) -> User {
    let response = await http_get(f"/users/{id}");
    await parse_user(response)
}
```

- `async fn`. 비동기 함수 선언.
- 본체 안에서 `await` 로 다른 비동기 결과를 기다린다.
- `-> R` 자리는 완료 시 산출되는 값의 타입. 함수의 실제 반환 타입은 _Future<R>_ 로 자동 wrapping 된다.

```nok
let f = fetch_user(42i64);        // f: Future<User>
let u = await f;                   // 결과를 기다림
```

호출 자체는 즉시 반환되며, _Future_ 객체만 만들어진다. 실제 작업은 `await` 시점에 시작된다 (또는 스케줄러가 미리 시작).

## 4.12.4. 인라인 `async` 블록: 값

```nok
let f = async {
    let a = await op1();
    let b = await op2();
    a + b
};

let r = await f;
```

- 식 자리에서 즉시 _Future<T>_ 값을 만든다.
- 본체는 [블록 표현식](../003-expressions/309-block-expressions.md).

이 형태는 짧은 비동기 작업을 익명 인라인으로 적기 위한 것이다.

## 4.12.5. `async` 클로저

```nok
let task = async || {
    let data = await fetch();
    process(data)
};

let f1 = task();                   // f1: Future<...>
let f2 = task();                   // 매번 새 Future 생성
```

- `async ||`. 호출 가능한 비동기 클로저.
- 호출할 때마다 새 _Future_ 가 생성된다.

## 4.12.6. 세 형태의 대응

```
일반 함수    fn f() { ... }       ↔  || { ... }       ↔  { ... }
yield 형     yield fn f() { ... } ↔  yield || { ... } ↔  yield { ... }
async 형     async fn f() { ... } ↔  async || { ... } ↔  async { ... }
```

수식어 (`yield` 나 `async`) 가 같은 함수, 블록, 클로저 형태에 붙어 제너레이터나 _Future_ 로 일관되게 변환된다.

## 4.12.7. Future 트레잇

비동기 함수, 블록, 클로저의 결과는 표준 라이브러리의 `Future<T>` 트레잇을 충족하는 객체다. 정확한 트레잇 정의와 메서드 (`then`, `map`, `join` 등) 는 표준 라이브러리에서 정의한다.

## 4.12.8. `await` 의 의미론

```nok
async fn f() -> i32 {
    let a = await op1();
    let b = await op2();
    a + b
}
```

본체 평가:

1. `op1()` 호출. _Future_ 반환.
2. `await`. 그 Future 의 완료를 기다린다. 완료될 때까지 현재 함수 (f) 도 일시 중단.
3. Future 완료 시 결과를 `a` 에 바인딩하고 평가 재개.
4. `op2()` 와 두 번째 `await` 도 같은 패턴.
5. 모든 `await` 완료 후 `a + b` 를 평가해 함수의 결과로 반환.

이 일시 중단과 재개는 [제너레이터 (`yield`)](411-generators.md) 와 같은 코루틴 메커니즘 위에서 작동한다.

## 4.12.9. 동시 실행: 라이브러리 함수

여러 Future 를 동시에 진행하려면 표준 라이브러리 함수를 쓴다.

```nok
let (a, b) = await join(op1(), op2());      // 둘 다 끝나기를 기다림
let first = await race(op1(), op2());       // 먼저 끝나는 것
let cancelled = await timeout(d, op1());    // 시간 제한
```

`join`, `race`, `timeout`, `select` 등은 언어 키워드가 아니라 라이브러리 함수다. [언어와 표준 라이브러리의 경계](../000-meta/004-language-vs-stdlib.md) 의 원칙에 따른다.

자세한 시그니처는 표준 라이브러리에서 정의.

## 4.12.10. `await` 가 가능한 위치

| 위치                         | `await` 가능?                   |
| ---------------------------- | ------------------------------- |
| `async fn` 본체              | OK                              |
| `async { ... }` 본체         | OK                              |
| `async \|...\| { ... }` 본체 | OK                              |
| 일반 `fn` 본체               | ERROR                           |
| 일반 `{ ... }` 블록          | ERROR                           |
| `yield` 함수나 블록 본체     | ERROR (단, `async yield` 는 OK) |

## 4.12.11. 비동기 함수 호출의 즉시 반환

```nok
let f = fetch_data();             // 즉시 반환 — 작업은 시작되었거나 스케줄에 있음
do_other_work();                   // 다른 작업 병행
let data = await f;                // 결과 필요 시점에 기다림
```

이 패턴은 시간이 걸리는 작업을 호출 직후가 아니라 결과가 필요한 시점에 기다리도록 한다. 효율적인 동시성의 기본 패턴이다.

## 4.12.12. task 와의 관계

비동기 작업을 별도 실행 단위로 띄우려면 표준 라이브러리의 `task::spawn` 을 쓴다.

```nok
let handle = task::spawn { compute() };       // 새 task 로 실행
let result = await handle;                     // 완료 대기
```

`task::spawn` 은 그 본체 클로저를 스케줄러에 등록하고, handle (Future 와 같은 인터페이스) 을 반환한다.

`task::spawn` 은 라이브러리 함수이며 `async` 와 직교한다. 함수에 `async` 를 붙이는 것 자체는 "이 함수가 비동기다" 를 표시할 뿐이며, 실제 별도 실행에는 `task::spawn` 같은 명시적 호출이 필요하다.

## 4.12.13. 메서드로서의 `async`

```nok
impl Database {
    async fn query(self, sql: str) -> ResultSet { ... }
}

let r = await db.query("SELECT ...");
```

[메서드](409-method.md) 의 일반 메서드와 같이, `async` 를 메서드에도 붙일 수 있다.

## 4.12.14. 에러 처리

비동기 함수도 [에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 `try`, `catch`, `throw`, `finally` 를 그대로 쓸 수 있다. `throws` 시그니처와의 결합은 [`throws` 시그니처](414-throws.md) 에서 다룬다.

```nok
async fn safe_fetch() throws -> Data {
    try {
        await fetch_or_throw()
    } catch e: NetworkError {
        await fetch_fallback()
    }
}
```
