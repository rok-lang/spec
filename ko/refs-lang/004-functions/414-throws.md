# 4.14. `throws` 시그니처

[에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 `throw`, `try`, `catch`, `finally` 와 결합되는 함수 시그니처의 `throws` 키워드 형태를 정의합니다.

## 4.14.1. 형태

[함수 선언](401-function-declaration.md) 의 `fn_decl` 에는 `throws_clause?` 가 매개변수 괄호 뒤이자 반환 타입 화살표 앞 위치에 결합됩니다.

```
throws_clause   ::= "throws" ("(" type_expr ")")?
```

따라서 `fn_decl` 의 전체 형태는 다음과 같이 표현됩니다.

```
fn_decl ::= visibility? fn_modifiers "fn" identifier "(" param_list? ")" throws_clause? return_type? block
```

여기서 `fn_modifiers` 는 [4.1 함수 선언](401-function-declaration.md) 에서 정의한 `unsafe? async? yield?` 입니다.

```rok
fn name(args) -> R                              // non-throwing
fn name(args) throws -> R                       // untyped — 임의의 Error
fn name(args) throws (E) -> R                   // typed — 단일 Error 타입
fn name(args) throws (A | B | C) -> R           // typed — 합 타입 Error
```

`throws` 키워드는 [키워드](../001-lexical/105-keywords.md) 항목에 정의되어 있으며, 매개변수 괄호 뒤이자 반환 타입 화살표 앞에 위치합니다. 선택적인 괄호 안에는 던질 수 있는 에러의 타입을 명시할 수 있습니다.

## 4.14.2. non-throwing 함수

```rok
fn add(x: i32, y: i32) -> i32 { x + y }
```

`throws` 가 없는 함수에서는 본체에서 `throw` 식을 사용할 수 없으며, 사용 시 컴파일 에러가 발생합니다. 호출자 또한 `try` 와 `catch` 로 감쌀 의무가 없습니다.

```rok
fn bad() -> i32 {
    throw Error("nope");          // ERROR — non-throwing 함수에서 throw
}
```

이러한 강제는 어떤 함수가 예외를 던질 수 있는지를 시그니처에 명시하도록 만들며, 그 결과 조용한 예외 전파를 차단해 줍니다.

## 4.14.3. untyped `throws` 단독

```rok
fn parse(s: str) throws -> i32 { ... }
```

이 형태에서는 어떤 타입의 예외든 던질 수 있고, 호출자는 `catch` 의 가지 패턴으로 매칭하게 됩니다. 모든 타입을 다루려면 catch-all 또는 와일드카드를 사용하면 됩니다.

untyped 형태는 공개 API 와 같이 예외 종류가 호출자에게 닫혀 있지 않은 경우에 권장됩니다.

## 4.14.4. typed `throws` 의 단일 타입과 합 타입

```rok
fn fetch_user(id: i64) throws (NotFound) -> User { ... }
fn parse_or_load(s: str) throws (ParseError | IOError | TimeoutError) -> Data { ... }
```

괄호 안에 던질 수 있는 정확한 타입을 명시할 수 있으며, 합 타입 (`A | B | C`) 도 그대로 사용할 수 있습니다.

typed 형태는 내부 또는 라이브러리 API, 임베디드 환경, 그리고 예외 종류가 닫혀 있는 경우에 적합합니다. 이 경우 호출자가 어떤 `catch` 가지를 두어야 하는지를 컴파일러가 검증해 줍니다.

```rok
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

함수의 본체에서 던지는 모든 예외는 시그니처의 `throws` 와 호환되어야 합니다.

```rok
fn f() throws (A) -> i32 {
    throw A_instance;             // OK
    throw B_instance;             // ERROR — A 만 가능
}
```

typed `throws` 의 경우 본체의 모든 `throw` 가 시그니처의 타입에 들어맞아야 하지만, untyped `throws` 의 경우 본체의 `throw` 는 자유롭게 사용할 수 있습니다.

검사는 `throw expr` 의 `expr` 타입과 호출되는 `throws` 함수들의 `throws` 타입을 모두 모아서 수행하게 됩니다.

## 4.14.6. 호출자의 의무

`throws` 함수를 호출할 때 호출자는 다음 중 하나의 조건을 만족해야 합니다. 첫째, 자신도 `throws` 시그니처를 가져 예외를 상위로 전파하겠다고 약속하거나, 둘째, `try` 와 `catch` 로 감싸서 직접 잡아야 합니다.

직접 호출하면서 두 가지 모두 하지 않으면 컴파일 에러가 발생합니다.

```rok
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

Rust 의 `?` 연산자처럼 자동으로 전파되는 식 단축은 rok 에 존재하지 않습니다. `throws` 함수의 호출은 명시적인 `try { ... }` 안에 있거나, 호출자 자체가 `throws` 여야 합니다.

```rok
let x = parse(s);                 // 그냥 호출 — 호출자가 throws 거나 try 안에 있어야
```

이 결정은 예외 흐름이 코드에 명확히 드러나도록 하기 위함이며, 자동 전파의 함정 (Rust 의 `?` 가 어느 함수까지 전파하는지의 모호함) 을 피하기 위함입니다.

## 4.14.8. `throws` 와 `async`, `yield` 의 결합

세 수식어는 모두 같은 함수에 결합할 수 있습니다.

```rok
async fn fetch(url: str) throws (NetworkError) -> Data { ... }

yield fn lines(s: str) throws (ParseError) -> str { ... }

async yield fn read_paginated(url: str) throws (NetworkError) -> Page { ... }
```

`throws` 는 시그니처 위치의 키워드로서, `async` 또는 `yield` 와 직교적으로 결합됩니다.

## 4.14.9. `throws` 가 합 타입 시그니처에 미치는 영향

```rok
async fn fetch_data() throws (NetworkError) -> Data { ... }
```

위 함수의 호출 결과 타입이 `Future<Data>` 인지 아니면 `Future<Data | NetworkError>` 인지 궁금할 수 있습니다.

답은 **`Future<Data>`** 입니다. 예외는 `Future` 의 결과에 반영되는 것이 아니라, `await` 시점에 던져지는 별도의 메커니즘을 통해 전파되기 때문입니다.

```rok
let f = fetch_data();              // f: Future<Data>
try {
    let d = await f;              // 여기서 NetworkError 던질 수 있음
} catch e: NetworkError {
    ...
}
```

이 결정 덕분에 `Future` 가 결과와 에러라는 두 변형을 갖는 `enum` 으로 변하지 않으며, 깔끔한 단일 타입으로 유지될 수 있습니다.

`yield` 와 `async yield` 도 동일한 패턴을 따릅니다. `Iterator<T>` 와 `Stream<T>` 의 `T` 는 성공 값의 타입을 나타내고, `throws` 의 에러는 별도의 메커니즘으로 전파됩니다.

## 4.14.10. 메서드의 `throws`

[메서드](409-method.md) 의 일반 메서드와 마찬가지로 `throws` 시그니처를 가질 수 있습니다.

```rok
impl Database {
    fn query(self, sql: str) throws (SqlError) -> ResultSet { ... }
}
```

호출 시 호출자의 의무와 전파 규칙은 자유 함수와 동일하게 적용됩니다.

## 4.14.11. fatal 과 catchable 에러

[에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 fatal 에러는 어떤 `throws` 시그니처에도 들어가지 않습니다. 시그니처 검사는 catchable 에러만을 대상으로 수행되기 때문입니다.

fatal 에러는 `throws` 시그니처가 없는 함수도 던질 수 있으며, `catch` 도 매칭하지 않습니다. 이는 프로세스 종료에 가까운 의미를 지닙니다.
