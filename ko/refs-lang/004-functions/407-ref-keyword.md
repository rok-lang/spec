# 4.7. `ref` 키워드

`ref`는 "이 자리의 변경이 호출자(또는 바깥)의 변수에 반영된다"는 표시입니다. 매개변수와 호출 인자에서는 copy-in / copy-out으로 구현하고, 캡처 리스트에서는 공유 셀로 구현합니다. 일급 참조 값은 아닙니다.

## 4.7.1. 세 등장 위치

```
1. 함수 매개변수:    fn f(ref x: i32) { x = ... }
2. 클로저 캡처 리스트: |...| { ... } 앞의 [ref x]
3. 호출 시:           f(ref x)
```

세 위치 모두 의미는 같습니다. 변경이 바깥 변수에 반영됩니다.

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

매개변수 이름 앞에 `ref`를 붙이면 본체에서 그 매개변수를 재할당한 결과가 호출자의 변수에 반영됩니다. 매개변수 자체의 타입(위 예의 `i32`)은 일반 매개변수와 같습니다.

### 의미: copy-in / copy-out

`ref` 매개변수는 copy-in / copy-out입니다. 호출할 때 호출자 자리(변수, 필드, 원소)의 값을 매개변수로 복사해 들입니다. 피호출자는 그 값으로 작업합니다. 호출이 끝나면 매개변수의 최종 값을 호출자 자리에 되씁니다.

- **되쓰기 시점**: 정상 반환할 때뿐 아니라 피호출자가 에러를 던질 때도 되씁니다. 그래서 `throw` 전에 바꾼 내용은 호출자에게 보입니다.
- **배타성**: 호출이 진행되는 동안 같은 자리에 다른 경로로 접근하면 안 됩니다. 컴파일러가 이를 검사합니다. 같은 자리를 `ref`로 두 번 넘기거나, 한 인자로는 `ref`로 넘기고 다른 인자로 읽으면 컴파일 에러입니다. 서로 다른 필드(`ref p.x`, `ref p.y`)는 겹치지 않으므로 허용합니다.

```rok
fn bump_then_fail(ref n: i32) throws {
    n = n + 1;
    throw MyError {};
}
let a = 1i32;
try bump_then_fail(ref a) catch _ { };   // a = 2 (throw 때도 되씀)
swap(ref a, ref a);                       // ERROR: 같은 자리를 두 번 ref
add(ref a, a);                            // ERROR: ref 로 넘긴 자리를 다른 인자로 읽음
swap(ref p.x, ref p.y);                   // OK: 서로 다른 필드
```

> [!IMPORTANT]
> 호출자는 호출 지점에 `ref`를 명시해야 합니다. 명시하지 않고 호출하면 컴파일 에러입니다. 어떤 변수가 바뀔 수 있는지 눈에 보이게 하려는 규칙입니다.

## 4.7.3. 호출 시 명시(C# 식)

```rok
let x = 5i32;
let y = 10i32;
increment(ref x);                 // x = 6
swap(ref x, ref y);
```

호출자는 `ref`로 선언한 매개변수 자리에만, 그리고 그런 자리마다 `ref`를 명시해야 합니다. `ref`를 빠뜨리거나 `ref`가 아닌 매개변수에 `ref`를 붙이면 컴파일 에러입니다. `ref self` 메서드를 경로로 부를 때도 같은 규칙을 적용합니다(`Counter::increment(ref c)`, 4.7.5).

이 규칙 덕분에 호출 지점에서 어떤 변수가 바뀔 수 있는지 눈에 보입니다. C#의 `ref` 패턴과 같은 이유입니다.

```rok
fn read(x: i32) { ... }
fn write(ref x: i32) { ... }

read(a);                          // 일반: a 가 바뀌지 않음이 명확
write(ref a);                      // ref: a 가 바뀔 수 있음이 명확
```

## 4.7.4. 클로저 캡처 리스트

```rok
let count = 0;
let inc = [ref count] || { count = count + 1 };
```

[캡처 의미론](406-capture-semantics.md)에서 정의한 `ref` 캡처에 따라 캡처 리스트의 `ref` 변수는 공유 셀에 담깁니다(boxing). 바깥 스코프와 클로저가 같은 셀을 읽고 쓰므로, 클로저가 변수의 스코프보다 오래 살아도 됩니다.

