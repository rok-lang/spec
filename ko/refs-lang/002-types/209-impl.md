# 2.9. `impl`

`impl` 블록은 타입에 메서드를 붙이거나 타입에 트레잇을 구현하는 컨테이너입니다.

## 2.9.1. 두 형태

`impl` 블록의 문법에는 두 가지 형태가 있습니다.

```
impl_block      ::= "impl" generics? type_expr "{" impl_item* "}"
                  | "unsafe"? "impl" generics? type_expr "for" type_expr where_clause? "{" impl_item* "}"
                  | "impl" generics? "!" type_expr "for" type_expr where_clause? "{" "}"
impl_item       ::= fn_decl
                  | "type" identifier "=" type_expr ";"
                  | const_decl
```

`impl <`의 `<`는 뒤에 제네릭 매개변수가 올 때만(`<>`, `<T>`, `<T:`, `<T,`, `<T =`, `<const N`) `generics`를 엽니다. 그 밖에는 대상 타입의 시작입니다. 예를 들어 고유 `impl <A as P>::Out { .. }`가 그렇습니다.

### 고유(inherent) `impl`: 메서드 부착

고유 `impl`의 예는 다음과 같습니다.

```rok
impl Point {
    fn origin() -> Point { Point { x: 0.0f64, y: 0.0f64 } }
    fn distance(self, other: Point) -> f64 { ... }
}
```

이 형태는 `impl 타입 { ... }` 모양이고, 본체는 함수 정의의 모음입니다. 여기서 정의한 함수는 그 타입의 메서드가 됩니다.

고정 배열과 튜플에도 고유 `impl`을 둘 수 있습니다. 표준 라이브러리는 이 방법으로 배열에 `len()` / `is_empty()`를 붙입니다.

```rok
impl<T, const N: usize> [T; N] {
    fn len(self) -> usize { N }
}

impl<A, B> (A, B) {
    fn swap(self) -> (B, A) { (self.1, self.0) }
}
```

### 트레잇 `impl`. 트레잇 구현

트레잇 `impl`의 예는 다음과 같습니다.

```rok
impl Describe for Point {
    fn describe(self) -> str { f"({self.x}, {self.y})" }
}

impl Add for Point {
    type Output = Point;
    fn add(self, rhs: Point) -> Point { ... }
}
```

이 형태는 `impl 트레잇 for 타입 { ... }` 모양입니다. 본체에는 [`trait`](208-trait.md)가 요구한 메서드의 본체와 연관 타입의 구체 정의가 옵니다. 트레잇에 기본 메서드가 있는 항목은 본체에서 생략할 수 있습니다. 생략하면 기본 본체를 씁니다.

