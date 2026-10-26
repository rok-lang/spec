# 4.14. `throws` 시그니처

[에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 `throw`, `try`, `catch`, `finally` 와 결합되는 함수 시그니처의 `throws` / `rethrows` 형태를 정의합니다.

> [!IMPORTANT]
> Rok 의 에러 모델은 `throws` 하나입니다. 에러를 던질 수 있는 함수는 시그니처에 `throws` 를 적고, 호출하는 쪽은 그 호출을 `try` 로 표시합니다. 에러를 값으로 담아 두어야 할 때에만 표준 라이브러리의 [`Result<T, E>`](../../refs-standard-libraries/200-core/214-error.md) 를 씁니다. 규칙은 한 줄입니다. **던지면 `throws`, 담아 두면 `Result`.**

## 4.14.1. 형태

[함수 선언](401-function-declaration.md) 의 `fn_decl` 에는 `throws_clause?` 가 매개변수 괄호 뒤이자 반환 타입 화살표 앞 위치에 결합됩니다.

```
throws_clause   ::= "throws" ("(" type_expr ")")?
                  | "rethrows"
```

따라서 `fn_decl` 의 전체 형태는 다음과 같이 표현됩니다.

```
fn_decl ::= visibility? fn_modifiers "fn" identifier "(" param_list? ")" throws_clause? return_type? block
```

여기서 `fn_modifiers` 는 [4.1 함수 선언](401-function-declaration.md) 에서 정의한 `unsafe? async? yield?` 입니다.

```rok
fn name(args) -> R                              // non-throwing
fn name(args) throws -> R                       // untyped - throws (any Error) 의 축약
fn name(args) throws (E) -> R                   // typed - 단일 에러 타입
fn name(args) throws (A | B | C) -> R           // typed - 합 타입 에러
fn name(f: fn(T) throws -> U) rethrows -> R     // 인자로 받은 함수가 던질 때만 던짐
```

`throws` 와 `rethrows` 는 매개변수 괄호 `)` 뒤에서만 키워드인 문맥 키워드이며 ([1.5.4](../001-lexical/105-keywords.md#154-문맥-키워드)), 반환 타입 화살표 앞에 위치합니다. `throws` 뒤의 선택적인 괄호 안에는 던질 수 있는 에러의 타입을 명시합니다.

## 4.14.2. non-throwing 함수

```rok
fn add(x: i32, y: i32) -> i32 { x + y }
```

`throws` 가 없는 함수에서는 에러가 함수 밖으로 나갈 수 없습니다. 본체의 `throw` 나 `try` 로 표시한 호출에서 나온 에러가 함수 안의 `catch` 로 모두 처리되지 않으면 컴파일 에러가 발생합니다. 호출자는 이 함수를 `try` 없이 호출합니다.

```rok
fn bad() -> i32 {
    throw ValueError { msg: "nope" };   // ERROR - non-throwing 함수 밖으로 에러가 나감
}
```

이러한 강제는 어떤 함수가 에러를 던질 수 있는지를 시그니처에 명시하도록 만들며, 그 결과 조용한 에러 전파를 차단합니다.

## 4.14.3. untyped `throws`

```rok
fn parse(s: str) throws -> i32 { ... }
```

untyped `throws` 는 `throws (any Error)` 의 축약입니다. 이 함수는 표준 라이브러리의 [`Error`](../../refs-standard-libraries/200-core/214-error.md) 트레잇을 구현하는 어떤 타입의 에러든 던질 수 있으며, 던진 값은 [제약 붙은 `any`](../002-types/202-any-slot.md) 인 `any Error` 값으로 전달됩니다. 호출자는 `catch` 의 타입 패턴으로 구체 타입을 좁혀 매칭하거나, `catch e: any Error` 또는 catch-all 로 모두 받습니다.

untyped 형태는 공개 API 와 같이 에러 종류가 호출자에게 닫혀 있지 않은 경우에 권장됩니다.

## 4.14.4. typed `throws` 의 단일 타입과 합 타입

```rok
fn fetch_user(id: i64) throws (NotFound) -> User { ... }
fn parse_or_load(s: str) throws (ParseError | IOError | TimeoutError) -> Data { ... }
```

괄호 안에 던질 수 있는 정확한 타입을 명시할 수 있으며, 합 타입 (`A | B | C`) 도 그대로 사용할 수 있습니다.

typed 형태는 내부 또는 라이브러리 API, 임베디드 환경, 그리고 에러 종류가 닫혀 있는 경우에 적합합니다. 이 경우 컴파일러는 호출자의 `catch` 가지가 에러 타입을 모두 덮는지 검증할 수 있으며, 모두 덮으면 그 식은 더 이상 에러를 던지지 않는 것으로 취급합니다.

```rok
fn caller() -> Data {             // non-throwing - 세 타입을 모두 처리
    try parse_or_load(input)
    catch e: ParseError { ... }
    catch e: IOError { ... }
    catch e: TimeoutError { ... }
}
```

## 4.14.5. `throws` 와 본체의 일치

함수 밖으로 나갈 수 있는 모든 에러는 시그니처의 `throws` 와 호환되어야 합니다. 에러의 출처는 본체의 `throw` 문과 `try` 로 표시한 throwing 호출이며, 함수 안의 `catch` 가 처리한 에러는 검사 대상에서 빠집니다.

```rok
fn f() throws (A) -> i32 {
    throw A_instance;             // OK
    throw B_instance;             // ERROR - A 만 가능
}
```

호환 규칙은 다음과 같습니다. typed `throws (E)` 는 출처의 에러 타입이 모두 `E` 의 항이어야 합니다. untyped `throws` 는 출처의 에러 타입이 모두 `Error` 를 구현해야 하며, 그 값은 `any Error` 로 전달됩니다.

에러 타입 사이의 자동 변환은 없습니다. 하위 함수의 에러를 다른 타입으로 바꿔 전파하려면 `catch` 에서 감싸 다시 던집니다.

```rok
fn load_config(path: str) throws (ConfigError) -> Config {
    try parse_toml(read_text(path))
    catch e: IOError { throw ConfigError::io(e); }
    catch e: ParseError { throw ConfigError::syntax(e); }
}
```

## 4.14.6. 호출자의 의무

`throws` 함수를 호출하는 식은 반드시 `try` 가 덮는 식 안에 있어야 합니다. `try` 없이 throwing 함수를 호출하면 컴파일 에러가 발생합니다. `try` 는 그 오른쪽 식 전체를 덮으므로, `try a(b())` 는 `a` 와 `b` 두 호출을 모두 덮고, `try { a(); b(); }` 는 블록 안의 호출을 모두 덮습니다 ([3.14.2](../003-expressions/314-error-control-flow.md#3142-try-호출-지점-표시)).

`try` 로 표시한 호출에서 나온 에러는 가장 가까운 `catch` 가 처리하며, 처리되지 않은 에러는 둘러싼 함수로 전파됩니다. 이때 둘러싼 함수의 `throws` 가 그 에러 타입을 받아야 합니다 (4.14.5).

```rok
fn caller() {                     // non-throwing
    parse(s);                     // ERROR - throwing 호출에 try 표시 없음
    try parse(s);                 // ERROR - 에러가 전파되는데 caller 는 throws 가 아님
}

fn caller2() throws {             // 전파
    try parse(s);                 // OK
}

fn caller3() {                    // 잡음
    try parse(s) catch _ {
        ...
    };
}

fn caller4() -> i32 {             // 에러를 버리거나 panic 으로 바꿈
    let a = (try? parse(s)) ?? 0; // try? - 에러면 none
    let b = try! parse(t);        // try! - 에러면 panic
    a + b
}
```

이로써 에러가 흐를 수 있는 지점은 모두 코드에 `try` 로 드러나며, 에러가 어디까지 전파되는지는 함수 시그니처의 `throws` 로 드러납니다.

## 4.14.7. `rethrows`

```rok
fn map<U>(self, f: fn(T) throws -> U) rethrows -> List<U> {
    let out: List<U> = [];
    for x in self {
        out.push(try f(x));
    }
    out
}
```

`rethrows` 는 함수형 매개변수를 받는 고차 함수를 위한 형태입니다. `rethrows` 함수는 인자로 받은 함수나 클로저가 던질 때에만 던집니다.

- 호출 지점에서 throwing 함수형 매개변수에 넘긴 인자가 모두 non-throwing 이면, 그 호출은 non-throwing 이며 `try` 없이 부릅니다.
- 인자 중 하나라도 throwing 이면 그 호출은 throwing 이며, 던지는 에러 타입은 그 인자들의 에러 타입입니다. 호출자는 `try` 로 표시합니다.
- `rethrows` 함수의 본체에서 함수 밖으로 나갈 수 있는 에러는 함수형 매개변수의 호출에서 나온 에러뿐입니다. 본체가 자기 에러를 직접 던지면 컴파일 에러입니다.

```rok
let lens = names.map(|s| s.len());              // non-throwing 클로저 - try 없음
let nums = try names.map(|s| try parse(s));     // throwing 클로저 - try 필요
```

`rethrows` 함수를 값으로 쓰면, 기대 타입 (함수 타입) 이 어느 인스턴스를 쓸지 정합니다. 기대 타입이 없으면 에러 매개변수는 `any Error` 로 정해집니다.

## 4.14.8. 함수 타입의 `throws`

`throws` 는 [함수 타입](404-function-type.md) 에도 붙습니다.

```rok
fn(T) -> U                    // non-throwing
fn(T) throws -> U             // untyped - throws (any Error)
fn(T) throws (E) -> U         // typed
```

non-throwing 함수 값은 같은 매개변수와 반환 타입의 throwing 함수 타입 자리에 넣을 수 있으며, `throws (A)` 함수 값은 `throws (A | B)` 자리나 (`A` 가 `Error` 를 구현하면) untyped `throws` 자리에 넣을 수 있습니다. 반대 방향은 허용되지 않습니다. 자세한 내용은 [4.4](404-function-type.md) 에서 다룹니다.

## 4.14.9. `throws` 와 `async`, `yield` 의 결합

세 수식어는 모두 같은 함수에 결합할 수 있습니다.

```rok
async fn fetch(url: str) throws (NetworkError) -> Data { ... }

yield fn lines(s: str) throws (ParseError) -> str { ... }

async yield fn read_paginated(url: str) throws (NetworkError) -> Page { ... }
```

`throws` 는 시그니처 위치의 키워드로서, `async` 또는 `yield` 와 직교적으로 결합됩니다.

## 4.14.10. `throws` 가 합 타입 시그니처에 미치는 영향

```rok
async fn fetch_data() throws (NetworkError) -> Data { ... }
```

위 함수의 호출 결과가 `Data | NetworkError` 를 담는 Future 인지 궁금할 수 있습니다.

답은 **`TryAsyncFuture<Data, NetworkError>`** 입니다 ([4.1.11](401-function-declaration.md#코루틴-함수의-결과-타입)). 에러는 결과 타입에 섞이지 않고, `await` 지점에서 에러 채널로 전달됩니다. 따라서 `await` 하는 쪽이 `try` 로 표시합니다. `TryAsyncFuture` 는 일반 `Future` 트레잇을 구현하지 않습니다.

```rok
let f = fetch_data();              // f: TryAsyncFuture<Data, NetworkError>
let d = try await f catch e: NetworkError {
    ...
};
```

이 결정 덕분에 결과가 결과와 에러라는 두 변형을 갖는 `enum` 으로 변하지 않습니다.

`yield` 와 `async yield` 도 동일한 패턴을 따릅니다. `throws (E)` 인 제너레이터는 `TryGenIter<T, E>`, 스트림은 `TryAsyncStream<T, E>` 이며, 에러는 `next` 지점에서 에러 채널로 전달됩니다.

### `try for`

`next` 가 던지는 iterator (`TryGenIter` 등) 를 도는 `for` 는 throwing 식이므로 `try` 로 덮어야 합니다. `for await` 도 마찬가지입니다.

```rok
try for line in lines("a.txt") {         // 각 next 의 IoError 가 전파됨
    process(line);
}
try for await ev in events() { handle(ev); }
for line in lines("a.txt") { ... }       // ERROR - try 없음
```

## 4.14.11. 메서드의 `throws`

[메서드](409-method.md) 의 일반 메서드와 마찬가지로 `throws` 시그니처를 가질 수 있습니다.

```rok
impl Database {
    fn query(self, sql: str) throws (SqlError) -> ResultSet { ... }
}
```

호출 시 호출자의 의무와 전파 규칙은 자유 함수와 동일하게 적용됩니다.

트레잇을 구현하는 impl 메서드의 `throws` 는 트레잇 메서드의 `throws` 와 같아야 합니다. impl 이 `throws` 를 새로 붙이거나 에러 타입을 넓히면 컴파일 에러입니다.

연산자 트레잇 메서드 (`Add::add` 등) 는 `throws` 없이 선언되므로 연산자는 에러를 던지지 않습니다. 실패할 수 있는 연산은 이름 있는 메서드로 두거나 `Result` 를 반환합니다 ([stdlib 2.3](../../refs-standard-libraries/200-core/203-ops.md)).

## 4.14.12. 값 기반 구현 모델

throwing 함수는 결과를 값으로 돌려줍니다. 반환값은 성공 여부 플래그와, 성공 값 또는 에러 값을 담는 payload 의 쌍입니다. 에러 채널은 시그니처로 구분되므로 성공 타입과 에러 타입이 같은 타입이어도 문제가 없습니다. 호출 지점의 `try` 는 플래그를 검사해 분기하며, 에러면 가장 가까운 `catch` 로 가거나 둘러싼 함수의 에러 결과로 그대로 반환합니다.

이 모델에서는 에러가 나지 않는 경로의 비용이 분기 하나뿐이고, `try` 나 `catch` 에 진입하는 비용은 없습니다. 스택을 거슬러 올라가는 unwinding 이나 런타임의 에러 전달 함수도 쓰지 않습니다. 정확한 반환값 레이아웃은 [8.6.5](../008-abi/806-calling-convention.md#865-throwing-함수의-반환-값-기반-에러-전달) 에서, lowering 은 [8.9.6](../008-abi/809-lowering-control.md#896-try--catch--finally-값-기반-분기) 에서 정의합니다.

## 4.14.13. panic 과 fatal 에러

panic 은 `throws` 의 에러가 아닙니다. 표준 라이브러리의 `panic_fn`, `try!` 의 실패, 0 으로 나누기, 정수 overflow, 경계 검사 실패 같은 panic 과 [에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 fatal 에러는 어떤 `throws` 시그니처에도 들어가지 않으며, `catch` 로 잡을 수 없습니다. 런타임은 다음을 표준 에러 (stderr) 에 출력하고 프로세스를 종료 코드 `101` 로 종료합니다.

```
thread '<name>' panicked at <path>:<line>:<col>:
<message>
```

- `<name>` 은 스레드 이름이며, 주 스레드는 `main`, 이름 없는 스레드는 `<unnamed>` 입니다.
- 위치는 실패한 식입니다. [`#[track_caller]`](../006-meta-system/602-built-in-attributes.md#6212-track_caller-호출-위치-전달) 함수 사슬 안이면 그 사슬 밖의 첫 호출 위치입니다.
- backtrace 는 정의하지 않습니다.
