# 4.4. 함수 타입

함수와 클로저는 값이며, 그 값에는 타입이 있습니다. 이 문서는 [타입 표현식](../002-types/201-type-expressions.md)의 함수 타입 형태를 자세히 다룹니다.

## 4.4.1. 형태

함수 타입의 모양은 다음과 같습니다.

```rok
fn(매개변수타입들) -> 반환타입
fn(i32, i32) -> i32
fn(str) -> bool
fn() -> ()
```

`fn` 키워드 뒤에 괄호로 감싼 매개변수 타입 목록이 옵니다. 그 뒤에 `->`와 반환 타입이 옵니다. 매개변수 이름은 적지 않고 타입만 적습니다. 반환 타입을 생략하면 unit으로 봅니다. 그래서 `fn(i32, i32)`는 `fn(i32, i32) -> ()`와 같습니다.

에러를 던질 수 있는 함수의 타입은 함수 선언과 같은 자리에 [`throws`](414-throws.md)를 적습니다.

```rok
fn(str) throws -> i32               // untyped: throws (any Error)
fn(str) throws (ParseError) -> i32  // typed
fn(Path) throws (IOError)           // 반환 타입 생략: unit
```

`unsafe fn(..) -> R`은 호출할 때 `unsafe`가 필요한 함수의 타입입니다. `unsafe fn`과 C 함수를 값으로 쓰면 이 타입이 됩니다. 안전한 `fn` 값은 `unsafe fn` 자리에 들어갈 수 있지만 반대는 안 됩니다([4.15](415-unsafe.md#unsafe-fn-타입)).

`throws`는 함수 타입의 일부입니다. 그래서 에러를 던지는 함수 타입의 값을 호출할 때도 `try`로 표시해야 합니다. `rethrows`는 함수 선언에만 쓰며 함수 타입에는 쓰지 않습니다.

## 4.4.2. 함수 / 클로저 값의 타입

함수든 클로저든 같은 형식의 함수 타입으로 나타냅니다.

```rok
fn add(x: i32, y: i32) -> i32 { x + y }
let f = add;                                    // f: fn(i32, i32) -> i32

let g = |x: i32| -> i32 { x * 2 };              // g: fn(i32) -> i32
let h: fn(i32) -> i32 = |x| x + 1;              // 명시 타입
```

함수와 클로저는 같은 함수 타입입니다. 그래서 함수 타입이 필요한 자리에는 어느 쪽이든 들어갈 수 있습니다.

## 4.4.3. 변수에 대입

함수 타입을 명시한 변수에는 함수나 클로저를 자유롭게 대입할 수 있습니다.

```rok
let f: fn(i32) -> i32 = |x| x * 2;
let g: fn(str, i32) -> str = format;
```

이처럼 함수 타입 슬롯에는 함수와 클로저를 모두 대입할 수 있습니다.

## 4.4.4. 함수 인자 / 반환값으로 사용

함수 타입은 주로 다른 함수의 인자나 반환값으로 쓰입니다.

```rok
fn apply(f: fn(i32) -> i32, x: i32) -> i32 { f(x) }
fn make_adder(n: i32) -> fn(i32) -> i32 { |x| x + n }
```

첫 번째는 함수를 인자로 받는 고차 함수입니다. 두 번째는 함수를 결과로 반환합니다. 함수 타입은 주로 이 두 패턴에 쓰입니다.

## 4.4.5. 컬렉션 안의 함수

함수 타입도 다른 일반 타입처럼 컬렉션의 원소 타입이 될 수 있습니다.

```rok
let handlers: List<fn(Event) -> ()> = [...];
let dispatch: Dict<str, fn(Request) -> Response> = {...};
```

함수를 컬렉션에 담아 두면 동작을 데이터처럼 다룰 수 있습니다.

## 4.4.6. 캡처가 있는 클로저

캡처가 있는 클로저도 같은 함수 타입으로 나타냅니다. 호출하는 쪽에는 같은 인터페이스로 보입니다.

```rok
let n = 10;
let f: fn(i32) -> i32 = |x| x + n;     // n 을 캡처하지만 타입은 fn(i32) -> i32
```

캡처는 클로저 객체 내부에 붙어 있다가 호출할 때 자동으로 쓰입니다. 그래서 호출하는 코드는 캡처가 있는지 신경 쓰지 않아도 되고, 캡처는 타입에도 드러나지 않습니다.

Rust의 `Fn`, `FnMut`, `FnOnce` 트레잇 구분은 Rok에 없습니다. 캡처가 기본적으로 값 기반이어서 의미가 덜 복잡하기 때문입니다. 캡처 의미의 자세한 내용은 [클로저](405-closure.md)에서 정의합니다.

### `Send` / `Sync` 바운드

함수 타입 끝에 `+ Send` / `+ Sync`를 붙이면 그 값이 스레드 경계를 넘을 수 있다는 사실을 타입에 적습니다([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync-스레드-경계-마커)).

```rok
fn(i32) -> i32 + Send
fn() + Send + Sync
```

- 클로저 리터럴이 `+ Send` 자리에 오면, 캡처한 값은 모두 `Send + Sync`여야 합니다. 캡처는 만든 쪽도 계속 쓰는 핸들이므로 두 스레드가 모두 접근할 수 있습니다. 그래서 `fn(..) + Send` 값은 `Sync`이기도 합니다.
- spawn 함수(`task::spawn`, `thread::spawn`, `par::*`)는 `+ Send` 클로저와 future를 받습니다. 그래서 이 함수로 띄운 코드는 `Sync`가 아닌 값을 캡처할 수 없습니다.
- `fn` 타입 변수를 이 자리에 쓰려면 그 변수의 타입에 이미 같은 바운드가 있어야 합니다. `fn() -> T` 값은 `fn() -> T + Send` 자리에 들어가지 않습니다.
- 이름 있는 함수는 캡처가 없으므로 항상 `Send + Sync`입니다.

```rok
let c: Cell<i32> = Cell::new(0);
let f: fn() + Send = || print("hi");     // OK: 캡처 없음
let g: fn() + Send = || c.set(1);        // ERROR: Cell 은 Send 가 아님
let h: fn() = || print("hi");
let k: fn() + Send = h;                  // ERROR: h 의 타입에 + Send 없음
```

## 4.4.7. 매개변수 / 반환의 점진성

이 표기법으로는 함수 타입에 제네릭 매개변수를 직접 둘 수 없습니다. 제네릭 함수의 값은 쓰는 지점에서 단형화되어 구체 타입의 함수 값이 됩니다.

```rok
fn id<T>(x: T) -> T { x }

let f = id;                       // ERROR: id 자체는 제네릭, 어떤 T 인지 결정 안 됨
let f: fn(i32) -> i32 = id;       // OK: i32 로 단형화
let g = id::<i32>;                // OK: turbofish 로 단형화
```

함수 타입에 동적 슬롯을 두려면 매개변수나 반환에 `any`를 적습니다.

```rok
fn(any) -> any
```

이 형태는 어떤 타입이든 받고 어떤 타입이든 반환하는 함수의 타입입니다. 다만 동적 디스패치 비용이 듭니다.

## 4.4.8. divergent 함수의 타입

[반환](403-return.md)에서 다룬 divergent 함수의 타입은 반환 타입 자리에 `!`를 적습니다(`fn() -> !`). `fn(A) -> !` 값은 매개변수가 같은 어떤 `fn(A) -> R` 자리에도 들어갈 수 있습니다. 에러를 던지는 `fn(A) throws (E) -> !`는 `fn(A) throws (E) -> R`에 맞습니다.

```rok
let f: fn() -> i32 = || panic("never");     // fn() -> ! 값 → fn() -> i32 자리
```

### 함수 값의 표현

함수 값은 `{code, env}` 쌍입니다. `code`는 코드 포인터이고 `env`는 캡처 환경입니다. 이름 있는 함수의 값은 `env`가 null입니다. `extern "C"` 선언에서 함수 타입 매개변수의 타입은 `extern "C" fn(..)`입니다. 이 타입은 `env` 없는 C 함수 포인터이고, Rok 함수 값과 서로 변환되지 않습니다([5.9.4](../005-modules/509-extern-block.md#c-함수-포인터-타입)). 메서드 경로(`S::new`, `i32::add`)도 함수 값이 됩니다([3.5.3](../003-expressions/305-member-access-and-method-call.md#메서드-경로-값)).

## 4.4.9. 함수 타입 사이의 부분형 관계(없음)

> [!IMPORTANT]
> Rust나 TypeScript 같은 언어에 있는 함수 타입 부분형(subtyping)은 Rok에 없습니다. 한 함수 타입을 더 일반적이거나 더 구체적인 함수 타입으로 자동 변환하지 않습니다. 매개변수와 반환 타입은 정확히 일치해야 합니다.

```rok
let f: fn(i32 | f64) -> i32 = |x: i32| x;     // ERROR: 매개변수 타입이 정확히 일치 안 함
```

함수 타입 안의 매개변수와 반환 타입은 정확히 일치해야 하며, 합 타입으로 넓히지 않습니다.

예외는 4.4.8의 divergent 반환과 에러 채널뿐입니다. 매개변수와 반환 타입이 같으면, 던지는 에러가 더 적은 함수 값을 더 많이 던질 수 있는 함수 타입 자리에 넣을 수 있습니다. 이 변환(non-throwing → throwing, 좁은 에러 → 넓은 에러)은 컴파일러가 만드는 어댑터로 구현할 수 있습니다.

```rok
fn parse_typed(s: str) throws (ParseError) -> i32 { ... }

let p: fn(str) throws (ParseError) -> i32 = |s| s.len() as i32;   // OK: non-throwing → throwing
let q: fn(str) throws -> i32 = parse_typed;   // OK: throws (ParseError) → throws (any Error)
let r: fn(str) -> i32 = parse_typed;          // ERROR: throwing → non-throwing
```

`throws (A)` 함수 값은 `throws (A | B)` 자리에 넣을 수 있습니다. `A`가 `Error`를 구현하면 에러 타입을 적지 않은 `throws` 자리에도 넣을 수 있습니다.

이 변환은 함수 타입 변수와 캡처가 있는 클로저를 포함한 모든 함수 값에 적용됩니다. 컴파일러는 원래 값을 감싸는 어댑터 `{code, env}`를 만듭니다.

```rok
let g: fn(str) throws (ParseError) -> i32 = parse_typed;
let h: fn(str) throws -> i32 = g;             // OK: 어댑터로 감쌈
```

이렇게 정한 이유는 명시성입니다. Rok의 방침은 함수 변환을 [형 변환](../002-types/214-cast-and-conversion.md)의 명시적 형태로 직접 적는 것입니다.

## 4.4.10. 함수 타입의 평가

함수 타입은 값을 만들지 않는 타입입니다. 어떤 모양의 함수를 받아들일지 정할 뿐이고, 따로 평가 동작이 없습니다.

함수 호출 동작은 [멤버 접근과 메서드 호출](../003-expressions/305-member-access-and-method-call.md)에서 설명하는 일반 호출 규칙(`f(...)` 형태)을 따릅니다.

## 4.4.11. 매개변수 이름의 부재

함수 타입에는 매개변수 이름이 들어가지 않습니다. 정의(`fn name(x: i32)`)에는 이름이 있지만, 타입 표기(`fn(i32)`)에는 위치만 남습니다.

```rok
fn(i32, i32) -> i32                // OK
fn(x: i32, y: i32) -> i32          // ERROR: 함수 타입에 이름 없음
```

호출할 때 named 인자([매개변수](402-parameters.md))는 함수 정의의 이름을 참조합니다. 그래서 함수 타입을 거쳐 호출하면 named 인자가 동작하지 않습니다. 값으로 호출할 때는 위치 인자만 쓸 수 있습니다.

```rok
fn make_user(name: str, age: i32) -> User { ... }
let f: fn(str, i32) -> User = make_user;

make_user(name: "A", age: 30i32);    // named OK: 정의 이름 사용
f("A", 30i32);                        // 위치 OK
f(name: "A", age: 30i32);             // ERROR: 타입에 이름 없음
```
