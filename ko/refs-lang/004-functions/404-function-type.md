# 4.4. 함수 타입

함수와 클로저는 값이며, 그 값에는 타입이 있습니다. 본 문서에서는 [타입 표현식](../002-types/201-type-expressions.md) 의 함수 타입 형태에 대해 자세히 다루어 보겠습니다.

## 4.4.1. 형태

함수 타입은 다음과 같은 모양을 가집니다.

```rok
fn(매개변수타입들) -> 반환타입
fn(i32, i32) -> i32
fn(str) -> bool
fn() -> ()
```

기본적으로 `fn` 키워드 뒤에 괄호로 둘러싼 매개변수 타입 목록이 오고, 그 뒤로 `->` 와 반환 타입이 따라옵니다. 이때 매개변수 이름은 적지 않고 타입만 기재한다는 점에 유의합니다. 또한 반환 타입을 생략하면 unit 으로 간주되므로, `fn(i32, i32)` 는 사실상 `fn(i32, i32) -> ()` 와 동일하게 취급됩니다.

에러를 던질 수 있는 함수의 타입은 함수 선언과 같은 자리에 [`throws`](414-throws.md) 를 적습니다.

```rok
fn(str) throws -> i32               // untyped — throws (any Error)
fn(str) throws (ParseError) -> i32  // typed
fn(Path) throws (IOError)           // 반환 타입 생략 — unit
```

`throws` 는 함수 타입의 일부이므로, throwing 함수 타입의 값을 호출할 때에도 `try` 로 표시해야 합니다. `rethrows` 는 함수 선언에만 쓰며 함수 타입에는 쓰지 않습니다.

## 4.4.2. 함수 / 클로저 값의 타입

함수든 클로저든 모두 같은 형식의 함수 타입으로 표현됩니다.

```rok
fn add(x: i32, y: i32) -> i32 { x + y }
let f = add;                                    // f: fn(i32, i32) -> i32

let g = |x: i32| -> i32 { x * 2 };              // g: fn(i32) -> i32
let h: fn(i32) -> i32 = |x| x + 1;              // 명시 타입
```

위에서 보시다시피 함수와 클로저는 동일한 함수 타입으로 표현되므로, 함수 타입을 요구하는 자리라면 어느 쪽이든 적합하게 들어갈 수 있습니다.

## 4.4.3. 변수에 대입

함수 타입이 명시된 변수에 함수나 클로저를 자유롭게 대입할 수 있습니다.

```rok
let f: fn(i32) -> i32 = |x| x * 2;
let g: fn(str, i32) -> str = format;
```

이처럼 함수 타입 슬롯에는 함수도 클로저도 모두 대입이 가능합니다.

## 4.4.4. 함수 인자 / 반환값으로 사용

함수 타입은 다른 함수의 인자나 반환값으로 활용될 때 그 진가를 발휘합니다.

```rok
fn apply(f: fn(i32) -> i32, x: i32) -> i32 { f(x) }
fn make_adder(n: i32) -> fn(i32) -> i32 { |x| x + n }
```

첫 번째 패턴은 고차 함수로서 함수를 인자로 받는 형태이고, 두 번째 패턴은 함수를 결과로 반환하는 형태입니다. 이 두 가지 패턴이야말로 함수 타입의 핵심 쓰임새라고 할 수 있습니다.

## 4.4.5. 컬렉션 안의 함수

함수 타입도 다른 일반 타입과 마찬가지로 컬렉션의 원소 타입이 될 수 있습니다.

```rok
let handlers: List<fn(Event) -> ()> = [...];
let dispatch: Dict<str, fn(Request) -> Response> = {...};
```

이렇게 함수 타입을 컬렉션 안에 담아 두면 다양한 동작을 데이터처럼 다룰 수 있게 됩니다.

## 4.4.6. 캡처가 있는 클로저

캡처가 있는 클로저 또한 같은 함수 타입으로 표현됩니다. 사용자 입장에서 보기에는 동일한 인터페이스로 보이게 됩니다.