`ref` 매개변수(`ref self` 포함)도 같은 방식으로 캡처합니다(`[ref n]`, `[ref self]`). 함수 본체와 클로저는 매개변수의 셀 하나를 공유합니다. 그래서 클로저가 쓴 값도 매개변수의 최종 값에 들어가고 호출자에게 되쓰입니다([4.7.2](#472-함수-매개변수)). 호출이 끝난 뒤에 클로저가 쓴 값은 호출자에게 닿지 않습니다.

```rok
fn bump(ref c: Counter) {
    let add = [ref c] |x: i32| { c.count = c.count + x; };
    add(5);
}
let c = Counter { count: 1 };
bump(ref c);                      // c.count = 6
```

## 4.7.5. 메서드 수신자의 예외

[`impl`](../002-types/209-impl.md)의 메서드에는 `ref self` 매개변수를 둘 수 있습니다.

```rok
impl Counter {
    fn increment(ref self) { self.count = self.count + 1; }
}
```

이 경우에는 호출 지점에 `ref`를 적지 않아도 됩니다.

```rok
let c = Counter { count: 0 };
c.increment();                    // 'ref' 명시 없음: 메서드 receiver 의 자동 처리
```

이 규칙은 일반 `ref` 매개변수와 달리 메서드 수신자에만 있습니다. 메서드 호출 문법 자체가 수신자를 자동으로 [디스패치](../000-meta/007-glossary.md#dispatch-디스패치)하는 메커니즘입니다. 그래서 표시를 더 붙이면 눈에 거슬리기만 합니다.

```rok
fn external_inc(ref c: Counter) { ... }
external_inc(ref c);              // 일반 함수 호출은 'ref' 명시가 필요
Counter::increment(ref c);        // ref self 메서드를 경로로 부를 때도 ref 필요
```

### `ref self`의 의미: copy-in / copy-out

`ref self`는 copy-in / copy-out으로 동작합니다. 호출할 때 수신자의 값을 `self`로 복사해 들입니다. 호출이 끝나면 `self`의 최종 값을 수신자 자리에 되씁니다.

- **배타성**: 호출이 진행되는 동안 수신자 변수에 다른 경로(다른 인자, 캡처한 클로저 등)로 접근하면 안 됩니다. 컴파일러가 이를 알아볼 수 있으면 컴파일 에러입니다. Swift의 배타성 규칙과 같습니다.
- **임시 수신자**: 수신자가 임시 값(`make().increment()`)이면 되쓴 결과는 버려집니다.

```rok
c.increment();                     // c 를 복사해 들이고, 끝나면 c 에 되씀
c.add_from(|| c.count);            // ERROR: 호출 중 c 를 클로저로 접근
```

## 4.7.6. `ref`와 타입

`ref`는 수식어이지 타입이 아닙니다. [타입 표현식](../002-types/201-type-expressions.md) 어디에도 나오지 않습니다.

```rok
let x: ref i32 = ...;             // ERROR: ref 는 타입이 아님
struct S { x: ref i32; }          // ERROR: 같은 이유
List<ref i32>                      // ERROR: 같은 이유
```

그래서 `ref`는 다음 위치에만 올 수 있습니다.

- 함수와 메서드의 매개변수 선언.
- 클로저 캡처 리스트.
- 함수와 메서드의 호출 인자.

`ref`를 반환 타입에 쓸 수 없으므로 Rok에는 참조를 돌려주는 함수가 없습니다(Rust의 `deref_mut` 같은 형태 없음). 안쪽 값을 바꾸는 API는 `set` / `update(f)`처럼 값으로 주고받습니다.

## 4.7.7. 참조 자체를 일급 값으로 취급할 수 없습니다

```rok
let r = ref x;                    // ERROR: ref 식 자체는 표현식이 아님
let r: SomePointerType = ref x;   // ERROR
```

`ref`는 값을 만드는 식이 아니라 호출, 매개변수, 캡처 리스트에 붙는 수식어 토큰일 뿐입니다.

코루틴 함수(`yield fn`, `async fn`, `async yield fn`)는 `ref` 매개변수를 둘 수 없습니다. 두면 컴파일 에러입니다. 호출이 반환된 뒤에도 본체가 계속 실행되므로 되쓸 시점이 없기 때문입니다.

같은 이유로 `ref` 매개변수가 있는 함수는 함수 값으로 쓸 수 없습니다(`let f = increment;`는 에러). 클로저에도 `ref` 매개변수를 둘 수 없습니다. 함수 타입 `fn(..)`에는 `ref`를 표현할 자리가 없기 때문입니다.

일급 참조가 필요하면(예: [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface)) [타입 표현식](../002-types/201-type-expressions.md)의 원시 포인터 타입(`Ptr<T>`와 `ConstPtr<T>`)을 씁니다. 원시 포인터는 [표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md)의 `unsafe` 영역입니다.

## 4.7.8. 정리: 값 전달과 `ref` 전달

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

## 4.7.9. struct와 참조 핸들

`struct`는 값 타입입니다. 값으로 넘기면 struct가 복사되므로 피호출자가 필드에 쓴 내용은 피호출자 안에만 남습니다. 호출자의 struct를 바꾸려면 `ref`(또는 `ref self`)로 받거나 새 값을 반환합니다. 참조 핸들(`List`, `Dict` 등, [stdlib 2.9](../../refs-standard-libraries/200-core/209-collections.md#참조-핸들))은 핸들로 복사되므로 원소 변경이 호출자에게 보입니다.

```rok
fn set_x(p: Point) { p.x = 10.0; }          // 지역 사본: 호출자의 p 는 그대로
fn set_x_ref(ref p: Point) { p.x = 10.0; }  // 호출자의 p 에 되씀
fn fill(xs: List<i32>) { xs.push(1); }      // 같은 List: 호출자도 봄
fn reset(ref xs: List<i32>) { xs = []; }    // 호출자의 변수가 새 List 를 가짐
```

| 시나리오                           | 필요한 키워드       |
|------------------------------------|---------------------|
| 호출자 struct의 필드 변경          | `ref`               |
| 컬렉션의 원소 변경                 | 일반 값 pass로 충분 |
| 호출자의 변수가 다른 값을 담게     | `ref`               |

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

위처럼 한 호출에 `ref` 매개변수를 여러 개 둘 수 있습니다.

호출 지점에서 같은 자리를 두 `ref`로 넘기면 4.7.2의 배타성 규칙에 따라 컴파일 에러입니다.

```rok
swap(ref x, ref x);               // ERROR: 같은 자리를 두 번 ref
```
