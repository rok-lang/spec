# 5.6. 가시성

각 항목(함수, 타입, 트레잇, 모듈, `const`, `static`)에는 가시성이 있습니다. 가시성은 그 항목을 외부에 어디까지 노출할지 정합니다.

```
visibility      ::= "pub" ("(" reach ")")?
reach           ::= "pkg" | "sub"
```

## 5.6.1. 네 단계

가시성은 네 단계입니다.

| 키워드     | 의미                              |
|------------|-----------------------------------|
| (없음)     | 모듈 비공개. 같은 모듈 안에서만.  |
| `pub(sub)` | 같은 모듈 트리의 하위 모듈에서만. |
| `pub(pkg)` | 같은 패키지 안에서만.             |
| `pub`      | 외부 패키지에서도.                |

가장 좁은 범위는 기본값인 private이고, 가장 넓은 범위는 `pub`입니다.

`pkg`와 `sub`은 `pub(...)` 괄호 안에서만 한정자로 동작하는 문맥 키워드입니다. 다른 위치에서는 일반 식별자입니다([1.5.4 가시성 한정자](../001-lexical/105-keywords.md#가시성-한정자-pkg--sub)).

## 5.6.2. 기본: 가시성 키워드 없음

가시성 키워드가 없으면 항목은 모듈 비공개입니다. 같은 모듈 안에서만 보입니다.

```rok
mod m {
    fn helper() -> i32 { 42i32 }   // 모듈 비공개 (기본)
    pub fn api() -> i32 { helper() }
}

let r = m::api();                   // OK
let h = m::helper();                // ERROR - 모듈 비공개
```

비공개 `mod`는 그 모듈을 선언한 부모 모듈과 그 부모의 하위 모듈에서 보입니다.

## 5.6.3. `pub`: 가장 넓은 공개

가장 넓게 공개하려면 `pub`을 붙입니다.

```rok
pub fn calculate(x: i32) -> i32 { ... }
pub struct Point { pub x: f64; pub y: f64; }
pub trait Display { ... }
```

이렇게 선언한 항목은 다른 패키지에서도 `use`로 가져와 쓸 수 있습니다.

## 5.6.4. `pub(pkg)`: 패키지 내부

`pub(pkg)`는 패키지 안에서만 공개합니다.

```rok
pub(pkg) fn internal_helper() -> i32 { ... }
```

같은 패키지 안의 어느 모듈에서나 보이지만, 외부 패키지에서는 보이지 않습니다. 의존 패키지와 `rok::`의 각 계층도 별개의 패키지입니다([5.7.14](507-paths.md#5714-패키지-사이-경로)).

이 단계는 패키지 내부의 라이브러리 도구용입니다. 공개 API는 아니지만 여러 모듈이 공유해야 하는 함수와 타입에 씁니다.

## 5.6.5. `pub(sub)`: 하위 모듈

`pub(sub)`는 모듈 트리의 한 줄기 안에서만 공유할 때 씁니다.

```rok
mod parent {
    pub(sub) fn shared_within_subtree() -> i32 { ... }

    pub mod child {
        pub fn use_it() -> i32 { sup::shared_within_subtree() }   // OK
    }
}
```

`parent`와 `parent`의 모든 자손 모듈에서만 보입니다. 부모 모듈이나 형제 모듈에서는 보이지 않습니다.

이 단계는 모듈 그룹 내부의 도구용입니다. 모듈 트리의 한 줄기 안에서만 공유합니다.

## 5.6.6. 단계의 포함 관계

가시성은 단조 증가하는 격자(lattice)를 이룹니다.

```
pub  ⊇  pub(pkg)  ⊇  pub(sub)  ⊇  private
```

`pub(sub)`의 포함 관계는 관련 모듈, 곧 정의한 모듈 자신과 그 자손 사이에서만 의미가 있습니다.

- `pub`은 모든 호출자에게 보입니다.
- `pub(pkg)`는 같은 패키지 안에서만 보입니다.
- `pub(sub)`는 같은 모듈 트리 안에서만 보입니다.
- private은 같은 모듈에서만 보입니다.

## 5.6.7. glob import와 가시성

glob import(`*`, `**`, 묵시적 `use rok::core::**` 포함)는 가져오는 모듈에서 보이는 항목만 가져옵니다. 도달 범위(reach)가 그 모듈을 포함하는 항목만 대상입니다. 보이지 않는 항목은 이름을 가리지도 않고 충돌하지도 않습니다. 항목별 glob-block 표시는 없습니다. 외부 노출 표면을 좁히려면 별도의 `prelude` 모듈을 만들고 `pub use`로 골라 담으세요.

## 5.6.8. `struct` 필드의 가시성

필드마다 가시성을 따로 붙일 수 있습니다.

```rok
struct Account {
    pub id: i64;                    // 외부 읽기 가능
    pub balance: f64;               // 외부 읽기 가능
        secret: str;                // 모듈 비공개
}
```

필드 가시성은 `struct` 자체의 가시성과 별개로 동작합니다.

```rok
pub struct Counter {                // struct 자체는 외부 가시
        count: i32 = 0i32;          // 필드는 모듈 비공개 (기본)
}

let c = Counter { count: 0i32 };    // ERROR - 필드 비공개라 외부에서 직접 만들 수 없음
let c = Counter::new();             // OK - 팩토리 메서드 (있다면)
```

이 방식은 타입은 외부에 노출하고 직접 인스턴스화는 막는 표준 패턴입니다(newtype, opaque type).

### tuple struct의 필드

tuple struct의 각 필드에도 자기 가시성이 있습니다. 기본은 비공개입니다.

```rok
pub struct S(pub i32, i32);         // 0 번은 pub, 1 번은 비공개
```

- `S(a, b)`로 만들려면 사용처에서 모든 필드가 보여야 합니다. 비공개 필드가 하나라도 있으면 그 가시성 범위 밖에서는 생성자 함수(`NonZero::new`)를 씁니다.
- `x.N`을 읽거나 대입하려면 N번 필드가 보여야 합니다.
- 패턴 `S(p0, p1, ..)`에서 하위 패턴이 `_`가 아닌 자리는 보여야 합니다. 이름 있는 필드의 struct 패턴도 같습니다(건드리는 필드만 검사).
- `pub` 바로 뒤의 `(`는 가시성 제한(`pub(pkg)` 등)으로 읽습니다. 튜플 타입의 필드를 공개하려면 괄호를 한 번 더 씁니다. `struct S(pub ((i32, i32)));`

## 5.6.9. `enum` 변형의 가시성

변형에는 별도의 가시성이 없습니다. `enum`의 가시성을 그대로 따릅니다. 변형 앞에 가시성 키워드를 붙이면 컴파일 에러입니다.

```rok
pub enum Shape {
    Circle(f64),                    // pub - Shape 를 따름
    Square(f64),
}

pub enum Bad {
    pub A,                          // ERROR - 변형에는 가시성을 적지 않음
}
```

변형의 페이로드 필드에도 필드별 가시성이 없고 `enum`을 따릅니다. `struct` 필드와 달리 변형 하나만 숨기면 외부의 `match`가 완전성을 만족할 수 없기 때문입니다. 표준 라이브러리의 `pub enum Result<T, E> { Ok(T), Err(E) }`에서도 이 규칙에 따라 `Ok` / `Err`가 공개됩니다.

## 5.6.10. `impl` 메서드의 가시성

고유 `impl`([`impl`](../002-types/209-impl.md))의 메서드는 기본적으로 모듈 비공개입니다. 외부에 노출하려면 `pub`으로 명시해야 합니다.

```rok
impl Account {
    pub fn balance(self) -> f64 { self.amount }
        fn _internal_check(self) { ... }
}
```

트레잇 `impl`의 메서드는 그 트레잇의 가시성을 따르므로 따로 표시할 수 없습니다.

## 5.6.11. `use`의 가시성

[`use`](505-use.md)로 하는 import에도 가시성이 있습니다.

```rok
use foo::bar;                       // 모듈 비공개 (기본)
pub use foo::bar;                   // 재내보내기 - 외부에서도 보임
pub(pkg) use foo::bar;              // 패키지 내부만
```

재내보내기는 [`use`](505-use.md)의 가시성 절과 함께 외부 노출 표면을 정합니다.

## 5.6.12. 가시성의 위치

가시성 키워드는 항목 선언의 첫 토큰입니다.

```rok
pub fn f() { ... }
pub(pkg) fn g() { ... }
pub struct S { ... }
pub mod m { ... }
pub const X: i32 = 0;
pub static Y: i32 = 0;
pub use foo::bar;
pub trait T { ... }
```

`fn`, `struct`, `enum`, `union`, `trait`, `impl`(`impl` 자체가 아니라 메서드마다 붙입니다), `mod`, `const`, `static`, `use`, `type`(별칭)의 모든 선언 형태에 붙일 수 있습니다.

## 5.6.13. 가시성과 이름 해소

[`use`](505-use.md)와 함께 볼 때, 항목이 현재 모듈에서 보이려면 다음 두 조건을 모두 만족해야 합니다.

1. 항목의 가시성이 현재 모듈에 닿아야 합니다. 경로의 모든 모듈 마디도 각각 보여야 합니다(`a::b::f`라면 `a`, `b`, `f`가 모두 보여야 함).
2. 항목이 직접 보여야 합니다(자식 모듈이나 `use`로 가져온 이름).

두 조건을 모두 만족하면 그 항목을 짧은 이름으로 쓸 수 있습니다.

항목이 보이지 않으면 컴파일러 진단이 그 이유를 알려 줍니다.

```
ERROR: 'foo::secret' is not visible from this module.
       It is declared as private - only visible within 'foo'.
       To use it externally, add 'pub' or other visibility keyword.
```
