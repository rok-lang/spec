# 4.1. 함수 선언

이 절에서는 `fn` 키워드 ([키워드](../001-lexical/105-keywords.md)) 로 시작하는 명명된 함수의 선언 형태를 정의합니다.

## 4.1.1. 형태

함수 선언의 문법은 다음과 같이 표현할 수 있습니다.

```
fn_decl       ::= visibility? fn_modifiers "fn" identifier "(" param_list? ")" throws_clause? return_type? block
fn_modifiers  ::= "unsafe"? "async"? "yield"?
return_type   ::= "->" type_expr
param_list    ::= param ("," param)* ","?
```

여기서 `visibility?` 는 [가시성](../005-modules/506-visibility.md) 의 `pub`, `pub(pkg)`, `pub(sub)` 중 하나이거나 생략 (모듈 비공개) 한 형태입니다. `fn_modifiers` 는 [unsafe](415-unsafe.md), [async](412-async.md), [yield (제너레이터)](411-generators.md) 세 수식어를 **고정된 순서로 0 또는 1 개씩** 적는 자리입니다 (자세한 규칙은 [4.1.11 fn 수식어의 순서와 조합](#4111-fn-수식어의-순서와-조합) 에서 정의). `throws_clause?` 의 의미는 [throws](414-throws.md) 에서 정의하며, 본 문서에서는 가시성과 modifier / throws 키워드 자체의 의미를 다루지 않습니다.

전체 모양을 한국어로 살펴보면 다음과 같은 형태가 됩니다.

```rok
fn 이름(매개변수들) -> 반환타입 {
    본체
}
```

각 구성 요소의 의미는 다음과 같습니다.

- `fn` 키워드로 시작합니다.
- 함수 이름은 [식별자](../001-lexical/104-identifiers.md) 의 형식을 따릅니다.
- 괄호로 둘러싼 매개변수 목록을 적으며, 자세한 규칙은 [매개변수](402-parameters.md) 에서 정의합니다.
- 반환 타입은 선택 사항으로, `->` 토큰 ([구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)) 뒤에 [타입 표현식](../002-types/201-type-expressions.md) 을 둡니다.
- 본체는 [블록 표현식](../003-expressions/309-block-expressions.md) 으로 작성합니다.

## 4.1.2. 가장 간단한 예

가장 단순한 형태의 함수 선언은 다음과 같이 작성할 수 있습니다.

```rok
fn greet() {
    print("hello");
}

fn add(x: i32, y: i32) -> i32 {
    x + y
}
```

## 4.1.3. 반환 타입 생략

반환 타입의 `->` 부분은 생략할 수 있으며, 이때 다음과 같은 의미를 갖습니다.

- 본체에서 반환 식의 타입을 추론하며, 본체의 마지막 식 타입이 반환 타입이 됩니다.
- 추론할 수 없거나 모호한 경우에는 컴파일 에러가 발생합니다.

다음 예에서는 반환 타입을 생략했지만, 마지막 식의 타입으로부터 `i32` 가 추론됩니다.

```rok
fn double(x: i32) {              // 반환 타입 생략
    x * 2                         // 추론: i32
}
// 위는 fn double(x: i32) -> i32 와 동일
```

본체가 값을 만들지 않는 경우 (마지막 식이 종결자로 끝나는 경우) 에는 반환 타입이 unit (`()`) 이 됩니다.

```rok
fn print_msg(s: str) {
    print(s);                     // 마지막에 종결자 - unit 반환
}
```

반환 타입을 명시했음에도 함수 본체와 명시 타입이 일치하지 않으면 컴파일 에러가 발생합니다.

```rok
fn bad() -> i32 {
    "hello"                       // ERROR - str 이 i32 아님
}
```

## 4.1.4. 매개변수 타입 생략

매개변수에 타입을 적지 않으면 암묵 제네릭으로 처리됩니다 ([`any` 슬롯](../002-types/202-any-slot.md), [제네릭](../002-types/210-generics.md)).

```rok
fn id(x) {                        // x: T (제네릭)
    x
}
// 호출 지점의 타입에 따라 단형화
let a = id(42i32);                // id<i32>
let b = id("hi");                 // id<str>
```

만약 매개변수가 동적 슬롯이 되기를 원한다면 명시적으로 `any` 를 적어 주시면 됩니다.

```rok
fn dyn_id(x: any) {               // x 는 any 슬롯 (런타임 디스패치)
    x
}
```

## 4.1.5. 본체

함수 본체는 [블록 표현식](../003-expressions/309-block-expressions.md) 의 규칙을 그대로 따르며, 다음 성질을 가집니다.

- 0 개 이상의 문장을 포함합니다.
- 마지막 식은 0 또는 1 개 (종결자 없이) 둘 수 있고, 그 식의 값이 함수의 반환값이 됩니다.
- 마지막에 `return` ([`return`](403-return.md)) 으로 명시 반환할 수 있습니다.

## 4.1.6. 빈 본체

본체가 비어 있는 함수는 unit 을 반환하는 함수가 됩니다.

```rok
fn nothing() { }                  // 본체 비어 있음 - unit 반환
```

## 4.1.7. 반환 타입과 본체 마지막 식

반환 타입이 명시된 함수에서 마지막 식이 종결자 없이 놓이면 그 식이 반환값이 됩니다.

```rok
fn f() -> i32 {
    42i32                         // 마지막 식이 반환값
}
```

위 예에서 `42i32` 는 종결자가 없는 식이므로 함수의 반환값이 됩니다. 만약 종결자를 붙여 (`42i32;`) 작성하면 값이 폐기되고 본체가 unit 을 반환하므로, 명시한 `-> i32` 와 충돌하여 컴파일 에러가 발생합니다.

```rok
fn f() -> i32 {
    42i32;                        // ERROR - 종결자가 있어 값이 폐기됨
                                  // (반환 타입 i32 와 안 맞음)
}
```

## 4.1.8. 함수 본체에서의 명시적 `return`

본체 안에서는 `return` 키워드를 사용해 명시 반환할 수 있으며, 자세한 내용은 [`return`](403-return.md) 에서 정의합니다.

```rok
fn early(x: i32) -> i32 {
    if x < 0 { return -1; }
    x * 2
}
```

## 4.1.9. 가시성

함수에 가시성 키워드 (`pub`, `pub(pkg)`, `pub(sub)`) 를 붙이면 외부에 노출되는 범위를 조절할 수 있습니다.

```rok
pub fn area(r: f64) -> f64 { ... }              // 외부 패키지에서 호출 가능
pub(pkg) fn helper(x: i32) -> i32 { ... }       // 같은 패키지 안에서만
fn _internal(s: str) { ... }                     // 모듈 비공개 (기본)
```

각 가시성 단계의 정확한 의미는 [가시성](../005-modules/506-visibility.md) 에서 정의합니다.

## 4.1.10. 같은 모듈 내 중복 정의 금지

같은 모듈 안에서 같은 이름의 함수를 두 번 선언하면 컴파일 에러가 발생합니다. rok 에는 오버로딩이 없습니다.

```rok
fn add(x: i32, y: i32) -> i32 { x + y }
fn add(x: f64, y: f64) -> f64 { x + y }   // ERROR - 중복
```

타입에 따라 다르게 동작하도록 만들고 싶다면 트레잇과 제네릭을 함께 사용하여 표현해 주시면 됩니다 ([`trait`](../002-types/208-trait.md)).

모듈의 함수, `const`, `static` 은 하나의 값 이름공간을 공유합니다: 같은 모듈의 다른 것과 이름이 같은 `const`, `static`, 함수도 컴파일 에러입니다. tuple 구조체나 unit 구조체의 생성자도 그 이름공간에 있습니다 ([2.3.7](../002-types/203-struct.md#237-같은-모듈-내-중복-정의-금지)): `fn Point`, `const Point`, `static Point` 옆의 `struct Point(i32, i32);` 는 중복 정의입니다. arity, `async`, 타입 매개변수가 달라도 두 함수가 구별되지 않습니다. 인자 개수를 다르게 받으려면 한 함수의 뒤쪽 매개변수에 기본값을 줍니다 ([4.2.3](402-parameters.md#423-기본값)).

## 4.1.11. fn 수식어의 순서와 조합

함수에 붙는 수식어 (`unsafe`, `async`, `yield`) 는 **반드시 다음 순서**로 적어야 합니다.

```
unsafe → async → yield
```

이 순서를 어기면 syntax error 가 발생합니다. 같은 수식어를 두 번 적는 것 또한 grammar 차원에서 불가능합니다 (`unsafe?` 의 `?` 가 0 또는 1 개만 허용).

```rok
unsafe async yield fn ok() -> i32 { ... }   // OK - 정해진 순서
async unsafe fn bad1() { ... }           // ERROR - async 가 unsafe 앞에 옴
yield async fn bad2() { ... }            // ERROR - yield 가 async 앞에 옴
unsafe unsafe fn bad3() { ... }          // ERROR - unsafe 중복
```

이러한 강제 규칙은 같은 의미의 함수가 항상 같은 모양으로 적히도록 보장하여, formatter / linter / 사용자 가독성을 모두 단순하게 합니다.

### 가능한 8 개 조합

세 수식어가 모두 선택적이므로 총 8 개의 조합이 가능하며, 각각 다음과 같이 정의됩니다.

| 조합                    | 호출 결과                              | 의미                 | 정의 위치                          |
|-------------------------|----------------------------------------|----------------------|------------------------------------|
| `fn`                    | `R`                                    | 일반 함수            | [4.1](401-function-declaration.md) |
| `unsafe fn`             | `R`                                    | unsafe 함수          | [4.15](415-unsafe.md)              |
| `async fn`              | `AsyncFuture<R>`                   | 비동기 함수          | [4.12](412-async.md)               |
| `yield fn`              | `GenIter<R>`                       | 제너레이터           | [4.11](411-generators.md)          |
| `async yield fn`        | `AsyncStream<R>`                   | 비동기 스트림        | [4.13](413-stream.md)              |
| `unsafe async fn`       | `AsyncFuture<R>`                   | unsafe 비동기 함수   | (각 modifier 의 합)                |
| `unsafe yield fn`       | `GenIter<R>`                       | unsafe 제너레이터    | (각 modifier 의 합)                |
| `unsafe async yield fn` | `AsyncStream<R>`                   | unsafe 비동기 스트림 | (각 modifier 의 합)                |

`unsafe` 가 함께 붙은 조합의 의미는 단순합니다. 함수 본체 전체가 unsafe scope 가 되며 (4.15 의 정의), 결과 타입과 코루틴 동작은 함께 붙은 다른 수식어 (`async` / `yield`) 의 정의를 그대로 따릅니다. 즉 `unsafe` 는 본체 안에서 raw pointer 역참조나 `unsafe fn` 호출을 허용하는 효과만 더합니다.

### 코루틴 함수의 결과 타입

`async` / `yield` 가 붙은 함수 (와 블록, 클로저) 의 호출 결과는 표준 라이브러리 타입입니다. 선언의 `-> R` 은 원소 / 결과 타입 `T` 를 적습니다.

| 형태             | 결과 타입         | 구현 트레잇            | `throws (E)` 일 때        |
|------------------|-------------------|------------------------|---------------------------|
| `yield fn`       | `GenIter<T>`      | `Iterator<Item = T>`   | `TryGenIter<T, E>`        |
| `async fn`       | `AsyncFuture<T>`  | `Future<Output = T>`   | `TryAsyncFuture<T, E>`    |
| `async yield fn` | `AsyncStream<T>`  | `Stream<Item = T>`     | `TryAsyncStream<T, E>`    |

`Try*` 변형은 에러를 다음 진행 지점에서 던집니다. 예를 들어 `TryGenIter<T, E>` 의 `next` 는 `next(ref self) throws (E) -> T | none` 이고, `TryAsyncFuture` 는 `try await` 에서 던집니다. 이 변형들은 일반 `Iterator` / `Future` / `Stream` 트레잇을 구현하지 않습니다 ([4.14.10](414-throws.md#41410-throws-가-합-타입-시그니처에-미치는-영향)).

```rok
async fn fetch_user(id: i64) -> User { ... }
let f: AsyncFuture<User> = fetch_user(42i64);

yield fn lines(path: str) throws (IoError) -> str { ... }
let it: TryGenIter<str, IoError> = lines("a.txt");
```

### 클로저와 블록 형태

`async` 와 `yield` 의 클로저 / 블록 형태에서도 같은 순서 규칙이 적용됩니다. 예를 들어 Stream 을 만드는 클로저는 항상 `async yield ||` 로 적으며, `yield async ||` 는 syntax error 입니다. 자세한 클로저 / 블록 형태는 각 modifier 문서를 참고합니다.

## 4.1.12. 매개변수 / 반환 / 본체 사이의 평가 순서

호출이 일어날 때의 평가 순서는 다음과 같습니다.

1. 인자 표현식을 평가합니다 (좌→우 순서).
2. 매개변수 매칭을 수행하고 함수 본체 평가를 시작합니다.
3. 본체를 평가하며, 종료 시 마지막 식의 값 또는 `return` 의 값이 결과가 됩니다.
4. 호출 지점에 결과를 전달합니다.

세부 평가 순서는 [매개변수](402-parameters.md) 의 매개변수 평가 순서 절에서 정의합니다.