```rok
let n = 10;
let f: fn(i32) -> i32 = |x| x + n;     // n 을 캡처하지만 타입은 fn(i32) -> i32
```

캡처는 클로저 객체에 내부적으로 부속되며, 호출 시점에 자동으로 사용됩니다. 따라서 사용자 코드는 캡처의 유무를 신경 쓸 필요가 없으며, 캡처는 타입에도 드러나지 않습니다.

참고로 Rust 의 `Fn`, `FnMut`, `FnOnce` 트레잇 구분은 rok 에는 존재하지 않습니다. 캡처가 값 기반을 기본으로 하기 때문에 의미상의 복잡성이 줄어들기 때문입니다. 자세한 캡처 의미에 대해서는 [클로저](405-closure.md) 에서 정의하고 있으니 참조합니다.

### `Send` / `Sync` bound

함수 타입 끝에 `+ Send` / `+ Sync` 를 붙이면, 그 값이 스레드 경계를 넘을 수 있음을 타입에 적습니다 ([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync--스레드-경계-marker)).

```rok
fn(i32) -> i32 + Send
fn() + Send + Sync
```

- 클로저 리터럴이 `+ Send` 자리에 오면, 컴파일러는 캡처한 값이 모두 `Send` 인지 검사합니다 (`+ Sync` 는 `Sync` 인지).
- `fn` 타입 변수를 넣으려면 그 변수의 타입에 이미 같은 bound 가 있어야 합니다. `fn() -> T` 값은 `fn() -> T + Send` 자리에 들어가지 않습니다.
- 이름 있는 함수는 캡처가 없으므로 항상 `Send + Sync` 입니다.

```rok
let c: Cell<i32> = Cell::new(0);
let f: fn() + Send = || print("hi");     // OK — 캡처 없음
let g: fn() + Send = || c.set(1);        // ERROR — Cell 은 Send 가 아님
let h: fn() = || print("hi");
let k: fn() + Send = h;                  // ERROR — h 의 타입에 + Send 없음
```

## 4.4.7. 매개변수 / 반환의 점진성

함수 타입에 제네릭 매개변수를 두는 것은 본 표기법에서 직접 표현되지 않습니다. 제네릭 함수의 값은 호출 지점에서 단형화되어 구체적인 타입을 가진 함수 값이 됩니다.

```rok
fn id<T>(x: T) -> T { x }

let f = id;                       // ERROR — id 자체는 제네릭, 어떤 T 인지 결정 안 됨
let f: fn(i32) -> i32 = id;       // OK — i32 로 단형화
let g = id::<i32>;                // OK — turbofish 로 단형화
```

만약 함수 타입에 동적 슬롯을 두고자 한다면 매개변수나 반환에 `any` 를 적어 주시면 됩니다.

```rok
fn(any) -> any
```

이 형태는 어떤 타입이든 받고 어떤 타입이든 반환할 수 있는 함수의 타입입니다. 다만 이 경우에는 동적 디스패치 비용이 든다는 점을 염두에 두셔야 합니다.

## 4.4.8. divergent 함수의 타입

[반환](403-return.md) 에서 다룬 divergent 함수의 타입은 반환 타입 자리에 `!` 를 적습니다 (`fn() -> !`). `fn(A) -> !` 값은 매개변수가 같은 임의의 `fn(A) -> R` 자리에 들어갈 수 있습니다.

```rok
let f: fn() -> i32 = || panic("never");     // fn() -> ! 값 → fn() -> i32 자리
```

### 함수 값의 표현

