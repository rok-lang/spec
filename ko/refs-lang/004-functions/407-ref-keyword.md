# 4.7. `ref` 키워드

`ref` 는 변수 바인딩에 대한 reference 를 뜻하는 단일 키워드다. 세 위치에서 일관된 의미로 등장한다.

## 4.7.1. 세 등장 위치

```
1. 함수 매개변수:    fn f(ref x: i32) { x = ... }
2. 클로저 캡처 리스트: |...| { ... } 앞의 [ref x]
3. 호출 시:           f(ref x)
```

세 위치 모두 변수 바인딩 자체에 대한 참조라는 같은 의미다. 한 메커니즘으로 통일된다.

## 4.7.2. 함수 매개변수

```nok
fn increment(ref n: i32) {
    n = n + 1;
}

fn swap(ref a: i32, ref b: i32) {
    let t = a;
    a = b;
    b = t;
}
```

- 매개변수 이름 앞에 `ref`.
- 본체에서 그 매개변수에 대한 재할당이 호출자의 변수에 반영된다.
- 매개변수 자체의 타입 (위 예의 `i32`) 은 일반과 같다.

> [!IMPORTANT]
> `ref` 는 호출자가 호출 지점에 명시한다. 명시 없이 호출하면 컴파일 에러. 어떤 변수가 바뀔 수 있는지 시각적으로 드러나게 한다.

## 4.7.3. 호출 시 명시 (C# 식)

```nok
let x = 5i32;
let y = 10i32;
increment(ref x);                 // x = 6
swap(ref x, ref y);
```

- 함수 매개변수가 `ref` 로 선언된 그 자리에 호출자가 `ref` 를 명시해야 한다.
- `ref` 명시 없이 호출하면 컴파일 에러.

이 강제는 호출 지점에서 어떤 변수가 바뀔 수 있는지 시각적으로 드러나게 한다. C# 의 `ref` 패턴과 같은 동기.

```nok
fn read(x: i32) { ... }
fn write(ref x: i32) { ... }

read(a);                          // 일반 — a 가 바뀌지 않음이 명확
write(ref a);                      // ref — a 가 바뀔 수 있음이 명확
```

## 4.7.4. 클로저 캡처 리스트

```nok
let count = 0;
let inc = [ref count] || { count = count + 1 };
```

[캡처 의미론](406-capture-semantics.md) 에서 정의된 `ref` 캡처. 캡처 리스트 안의 `ref` 는 외부 변수의 바인딩을 참조한다.

## 4.7.5. 메서드 receiver 의 예외

[`impl`](../002-types/209-impl.md) 의 메서드는 `ref self` 매개변수를 가질 수 있다.

```nok
impl Counter {
    fn increment(ref self) { self.count = self.count + 1; }
}
```

이 경우 호출 지점에서는 `ref` 명시가 필요 없다.

```nok
let c = Counter { count: 0 };
c.increment();                    // 'ref' 명시 없음 — 메서드 receiver 의 자동 처리
```

이는 일반 `ref` 매개변수와 다른 메서드 receiver 의 특수 규칙이다. 메서드 호출 문법 자체가 receiver 를 자동 [dispatch](../000-meta/007-glossary.md#dispatch-디스패치) 하는 메커니즘이라, 추가 명시는 시각적 잡음이 된다.

```nok
fn external_inc(ref c: Counter) { ... }
external_inc(ref c);              // 일반 함수 호출은 'ref' 명시가 필요
```

## 4.7.6. `ref` 의 타입 측면

`ref` 는 수식어이지 타입이 아니다. [타입 표현식](../002-types/201-type-expressions.md) 어디에도 등장하지 않는다.

```nok
let x: ref i32 = ...;             // ERROR — ref 는 타입이 아님
struct S { x: ref i32; }          // ERROR — 같은 이유
List<ref i32>                      // ERROR — 같은 이유
```

`ref` 는 다음 위치에만 등장할 수 있다.

- 함수와 메서드의 매개변수 선언.
- 클로저 캡처 리스트.
- 함수와 메서드의 호출 인자.

## 4.7.7. 참조 자체를 일급 값으로 취급할 수 없음

```nok
let r = ref x;                    // ERROR — ref 식 자체는 표현식이 아님
let r: SomePointerType = ref x;   // ERROR
```

`ref` 는 값을 만드는 식이 아니다. 호출, 매개변수, 캡처 리스트에 붙는 수식어 토큰일 뿐이다.

일급 참조가 필요한 경우 (예: [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface)) 는 [타입 표현식](../002-types/201-type-expressions.md) 의 원시 포인터 타입 (`Ptr<T>` 와 `ConstPtr<T>`) 을 사용한다. 이는 [표준 라이브러리 계층](../000-meta/005-stdlib-tiers.md) 의 `unsafe` 영역.

## 4.7.8. 값 pass 와 `ref` pass 의 의미론 정리

같은 호출 시점의 두 형태:

```nok
fn by_value(x: i32) { x = 10i32; }
fn by_ref(ref x: i32) { x = 10i32; }

let a = 5i32;
by_value(a);                      // a = 5 (변화 없음)
by_ref(ref a);                    // a = 10
```

| 측면                      | 값 pass (기본) | `ref` pass (`ref`) |
| ------------------------- | -------------- | ------------------ |
| 매개변수 안 재할당의 효과 | 로컬만         | 호출자의 변수      |
| 명시 호출 토큰            | 없음           | `ref` 필수         |
| 시각 신호                 | 일반 호출      | 변경 가능 표시     |

## 4.7.9. 힙 객체와의 결합

힙 객체 (`struct`, List, Dict 등) 의 필드나 원소 변경은 일반 값 pass 에서도 호출자에게 보인다 (공유 참조).

```nok
fn modify_fields(p: Point) {
    p.x = 10.0f64;                // 같은 힙 객체의 필드 — 호출자도 봄
}

fn replace(ref p: Point) {
    p = Point { x: 99.0, y: 99.0 };   // 호출자의 변수가 새 Point 를 가리킴
}
```

| 시나리오                             | 필요한 키워드       |
| ------------------------------------ | ------------------- |
| 객체의 필드만 변경                   | 일반 값 pass — 충분 |
| 호출자의 변수가 다른 객체를 가리키게 | `ref` 명시          |

이 구분은 [멤버 접근과 메서드 호출](../003-expressions/305-member-access-and-method-call.md) 의 객체 변경 의미와 일관된다.

## 4.7.10. 두 바인딩에 대한 `ref` 동시 사용

```nok
fn swap(ref a: i32, ref b: i32) {
    let t = a;
    a = b;
    b = t;
}

let x = 1i32;
let y = 2i32;
swap(ref x, ref y);               // x = 2, y = 1
```

같은 호출에서 여러 `ref` 매개변수를 둘 수 있다.

호출 지점에서 같은 변수에 두 `ref` 를 넘기는 것은 컴파일러가 가능하면 검출한다 (별칭 분석). 검출되지 않으면 동작은 정의되지 않으며, 사용자 책임이다.

```nok
swap(ref x, ref x);               // 권장 안 함 — 별칭
```

이 검사는 향후 별도 분석이나 진단으로 강화될 수 있다.
