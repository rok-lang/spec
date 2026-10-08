# 3.14. 에러 제어 흐름

`try`, `catch`, `throw`, `finally` 는 예외를 발생시키고 받아 처리하는 제어 흐름이다.

```
try_expr    ::= "try" block catch_clause+ finally_clause?
              | "try" block finally_clause
catch_clause ::= "catch" identifier (":" type_expr)? block
finally_clause ::= "finally" block
throw_expr  ::= "throw" expr
```

## 3.14.1. `throw`: 예외 발생

```nok
throw err;
```

- `throw` 키워드 + 임의의 식.
- 식의 결과 값이 예외 값으로 던져진다.
- 예외가 던져지면 둘러싸는 가장 가까운 `try` 의 `catch` 까지 스택을 풀고 도달한다.

`throw` 식 자체는 divergent 이며, 그 자리에서 평가 흐름이 빠져나간다.

```nok
let x = if cond { 42 } else { throw Error("not allowed"); };
// x: i32 (else 분기 divergent)
```

## 3.14.2. `try` 와 `catch`

```nok
try {
    body
} catch e {
    handler
}
```

- `try` 키워드 + 본체 블록 + `catch` 키워드 + 바인딩 식별자 + 본체 블록.
- 본체 평가 중 예외가 발생하면 `catch` 본체로 점프하고 예외 값을 식별자에 바인딩한다.
- 본체가 정상 종료하면 `catch` 본체는 실행되지 않는다.

### 타입 매칭 `catch`

```nok
try {
    body
} catch e: TypeError {
    ...
} catch e: ValueError {
    ...
}
```

- 여러 `catch` 절을 두고 예외 값의 타입별로 처리할 수 있다.
- 위에서 아래로 순서대로 매칭.
- 어느 절도 매칭하지 않으면 예외는 그대로 다시 전파된다.

### 패턴 `catch`

```nok
try {
    body
} catch HttpError { code, msg } {
    ...
}
```

[패턴](311-patterns.md) 의 모든 패턴 형태를 `catch` 자리에서 사용할 수 있다. 위 타입 매칭도 패턴의 한 형태일 뿐이다 (`e: TypeError` 는 타입 검사 패턴).

## 3.14.3. `finally`

```nok
try {
    body
} finally {
    cleanup
}
```

또는

```nok
try {
    body
} catch e: Error {
    handler
} finally {
    cleanup
}
```

- `finally` 본체는 정상 종료, `catch` 처리, 미처리 예외 전파 등 모든 경우에 실행된다.
- 자원 해제, 락 해제, 임시 상태 복원 등에 쓴다.
- `catch` 가 예외를 다시 던지거나 `finally` 본체에서 새 예외가 발생하면, 새 예외가 우선한다.

## 3.14.4. 표현식으로서의 값

`try`, `catch`, `finally` 도 표현식이다.

```nok
let x = try {
    parse(input)            // 정상 종료 시 이 값
} catch _ {
    default                 // catch 시 이 값
};
```

- `try` 본체와 모든 `catch` 본체의 결과 타입이 다르면 [표현식 개요](README.md) 의 규칙에 따라 합 타입으로 합쳐져 결과 타입이 된다.
- `finally` 본체는 값을 만들지 않으며, `try` 결과에 영향을 주지 않는다.

## 3.14.5. `throw` 의 식별자와 값

`throw err` 의 `err` 는 임의의 표현식이며, 어떤 타입의 값이든 던질 수 있다. 표준 라이브러리에서 에러 타입 패밀리를 정의하지만, 언어 자체는 던지는 값의 타입에 제약을 두지 않는다.

표준 라이브러리의 `Error` 트레잇 (또는 그 슈퍼타입) 을 충족하는 값을 던지는 것이 관용이며, 표준 `catch` 가 이를 매칭하기에도 유리하다.

## 3.14.6. 함수 시그니처의 `throws`

함수가 예외를 던질 수 있다는 사실을 시그니처에 명시하는 형태가 있다.

```nok
fn parse(s: str) throws -> i32 { ... }              // untyped, 어떤 Error 든
fn parse(s: str) throws (TypeError) -> i32 { ... }  // 단일 타입
fn parse(s: str) throws (A | B | C) -> i32 { ... }  // 합 타입
```

- `throws` 키워드 ([키워드](../001-lexical/105-keywords.md) 에서 언급한 `throw` 와는 별개의 토큰. 정확한 키워드 등재는 함수 사양에서 다룬다).
- 시그니처에 `throws` 가 없는 함수는 예외를 던질 수 없다. 본체에서 `throw` 를 시도하면 컴파일 에러다.

`throws (T)` 의 `T` 가 합 타입이면 함수가 던질 수 있는 예외의 닫힌 집합이 된다. 이 정보는 호출자의 `catch` 작성에 도움이 된다.

자세한 함수 시그니처와 `throws` 의 의미는 [`throws` 시그니처](../004-functions/414-throws.md) 에서 정의한다. 본 문서는 제어 흐름의 형태까지만 다룬다.

## 3.14.7. 치명적 (fatal) 과 잡을 수 있는 (catchable)

표준 라이브러리는 두 종류의 에러를 구분한다.

- Fatal. `StackOverflowError`, `OutOfMemoryError`, `InternalError` 같이 일반적으로 잡을 수 없는 에러. `catch` 절이 매칭하더라도 처리하지 않고 그대로 전파한다.
- Catchable. 일반 에러. `try` 와 `catch` 로 잡을 수 있다.

이 구분의 정확한 기준은 표준 라이브러리에서 정의한다.

## 3.14.8. 평가 순서

`try`, `catch`, `finally` 의 평가는:

1. `try` 본체 평가.
2. 본체에서 예외 발생 시 매칭하는 `catch` 절을 찾아 그 본체 평가.
3. `finally` 본체가 있으면 무조건 평가.
4. `catch` 매칭이 없으면 예외를 더 바깥으로 전파.

`throw` 식이 평가될 때 그 식의 인자 평가가 먼저 끝난 뒤 예외가 던져진다.

## 3.14.9. `try` 식 단독 사용 (없음)

```
let x = try compute();         // 채택 안 함
```

Rust 나 Swift 의 `try expr` 식 (한 식의 `throws` 를 명시) 형태는 도입하지 않는다. 대신 명시적 `try { ... } catch ... { ... }` 만 사용한다. 흐름이 더 분명하고 시각적으로도 명확하다.
