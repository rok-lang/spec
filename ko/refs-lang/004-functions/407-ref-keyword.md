# 4.7. `ref` 키워드

`ref` 는 "이 자리의 변경이 호출자 (또는 바깥) 의 변수에 반영된다" 는 것을 표시하는 키워드입니다. 매개변수와 호출 인자에서는 copy-in / copy-out 으로, 캡처 리스트에서는 공유 셀로 구현되며, 일급 참조 값은 아닙니다.

## 4.7.1. 세 등장 위치

```
1. 함수 매개변수:    fn f(ref x: i32) { x = ... }
2. 클로저 캡처 리스트: |...| { ... } 앞의 [ref x]
3. 호출 시:           f(ref x)
```

세 위치 모두 "변경이 바깥 변수에 반영된다" 는 같은 의미를 가집니다.

## 4.7.2. 함수 매개변수

```rok
fn increment(ref n: i32) {
    n = n + 1;
}

fn swap(ref a: i32, ref b: i32) {
    let t = a;
    a = b;
    b = t;
}
```

매개변수 이름 앞에 `ref` 를 붙이면, 본체에서 그 매개변수에 대한 재할당이 호출자의 변수에 반영됩니다. 매개변수 자체의 타입(위 예의 `i32`) 은 일반과 같습니다.

### 의미 — copy-in / copy-out

`ref` 매개변수는 copy-in / copy-out 입니다. 호출 시 호출자 자리 (변수, 필드, 원소) 의 값을 매개변수로 복사해 들이고, callee 는 그 값으로 작업하며, 호출이 끝나면 매개변수의 최종 값을 호출자 자리에 다시 씁니다.

- **되쓰기 시점**: 정상 반환할 때뿐 아니라 callee 가 에러를 던질 때도 되씁니다. 따라서 `throw` 전에 한 변경은 호출자에게 보입니다.
- **배타성**: 호출이 진행되는 동안 같은 자리를 다른 경로로 접근할 수 없으며, 컴파일러가 검사합니다. 같은 자리를 `ref` 로 두 번 넘기거나, 한 인자로는 `ref` 로 넘기고 다른 인자로 읽으면 컴파일 에러입니다. 서로 다른 필드 (`ref p.x`, `ref p.y`) 는 겹치지 않으므로 허용됩니다.

```rok
fn bump_then_fail(ref n: i32) throws {
    n = n + 1;
    throw MyError {};
}
let a = 1i32;
try bump_then_fail(ref a) catch _ { };   // a = 2 — throw 때도 되씀
swap(ref a, ref a);                       // ERROR — 같은 자리를 두 번 ref
add(ref a, a);                            // ERROR — ref 로 넘긴 자리를 다른 인자로 읽음
swap(ref p.x, ref p.y);                   // OK — 서로 다른 필드
```

> [!IMPORTANT]
> `ref` 는 호출자가 호출 지점에 명시해야 하며, 명시 없이 호출하면 컴파일 에러가 발생합니다. 이는 어떤 변수가 바뀔 수 있는지 시각적으로 드러나게 하기 위함입니다.

## 4.7.3. 호출 시 명시 (C# 식)

```rok
let x = 5i32;
let y = 10i32;
increment(ref x);                 // x = 6
swap(ref x, ref y);
```

함수 매개변수가 `ref` 로 선언된 자리에서, 그리고 그 자리에서만 호출자도 `ref` 를 명시해야 합니다. `ref` 를 빠뜨리거나 `ref` 가 아닌 매개변수에 `ref` 를 붙이면 컴파일 에러입니다. `ref self` 메서드를 경로로 부를 때도 같은 규칙이 적용됩니다 (`Counter::increment(ref c)`, 4.7.5).

이러한 강제는 호출 지점에서 어떤 변수가 바뀔 수 있는지 시각적으로 드러나게 해 주며, 이는 C# 의 `ref` 패턴과 같은 동기에서 비롯됩니다.

```rok
fn read(x: i32) { ... }
fn write(ref x: i32) { ... }

read(a);                          // 일반 — a 가 바뀌지 않음이 명확
write(ref a);                      // ref — a 가 바뀔 수 있음이 명확
```

## 4.7.4. 클로저 캡처 리스트

```rok
let count = 0;
let inc = [ref count] || { count = count + 1 };
```

[캡처 의미론](406-capture-semantics.md) 에서 정의된 `ref` 캡처에 따라, 캡처 리스트의 `ref` 변수는 공유 셀에 담깁니다 (boxing). 바깥 스코프와 클로저가 같은 셀을 읽고 쓰므로, 클로저가 변수의 스코프보다 오래 살아도 됩니다.

## 4.7.5. 메서드 receiver 의 예외

[`impl`](../002-types/209-impl.md) 의 메서드는 `ref self` 매개변수를 가질 수 있습니다.

```rok
impl Counter {
    fn increment(ref self) { self.count = self.count + 1; }
}
```

이 경우에는 호출 지점에서 `ref` 명시가 필요하지 않습니다.

```rok
let c = Counter { count: 0 };
c.increment();                    // 'ref' 명시 없음 — 메서드 receiver 의 자동 처리
```

