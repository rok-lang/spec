# 4.1. 함수 선언

이 절은 `fn` 키워드([키워드](../001-lexical/105-keywords.md))로 시작하는 이름 있는 함수의 선언 형태를 정의합니다.

## 4.1.1. 형태

함수 선언의 문법은 다음과 같습니다.

```
fn_decl       ::= visibility? fn_modifiers "fn" identifier "(" param_list? ")" throws_clause? return_type? block
fn_modifiers  ::= "unsafe"? "async"? "yield"?
return_type   ::= "->" type_expr
param_list    ::= param ("," param)* ","?
```

여기서 `visibility?`는 [가시성](../005-modules/506-visibility.md)의 `pub`, `pub(pkg)`, `pub(sub)` 중 하나입니다. 생략하면 모듈 비공개입니다. `fn_modifiers`는 [unsafe](415-unsafe.md), [async](412-async.md), [yield (제너레이터)](411-generators.md) 세 수식어를 **정해진 순서로 0개 또는 1개씩** 적는 자리입니다. 자세한 규칙은 [4.1.11 fn 수식어의 순서와 조합](#4111-fn-수식어의-순서와-조합)에 있습니다. `throws_clause?`의 의미는 [throws](414-throws.md)에서 정의합니다. 이 문서는 가시성, 수식어, throws 키워드 자체의 의미는 다루지 않습니다.

형식을 따지지 않고 전체 모양을 적으면 다음과 같습니다.

```rok
fn 이름(매개변수들) -> 반환타입 {
    본체
}
```

구성 요소의 의미는 다음과 같습니다.

- `fn` 키워드로 시작합니다.
- 함수 이름은 [식별자](../001-lexical/104-identifiers.md)의 형식을 따릅니다.
- 이어서 괄호로 감싼 매개변수 목록이 옵니다. 자세한 규칙은 [매개변수](402-parameters.md)에서 정의합니다.
- 반환 타입은 선택 사항입니다. `->` 토큰([구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)) 뒤에 [타입 표현식](../002-types/201-type-expressions.md)이나 불투명 `some Tr`([4.1.13](#4113-불투명-반환-타입-some-trait))을 둡니다.
- 본체는 [블록 표현식](../003-expressions/309-block-expressions.md)입니다.

## 4.1.2. 가장 간단한 예

가장 단순한 함수 선언은 다음과 같습니다.

```rok
fn greet() {
    print("hello");
}

fn add(x: i32, y: i32) -> i32 {
    x + y
}
```

## 4.1.3. 반환 타입 생략

반환 타입의 `->` 부분은 생략할 수 있습니다. 그 뜻은 함수의 [가시성](../005-modules/506-visibility.md)에 따라 다릅니다.

모듈 비공개 함수나 `pub(sub)` 함수에서는 다음과 같습니다.

- 컴파일러가 본체에서 반환 타입을 추론합니다. 본체의 마지막 식 타입이 반환 타입이 됩니다.
- 추론할 수 없거나 모호하면 컴파일 에러입니다.

다음 예는 반환 타입을 생략했지만, 마지막 식의 타입에서 `i32`를 추론합니다.

```rok
fn double(x: i32) {              // 반환 타입 생략
    x * 2                         // 추론: i32
}
// 위는 fn double(x: i32) -> i32 와 동일
```

본체가 값을 만들지 않으면(마지막 식이 종결자로 끝나면) 반환 타입은 unit(`()`)입니다.

```rok
fn print_msg(s: str) {
    print(s);                     // 마지막에 종결자: unit 반환
}
```

`pub`나 `pub(pkg)` 함수는 다른 모듈이 호출하므로 시그니처를 적어 두며, 본체를 고쳐도 시그니처는 바뀌지 않습니다. 여기서 `->`를 생략하면 본체가 무엇을 하든, 반환하지 않는 본체라도 `()`입니다. 본체가 값을 내면(마지막 식이나 `return v`의 타입이 `()`가 아니면) 컴파일 에러이고, 에러는 적어야 할 타입을 알려 줍니다.

```rok
pub fn log(s: str) {              // () 반환
    print(s);
}

pub fn double(x: i32) {           // ERROR: 본체가 i32 를 냄, `-> i32` 를 적을 것
    x * 2
}

pub fn double2(x: i32) -> i32 {   // OK
    x * 2
}
```

고유 `impl`의 메서드는 자기 가시성에 따라 같은 규칙을 따릅니다. 트레잇 `impl`의 메서드는 반환 타입이 선언과 같아야 하므로 `->`를 트레잇 선언처럼 적습니다([2.9.1](../002-types/209-impl.md)). 트레잇에서 타입을 받는 것은 타입을 적지 않은 매개변수뿐입니다([4.2.2](402-parameters.md#422-타입-생략)).

반환 타입을 명시했는데 함수 본체가 그 타입과 맞지 않으면 컴파일 에러입니다.

```rok
fn bad() -> i32 {
    "hello"                       // ERROR: str 이 i32 아님
}
```

## 4.1.4. 매개변수 타입 생략

모듈 비공개 함수나 `pub(sub)` 함수에서 매개변수에 타입을 적지 않으면 암묵 제네릭이 됩니다([`any` 슬롯](../002-types/202-any-slot.md), [제네릭](../002-types/210-generics.md)).

```rok
fn id(x) {                        // x: T (제네릭)
    x
}
// 호출 지점의 타입에 따라 단형화
let a = id(42i32);                // id<i32>
let b = id("hi");                 // id<str>
```

`pub`나 `pub(pkg)` 함수는 모든 매개변수의 타입을 적습니다. 타입이 없는 매개변수는 컴파일 에러입니다([4.2.2](402-parameters.md#422-타입-생략)).

매개변수를 동적 슬롯으로 두려면 `any`를 명시적으로 적습니다.

```rok
fn dyn_id(x: any) {               // x 는 any 슬롯 (런타임 디스패치)
    x
}
```

## 4.1.5. 본체

함수 본체는 [블록 표현식](../003-expressions/309-block-expressions.md)의 규칙을 그대로 따릅니다. 성질은 다음과 같습니다.

- 문장을 0개 이상 담습니다.
- 마지막 식은 종결자 없이 0개 또는 1개 둘 수 있습니다. 그 식의 값이 함수의 반환값이 됩니다.
- 마지막에 `return`([`return`](403-return.md))으로 명시 반환할 수 있습니다.

## 4.1.6. 빈 본체

본체가 빈 함수는 unit을 반환합니다.

```rok
fn nothing() { }                  // 본체 비어 있음: unit 반환
```

## 4.1.7. 반환 타입과 본체 마지막 식

반환 타입을 명시한 함수에서 마지막 식을 종결자 없이 두면 그 식이 반환값이 됩니다.

```rok
fn f() -> i32 {
    42i32                         // 마지막 식이 반환값
}
```

위 예에서 `42i32`는 종결자가 없는 식이므로 함수의 반환값이 됩니다. 종결자를 붙여 `42i32;`로 쓰면 값이 버려지고 본체는 unit을 반환합니다. 그러면 명시한 `-> i32`와 맞지 않으므로 컴파일 에러입니다.

```rok
fn f() -> i32 {
    42i32;                        // ERROR: 종결자가 있어 값이 폐기됨
                                  // (반환 타입 i32 와 안 맞음)
}
```

## 4.1.8. 함수 본체의 명시적 `return`

본체 안에서는 `return` 키워드로 명시 반환할 수 있습니다. 자세한 내용은 [`return`](403-return.md)에서 정의합니다.

```rok
fn early(x: i32) -> i32 {
    if x < 0 { return -1; }
    x * 2
}
```

## 4.1.9. 가시성

함수에 가시성 키워드(`pub`, `pub(pkg)`, `pub(sub)`)를 붙이면 함수가 바깥에 드러나는 범위를 정할 수 있습니다.

```rok
pub fn area(r: f64) -> f64 { ... }              // 외부 패키지에서 호출 가능
pub(pkg) fn helper(x: i32) -> i32 { ... }       // 같은 패키지 안에서만
fn _internal(s: str) { ... }                     // 모듈 비공개 (기본)
```

각 가시성 단계의 정확한 의미는 [가시성](../005-modules/506-visibility.md)에서 정의합니다.

## 4.1.10. 같은 모듈 내 중복 정의 금지

같은 모듈 안에서 같은 이름의 함수를 두 번 선언하면 컴파일 에러입니다. Rok에는 오버로딩이 없습니다.

```rok
fn add(x: i32, y: i32) -> i32 { x + y }
fn add(x: f64, y: f64) -> f64 { x + y }   // ERROR: 중복
```

타입에 따라 동작을 다르게 하려면 트레잇과 제네릭을 함께 씁니다([`trait`](../002-types/208-trait.md)).

모듈의 함수, `const`, `static`은 값 이름공간 하나를 공유합니다. 그래서 같은 모듈에서 이 셋 중 다른 것과 이름이 같은 `const`, `static`, 함수도 컴파일 에러입니다. 튜플 struct나 unit struct의 생성자도 이 이름공간에 있습니다([2.3.7](../002-types/203-struct.md#237-같은-모듈-내-중복-정의-금지)). `fn Point`, `const Point`, `static Point` 옆의 `struct Point(i32, i32);`는 중복 정의입니다. 인자 개수(arity)나 `async` 여부, 타입 매개변수가 달라도 두 함수는 구별되지 않습니다. 인자 개수를 다르게 받으려면 한 함수의 뒤쪽 매개변수에 기본값을 줍니다([4.2.3](402-parameters.md#423-기본값)).

## 4.1.11. fn 수식어의 순서와 조합

함수에 붙는 수식어(`unsafe`, `async`, `yield`)는 **다음 순서로 적어야** 합니다.

```
unsafe → async → yield
```

순서를 어기면 구문 에러입니다. 같은 수식어를 두 번 적을 수도 없습니다. 문법에서 `unsafe?`의 `?`는 0개 또는 1개만 허용합니다.

```rok
unsafe async yield fn ok() -> i32 { ... }   // OK: 정해진 순서
async unsafe fn bad1() { ... }           // ERROR: async 가 unsafe 앞에 옴
yield async fn bad2() { ... }            // ERROR: yield 가 async 앞에 옴
unsafe unsafe fn bad3() { ... }          // ERROR: unsafe 중복
```

이 규칙 덕분에 같은 의미의 함수는 항상 같은 모양으로 적힙니다. 그래서 포매터와 린터가 단순해지고 사람이 읽기도 쉬워집니다.

### 가능한 8개 조합

세 수식어가 모두 선택이므로 조합은 8개입니다. 각 조합은 다음과 같습니다.

| 조합                    | 호출 결과                              | 의미                 | 정의 위치                          |
|-------------------------|----------------------------------------|----------------------|------------------------------------|
| `fn`                    | `R`                                    | 일반 함수            | [4.1](401-function-declaration.md) |
| `unsafe fn`             | `R`                                    | unsafe 함수          | [4.15](415-unsafe.md)              |
| `async fn`              | `AsyncFuture<R>`                   | 비동기 함수          | [4.12](412-async.md)               |
| `yield fn`              | `GenIter<R>`                       | 제너레이터           | [4.11](411-generators.md)          |
| `async yield fn`        | `AsyncStream<R>`                   | 비동기 스트림        | [4.13](413-stream.md)              |
| `unsafe async fn`       | `AsyncFuture<R>`                   | unsafe 비동기 함수   | (각 수식어의 합)                  |
| `unsafe yield fn`       | `GenIter<R>`                       | unsafe 제너레이터    | (각 수식어의 합)                  |
| `unsafe async yield fn` | `AsyncStream<R>`                   | unsafe 비동기 스트림 | (각 수식어의 합)                  |

`unsafe`가 붙은 조합의 의미는 단순합니다. 함수 본체 전체가 unsafe 범위가 됩니다(4.15의 정의). 결과 타입과 코루틴 동작은 함께 붙은 다른 수식어(`async` / `yield`)의 정의를 그대로 따릅니다. `unsafe`가 더하는 것은 본체 안에서 `unsafe` 블록이 허용하는 연산뿐입니다([4.15.2](415-unsafe.md#4152-unsafe----블록-표현식)). `unsafe fn` 호출, 원시 포인터 역참조, `union` 필드 읽기, 넘겨준 뒤의 값 사용이 그것입니다.

### 코루틴 함수의 결과 타입

`async` / `yield`가 붙은 함수(블록, 클로저 포함)를 호출하면 표준 라이브러리 타입의 값이 나옵니다. 선언의 `-> R`에는 원소 / 결과 타입 `T`를 적습니다.

| 형태             | 결과 타입         | 구현 트레잇            | `throws (E)`일 때        |
|------------------|-------------------|------------------------|---------------------------|
| `yield fn`       | `GenIter<T>`      | `Iterator<Item = T>`   | `TryGenIter<T, E>`        |
| `async fn`       | `AsyncFuture<T>`  | `Future<Output = T>`   | `TryAsyncFuture<T, E>`    |
| `async yield fn` | `AsyncStream<T>`  | `Stream<Item = T>`     | `TryAsyncStream<T, E>`    |

이 타입들은 `iter`(`GenIter`, `TryGenIter`), `future`(`AsyncFuture`, `TryAsyncFuture`), `stream`(`AsyncStream`, `TryAsyncStream`)에 있고 prelude에는 없습니다. 모듈을 거쳐(`future::AsyncFuture<T>`) 쓰거나 `use`로 가져옵니다. `Try*` 변형은 에러를 다음 진행 지점에서 던집니다. 예를 들어 `TryGenIter<T, E>`의 `next`는 `next(ref self) throws (E) -> T | none`이고, `TryAsyncFuture`는 `try await`에서 던집니다. 이 변형은 일반 `Iterator` / `Future` / `Stream` 트레잇을 구현하지 않습니다([4.14.10](414-throws.md#41410-throws가-합-타입-시그니처에-미치는-영향)). 대신 `TryGenIter`는 `TryIterator`를, `TryAsyncStream`은 `TryStream`을 구현합니다([stdlib 2.7.13](../../refs-standard-libraries/200-core/207-iter.md#2713-tryiterator-실패할-수-있는-반복), [2.8.11](../../refs-standard-libraries/200-core/208-stream.md#2811-trystream-실패할-수-있는-스트림)).

```rok
async fn fetch_user(id: i64) -> User { ... }
let f: future::AsyncFuture<User> = fetch_user(42i64);

yield fn lines(path: str) throws (IoError) -> str { ... }
let it: iter::TryGenIter<str, IoError> = lines("a.txt");
```

### 클로저와 블록 형태

`async`와 `yield`의 클로저 / 블록 형태에도 같은 순서 규칙을 적용합니다. 예를 들어 Stream을 만드는 클로저는 항상 `async yield ||`로 적습니다. `yield async ||`는 클로저 형태가 아닙니다. 컴파일러는 이를 `async ||` 클로저에 `yield`를 적용한 것으로 읽으며, 이는 에러(E0059, 제너레이터 밖의 `yield`)이고 help가 `async yield ||`를 알려 줍니다. 클로저와 블록 형태의 자세한 내용은 각 수식어 문서에 있습니다.

## 4.1.12. 매개변수 / 반환 / 본체 사이의 평가 순서

호출할 때의 평가 순서는 다음과 같습니다.

1. 인자 표현식을 평가합니다(좌→우 순서).
2. 매개변수를 매칭하고 함수 본체를 평가하기 시작합니다.
3. 본체를 평가합니다. 끝나면 마지막 식의 값이나 `return`의 값이 결과가 됩니다.
4. 호출 지점에 결과를 전달합니다.

세부 평가 순서는 [매개변수](402-parameters.md)의 매개변수 평가 순서 절에서 정의합니다.

## 4.1.13. 불투명 반환 타입: `some Trait`

```
opaque_ret ::= "some" (bound | "(" bound ("+" bound)* ")")
bound      ::= type_path | fn_type
```

함수의 반환 타입(`->` 뒤, `throws (E) ->` 뒤 포함)은 `some Tr`일 수 있습니다. `Tr`을 가진 구체 타입 하나를 뜻하며, 본문이 고르고 호출자에게는 숨깁니다. `Tr`은 바운드처럼 연관 타입 바인딩을 쓴 트레잇(`some Iterator<Item = str>`)이거나 함수 타입(`some fn(i32) -> i32`)입니다. 바운드가 여럿이면 괄호에 넣습니다: `some (Iterator<Item = i32> + Clone)`.

```rok
pub fn evens(n: i32) -> some Iterator<Item = i32> {
    (0..n).filter(|x| x % 2 == 0)
}

fn adder(k: i32) -> some fn(i32) -> i32 {
    |x| x + k                     // x: i32, 바운드에서
}

let total = evens(10).map(|x| x * x).sum();
```

- 구체 타입은 본문 값의 타입이며, `pub` 함수에서도 [4.1.3](#413-반환-타입-생략)처럼 추론합니다. 호출은 정적입니다. [제네릭](../002-types/210-generics.md)처럼 태그도 박싱도 없습니다.
- 함수마다 구체 타입은 하나입니다. 반환 경로의 타입이 다르면 에러(`the return paths of `f` have different types ..`)이고, 도움말은 `` to return several types, write `-> any Tr` ``입니다.
- 구체 타입은 모든 바운드를 가져야 합니다. 아니면 바운드 자리에 `` `f` returns `ListIter<i32>`, which is not `Iterator<Item = str>` `` 에러가 납니다.
- 호출자는 바운드만 봅니다. 바운드(와 그 슈퍼트레잇)의 메서드를 부르고, 바운드가 반복자면 반복하고, 같은 바운드의 제네릭 매개변수 자리에 넘기고, 바운드가 함수 타입이면 호출할 수 있습니다. 구체 타입을 이름 붙이거나 쓰면 에러입니다: `` no method `contains` on `some Iterator<Item = i32>`: callers see only its bounds ``, `` `let b` expects `Range<i32>`, found `some Iterator<Item = i32>` ``.
- 같은 함수는 늘 같은 타입을 반환하므로 두 호출의 결과는 한 타입입니다. 다른 함수의 결과는 다른 타입입니다.
- `some`은 함수의 반환 타입이나 매개변수 타입([4.1.14](#4114-some-매개변수))에 씁니다. 다른 자리(`let` 주석, 필드, 타입 인자, 클로저 매개변수)에서는 무엇을 쓸지 알려 주는 에러입니다: 변수는 주석 없이, 필드나 타입 인자는 제네릭 `<T: Tr>`, 여러 타입의 값은 `any Tr`.
- 패키지를 넘을 때는 함수 본문이 패키지 인터페이스와 함께 가므로 의존하는 빌드가 구체 타입을 압니다. 그 사용자에게는 여전히 바운드만 보입니다.
- `Fn` 트레잇은 없습니다. 함수 바운드는 함수 타입 `fn(A) -> R`입니다.

## 4.1.14. `some` 매개변수

매개변수의 타입은 `some Tr`일 수 있습니다(바운드는 [4.1.13](#4113-불투명-반환-타입-some-trait)과 같습니다). 이름 없는 타입 매개변수입니다. `fn show(x: some Display)`는 `T`에 이름이 없는 `fn show<T: Display>(x: T)`입니다.

```rok
fn show(x: some Display) -> str { f"<{x}>" }
fn both(a: some Display, b: some Display) -> str { f"{a}{b}" }
fn count(it: some Iterator<Item = i32>) -> usize { it.count() }
fn twice(f: some fn(i32) -> i32, x: i32) -> i32 { f(f(x)) }

let s = both(1, 2.5);            // a: i64, b: f64
let n = count([1, 2, 3].iter());
let t = twice(|x| x * 3, 2);     // x: i32, 바운드에서
```

- 호출마다 매개변수의 정적 타입은 하나, 인자의 타입입니다. [제네릭](../002-types/210-generics.md)처럼 태그도 박싱도 없습니다. 인자는 모든 바운드를 가져야 합니다.
- `some` 매개변수는 저마다 따로인 타입 매개변수입니다. `both(1, 2.5)`에서 `a`와 `b`의 타입은 다릅니다.
- 본문은 이름 있는 매개변수 `T: Tr`처럼 바운드만 봅니다.
- `some fn(A) -> R`은 함수 값(함수나 클로저)을 받고, 클로저의 매개변수는 그 타입을 받습니다.
- 자유 함수, 메서드, 트레잇 메서드가 이름 있는 타입 매개변수와 함께 `some` 매개변수를 가질 수 있습니다.
- 그 타입에는 이름이 없습니다. 호출의 `::<..>`는 이름 있는 타입 매개변수만 줍니다. 더 쓰면 에러이고, 도움말은 `some` 매개변수의 타입은 인자의 타입이니 인자를 넘기거나, `::<..>`로 고르려면 `<T: Tr>`를 선언하라고 합니다.

`any Tr`과 `some Tr`은 짝입니다. `any Tr`은 동적 슬롯으로, `Tr`을 가진 아무 타입의 값을 그때그때 담고 호출은 실행 중에 디스패치합니다([2.2](../002-types/202-any-slot.md)). `some Tr`은 숨긴 정적 타입 하나로, 반환이면 피호출자의 본문이, 매개변수면 호출의 인자가 정하고 호출은 정적으로 디스패치합니다.
