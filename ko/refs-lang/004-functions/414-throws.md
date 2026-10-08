# 4.14. `throws` 시그니처

[에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 `throw`, `try`, `catch`, `finally` 와 결합되는 함수 시그니처의 `throws` 키워드 형태를 정의한다.

## 4.14.1. 형태

```
throws_clause   ::= "throws" ("(" type_expr ")")?
fn_decl_throws  ::= "fn" identifier "(" param_list? ")" throws_clause? return_type? block
```

```nok
fn name(args) -> R                              // non-throwing
fn name(args) throws -> R                       // untyped — 임의의 Error
fn name(args) throws (E) -> R                   // typed — 단일 Error 타입
fn name(args) throws (A | B | C) -> R           // typed — 합 타입 Error
```

- `throws` 키워드 ([키워드](../001-lexical/105-keywords.md)). 매개변수 괄호 뒤, 반환 타입 화살표 앞.
- 선택적 괄호 안에 던질 수 있는 에러의 타입을 명시.

## 4.14.2. non-throwing 함수

```nok
fn add(x: i32, y: i32) -> i32 { x + y }
```

- `throws` 없는 함수.
- 본체에서 `throw` 식을 쓸 수 없다. 컴파일 에러.
- 호출자가 `try` 와 `catch` 로 감쌀 의무도 없다.

```nok
fn bad() -> i32 {
    throw Error("nope");          // ERROR — non-throwing 함수에서 throw
}
```

이 강제는 어떤 함수가 예외를 던질 수 있는지를 시그니처에 명시하도록 한다. 조용한 예외 전파를 차단한다.

## 4.14.3. untyped `throws` 단독

```nok
fn parse(s: str) throws -> i32 { ... }
```

- 어떤 타입의 예외든 던질 수 있다.
- 호출자는 `catch` 의 가지 패턴으로 매칭한다. 모든 타입을 다루려면 catch-all 또는 와일드카드.

untyped 는 공개 API, 예외 종류가 호출자에게 닫히지 않은 경우에 권장.

## 4.14.4. typed `throws` 의 단일 타입과 합 타입

```nok
fn fetch_user(id: i64) throws (NotFound) -> User { ... }
fn parse_or_load(s: str) throws (ParseError | IOError | TimeoutError) -> Data { ... }
```

- 괄호 안에 던질 수 있는 정확한 타입을 명시.
- 합 타입 (`A | B | C`) 도 그대로 쓸 수 있다.

typed 는 내부나 라이브러리 API, 임베디드, 예외 종류가 닫혀 있는 경우에 적합. 호출자가 어떤 `catch` 가지를 둬야 할지 컴파일러가 검증할 수 있다.

```nok
fn caller() throws -> Data {
    try {
        await parse_or_load(input)
    } catch e: ParseError {
        ...
    } catch e: IOError {
        ...
    } catch e: TimeoutError {
        ...
    }
}
```

## 4.14.5. `throws` 와 본체의 일치

함수의 본체에서 던지는 모든 예외가 시그니처의 `throws` 와 호환되어야 한다.

```nok
fn f() throws (A) -> i32 {
    throw A_instance;             // OK
    throw B_instance;             // ERROR — A 만 가능
}
```

- typed `throws`. 본체의 모든 `throw` 가 시그니처의 타입에 들어맞아야 한다.
- untyped `throws`. 본체의 `throw` 는 자유다.

검사는 `throw expr` 의 `expr` 타입과 호출되는 `throws` 함수들의 `throws` 타입을 모두 모아서 수행한다.

## 4.14.6. 호출자의 의무

`throws` 함수를 호출할 때 호출자는 다음 중 하나여야 한다.

- 자신도 `throws` 시그니처를 갖는다. 예외를 상위로 전파한다는 약속.
- `try` 와 `catch` 로 감싸서 잡는다.

직접 호출하면서 둘 다 안 하면 컴파일 에러.

```nok
fn caller() {                     // non-throwing
    parse(s);                     // ERROR — parse() 가 throws 인데 catch 없음
}

fn caller2() throws {             // 전파
    parse(s);                     // OK
}

fn caller3() {                    // 잡음
    try {
        parse(s);
    } catch _ {
        ...
    }
}
```

## 4.14.7. `throws` 의 자동 전파 (없음)

Rust 의 `?` 연산자 같은 자동 전파 식 단축은 nok 에 없다. `throws` 함수의 호출은 명시적 `try { ... }` 안에 있거나, 호출자 자체가 `throws` 여야 한다.

```nok
let x = parse(s);                 // 그냥 호출 — 호출자가 throws 거나 try 안에 있어야
```

이 결정은 예외 흐름이 코드에 명확히 드러나도록 한다. 자동 전파의 함정 (Rust 의 `?` 가 어느 함수까지 전파하는지의 모호함) 을 피한다.

## 4.14.8. `throws` 와 `async`, `yield` 의 결합

세 수식어 모두 같은 함수에 결합할 수 있다.

```nok
async fn fetch(url: str) throws (NetworkError) -> Data { ... }

yield fn lines(s: str) throws (ParseError) -> str { ... }

async yield fn read_paginated(url: str) throws (NetworkError) -> Page { ... }
```

`throws` 는 시그니처 위치의 키워드로, `async` 나 `yield` 와 직교적으로 결합한다.

## 4.14.9. `throws` 가 합 타입 시그니처에 미치는 영향

```nok
async fn fetch_data() throws (NetworkError) -> Data { ... }
```

위 함수의 호출 결과 타입은 `Future<Data>` 일까, `Future<Data | NetworkError>` 일까.

답: **`Future<Data>`**. 예외는 `Future` 의 결과에 반영되는 것이 아니라, `await` 시점에 던져지는 별도 메커니즘으로 전파된다.

```nok
let f = fetch_data();              // f: Future<Data>
try {
    let d = await f;              // 여기서 NetworkError 던질 수 있음
} catch e: NetworkError {
    ...
}
```

이 결정은 `Future` 가 결과와 에러 두 변형을 갖는 `enum` 이 되지 않게 한다. 깔끔한 단일 타입으로 유지된다.

`yield` 와 `async yield` 도 같은 패턴. `Iterator<T>` 와 `Stream<T>` 의 `T` 는 성공 값의 타입이고, `throws` 의 에러는 별도 메커니즘으로 전파.

## 4.14.10. 메서드의 `throws`

[메서드](409-method.md) 의 일반 메서드와 같이 `throws` 시그니처를 가질 수 있다.

```nok
impl Database {
    fn query(self, sql: str) throws (SqlError) -> ResultSet { ... }
}
```

호출 시 호출자의 의무와 전파 규칙은 자유 함수와 같다.

## 4.14.11. fatal 과 catchable 에러

[에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 fatal 에러는 어떤 `throws` 시그니처에도 들어가지 않는다. 시그니처 검사는 catchable 에러만 대상으로 한다.

fatal 에러는 `throws` 시그니처가 없는 함수도 던질 수 있고, `catch` 도 매칭하지 않는다. 프로세스 종료에 가까운 의미를 갖는다.
