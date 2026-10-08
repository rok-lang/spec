# 3.14. 에러 제어 흐름

`try`, `catch`, `throw`, `finally` 는 예외를 발생시키고 받아 처리하는 제어 흐름입니다.

```
try_expr      ::= "try" block catch_clause+ finally_clause?
                | "try" block finally_clause
catch_clause  ::= "catch" identifier (":" type_expr)? block
finally_clause ::= "finally" block
throw_expr    ::= "throw" expr
question_expr ::= expr "?"
```

## 3.14.1. `throw`: 예외 발생

```rok
throw err;
```

`throw` 키워드 뒤에 임의의 식이 따라오는 형태이고, 그 식의 결과 값이 예외 값으로 던져집니다. 예외가 던져지면 둘러싸는 가장 가까운 `try` 의 `catch` 까지 스택을 풀고 도달하게 됩니다.

`throw` 식 자체는 divergent 이기 때문에, 그 자리에서 평가 흐름이 그대로 빠져나갑니다.

```rok
let x = if cond { 42 } else { throw Error("not allowed"); };
// x: i32 (else 분기 divergent)
```

## 3.14.2. `try` 와 `catch`

```rok
try {
    body
} catch e {
    handler
}
```

`try` 키워드와 본체 블록, `catch` 키워드와 바인딩 식별자, 본체 블록이 차례로 이어지는 형태입니다. 본체를 평가하는 도중 예외가 발생하면 `catch` 본체로 점프하면서 예외 값을 식별자에 바인딩해 주고, 본체가 정상적으로 종료되면 `catch` 본체는 실행되지 않습니다.

### 타입 매칭 `catch`

```rok
try {
    body
} catch e: TypeError {
    ...
} catch e: ValueError {
    ...
}
```

여러 `catch` 절을 두면 예외 값의 타입별로 나눠서 처리할 수 있습니다. 매칭은 위에서 아래로 순서대로 진행되며, 어느 절도 매칭되지 않으면 예외는 그대로 다시 전파됩니다.

### 패턴 `catch`

```rok
try {
    body
} catch HttpError { code, msg } {
    ...
}
```

[패턴](311-patterns.md) 의 모든 패턴 형태를 `catch` 자리에서 사용할 수 있습니다. 위에서 살펴본 타입 매칭도 사실은 패턴의 한 형태일 뿐이고, `e: TypeError` 는 타입 검사 패턴에 해당합니다.

## 3.14.3. `finally`

```rok
try {
    body
} finally {
    cleanup
}
```

또는 다음과 같이 쓸 수도 있습니다.

```rok
try {
    body
} catch e: Error {
    handler
} finally {
    cleanup
}
```

`finally` 본체는 정상 종료, `catch` 처리, 미처리 예외 전파 등 어떤 경우에도 빠짐없이 실행됩니다. 그래서 자원 해제, 락 해제, 임시 상태 복원 같은 작업에 활용하기 좋습니다. 만약 `catch` 가 예외를 다시 던지거나 `finally` 본체에서 새로운 예외가 발생한다면, 새 예외가 우선하게 됩니다.

## 3.14.4. 표현식으로서의 값

`try`, `catch`, `finally` 도 모두 표현식입니다.

```rok
let x = try {
    parse(input)            // 정상 종료 시 이 값
} catch _ {
    default                 // catch 시 이 값
};
```

`try` 본체와 모든 `catch` 본체의 결과 타입이 서로 다르면 [표현식 개요](README.md) 의 규칙에 따라 합 타입으로 합쳐져 결과 타입이 정해집니다. `finally` 본체는 값을 만들지 않으며, `try` 결과에도 영향을 주지 않습니다.

## 3.14.5. `throw` 의 식별자와 값

`throw err` 의 `err` 는 임의의 표현식이고, 어떤 타입의 값이든 던질 수 있습니다. 표준 라이브러리에서 에러 타입 패밀리를 정의해 두긴 하지만, 언어 자체는 던지는 값의 타입에 따로 제약을 두지 않습니다.

