# 3.14. 에러 제어 흐름

`throw`, `try`, `catch`, `finally`는 에러를 일으키고, 에러가 흐를 수 있는 지점을 표시하고, 에러를 받아 처리하는 제어 흐름입니다. 후위 `?`는 `T | none` 값이 `none`이면 호출자에게 조기 반환합니다.

```
throw_stmt     ::= "throw" expr ";"
try_expr       ::= ("try" | "try?" | "try!") expr
catch_expr     ::= expr catch_clause+ finally_clause?
                 | expr finally_clause
catch_clause   ::= "catch" pattern? block            // struct 패턴도 됨: catch E { f, .. } { .. }
finally_clause ::= "finally" block
question_expr  ::= expr "?"
```

`catch`와 `finally`는 식이나 블록 뒤의 후위 절 자리에서만 키워드가 되는 문맥 키워드입니다([1.5.4](../001-lexical/105-keywords.md#154-문맥-키워드)). 에러를 던질 수 있는 함수의 시그니처(`throws`, `rethrows`)는 [`throws` 시그니처](../004-functions/414-throws.md)에서 정의합니다. 이 문서는 식과 문장 수준의 제어 흐름을 다룹니다.

## 3.14.1. `throw`: 에러 발생

```rok
throw err;
```

`throw` 키워드 뒤에 아무 식이나 오는 문장입니다. 그 식의 결과 값을 에러 값으로 던집니다. 던진 에러는 그 문장을 둘러싼 가장 가까운 `catch` 식이 처리합니다. 처리되지 않으면 둘러싼 함수의 에러 결과가 됩니다. 이때 둘러싼 함수의 `throws` 시그니처가 그 에러 타입을 받아야 합니다([4.14.5](../004-functions/414-throws.md#4145-throws와-본체의-일치)).

`throw` 문 뒤로는 흐름이 이어지지 않으므로 `throw` 문으로 끝나는 블록은 divergent입니다.

```rok
let x = if cond { 42i32 } else { throw ValueError { msg: "not allowed" }; };
// x: i32 (else 분기 divergent)
```

## 3.14.2. `try`: 호출 지점 표시

```rok
let n = try parse(s);
```

`try`는 전위 식 연산자입니다. 에러를 던지는 함수(`throws` 시그니처가 있는 함수)의 호출은 `try`가 덮는 식 안에 있어야 합니다. 그렇지 않으면 컴파일 에러입니다. 이 규칙 덕분에 에러가 흐를 수 있는 지점이 코드에 모두 드러납니다.

`try e`의 값과 타입은 `e`의 값과 타입 그대로입니다. `e` 안의 호출이 에러를 던지면 가장 가까운 `catch`가 처리합니다. `catch`가 없거나 매칭되는 절이 없으면 에러는 둘러싼 함수로 전파됩니다.

### 덮는 범위

`try`는 오른쪽 식 전체를 덮습니다. `try` 하나로 여러 호출을 함께 표시할 수 있습니다.

```rok
try a(b());                 // a, b 모두 덮음
try f().g().h();            // 체인 전체를 덮음
try {                       // 블록도 식이므로 블록 안의 호출을 모두 덮음
    let cfg = load(path);
    apply(cfg);
};
```

`try`의 피연산자는 오른쪽으로 식이 끝나는 곳까지 이어집니다. `catch` / `finally` 절, 문장 종결자 `;`, 닫는 괄호, 콤마 앞에서 멈춥니다. 따라서 `try a + b()`는 `try (a + b())`이고, `try parse(s) catch _ { 0 }`의 `catch`는 `try parse(s)` 전체에 붙습니다. 정확한 결합 규칙은 [우선순위와 결합성](302-precedence-and-associativity.md)에서 정의합니다.

`try`가 이항 연산자의 오른쪽 피연산자나 함수 인자 자리에 오면 그 피연산자만 덮습니다.

```rok
let total = base + try fetch_extra();
log(try describe(item));
```

### `await`와 함께

에러를 던지는 `async` 함수의 에러는 `await` 지점에서 전달됩니다. 그래서 `await` 식을 `try`로 덮습니다([4.14.10](../004-functions/414-throws.md#41410-throws가-합-타입-시그니처에-미치는-영향)). 제너레이터의 에러도 마찬가지로 `next` 지점에서 받습니다.

```rok
let data = try await fetch(url);
```

## 3.14.3. `try?`와 `try!`

`try`의 두 변형은 에러를 둘러싼 함수로 전파하지 않고, 그 자리에서 다른 형태로 바꿉니다.

| 형태     | `e: T` 일 때의 결과 타입 | 에러가 나면                   |
|----------|--------------------------|-------------------------------|
| `try e`  | `T`                      | 가장 가까운 `catch` 또는 전파 |
| `try? e` | `T \| none`              | 에러를 버리고 `none`          |
| `try! e` | `T`                      | panic (3.14.8)                |

```rok
let port = (try? env::var("PORT")) ?? "8080";
let cfg = try! load_builtin_config();      // 실패하면 프로그램 버그
```

`try?` / `try!`의 `?` / `!`는 `try` 바로 뒤에 붙여야 합니다. `try !x`는 `try (!x)`입니다. `try?`와 `try!`는 에러를 전파하지 않으므로 둘러싼 함수가 `throws`가 아니어도 됩니다. 덮는 범위는 `try`와 같습니다. `try?`의 결과에 `??`를 이어 쓰려면 위 예처럼 괄호로 범위를 닫습니다. 괄호가 없으면 `try? f() ?? d`는 `try? (f() ?? d)`이므로 에러가 나면 `d`가 아니라 `none`이 됩니다. 컴파일러는 이 형태에 경고합니다(린트 `try_coalesce_precedence`). `(try? f()) ?? d`나 `try? (f() ?? d)`로 적습니다.

## 3.14.4. `catch`: 후위 처리 식

```rok
let n = try parse(s) catch e: ParseError { 0 } catch _ { -1 };
```

`catch` 절은 왼쪽 식에 붙는 후위 절입니다. 왼쪽 식을 평가하는 중에 에러가 나면 `catch` 절을 위에서 아래로 검사하고, 처음 매칭되는 절의 블록을 평가합니다. 왼쪽 식이 정상적으로 끝나면 `catch` 블록은 실행되지 않습니다. 왼쪽 식에서 나오는 에러는 `try`로 표시한 에러를 던지는 호출과 `throw` 문이 만든 것입니다.

어느 절도 매칭되지 않으면 에러는 다시 전파됩니다. 이때 둘러싼 함수는 그 에러 타입을 `throws`로 선언해야 합니다.

### 블록 형태

`try`가 블록을 덮고 그 식에 `catch` / `finally`가 붙는 형태도 같은 규칙으로 해석합니다.

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

`catch` 뒤에는 [패턴](311-patterns.md)의 모든 패턴 형태를 쓸 수 있습니다. `e: TimeoutError`는 타입 검사 패턴이며, 매칭된 분기 안에서 `e`는 그 타입으로 좁혀집니다. `catch e: A | B`는 잡은 에러를 합 타입 `A | B`(에러 합 타입의 부분 집합)로 바인딩합니다. 타입을 적지 않은 `throws`의 에러는 [`any Error`](../002-types/202-any-slot.md) 값입니다. 그래서 모든 에러를 한 이름으로 받으려면 `catch e: any Error`나 `catch e`를 씁니다. 트레잇 이름만 쓴 `catch e: Error`는 [트레잇-as-타입 금지](../002-types/212-trait-as-type-banned.md)에 따라 컴파일 에러입니다.

패턴을 생략한 `catch { ... }`는 `catch _ { ... }`와 같습니다.

### 완전성

왼쪽 식이 던질 수 있는 에러 타입을 `catch` 절이 모두 덮으면, 그 `catch` 식은 더 이상 에러를 던지지 않습니다. 타입을 적은 `throws (A | B)` 호출이면 `A`와 `B`를 모두 덮으면 됩니다. 타입을 적지 않은 `throws`이면 `catch e`, `catch _`, `catch e: any Error`처럼 모든 값을 받는 절이 있어야 합니다.

```rok
fn parse_or_zero(s: str) -> i32 {          // non-throwing
    try parse(s) catch _ { 0 }             // 모든 에러를 덮음
}
```

### 도달하지 않는 `try`와 `catch` 경고

컴파일러는 다음 경우에 경고합니다.

- 에러를 던지는 호출을 하나도 덮지 않는 `try`
- 어떤 에러도 도달하지 않는 `catch` 절이나 분기

경고가 난 절도 타입 검사는 그대로 거칩니다.

## 3.14.5. `finally`

```rok
try {
    body
} finally {
    cleanup
}
```

다음처럼 `catch`와 함께 쓸 수도 있습니다.

```rok
try {
    body
} catch e: IOError {
    handler
} finally {
    cleanup
}
```

`finally` 블록은 왼쪽 식을 벗어나는 모든 경우에 실행됩니다. 정상 종료, `catch` 처리, 처리되지 않은 에러의 전파, `return` / `break` / `continue`로 빠져나가는 경우가 모두 포함됩니다. 그래서 자원 해제, 락 해제, 임시 상태 복원 같은 작업에 알맞습니다. `catch` 블록이 새 에러를 던지거나 `finally` 블록 안에서 에러를 던지면 새 에러가 우선합니다.

panic과 fatal 에러(3.14.8)는 프로세스를 종료하므로 `finally`를 실행하지 않습니다.

## 3.14.6. 식의 값과 타입

`try`, `catch`, `finally`는 모두 식입니다.

```rok
let x = try parse(input) catch _ { default };
```

`catch` 식의 값은 왼쪽 식이 정상적으로 끝나면 그 값이고, 에러를 잡으면 해당 핸들러 블록의 값입니다. 왼쪽 식과 모든 핸들러 블록은 타입이 같아야 합니다([표현식 개요](README.md)의 분기 결과 규칙). 핸들러 블록이 `throw`나 `return`으로 끝나 divergent이면 그 블록은 타입 검사에서 빠집니다. `finally` 블록은 값을 만들지 않으며, 식의 결과에도 영향을 주지 않습니다.

## 3.14.7. 던지는 값과 함수 시그니처

`throw err`의 `err`는 임의의 식입니다. 타입을 적은 `throws (E)` 함수는 `E`의 항에 해당하는 값을 던질 수 있습니다. 타입을 적지 않은 `throws` 함수는 `throws (any Error)`의 줄임입니다. 그래서 표준 라이브러리의 `Error` 트레잇을 구현하는 값을 던질 수 있습니다.

```rok
fn parse(s: str) throws -> i32 { ... }              // untyped, any Error
fn parse(s: str) throws (TypeError) -> i32 { ... }  // 단일 타입
fn parse(s: str) throws (A | B | C) -> i32 { ... }  // 합 타입
```

시그니처에 `throws`가 없는 함수 밖으로는 에러가 나갈 수 없습니다. `throws (T)`의 `T`가 합 타입이면 그 타입은 함수가 던질 수 있는 에러의 닫힌 집합입니다. 호출하는 쪽은 그 집합을 보고 `catch` 절을 쓰고 완전성을 검사합니다.

함수 시그니처와 `throws`의 의미는 [`throws` 시그니처](../004-functions/414-throws.md)에서 자세히 정의합니다.

## 3.14.8. panic과 fatal 에러

`throws`의 에러와 달리, 다음은 값으로 전달되지 않으며 `catch`로 잡을 수 없습니다.

- panic. 표준 라이브러리의 `panic_fn` 호출, `try!`의 실패, 0으로 나누기, 정수 오버플로, 경계 검사 실패 등이 여기에 속합니다.
- Fatal. `StackOverflowError`, `OutOfMemoryError`, `InternalError`처럼 실행을 계속할 수 없는 상태입니다.

두 경우 모두 런타임은 메시지를 표준 에러(stderr)에 출력하고 종료 코드 `101`로 프로세스를 종료합니다. 어떤 상황이 panic이나 fatal에 해당하는지 정확한 목록은 표준 라이브러리에서 정의합니다.

실패한 `try!`는 `try!`의 위치에서 panic을 일으킵니다. 메시지는 `main` 밖으로 나간 에러와 같은 방식으로 에러를 보여 줍니다([4.16.4](../004-functions/416-entry-point.md): `Display`, 없으면 `Debug`, 없으면 타입 이름). 이어서 그 에러가 난 위치를 보여 줍니다.

```text
thread 'main' panicked at src/main.rok:12:13:
try! failed: no such key `port`
  raised at src/main.rok:12:18
```

## 3.14.9. 평가 순서

`catch` 식과 `finally`는 다음 순서로 평가합니다.

1. 왼쪽 식을 평가합니다.
2. 왼쪽 식에서 에러가 나면 매칭되는 `catch` 절을 찾아 그 블록을 평가합니다.
3. `finally` 블록이 있으면 왼쪽 식을 어떻게 벗어나든 평가합니다.
4. 매칭되는 `catch` 절이 없으면 `finally`를 실행한 뒤 에러를 바깥으로 전파합니다.

`throw` 문을 평가할 때는 그 식의 평가를 먼저 마친 뒤 에러를 던집니다.

## 3.14.10. `?` 연산자: `none` 조기 반환

`?`는 `T | none` 값을 풀어 쓰고, 값이 `none`이면 둘러싼 함수에서 바로 `none`을 반환하는 후위 단항 연산자입니다.

```rok
fn first_word_len(text: str | none) -> usize | none {
    let t = text?;                      // none → 즉시 return none
    let w = t.split(" ").next()?;
    w.len()
}
```

`e?`에서 `e: T | none`이면 식의 값은 `T`입니다. `e`가 `none`이면 둘러싼 함수는 `return none`으로 바로 반환합니다. 따라서 둘러싼 함수의 반환 타입에 `none` 항이 있어야 합니다(`_ | none`). 없으면 컴파일 에러입니다. 후위 연산자이므로 메서드 체인의 끝에 자연스럽게 둘 수 있습니다.

### 풀어 쓰기 규칙

`?`는 트레잇을 거치지 않고 컴파일러가 직접 풀어 씁니다.

```
expr?
```

위 식은 다음과 같이 풀립니다.

```rok
match expr {
    none => return none,
    v => v,
}
```

### 적용 범위

`?`는 `T | none` 전용입니다. 에러를 전파하는 수단이 아니며, 표준 라이브러리의 `Result` 값이나 에러를 던지는 호출에는 쓸 수 없습니다. 에러 전파는 `throws` 시그니처와 `try`로 표시합니다.

```rok
fn load() -> Result<Config, IOError> {
    let raw = read_result(path)?;     // ERROR: `?` 는 `T | none` 전용입니다.
                                      //   에러 전파는 `throws` 와 `try` 를 쓰세요.
    ...
}
```

클로저 본체 안의 `?`는 클로저의 반환 타입을 기준으로 동작합니다. 바깥 함수의 반환 타입과는 관계가 없습니다.

```rok
let lens = users.map(|u| -> usize | none { let nick = u.nickname?; nick.len() });
```

`none`을 대체값으로 채우거나 멤버 접근을 그대로 통과시키려면 [옵셔널 체이닝과 합체](308-optional-chaining-and-coalescing.md)의 `?.`와 `??`를 씁니다.