이는 일반 `ref` 매개변수와 다른 메서드 receiver 만의 특수 규칙입니다. 메서드 호출 문법 자체가 receiver 를 자동 [dispatch](../000-meta/007-glossary.md#dispatch-디스패치)하는 메커니즘이기 때문에, 추가적인 명시는 시각적 잡음이 되어 버립니다.

```rok
fn external_inc(ref c: Counter) { ... }
external_inc(ref c);              // 일반 함수 호출은 'ref' 명시가 필요
Counter::increment(ref c);        // ref self 메서드를 경로로 부를 때도 ref 필요
```

### `ref self` 의 의미 — copy-in / copy-out

`ref self` 는 copy-in / copy-out 으로 동작합니다. 호출 시 receiver 의 값을 `self` 로 복사해 들이고, 호출이 끝나면 `self` 의 최종 값을 receiver 자리에 다시 씁니다.

- **배타성**: 호출이 진행되는 동안 receiver 변수를 다른 경로 (다른 인자, 캡처한 클로저 등) 로 접근하면 안 됩니다. 컴파일러가 이를 볼 수 있으면 컴파일 에러입니다 (Swift 의 exclusivity 규칙과 같습니다).
- **임시 receiver**: receiver 가 임시 값 (`make().increment()`) 이면 되쓰기 결과는 버려집니다.

```rok
c.increment();                     // c 를 복사해 들이고, 끝나면 c 에 되씀
c.add_from(|| c.count);            // ERROR — 호출 중 c 를 클로저로 접근
```

## 4.7.6. `ref` 의 타입 측면

`ref` 는 수식어이지 타입이 아니며, [타입 표현식](../002-types/201-type-expressions.md) 어디에도 등장하지 않습니다.

```rok
let x: ref i32 = ...;             // ERROR — ref 는 타입이 아님
struct S { x: ref i32; }          // ERROR — 같은 이유
List<ref i32>                      // ERROR — 같은 이유
```

따라서 `ref` 는 다음 위치에만 등장할 수 있습니다.

- 함수와 메서드의 매개변수 선언.
- 클로저 캡처 리스트.
- 함수와 메서드의 호출 인자.

## 4.7.7. 참조 자체를 일급 값으로 취급할 수 없습니다

```rok
let r = ref x;                    // ERROR — ref 식 자체는 표현식이 아님
let r: SomePointerType = ref x;   // ERROR
```

`ref` 는 값을 만드는 식이 아니며, 호출과 매개변수, 캡처 리스트에 붙는 수식어 토큰일 뿐입니다.

같은 이유로 `ref` 매개변수를 가진 함수는 함수 값으로 쓸 수 없으며 (`let f = increment;` 는 에러), 클로저는 `ref` 매개변수를 가질 수 없습니다. 함수 타입 `fn(..)` 에는 `ref` 를 표현할 자리가 없기 때문입니다.

만약 일급 참조가 필요한 경우(예: [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface)) 에는 [타입 표현식](../002-types/201-type-expressions.md) 의 원시 포인터 타입(`Ptr<T>` 와 `ConstPtr<T>`) 을 사용하면 되며, 이는 [표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md) 의 `unsafe` 영역에 해당합니다.

## 4.7.8. 값 pass 와 `ref` pass 의 의미론 정리

같은 호출 시점에서의 두 형태를 비교해 보겠습니다.

```rok
fn by_value(x: i32) { x = 10i32; }
fn by_ref(ref x: i32) { x = 10i32; }

let a = 5i32;
by_value(a);                      // a = 5 (변화 없음)
by_ref(ref a);                    // a = 10
```

| 측면                      | 값 pass (기본) | `ref` pass (`ref`) |
|---------------------------|----------------|--------------------|
| 매개변수 안 재할당의 효과 | 로컬만         | 호출자의 변수      |
| 명시 호출 토큰            | 없음           | `ref` 필수         |
| 시각 신호                 | 일반 호출      | 변경 가능 표시     |

## 4.7.9. 힙 객체와의 결합

힙 객체(`struct`, List, Dict 등) 의 필드나 원소 변경은 일반 값 pass 에서도 호출자에게 보입니다 (공유 참조).

```rok
fn modify_fields(p: Point) {
    p.x = 10.0f64;                // 같은 힙 객체의 필드 — 호출자도 봄
}

fn replace(ref p: Point) {
    p = Point { x: 99.0, y: 99.0 };   // 호출자의 변수가 새 Point 를 가리킴
}
```

| 시나리오                             | 필요한 키워드       |
|--------------------------------------|---------------------|
| 객체의 필드만 변경                   | 일반 값 pass — 충분 |
| 호출자의 변수가 다른 객체를 가리키게 | `ref` 명시          |

이러한 구분은 [멤버 접근과 메서드 호출](../003-expressions/305-member-access-and-method-call.md) 의 객체 변경 의미와 일관되게 이어집니다.

## 4.7.10. 두 바인딩에 대한 `ref` 동시 사용

```rok
fn swap(ref a: i32, ref b: i32) {
    let t = a;
    a = b;
    b = t;
}

let x = 1i32;
let y = 2i32;
swap(ref x, ref y);               // x = 2, y = 1
```

위와 같이 같은 호출에서 여러 `ref` 매개변수를 둘 수 있습니다.

호출 지점에서 같은 자리를 두 `ref` 로 넘기면 4.7.2 의 배타성 규칙에 따라 컴파일 에러입니다.

```rok
swap(ref x, ref x);               // ERROR — 같은 자리를 두 번 ref
```