표준 라이브러리의 `Error` 트레잇 (또는 그 슈퍼타입) 을 충족하는 값을 던지는 것이 관용적인 사용법이고, 표준 `catch` 가 이러한 값을 매칭하기에도 더 유리합니다.

## 3.14.6. 함수 시그니처의 `throws`

함수가 예외를 던질 수 있다는 사실을 시그니처에 명시하는 형태가 있습니다.

```rok
fn parse(s: str) throws -> i32 { ... }              // untyped, 어떤 Error 든
fn parse(s: str) throws (TypeError) -> i32 { ... }  // 단일 타입
fn parse(s: str) throws (A | B | C) -> i32 { ... }  // 합 타입
```

여기서 쓰이는 `throws` 키워드는 [키워드](../001-lexical/105-keywords.md) 에서 언급한 `throw` 와는 별개의 토큰입니다 (정확한 키워드 등재는 함수 사양에서 다룹니다). 시그니처에 `throws` 가 없는 함수는 예외를 던질 수 없으며, 본체에서 `throw` 를 시도하면 컴파일 에러가 됩니다.

`throws (T)` 의 `T` 가 합 타입인 경우, 그 함수가 던질 수 있는 예외의 닫힌 집합이 됩니다. 이러한 정보는 호출자가 `catch` 를 작성할 때 큰 도움이 됩니다.

자세한 함수 시그니처와 `throws` 의 의미는 [`throws` 시그니처](../004-functions/414-throws.md) 에서 정의하고 있습니다. 본 문서에서는 제어 흐름의 형태까지만 다룹니다.

## 3.14.7. 치명적 (fatal) 과 잡을 수 있는 (catchable)

표준 라이브러리는 두 종류의 에러를 구분해서 다룹니다.

- Fatal. `StackOverflowError`, `OutOfMemoryError`, `InternalError` 처럼 일반적으로 잡을 수 없는 에러입니다. `catch` 절이 매칭되더라도 처리되지 않고 그대로 전파됩니다.
- Catchable. 일반적인 에러로, `try` 와 `catch` 로 잡을 수 있습니다.

이 구분의 정확한 기준은 표준 라이브러리에서 정의합니다.

## 3.14.8. 평가 순서

`try`, `catch`, `finally` 의 평가는 다음 순서로 이루어집니다.

1. `try` 본체를 평가합니다.
2. 본체에서 예외가 발생하면 매칭하는 `catch` 절을 찾아 그 본체를 평가합니다.
3. `finally` 본체가 있으면 무조건 평가합니다.
4. `catch` 매칭이 없으면 예외를 더 바깥으로 전파합니다.

`throw` 식이 평가될 때는 그 식의 인자 평가가 먼저 끝난 뒤에 예외가 던져집니다.

## 3.14.9. `?` 연산자 — 에러 / `none` 조기 전파

`?` 는 fallible 식 (실패할 수 있는 식) 의 결과를 풀어서 호출자에게 조기 전파하는 후위 단항 연산자입니다.

```rok
fn read_config() -> Result<Config, ConfigError> {
    let raw = read_file("config.toml")?;        // Err → 즉시 return Err
    let parsed = parse_toml(raw)?;
    let cfg = validate(parsed)?;
    Ok(cfg)
}
```

각 `?` 자리에서 우변 식이 평가되고, 그 결과가 정상 (`Ok` / 정상 값) 이면 식의 값으로 풀리며, 실패 (`Err` / `none`) 이면 함수가 즉시 그 실패값으로 반환합니다. 후위 위치라 메서드 체인의 자연스러운 끝에 둘 수 있습니다.

### 적용 가능한 타입

`?` 는 [표준 라이브러리](../../refs-standard-libraries/200-core/214-error.md) 의 `Try` 트레잇을 구현하는 모든 타입에 적용할 수 있으며, 표준 impl 은 다음 두 가지입니다.

