# 3.14. 에러 제어 흐름

`throw`, `try`, `catch`, `finally` 는 에러를 발생시키고, 에러가 흐를 수 있는 지점을 표시하고, 받아 처리하는 제어 흐름입니다. 후위 `?` 는 `T | none` 값의 `none` 을 호출자에게 조기 반환합니다.

```
throw_stmt     ::= "throw" expr ";"
try_expr       ::= ("try" | "try?" | "try!") expr
catch_expr     ::= expr catch_clause+ finally_clause?
                 | expr finally_clause
catch_clause   ::= "catch" pattern? block
finally_clause ::= "finally" block
question_expr  ::= expr "?"
```

`catch` 와 `finally` 는 식이나 블록 뒤의 후위 절 자리에서만 키워드인 문맥 키워드입니다 ([1.5.4](../001-lexical/105-keywords.md#154-문맥-키워드)). 에러를 던질 수 있는 함수의 시그니처 (`throws`, `rethrows`) 는 [`throws` 시그니처](../004-functions/414-throws.md) 에서 정의합니다. 본 문서는 식과 문장 수준의 제어 흐름을 다룹니다.

## 3.14.1. `throw`: 에러 발생

```rok
throw err;
```

`throw` 키워드 뒤에 임의의 식이 따라오는 문장이며, 그 식의 결과 값이 에러 값으로 던져집니다. 던져진 에러는 그 문장을 둘러싼 가장 가까운 `catch` 식이 처리하고, 처리되지 않으면 둘러싼 함수의 에러 결과가 됩니다. 이때 둘러싼 함수의 `throws` 시그니처가 그 에러 타입을 받아야 합니다 ([4.14.5](../004-functions/414-throws.md#4145-throws-와-본체의-일치)).

`throw` 문 뒤로는 흐름이 이어지지 않으므로, `throw` 문으로 끝나는 블록은 divergent 입니다.

```rok
let x = if cond { 42i32 } else { throw ValueError { msg: "not allowed" }; };
// x: i32 (else 분기 divergent)
```

## 3.14.2. `try`: 호출 지점 표시

```rok
let n = try parse(s);
```

`try` 는 접두 식 연산자입니다. throwing 함수 (`throws` 시그니처를 가진 함수) 의 호출은 반드시 `try` 가 덮는 식 안에 있어야 하며, 그렇지 않으면 컴파일 에러가 발생합니다. 이 규칙 덕분에 에러가 흐를 수 있는 지점이 코드에 모두 드러납니다.

`try e` 의 값과 타입은 `e` 의 값과 타입 그대로입니다. `e` 안의 throwing 호출이 에러를 내면, 그 에러는 가장 가까운 `catch` 가 처리하며, `catch` 가 없거나 매칭되지 않으면 둘러싼 함수로 전파됩니다.

### 덮는 범위

`try` 는 그 오른쪽 식 전체를 덮습니다. 하나의 `try` 로 여러 호출을 함께 표시할 수 있습니다.

```rok
try a(b());                 // a, b 모두 덮음
try f().g().h();            // 체인 전체를 덮음
try {                       // 블록도 식이므로 블록 안의 호출을 모두 덮음
    let cfg = load(path);
    apply(cfg);
};
```

`try` 의 피연산자는 오른쪽으로 식이 끝나는 곳까지 이어지며, `catch` / `finally` 절, 문장 종결자 `;`, 닫는 괄호, 콤마 앞에서 멈춥니다. 따라서 `try a + b()` 는 `try (a + b())` 이고, `try parse(s) catch _ { 0 }` 의 `catch` 는 `try parse(s)` 전체에 붙습니다. 정확한 결합 규칙은 [우선순위와 결합성](302-precedence-and-associativity.md) 에서 정의합니다.

`try` 가 이항 연산자의 오른쪽 피연산자나 함수 인자 자리에 오면 그 피연산자만 덮습니다.

```rok
let total = base + try fetch_extra();
log(try describe(item));
```

### `await` 와 함께

`async` throwing 함수의 에러는 `await` 지점에서 전달되므로, `await` 식을 `try` 로 덮습니다 ([4.14.10](../004-functions/414-throws.md#41410-throws-가-합-타입-시그니처에-미치는-영향)). 생성자의 에러도 같은 방식으로 `next` 지점에서 받습니다.

```rok
let data = try await fetch(url);
```

## 3.14.3. `try?` 와 `try!`

`try` 의 두 변형은 에러를 둘러싼 함수로 전파하지 않고 그 자리에서 다른 형태로 바꿉니다.

| 형태     | `e: T` 일 때의 결과 타입 | 에러가 나면                   |
|----------|--------------------------|-------------------------------|
| `try e`  | `T`                      | 가장 가까운 `catch` 또는 전파 |
| `try? e` | `T \| none`              | 에러를 버리고 `none`          |
| `try! e` | `T`                      | panic (3.14.8)                |

```rok
let port = (try? env::var("PORT")) ?? "8080";
let cfg = try! load_builtin_config();      // 실패하면 프로그램 버그
```

`try?` 와 `try!` 는 에러를 전파하지 않으므로, 둘러싼 함수가 `throws` 일 필요가 없습니다. 덮는 범위는 `try` 와 같습니다. `try?` 의 결과에 `??` 를 이어 쓰려면 위 예처럼 괄호로 범위를 닫습니다.

## 3.14.4. `catch`: 후위 처리 식

```rok
let n = try parse(s) catch e: ParseError { 0 } catch _ { -1 };
```

`catch` 절은 왼쪽 식에 붙는 후위 절입니다. 왼쪽 식을 평가하는 도중 에러가 나오면 `catch` 절을 위에서 아래로 검사하고, 처음 매칭되는 절의 블록을 평가합니다. 왼쪽 식이 정상적으로 끝나면 `catch` 블록은 실행되지 않습니다. 왼쪽 식에서 나오는 에러는 `try` 로 표시한 throwing 호출과 `throw` 문이 만든 것입니다.

어느 절도 매칭되지 않으면 에러는 다시 전파되며, 이때 둘러싼 함수가 그 에러 타입을 `throws` 해야 합니다.

### 블록 형태

`try` 가 블록을 덮고 `catch` / `finally` 가 그 식에 붙는 형태도 같은 규칙으로 해석됩니다.

```rok
try {
    body
} catch e {
    handler
}
```

### 패턴과 타입 매칭

```rok
try fetch(url)
catch e: TimeoutError { retry() }
catch HttpError { code, msg } { fail(code, msg) }
catch e: any Error { log(f"{e}"); fallback() }
```

`catch` 뒤에는 [패턴](311-patterns.md) 의 모든 패턴 형태를 쓸 수 있습니다. `e: TimeoutError` 는 타입 검사 패턴이며, 매칭된 분기 안에서 `e` 는 그 타입으로 좁혀집니다. `catch e: A | B` 는 잡은 에러를 그 합 타입 (에러 합 타입의 부분) `A | B` 로 바인딩합니다. untyped `throws` 의 에러는 [`any Error`](../002-types/202-any-slot.md) 값이므로, 모든 에러를 한 이름으로 받으려면 `catch e: any Error` 또는 `catch e` 를 씁니다. 트레잇 이름을 그대로 쓴 `catch e: Error` 는 [트레잇-as-타입 금지](../002-types/212-trait-as-type-banned.md) 에 따라 컴파일 에러입니다.

패턴을 생략한 `catch { ... }` 는 `catch _ { ... }` 와 같습니다.

### 완전성

왼쪽 식이 던질 수 있는 에러 타입을 모두 덮는 `catch` 절이 있으면, 그 `catch` 식은 더 이상 에러를 던지지 않는 식이 됩니다. typed `throws (A | B)` 의 호출이라면 `A` 와 `B` 를 모두 덮으면 되고, untyped `throws` 라면 `catch e`, `catch _`, `catch e: any Error` 처럼 모든 값을 받는 절이 있어야 합니다.

```rok
fn parse_or_zero(s: str) -> i32 {          // non-throwing
    try parse(s) catch _ { 0 }             // 모든 에러를 덮음
}
```

### 도달하지 않는 `try` 와 `catch` 경고

다음 경우 컴파일러는 경고를 냅니다.

- throwing 호출을 하나도 덮지 않는 `try`
- 어떤 에러도 도달하지 않는 `catch` 절이나 분기

경고가 나는 절도 타입 검사는 그대로 받습니다.

## 3.14.5. `finally`

```rok
try {
    body
} finally {
    cleanup
}
```

또는 다음과 같이 `catch` 와 함께 쓸 수도 있습니다.

```rok
try {
    body
} catch e: IOError {
    handler
} finally {
    cleanup
}
```

`finally` 블록은 왼쪽 식을 벗어나는 모든 경우에 실행됩니다. 정상 종료, `catch` 처리, 처리되지 않은 에러의 전파, 그리고 `return` / `break` / `continue` 로 빠져나가는 경우가 모두 포함됩니다. 그래서 자원 해제, 락 해제, 임시 상태 복원 같은 작업에 활용하기 좋습니다. `catch` 블록이 새 에러를 던지거나 `finally` 블록 안에서 에러가 던져지면, 새 에러가 우선합니다.

panic 과 fatal 에러 (3.14.8) 는 프로세스를 종료하므로 `finally` 를 실행하지 않습니다.

## 3.14.6. 식의 값과 타입

`try`, `catch`, `finally` 는 모두 식입니다.

```rok
let x = try parse(input) catch _ { default };
```

`catch` 식의 값은 왼쪽 식이 정상 종료하면 그 값이고, 에러를 잡으면 해당 핸들러 블록의 값입니다. 왼쪽 식과 모든 핸들러 블록의 타입은 같아야 하며 ([표현식 개요](README.md) 의 분기 결과 규칙), 핸들러 블록이 `throw` 나 `return` 으로 끝나 divergent 이면 그 블록은 타입 검사에서 빠집니다. `finally` 블록은 값을 만들지 않으며, 식의 결과에도 영향을 주지 않습니다.

## 3.14.7. 던지는 값과 함수 시그니처

`throw err` 의 `err` 는 임의의 식입니다. typed `throws (E)` 함수에서는 `E` 의 항에 해당하는 값을 던질 수 있고, untyped `throws` 함수는 `throws (any Error)` 의 축약이므로 표준 라이브러리의 `Error` 트레잇을 구현하는 값을 던질 수 있습니다.

```rok
fn parse(s: str) throws -> i32 { ... }              // untyped, any Error
fn parse(s: str) throws (TypeError) -> i32 { ... }  // 단일 타입
fn parse(s: str) throws (A | B | C) -> i32 { ... }  // 합 타입
```

시그니처에 `throws` 가 없는 함수 밖으로는 에러가 나갈 수 없습니다. `throws (T)` 의 `T` 가 합 타입인 경우, 그 함수가 던질 수 있는 에러의 닫힌 집합이 되며, 호출자가 `catch` 를 작성하고 완전성을 검사하는 데 쓰입니다.

자세한 함수 시그니처와 `throws` 의 의미는 [`throws` 시그니처](../004-functions/414-throws.md) 에서 정의합니다.

## 3.14.8. panic 과 fatal 에러

`throws` 의 에러와 달리, 다음은 값으로 전달되지 않으며 `catch` 로 잡을 수 없습니다.

- panic. 표준 라이브러리의 `panic_fn` 호출, `try!` 의 실패, 0 으로 나누기, 정수 overflow, 경계 검사 실패 등이 여기에 속합니다.
- Fatal. `StackOverflowError`, `OutOfMemoryError`, `InternalError` 처럼 계속 실행할 수 없는 상태입니다.

두 경우 모두 런타임은 메시지를 표준 에러 (stderr) 에 출력하고 프로세스를 종료 코드 `101` 로 종료합니다. 어느 상황이 panic 이나 fatal 에 해당하는지의 정확한 목록은 표준 라이브러리에서 정의합니다.

## 3.14.9. 평가 순서

`catch` 식과 `finally` 의 평가는 다음 순서로 이루어집니다.

1. 왼쪽 식을 평가합니다.
2. 왼쪽 식에서 에러가 나오면 매칭하는 `catch` 절을 찾아 그 블록을 평가합니다.
3. `finally` 블록이 있으면 왼쪽 식을 어떻게 벗어나든 평가합니다.
4. 매칭하는 `catch` 절이 없으면 `finally` 를 실행한 뒤 에러를 바깥으로 전파합니다.

`throw` 문이 평가될 때는 그 식의 평가가 먼저 끝난 뒤에 에러가 던져집니다.

## 3.14.10. `?` 연산자 — `none` 조기 반환

`?` 는 `T | none` 값을 풀어서 쓰고, `none` 이면 둘러싼 함수에서 곧바로 `none` 을 반환하는 후위 단항 연산자입니다.

```rok
fn first_word_len(text: str | none) -> usize | none {
    let t = text?;                      // none → 즉시 return none
    let w = t.split(" ").next()?;
    w.len()
}
```

`e?` 에서 `e: T | none` 이면 식의 값은 `T` 입니다. `e` 가 `none` 이면 둘러싼 함수가 `return none` 합니다. 따라서 둘러싼 함수의 반환 타입이 `none` 항을 포함해야 하며 (`_ | none`), 그렇지 않으면 컴파일 에러가 발생합니다. 후위 위치라 메서드 체인의 자연스러운 끝에 둘 수 있습니다.

### desugar 규칙

`?` 는 트레잇을 거치지 않고 컴파일러가 직접 풀어 씁니다.

```
expr?
```

위 식은 다음 형태로 풀어집니다.

```rok
match expr {
    none => return none,
    v => v,
}
```

### 적용 범위

`?` 는 `T | none` 전용입니다. 에러를 전파하는 수단이 아니며, 표준 라이브러리의 `Result` 값이나 throwing 호출에는 쓸 수 없습니다. 에러 전파는 `throws` 시그니처와 `try` 로 표시합니다.

```rok
fn load() -> Result<Config, IOError> {
    let raw = read_result(path)?;     // ERROR — `?` 는 `T | none` 전용입니다.
                                      //   에러 전파는 `throws` 와 `try` 를 쓰세요.
    ...
}
```

closure 본체 안의 `?` 는 closure 의 반환 타입에 대해 동작하며, 외부 함수의 반환 타입과는 무관합니다.

```rok
let lens = users.map(|u| -> usize | none { let nick = u.nickname?; nick.len() });
```

`none` 을 대체값으로 메우거나 멤버 접근을 통과시키는 연산은 [옵셔널 체이닝과 합체](308-optional-chaining-and-coalescing.md) 의 `?.` 와 `??` 를 사용합니다.
