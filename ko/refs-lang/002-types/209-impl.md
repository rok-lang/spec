# 2.9. `impl`

`impl` 블록은 어떤 타입에 메서드를 부착하거나, 어떤 타입에 트레잇을 구현하는 컨테이너입니다.

## 2.9.1. 두 형태

`impl` 블록의 문법은 다음과 같이 두 가지 형태를 가집니다.

```
impl_block      ::= "impl" generics? type_expr "{" impl_item* "}"
                  | "unsafe"? "impl" generics? type_expr "for" type_expr where_clause? "{" impl_item* "}"
                  | "impl" generics? "!" type_expr "for" type_expr where_clause? "{" "}"
impl_item       ::= fn_decl
                  | "type" identifier "=" type_expr ";"
                  | const_decl
```

### 고유 `impl` (inherent). 메서드 부착

먼저 고유 `impl` 의 예시부터 살펴보겠습니다.

```rok
impl Point {
    fn origin() -> Point { Point { x: 0.0f64, y: 0.0f64 } }
    fn distance(self, other: Point) -> f64 { ... }
}
```

이 형태는 `impl 타입 { ... }` 의 모습을 가지며, 본체는 함수 정의의 모음으로 이루어집니다. 그리고 이렇게 정의된 함수는 그 타입의 메서드가 됩니다.

고정 배열과 튜플에도 고유 `impl` 을 둘 수 있습니다. 표준 라이브러리는 이것으로 배열에 `len()` / `is_empty()` 를 줍니다.

```rok
impl<T, const N: usize> [T; N] {
    fn len(self) -> usize { N }
}

impl<A, B> (A, B) {
    fn swap(self) -> (B, A) { (self.1, self.0) }
}
```

### 트레잇 `impl`. 트레잇 구현

다음으로 트레잇 `impl` 의 예시를 살펴보겠습니다.

```rok
impl Display for Point {
    fn display(self) -> str { f"({self.x}, {self.y})" }
}

impl Add for Point {
    type Output = Point;
    fn add(self, rhs: Point) -> Point { ... }
}
```

이 형태는 `impl 트레잇 for 타입 { ... }` 의 모습을 가집니다. 본체에는 [`trait`](208-trait.md) 가 요구한 메서드의 본체와 연관 타입의 구체 정의가 옵니다. 한편 트레잇이 기본 메서드를 제공한 항목은 본체에서 생략할 수 있으며, 그러면 기본 본체를 사용하게 됩니다.

