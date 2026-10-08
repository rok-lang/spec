# 4.2. 매개변수

함수의 매개변수 형식과 호출 지점에서의 인자 매칭 규칙을 정의한다.

```
param           ::= self_param | regular_param
self_param      ::= "ref"? "self" (":" type_expr)?
regular_param   ::= "ref"? identifier (":" type_expr)? ("=" const_expr)?
```

## 4.2.1. 기본 형태

```nok
fn f(x: i32, y: i32) -> i32 { x + y }
```

각 매개변수는 `이름: 타입` 형태. 콤마로 구분.

매개변수 이름은 [식별자](../001-lexical/104-identifiers.md) 의 형식. 타입은 [타입 표현식](../002-types/201-type-expressions.md).

## 4.2.2. 타입 생략

타입을 적지 않으면 제네릭으로 처리된다 ([제네릭](../002-types/210-generics.md) 의 단형화 규칙).

```nok
fn id(x) { x }                                   // x 는 제네릭 — 호출 시 단형화
fn pair(x, y) { ... }                             // 매개변수 둘 다 제네릭
fn mixed(a: i32, b) { ... }                       // a 는 i32, b 는 제네릭
```

위 `id` 는 다음 명시적 [제네릭](../002-types/210-generics.md) 표기와 동등하다.

```nok
fn id<T>(x: T) -> T { x }
```

매개변수가 동적 슬롯이 되기를 원하면 명시적 `any` ([`any` 슬롯](../002-types/202-any-slot.md)).

```nok
fn handle(x: any) { ... }                         // x 는 any 슬롯
```

## 4.2.3. 기본값

매개변수에 기본값을 줄 수 있다.

```nok
fn retry(n: i32 = 3i32, delay: f64 = 0.1f64) { ... }

retry();                          // n=3, delay=0.1
retry(5i32);                       // n=5, delay=0.1
retry(5i32, 1.0f64);               // n=5, delay=1.0
```

규칙:

- `const` 표현식만 허용. 리터럴, `const` 변수, `const` 함수 호출. [`struct`](../002-types/203-struct.md) 의 필드 기본값과 동일한 제약.
- 호출 지점에서 생략된 매개변수만 평가된다. 매 호출마다 새로 평가.
- 결정론적. 같은 `const` 표현식은 항상 같은 값.
- 런타임 호출 (현재 시각·난수 등) 은 사용 불가. factory 함수 패턴으로 우회.

## 4.2.4. named 인자 호출

호출 지점에서 매개변수 이름으로 인자를 명시할 수 있다.

```nok
fn make_user(name: str, age: i32 = 0i32, email: str = "") -> User { ... }

make_user(name: "Alice", age: 30i32);
make_user(name: "Bob", email: "bob@example.com");      // age 는 기본값
make_user("Charlie", age: 25i32);                       // 첫 인자만 위치, 나머지는 named
```

규칙:

- 위치 인자가 먼저, named 인자가 뒤에.
- 한 호출에서 같은 매개변수에 위치와 named 를 둘 다 주면 에러.
- 모든 위치 인자가 매칭된 뒤, 남은 매개변수는 named 또는 기본값으로 채워야 한다.

## 4.2.5. variadic: `...T`

가변 개수의 인자를 받는 매개변수.

```nok
fn sum(xs: ...i32) -> i32 { ... }

sum();                            // xs = (빈)
sum(1i32);                         // xs = (1)
sum(1i32, 2i32, 3i32);             // xs = (1, 2, 3)
```

규칙:

- 매개변수 마지막 한 개만 variadic 가능.
- 함수 본체 안에서 `xs` 는 [타입 표현식](../002-types/201-type-expressions.md) 의 원소 슬라이스 뷰. 정확한 라이브러리 타입은 표준 라이브러리에서 정의.
- 호출 시 스택에 임시 배열을 만들어 슬라이스 포인터로 전달. 힙 할당 0.
- 본체에서 읽기, 인덱싱, 반복 가능. 변경이 필요하면 라이브러리 메서드로 컬렉션을 새로 만든다.

### 이종 variadic: `...any`

```nok
fn log_all(xs: ...any) { ... }
log_all(1i32, "hi", 3.14f64);     // 다양한 타입 가능
```

매개변수 타입을 `any` 로 두면 이종 인자를 받는 variadic 이 된다.

## 4.2.6. spread: `..args`

호출 지점에서 기존 컬렉션을 펼쳐 인자로 넘기는 형태.

```nok
let args = (1i32, 2i32, 3i32);
foo(..args);                      // foo(1, 2, 3)

let xs: List<i32> = [1i32, 2i32, 3i32];
sum(..xs);                         // sum(1, 2, 3) — variadic 받음
```

규칙:

- 튜플 spread: 컴파일 타임에 길이와 타입 검증.
- 리스트 spread: 런타임 길이. variadic 또는 위치 일치 시 OK.
- Dict spread: named 인자로 변환.
  ```nok
  let opts = {"lr": 0.01, "epochs": 100};
  train(..opts);                  // train(lr: 0.01, epochs: 100)
  ```
- 한 호출에 spread 와 일반 인자 자유 혼합 가능.
  ```nok
  foo(0, ..middle, 10);
  foo(..a, ..b, ..c);
  ```

## 4.2.7. 폐기: `_`

호출 결과를 명시적으로 폐기하는 표기.

```nok
_ = compute_with_side_effect();
```

- `_` 는 [식별자](../001-lexical/104-identifiers.md) 의 단일 밑줄.
- 좌변 자리에서 값을 받지만 어디에도 바인딩하지 않는다.
- 사용자가 의도적으로 결과를 무시한다는 신호.

## 4.2.8. 매개변수 평가 순서

호출 시 인자 표현식은 좌→우 순서로 평가된다.

```nok
foo(a(), b(), c());               // a() 먼저, b() 다음, c() 마지막
```

named 인자도 적힌 순서로 평가.

```nok
make_user(name: f(), age: g());   // f() 먼저, g() 다음
```

이 순서는 부작용이 있는 인자가 예측 가능한 순서로 실행되도록 보장한다.

## 4.2.9. 매개변수 가시 범위

매개변수는 함수 본체의 최상위 스코프에 도입된다. 본체 안에서는 일반 변수처럼 사용한다.

기본값 표현식은 매개변수가 없는 환경에서 평가된다. 같은 함수의 다른 매개변수를 참조할 수 없다.

```nok
fn f(a: i32, b: i32 = a + 1) { ... }   // ERROR — 기본값에서 a 참조 불가
```

이 제약은 [`struct`](../002-types/203-struct.md) 의 필드 기본값 제약과 동일한 동기를 갖는다. 결정론과 예측 가능 비용.

## 4.2.10. 정적 vs 점진의 매개변수 결정

| 적힌 형태 | 의미                                        |
| --------- | ------------------------------------------- |
| `x: i32`  | 정적. 호출 지점에서 i32 만 받음             |
| `x`       | 제네릭. 호출 지점 타입으로 단형화           |
| `x: any`  | 동적 슬롯. 어떤 타입이든 런타임 태그로 받음 |

세 형태가 한 함수 안에서 자유 혼합 가능.