impl 메서드는 트레잇 선언과 매개변수 개수가 같아야 합니다. 수신자를 뺀 각 매개변수에는 선언과 같은 `ref`가 붙어야 합니다. 값을 돌려주는지도 선언과 같아야 합니다. 이를 어기면 impl 위치에서 에러이고, 에러 메시지는 선언을 함께 보여 줍니다. impl 메서드의 `throws`는 트레잇 메서드의 `throws`와 같아야 합니다. impl이 `throws`를 새로 붙이거나 에러 타입을 넓힐 수 없습니다([4.14.11](../004-functions/414-throws.md#41411-메서드의-throws)).

`Send` / `Sync`는 자동 마커 트레잇입니다. 직접 선언하는 방법은 두 가지입니다([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync-스레드-경계-마커)).

- opt-in: `unsafe impl Send for X {}`. `unsafe impl`은 `Send` / `Sync`에만 쓸 수 있고, 이 두 트레잇을 직접 구현할 때는 써야 합니다.
- opt-out: `impl !Send for X {}`, `impl<T> !Send for Cell<T> {}`. 자동 구현을 막습니다. 본체는 비어 있어야 합니다. `!`는 `Send` / `Sync`에만 쓸 수 있습니다.

### `impl` 안의 상수

`impl` 블록 안의 `const N: T = v;`는 그 타입의 연관 상수입니다. 바깥에서는 `Type::N`으로, `impl` 안에서는 `Self::N`으로 읽습니다. 트레잇이 선언한 연관 상수([2.8.3a](208-trait.md#283a-연관-상수associated-const))는 트레잇 `impl`에서 값을 정합니다.

```rok
impl Circle {
    const UNIT: f64 = 1.0;
    fn unit() -> Circle { Circle { r: Self::UNIT } }
}
let r = Circle::UNIT;
```

`impl`의 대상이나 트레잇 인자는 투영(`<T as Tr>::X`)이어도 되고, 뒤에 선언된 항목을 가리켜도 됩니다. 컴파일러는 이런 이름을 모든 항목을 등록한 뒤에 해소합니다.

`impl`의 대상으로 타입 별칭을 쓸 수 있습니다. 제네릭 별칭이든 아니든 상관없습니다. 별칭은 펼쳐지고, 그 `impl`은 별칭이 가리키는 타입으로 적은 것과 같습니다. `type Al<T> = S<T>;`이면 `impl<T> Al<T> { .. }`는 `impl<T> S<T> { .. }`이고 `impl<T> Tr for Al<T>`는 `impl<T> Tr for S<T>`입니다. 인자 일부를 고정한 별칭(`type Names<V> = Dict<str, V>;`)으로 쓴 `impl`은 그 인스턴스만 덮습니다.

## 2.9.2. 메서드의 `self` 관례

본체의 함수가 인스턴스 메서드가 되려면 첫 매개변수가 `self`(또는 `ref self`)여야 합니다.

```rok
impl Point {
    fn origin() -> Point { ... }                    // 정적: Type::method() 호출
    fn distance(self, other: Point) -> f64 { ... }  // 인스턴스: instance.method() 호출
    fn replace_with(ref self, other: Point) { ... } // ref self: caller 의 변수 재할당 가능
}
```

`self` 매개변수의 정확한 의미와 `ref self`의 동작은 함수 사양에서 정의합니다. 이 문서는 `self`가 있는지 없는지로 정적 메서드와 인스턴스 메서드가 갈린다는 점만 밝힙니다.

## 2.9.3. 한 타입에 대한 여러 `impl` 블록

같은 타입의 고유 `impl`은 여러 블록으로 나눠도 됩니다.

```rok
impl Point {
    fn origin() -> Point { ... }
}

impl Point {
    fn distance(self, other: Point) -> f64 { ... }
}
```

블록을 나누는 것은 보기 좋게 정리하는 일일 뿐입니다. 결과는 한 본체에 모은 것과 같습니다.

다만 같은 타입에 같은 트레잇의 `impl`은 한 번만 둘 수 있습니다.

```rok
impl Display for Point { ... }
impl Display for Point { ... }   // ERROR: 중복 구현
```

## 2.9.4. 제네릭 `impl`

`impl`도 제네릭으로 선언할 수 있습니다.

```rok
impl<T> List<T> {
    fn len(self) -> usize { ... }
}

impl<T> Add for Pair<T> where T: Add<T> {
    type Output = Pair<T::Output>;
    fn add(self, rhs: Pair<T>) -> Pair<T::Output> { ... }
}
```

`impl` 뒤에 `<...>`로 타입 매개변수를 선언합니다. 이 매개변수는 대상 타입, 트레잇 인자, 연관 타입 어디에든 나올 수 있습니다. 트레잇 선언과 같은 방식으로 `where` 절로 추가 제약을 둘 수 있습니다.

제네릭 `impl`은 그 매개변수로 만들 수 있는 대상 인스턴스를 덮습니다. `impl<V> P<str, V>`는 `P<str, i32>`에는 메서드를 주지만 `P<i32, i32>`에는 주지 않습니다.

## 2.9.5. Orphan 규칙

한 패키지에서 정의한 트레잇을 다른 패키지에서 정의한 타입에 구현할 수 없습니다.

```rok
// 트레잇 Display, 타입 i32 가 모두 표준 라이브러리(다른 패키지) 정의
impl Display for i32 { ... }     // ERROR: 둘 다 외부 패키지
```

따라서 트레잇과 대상 타입 중 적어도 하나는 현재 패키지에 정의되어 있어야 합니다. 내장 타입(원시 타입, 튜플, 배열, 함수 타입)은 `rok::core`에서 정의한 것으로 봅니다.

이 규칙의 단위는 [소스 단위](../000-meta/003-source-units.md)의 패키지입니다. 그래서 같은 패키지 안의 서로 다른 모듈끼리는 자유롭게 `impl`할 수 있습니다.

이 규칙은 두 패키지가 같은 (트레잇, 타입) 쌍에 서로 충돌하는 `impl`을 만들지 못하게 막습니다. 그래서 일관성이 보장됩니다.

### 일관성(coherence)

한 (트레잇, 타입) 쌍에는 `impl`이 하나뿐입니다. 특수화(specialization)는 없습니다.

- 제네릭 `impl`은 쓰이는 타입마다 단형화됩니다.
- 구체 `impl`은 그 구체 타입이 제네릭 `impl`의 바운드를 만족할 때만 제네릭 `impl`과 겹칩니다.
- 두 제네릭 `impl`은 대상과 트레잇 인자가 단일화(unify)되면 겹칩니다. 컴파일러는 이를 선언 시점에 에러로 보고합니다. 단일화 뒤에도 제네릭으로 남는 매개변수의 바운드로는 둘을 구별하지 않습니다. 두 바운드를 모두 만족하는 타입이 아직 없더라도 에러입니다(Rust의 coherence와 같음).
- 겹침을 따질 때 `T?`와 `T | none`은 같은 타입입니다.
- 부정 추론은 알려진 타입에만 합니다.
  1. 단일화로 구체 타입이 된 매개변수는 그 바운드를 검사합니다. `impl<U: Num> X for Pair<U, i32>`와 `impl<U> X for Pair<str, U>`를 단일화하면 `U = str`입니다. `str: Num`이 성립하지 않으므로 둘은 겹치지 않습니다.
  2. 알려진 타입 생성자의 바운드(`List<T>: Iterator`)는 그 트레잇의 impl 중에 그 생성자나 맨 매개변수를 대상으로 하는 것이 없으면 둘을 구별합니다. 그래서 `impl<I: Iterator> IntoIterator for I`와 `impl<T> IntoIterator for List<T>`가 함께 있을 수 있습니다.

```rok
impl<T: Display> Describe for List<T> { ... }
impl<T: Hash> Describe for List<T> { ... }   // ERROR: i32 가 Display 와 Hash 를 모두 만족
impl<T: Shape> Area for T { ... }
impl<T: Color> Area for T { ... }            // ERROR: T 와 T 가 단일화됨 (바운드 무관)
```

```rok
impl<T: Display> Describe for T { ... }
impl Describe for i32 { ... }        // ERROR: i32: Display 라서 위 impl 과 겹침
```

## 2.9.6. 메서드 가시성

고유 `impl`의 메서드는 기본적으로 모듈 비공개입니다. 같은 모듈 안에서는 자유롭게 호출할 수 있습니다. 외부에 노출하려면 `pub`을 적어야 합니다.

```rok
impl Account {
    pub fn balance(self) -> f64 { self.amount }      // 외부 공개
        fn _internal_check(self) { ... }              // 모듈 내부만
}
```

이 규칙은 [`struct`](203-struct.md)와 [`enum`](204-enum.md)의 가시성 모델과 같습니다. `impl` 블록 자체는 보기 위한 묶음일 뿐 가시성 단위가 아닙니다. 가시성 단위는 언제나 모듈입니다.

트레잇 `impl`의 메서드는 그 트레잇의 가시성을 따릅니다. 가시성을 따로 표시할 수 없습니다.

## 2.9.7. 고유 `impl`과 트레잇 `impl`의 메서드 충돌

같은 메서드 이름이 고유 `impl`과 트레잇 `impl` 양쪽에 있으면 호출하는 쪽이 모호하지 않게 골라야 합니다. 정확한 해소 규칙은 [메서드](../004-functions/409-method.md#트레잇-사이의-이름-충돌)에서 정의합니다.

## 2.9.8. 동일 메서드 두 번 정의 금지

같은 타입에 같은 이름의 메서드를 두 번 정의하면 컴파일 에러입니다. 한 `impl` 블록 안이든, 여러 고유 `impl` 블록([2.9.3](#293-한-타입에-대한-여러-impl-블록))에 걸쳐서든 마찬가지입니다. 한 트레잇 `impl` 안에서도 같습니다. 오버로딩은 없습니다. 서로 다른 트레잇에서 온 같은 이름의 메서드는 있어도 됩니다([2.9.7](#297-고유-impl과-트레잇-impl의-메서드-충돌)).