impl 메서드의 `throws` 는 트레잇 메서드의 `throws` 와 같아야 합니다. impl 이 `throws` 를 새로 붙이거나 에러 타입을 넓힐 수 없습니다 ([4.14.11](../004-functions/414-throws.md#41411-메서드의-throws)).

`Send` / `Sync` 는 자동 marker 트레잇이며 두 가지 수동 선언이 있습니다 ([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync--스레드-경계-marker)).

- opt-in: `unsafe impl Send for X {}`. `unsafe impl` 은 `Send` / `Sync` 에만 쓸 수 있고, 이 두 트레잇의 수동 구현에는 반드시 필요합니다.
- opt-out: `impl !Send for X {}`, `impl<T> !Send for Cell<T> {}`. 자동 구현을 막으며, 본체는 비어 있어야 합니다. `!` 는 `Send` / `Sync` 에만 쓸 수 있습니다.

### `impl` 안의 상수

`impl` 블록 안의 `const N: T = v;` 는 그 타입의 연관 상수입니다. 바깥에서는 `Type::N` 으로, `impl` 안에서는 `Self::N` 으로 읽습니다. 트레잇이 선언한 연관 상수 ([2.8.3a](208-trait.md#283a-연관-상수-associated-const)) 는 트레잇 `impl` 에서 값을 정합니다.

```rok
impl Circle {
    const UNIT: f64 = 1.0;
    fn unit() -> Circle { Circle { r: Self::UNIT } }
}
let r = Circle::UNIT;
```

## 2.9.2. 메서드의 `self` 관례

본체의 함수가 인스턴스 메서드가 되려면 첫 매개변수가 `self` (또는 `ref self`) 여야 합니다.

```rok
impl Point {
    fn origin() -> Point { ... }                    // 정적 — Type::method() 호출
    fn distance(self, other: Point) -> f64 { ... }  // 인스턴스 — instance.method() 호출
    fn replace_with(ref self, other: Point) { ... } // ref self — caller 의 변수 재할당 가능
}
```

`self` 매개변수의 정확한 의미와 `ref self` 의 동작은 함수 사양에서 정의합니다. 본 문서에서는 `self` 유무가 정적 메서드와 인스턴스 메서드를 구분한다는 점까지만 짚고 넘어가겠습니다.

## 2.9.3. 한 타입에 대한 여러 `impl` 블록

같은 타입에 대해 고유 `impl` 은 여러 블록으로 나누어도 됩니다.

```rok
impl Point {
    fn origin() -> Point { ... }
}

impl Point {
    fn distance(self, other: Point) -> f64 { ... }
}
```

이 형태는 시각적이고 조직적인 묶음일 뿐이며, 결과는 한 본체에 모은 것과 같습니다.

다만 같은 타입에 대해 같은 트레잇의 `impl` 은 한 번만 가능합니다.

```rok
impl Display for Point { ... }
impl Display for Point { ... }   // ERROR — 중복 구현
```

## 2.9.4. 제네릭 `impl`

`impl` 도 제네릭으로 선언할 수 있습니다.

```rok
impl<T> List<T> {
    fn len(self) -> usize { ... }
}

impl<T> Add for Pair<T> where T: Add<T> {
    type Output = Pair<T::Output>;
    fn add(self, rhs: Pair<T>) -> Pair<T::Output> { ... }
}
```

`impl` 뒤에 `<...>` 로 타입 매개변수를 선언하며, 이 매개변수는 대상 타입과 트레잇 인자, 연관 타입 어디에든 등장할 수 있습니다. 또한 `where` 절로 추가 제약을 둘 수 있는데, 이는 트레잇 선언과 동일한 방식입니다.

## 2.9.5. Orphan 규칙

다른 패키지에서 정의된 트레잇을 또 다른 패키지에서 정의된 타입에 구현하는 것은 금지합니다.

```rok
// 트레잇 Display, 타입 i32 가 모두 표준 라이브러리(다른 패키지) 정의
impl Display for i32 { ... }     // ERROR — 둘 다 외부 패키지
```

따라서 트레잇이 현재 패키지 정의이거나, 아니면 대상 타입이 현재 패키지 정의이거나 둘 중 하나는 반드시 충족되어야 합니다.

이 규칙의 단위는 [소스 단위](../000-meta/003-source-units.md) 의 패키지입니다. 그러므로 같은 패키지 안의 다른 모듈끼리는 자유롭게 `impl` 할 수 있습니다.

이 규칙은 같은 (트레잇, 타입) 쌍에 대해 충돌하는 `impl` 이 두 패키지에서 동시에 만들어지지 않도록 하기 위한 장치이며, 일관성을 보장하기 위한 것입니다.

### 일관성 (coherence)

한 (트레잇, 타입) 쌍에는 `impl` 이 하나뿐이며, 특수화 (specialization) 는 없습니다.

- 구체 `impl` 이 제네릭 `impl` 과 겹치면 컴파일 에러입니다.
- 제네릭 `impl` 은 쓰이는 타입마다 단형화됩니다.
- 제네릭 `impl` 두 개가 겹치는 것도 에러이며, 선언 시점에 보고합니다. 두 `impl` 은 대상 (과 트레잇 인자) 이 단일화 (unify) 되면 겹칩니다. 바운드는 둘을 떼어 놓지 않으므로, 두 바운드를 모두 만족하는 타입이 아직 없어도 에러입니다 (Rust 의 coherence 와 같음).

```rok
impl<T: Display> Describe for List<T> { ... }
impl<T: Hash> Describe for List<T> { ... }   // ERROR — i32 가 Display 와 Hash 를 모두 만족
impl<T: Shape> Area for T { ... }
impl<T: Color> Area for T { ... }            // ERROR — T 와 T 가 단일화됨 (바운드 무관)
```

```rok
impl<T: Display> Describe for T { ... }
impl Describe for i32 { ... }        // ERROR — i32: Display 라서 위 impl 과 겹침
```

## 2.9.6. 메서드 가시성

고유 `impl` 의 메서드 가시성은 기본이 모듈 비공개입니다. 같은 모듈 안에서는 자유롭게 호출할 수 있으며, 외부에 노출하려면 `pub` 을 명시해야 합니다.

```rok
impl Account {
    pub fn balance(self) -> f64 { self.amount }      // 외부 공개
        fn _internal_check(self) { ... }              // 모듈 내부만
}
```

이러한 결정은 [`struct`](203-struct.md) 와 [`enum`](204-enum.md) 의 가시성 모델과 일치합니다. `impl` 블록 자체는 시각적 묶음일 뿐 가시성 단위가 아니며, 가시성 단위는 어디까지나 모듈입니다.

한편 트레잇 `impl` 의 메서드는 그 트레잇의 가시성을 따르므로, 따로 표시할 수 없습니다.

## 2.9.7. 고유 `impl` vs 트레잇 `impl` 의 메서드 충돌

같은 메서드 이름이 고유 `impl` 과 트레잇 `impl` 양쪽에 있으면, 호출 지점에서 모호하지 않은 방식으로 선택해야 합니다. 정확한 해소 규칙은 [메서드](../004-functions/409-method.md#트레잇-사이의-이름-충돌) 에서 정의합니다.

## 2.9.8. 동일 메서드 두 번 정의 금지

같은 `impl` 블록 안에서 같은 이름의 메서드를 두 번 정의하면 컴파일 에러가 됩니다 (오버로딩 없음).
