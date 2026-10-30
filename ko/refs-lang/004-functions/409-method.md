# 4.9. 메서드

이 문서는 [`impl`](../002-types/209-impl.md) 안에 정의하는 메서드의 매개변수와 호출 규약을 정의합니다. 함수 사양의 일부이며 메서드와 자유 함수가 어떻게 다른지를 중심으로 다룹니다.

## 4.9.1. 메서드의 정의 위치

메서드는 [`impl`](../002-types/209-impl.md) 블록 안에 정의합니다.

```rok
impl Point {
    fn origin() -> Point { Point { x: 0.0f64, y: 0.0f64 } }
    fn distance(self, other: Point) -> f64 { ... }
    fn translate(ref self, dx: f64, dy: f64) {
        self.x = self.x + dx;
        self.y = self.y + dy;
    }
}
```

## 4.9.2. 첫 매개변수의 형태에 따른 메서드 종류

### 정적 메서드: `self` 없음

```rok
impl Point {
    fn origin() -> Point { ... }              // 정적
    fn from_polar(r: f64, theta: f64) -> Point { ... }
}

let p = Point::origin();                       // Type::method() 호출
let q = Point::from_polar(1.0f64, 0.0f64);
```

정적 메서드는 첫 매개변수가 `self`도 `ref self`도 아닌 함수입니다. `Type::method(args)` 형태로 호출합니다. 이 호출은 [타입 경로](../002-types/213-type-paths.md)로 정적 항목에 접근하는 것입니다.

### 인스턴스 메서드: `self`

```rok
impl Point {
    fn distance(self, other: Point) -> f64 { ... }
    fn norm(self) -> f64 { ... }
}

let d = p.distance(q);                         // instance.method() 호출
let n = p.norm();
```