함수 값은 `{code, env}` 쌍입니다. `code` 는 코드 포인터이고 `env` 는 캡처 환경입니다. 이름 있는 함수의 값은 `env` 가 null 입니다. `extern "C"` 선언의 함수 타입 매개변수는 예외로, `env` 없는 C 함수 포인터이며 이름 있는 함수만 받습니다 ([5.9.4](../005-modules/509-extern-block.md#594-ffi-호환-타입)).

## 4.4.9. 함수 타입의 부분 집합 관계 (없음)

> [!IMPORTANT]
> Rust 나 TypeScript 같은 언어에서 볼 수 있는 함수 타입 부분형 (subtyping) 은 rok 에 없습니다. 즉, 한 함수 타입을 더 일반적이거나 더 구체적인 함수 타입으로 자동 변환하지 않습니다. 매개변수와 반환 타입은 정확히 일치해야 합니다.

```rok
let f: fn(i32 | f64) -> i32 = |x: i32| x;     // ERROR — 매개변수 타입이 정확히 일치 안 함
```

다시 한번 강조하면 매개변수와 반환 타입은 정확히 일치해야 합니다.

예외는 4.4.8 의 divergent 반환과 에러 채널뿐입니다. 매개변수와 반환 타입이 같다면, 던지는 에러가 더 적은 함수 값은 더 많이 던질 수 있는 함수 타입 자리에 넣을 수 있습니다. 이 변환 (non-throwing → throwing, 좁은 에러 → 넓은 에러) 은 컴파일러가 만드는 어댑터로 구현될 수 있습니다.

```rok
fn parse_typed(s: str) throws (ParseError) -> i32 { ... }

let p: fn(str) throws (ParseError) -> i32 = |s| s.len() as i32;   // OK — non-throwing → throwing
let q: fn(str) throws -> i32 = parse_typed;   // OK — throws (ParseError) → throws (any Error)
let r: fn(str) -> i32 = parse_typed;          // ERROR — throwing → non-throwing
```

`throws (A)` 함수 값은 `throws (A | B)` 자리에 넣을 수 있고, `A` 가 `Error` 를 구현하면 untyped `throws` 자리에도 넣을 수 있습니다.

이 변환은 함수 타입 변수와 캡처가 있는 클로저를 포함한 모든 함수 값에 적용됩니다. 컴파일러는 원래 값을 감싸는 어댑터 `{code, env}` 를 만듭니다.

```rok
let g: fn(str) throws (ParseError) -> i32 = parse_typed;
let h: fn(str) throws -> i32 = g;             // OK — 어댑터로 감쌈
```

이러한 결정의 동기는 명시성을 우선시하는 데 있습니다. 함수 변환은 [형 변환](../002-types/214-cast-and-conversion.md) 의 명시적인 형태로 사용자가 직접 적도록 하는 것이 rok 의 방침입니다.

## 4.4.10. 함수 타입의 평가

함수 타입은 값을 만들어 내지 않는 타입입니다. 어떤 모양의 함수가 받아들여질지에 대한 명세일 뿐이며, 별도의 평가 동작은 존재하지 않습니다.

함수의 호출 동작은 [멤버 접근과 메서드 호출](../003-expressions/305-member-access-and-method-call.md) 에서 설명하는 일반 호출 규칙 (`f(...)` 형태) 을 따릅니다.

## 4.4.11. 매개변수 이름의 부재

함수 타입에는 매개변수 이름이 들어가지 않습니다. 정의부 (`fn name(x: i32)`) 에는 이름이 있지만, 타입 표기 (`fn(i32)`) 에서는 위치만 남게 됩니다.

```rok
fn(i32, i32) -> i32                // OK
fn(x: i32, y: i32) -> i32          // ERROR — 함수 타입에 이름 없음
```

호출 시 named 인자 ([매개변수](402-parameters.md)) 는 함수 정의의 이름을 참조하게 됩니다. 따라서 함수 타입을 거쳐 호출하는 경우에는 named 인자가 작동하지 않으며, 값을 통한 호출에서는 위치 인자만 사용 가능합니다.

```rok
fn make_user(name: str, age: i32) -> User { ... }
let f: fn(str, i32) -> User = make_user;

make_user(name: "A", age: 30i32);    // named OK — 정의 이름 사용
f("A", 30i32);                        // 위치 OK
f(name: "A", age: 30i32);             // ERROR — 타입에 이름 없음
```