- `Result<T, E>` — `Err(e)` 면 호출자에게 전파, `Ok(v)` 면 `v` 로 풀림
- `T | none` — `none` 이면 호출자에게 전파, `v: T` 면 `v` 로 풀림

사용자도 자기 타입에 `Try` 와 `FromResidual` 을 구현하면 같은 자리에서 `?` 를 쓸 수 있습니다.

### desugar 규칙

```
expr?
```

위 식은 다음 형태로 풀어집니다.

```rok
match Try::branch(expr) {
    v: <Try::Output> => v,
    r: <Try::Residual> => return FromResidual::from_residual(r),
}
```

여기서 `Try` 의 `branch` 가 식의 값을 "정상 출력 (Output)" 과 "조기 종료 잔류값 (Residual)" 로 분기하고, `FromResidual::from_residual` 이 잔류값을 호출자가 기대하는 반환 타입으로 재구성합니다. `from_residual` 의 표준 구현은 [`From`](../../refs-standard-libraries/200-core/211-conversion.md) 트레잇을 통해 에러 타입 자동 변환을 지원합니다.

```rok
fn outer() -> Result<i32, OuterError> {
    let v = inner()?;       // inner: Result<i32, InnerError>
    Ok(v * 2)
    // ?  desugar:
    //   match Try::branch(inner()) {
    //       v: i32 => v,
    //       r: ResultResidual<InnerError> =>
    //           return FromResidual::from_residual(r),  // From<InnerError> for OuterError 호출
    //   }
}
```

### `throws` 함수 컨텍스트의 변형

함수 시그니처에 [`throws`](#3146-함수-시그니처의-throws) 가 적혀 있으면, `?` 의 잔류값은 `return` 대신 `throw` 로 lower 됩니다.

```rok
fn parse(s: str) throws (ParseError) -> i32 {
    let v = inner()?;       // Err(e) → throw e
    v * 2
}
```

이 변형은 [표준 라이브러리](../../refs-standard-libraries/200-core/214-error.md) 의 `Try` impl 이 함수 컨텍스트에 따라 다른 동작을 제공함으로써 표현됩니다. 본 사양은 후위 토큰의 syntax 와 desugar 골격까지를 정의하며, 트레잇 시그니처와 컨텍스트별 dispatch 의 권위는 표준 라이브러리에 있습니다.

### 적용 위치

`?` 는 식 자리 어디에든 둘 수 있습니다. 자주 등장하는 패턴은 다음과 같습니다.

```rok
let v = expr?;                 // let 우변
foo(expr?, bar);               // 함수 인자
expr1?.method(expr2?);          // 메서드 체인
using f = open(path)? { ... }  // using 자원 획득 자리
arr.iter().map(|x| step(x)?)   // closure 본체
```

closure 본체 안의 `?` 는 closure 의 반환 타입에 대해 동작합니다. 외부 함수의 반환 타입과는 무관합니다.

### 적용 불가능한 위치

`?` 는 함수 본체 (또는 closure 본체) 가 fallible 한 반환 타입을 갖지 않을 때 컴파일 에러가 됩니다.

```rok
fn no_return_err() -> i32 {
    let v = inner()?;          // ERROR — 함수가 Result 도 throws 도 아님
    v
}
```

이 검사는 `?` 가 만들어 내는 잔류값을 받을 자리가 없으면 의미가 없기 때문입니다.

## 3.14.10. `try` 식 단독 사용 (없음)

```
let x = try compute();         // 채택 안 함
```

Rust 나 Swift 의 `try expr` 식 (한 식의 `throws` 를 명시하는 형태) 은 도입하지 않습니다. 대신 명시적인 `try { ... } catch ... { ... }` 만 사용하는데, 이렇게 하는 편이 흐름이 더 분명하고 시각적으로도 명확하기 때문입니다.