이 형태의 메서드는 첫 매개변수가 `self`입니다. `self`는 그 인스턴스의 값입니다(`struct`이면 사본, [2.3.8](../002-types/203-struct.md#238-값-의미)). 본체에서는 `self.field`로 필드에 접근합니다. `instance.method(args)` 형태로 호출합니다.

### 인스턴스 메서드: `ref self`

```rok
impl Point {
    fn translate(ref self, dx: f64, dy: f64) {
        self.x = self.x + dx;
        self.y = self.y + dy;
    }

    fn replace_with(ref self, other: Point) {
        self = other;                          // 호출자의 변수가 other 가 됨
    }
}

let p = Point { x: 1.0, y: 2.0 };
p.translate(5.0f64, 0.0f64);                   // ref 명시 없이 호출
p.replace_with(other);                         // 마찬가지
```

이 형태의 메서드는 첫 매개변수가 `ref self`입니다. 본체 안에서 `self`와 그 필드에 대입할 수 있고, 그 결과는 호출자의 변수에 되쓰입니다([`ref` 키워드](407-ref-keyword.md) 참고). 호출자는 `ref`를 적지 않습니다. 컴파일러가 메서드 호출의 receiver를 자동으로 처리하기 때문입니다.

## 4.9.3. `self` 키워드의 두 사용

```rok
impl Point {
    fn distance(self, other: Point) -> f64 {
        let dx = self.x - other.x;             // 인스턴스 self 사용
        let dy = self.y - other.y;
        (dx * dx + dy * dy).sqrt()
    }
}
```

`self`([키워드](../001-lexical/105-keywords.md))는 두 자리에서 씁니다. 매개변수 위치에서는 인스턴스를 받는 매개변수입니다. 본체 안에서는 그 매개변수를 가리키는 식별자입니다.

매개변수 자리의 `self`에는 타입을 적지 않습니다. 타입은 자동으로 `Self`, 곧 현재 `impl`의 대상 타입이 됩니다.

## 4.9.4. `Self` 키워드: 타입 자리

`Self`([키워드](../001-lexical/105-keywords.md), 대문자)는 현재 `impl`의 대상 타입을 가리키는 타입 이름입니다.

```rok
impl Point {
    fn new(x: f64, y: f64) -> Self {           // Self == Point
        Self { x, y }                          // Self 로 인스턴스 생성도 가능
    }
}
```

둘을 분명히 구분해야 합니다. 소문자 `self`는 매개변수와 인스턴스 식별자이고, 대문자 `Self`는 타입 이름입니다.

## 4.9.5. 메서드 호출 수신자의 자동 `ref`

`ref self` 메서드는 호출하는 쪽이 `ref`를 적지 않아도 자동으로 `ref`가 붙습니다.

```rok
impl Counter {
    fn increment(ref self) { self.count = self.count + 1; }
}

let c = Counter { count: 0 };
c.increment();                                 // 'ref' 명시 없음
```

이것은 [`ref` 키워드](407-ref-keyword.md)의 일반 함수 호출 규칙(`ref`를 적어야 함)의 예외입니다. 이유는 두 가지입니다. 메서드 호출 문법 `x.method()` 자체가 receiver [dispatch](../000-meta/007-glossary.md#dispatch-디스패치) 메커니즘입니다. 또 receiver가 `self`든 `ref self`든 호출 형태가 같습니다. `ref`를 더 적으면 눈에 거슬리기만 합니다.

Rust, C#, Swift의 메서드 호출 관례와도 같습니다. 다만 같은 메서드를 경로로 부를 때는 receiver가 일반 인자이므로 `ref`를 적습니다(`Counter::increment(ref c)`, [4.7.3](407-ref-keyword.md#473-호출-시-명시c-식)).

## 4.9.6. 자유 함수와 메서드의 호출 형태

| 형태            | 호출                    |
|-----------------|-------------------------|
| 자유 함수       | `function_name(args)`   |
| 정적 메서드     | `Type::method(args)`    |
| 인스턴스 메서드 | `instance.method(args)` |

호출 형태만 보고도 그 함수가 어떤 종류인지 알 수 있습니다.

### 트레잇 사이의 이름 충돌

`x.method()`의 후보가 여러 개일 때는 다음 순서로 고릅니다.

1. 고유 `impl`의 메서드가 트레잇 메서드보다 우선합니다.
2. 트레잇 메서드끼리는 현재 패키지가 정의했거나 `use`로 직접 가져온 트레잇이 우선합니다. 암묵 `use rok::core::**`로만 보이는 트레잇은 그보다 뒤입니다. 현재 패키지는 호출이 적힌 실제 패키지입니다. 의존 패키지의 트레잇은 `use`로 가져오지 않으면 core의 트레잇보다 우선하지 않습니다.
3. 그래도 후보가 둘 이상이면 모호성 에러입니다. 진단은 `Trait::method(x)` 형태로 적으라고 안내합니다.

```rok
use pkg::geom::Union;              // 사용자 트레잇 - union 메서드
a.union(b);                        // pkg::geom::Union 이 core 의 Union 보다 우선
rok::core::ops::Union::union(a, b) // core 쪽을 쓰려면 명시
```

## 4.9.7. 메서드 안의 다른 메서드 호출

```rok
impl Point {
    fn distance(self, other: Point) -> f64 { ... }

    fn distance_squared(self, other: Point) -> f64 {
        let d = self.distance(other);          // 같은 인스턴스의 다른 메서드
        d * d
    }
}
```

`self.method()` 형태로 같은 인스턴스의 다른 메서드를 호출할 수 있습니다. 규칙은 일반 인스턴스 호출과 같습니다.

## 4.9.8. 메서드 가시성

메서드 가시성의 기본값은 [`impl`](../002-types/209-impl.md)에서 정의한 대로 모듈 비공개입니다.

```rok
impl Account {
    pub fn balance(self) -> f64 { ... }        // 외부 공개
        fn _internal(self) { ... }              // 모듈 비공개
}
```

가시성 단계의 정확한 규칙은 [가시성](../005-modules/506-visibility.md)에서 정의합니다.

## 4.9.9. 메서드 시그니처의 자유도

메서드의 매개변수, 반환, 본체는 모두 [매개변수](402-parameters.md)와 [반환](403-return.md)의 일반 함수 규칙을 따릅니다. 기본값, named 인자, variadic, spread를 메서드에서도 자유롭게 쓸 수 있습니다. 본체가 `throw`나 무한 루프로 항상 빠져나가는 divergent 메서드도 만들 수 있습니다. 제네릭 메서드에서는 `impl` 블록의 제네릭 매개변수와 메서드 자체의 제네릭을 함께 쓸 수 있습니다.

```rok
impl<T> List<T> {
    fn map<U>(self, f: fn(T) -> U) -> List<U> { ... }
}
```

메서드 자체의 제네릭은 함수 이름 뒤에 둡니다(`fn map<U>(...)`). 트레잇 `impl`에서 타입을 적지 않은 메서드 매개변수는 암묵 타입 매개변수가 되지 않습니다. 그 매개변수의 타입은 트레잇이 그 자리에 선언한 타입입니다. 이때 트레잇 매개변수와 연관 타입은 대입한 뒤의 타입을 씁니다.

## 4.9.10. 메서드 안의 `self` 관계 요약

| 첫 매개변수 | 의미                    | 호출 시                          | 본체에서 `self` 재할당       |
|-------------|-------------------------|----------------------------------|------------------------------|
| (없음)      | 정적 메서드             | `Type::method()`                 | -                            |
| `self`      | 인스턴스 메서드 (값)    | `instance.method()`              | 안 됨 (로컬만)               |
| `ref self`  | 인스턴스 메서드 (`ref`) | `instance.method()` (`ref` 자동) | 됨 (호출자에 반영)           |

메서드의 종류는 이 세 형태가 전부입니다.
