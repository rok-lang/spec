# 5.6. 가시성

함수, 타입, 트레잇, 모듈, `const`, `static` 같은 각 항목에 가시성을 부여하면 외부 노출 범위를 조절할 수 있습니다.

```
visibility      ::= "pub" ("(" reach ")")?
reach           ::= "pkg" | "sub"
```

## 5.6.1. 네 단계

가시성은 다음과 같이 네 단계로 나뉩니다.

| 키워드     | 의미                              |
|------------|-----------------------------------|
| (없음)     | 모듈 비공개. 같은 모듈 안에서만.  |
| `pub(sub)` | 같은 모듈 트리의 하위 모듈에서만. |
| `pub(pkg)` | 같은 패키지 안에서만.             |
| `pub`      | 외부 패키지에서도.                |

가장 좁은 범위가 기본값인 private 이고, 가장 넓은 범위가 `pub` 입니다.

`pkg` 와 `sub` 은 `pub(...)` 괄호 안에서만 한정자로 동작하는 컨텍스트 키워드여서, 그 외의 위치에서는 일반 식별자로 쓰입니다 ([1.5.4 가시성 한정자](../001-lexical/105-keywords.md#가시성-한정자-pkg--sub) 참고).

## 5.6.2. 기본: 가시성 키워드 없음

가시성 키워드를 적지 않으면 그 항목은 모듈 비공개가 되어 같은 모듈 안에서만 보입니다.

```rok
mod m {
    fn helper() -> i32 { 42i32 }   // 모듈 비공개 (기본)
    pub fn api() -> i32 { helper() }
}

let r = m::api();                   // OK
let h = m::helper();                // ERROR — 모듈 비공개
```

## 5.6.3. `pub`: 가장 넓은 공개

가장 넓은 공개 범위를 원할 때는 `pub` 을 붙입니다.

```rok
pub fn calculate(x: i32) -> i32 { ... }
pub struct Point { pub x: f64; pub y: f64; }
pub trait Display { ... }
```

이렇게 선언한 항목은 다른 패키지에서도 `use` 로 가져와 사용할 수 있습니다.

## 5.6.4. `pub(pkg)`: 패키지 내부

`pub(pkg)` 는 패키지 내부 공개를 의미합니다.

```rok
pub(pkg) fn internal_helper() -> i32 { ... }
```

같은 패키지 안의 어느 모듈에서나 가시화되지만, 외부 패키지에서는 보이지 않습니다.

이 단계는 패키지 내부 라이브러리 도구를 위한 것이라서, 공개 API 는 아니지만 모듈을 가로질러 공유해야 하는 함수와 타입에 사용합니다.

## 5.6.5. `pub(sub)`: 하위 모듈

`pub(sub)` 는 모듈 트리의 한 가지 안에서만 공유하고 싶을 때 씁니다.

```rok
mod parent {
    pub(sub) fn shared_within_subtree() -> i32 { ... }

    pub mod child {
        pub fn use_it() -> i32 { sup::shared_within_subtree() }   // OK
    }
}
```

`parent` 와 `parent` 의 모든 자손 모듈에서만 가시화되고, 부모 모듈이나 형제 모듈에서는 보이지 않습니다.

이 단계는 모듈 그룹 내부 도구를 위한 것이라서, 모듈 트리의 한 가지 안에서만 공유하는 용도로 사용합니다.

## 5.6.6. 단계의 포함 관계

가시성은 단조 증가 lattice 를 이룹니다.

```
pub  ⊇  pub(pkg)  ⊇  pub(sub)  ⊇  private
```

`pub(sub)` 의 포함 관계는 관련 모듈, 즉 정의 모듈 자신과 그 자손 안에서만 의미가 있습니다.

- `pub` 는 모든 호출자에게 보입니다.
- `pub(pkg)` 는 같은 패키지 안에서만 보입니다.
- `pub(sub)` 는 같은 모듈 트리 안에서만 보입니다.
- private 은 같은 모듈에서만 보입니다.

## 5.6.7. glob import 와 가시성

glob import (`*`, ``) 가 가져오는 항목은 그 호출자에 대해 가시한 항목입니다. 즉 reach 가 호출자를 포함해야 끌려옵니다. 별도의 per-item glob-block 표시는 두지 않으니, 외부 노출 표면을 좁히고 싶다면 별도의 `prelude` 모듈을 만들고 `pub use` 로 큐레이션하는 패턴을 쓰면 됩니다.

## 5.6.8. `struct` 필드의 가시성

각 필드에는 별도로 가시성을 부여할 수 있습니다.

```rok
struct Account {
    pub id: i64;                    // 외부 읽기 가능
    pub balance: f64;               // 외부 읽기 가능
        secret: str;                // 모듈 비공개
}
```

이때 부여한 필드 가시성은 `struct` 자체의 가시성과는 별개로 동작합니다.

```rok
pub struct Counter {                // struct 자체는 외부 가시
        count: i32 = 0i32;          // 필드는 모듈 비공개 (기본)
}

let c = Counter { count: 0i32 };    // ERROR — 필드 비공개라 외부에서 직접 만들 수 없음
let c = Counter::new();             // OK — 팩토리 메서드 (있다면)
```

이는 타입은 외부에 노출하되 직접 인스턴스화는 막는 표준 패턴입니다 (newtype, opaque type).

## 5.6.9. `enum` 변형의 가시성

각 변형도 별도의 가시성을 가집니다.

```rok
pub enum Result {
    pub Ok(i32),
    pub Err(str),
        Pending,                    // 모듈 비공개
}
```

변형 자체가 비공개라면 외부에서 매칭하거나 생성할 수 없습니다.

## 5.6.10. `impl` 메서드의 가시성

[`impl`](../002-types/209-impl.md) 의 고유 `impl` 메서드는 기본적으로 모듈 비공개여서, 외부에 노출하려면 `pub` 으로 명시해야 합니다.

```rok
impl Account {
    pub fn balance(self) -> f64 { self.amount }
        fn _internal_check(self) { ... }
}
```

트레잇 `impl` 의 메서드는 그 트레잇의 가시성을 따르기 때문에 별도로 표시할 수 없습니다.

## 5.6.11. `use` 의 가시성

[`use`](505-use.md) 의 import 도 가시성을 가집니다.

```rok
use foo::bar;                       // 모듈 비공개 (기본)
pub use foo::bar;                   // 재내보내기 — 외부에서도 보임
pub(pkg) use foo::bar;              // 패키지 내부만
```

재내보내기는 [`use`](505-use.md) 의 가시성 절과 결합해 외부 노출 표면을 결정합니다.

## 5.6.12. 가시성의 위치

가시성 키워드는 항목 선언의 첫 토큰으로 등장합니다.

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

`fn`, `struct`, `enum`, `union`, `trait`, `impl` (`impl` 자체에는 붙이지 않고 메서드별로 붙입니다), `mod`, `const`, `static`, `use`, `type` (별칭) 의 모든 선언 형태에 부착할 수 있습니다.

## 5.6.13. 가시성과 이름 해소

[`use`](505-use.md) 와 결합해 한 항목이 현재 모듈에서 보이려면 다음 두 조건을 모두 만족해야 합니다.

1. 항목의 가시성이 현재 모듈에서 도달 가능해야 합니다.
2. 항목이 직접 가시 (자식 모듈이나 `use` 로 가져온 이름) 해야 합니다.

두 조건을 모두 만족하면 그 항목을 짧은 이름으로 쓸 수 있습니다.

가시성이 부족한 항목은 컴파일러 진단이 보이지 않는 이유를 보고합니다.

```
ERROR: 'foo::secret' is not visible from this module.
       It is declared as private — only visible within 'foo'.
       To use it externally, add 'pub' or other visibility keyword.
```
