# 5.6. 가시성

각 항목 (함수, 타입, 트레잇, 모듈, `const`, `static`) 에 가시성을 부여해 외부 노출 범위를 조절한다.

```
visibility      ::= "pub" ("(" vis_qualifier ")")?
vis_qualifier   ::= "shy" | "sub" | "pkg"
```

## 5.6.1. 네 단계

| 키워드     | 의미                                 |
| ---------- | ------------------------------------ |
| (없음)     | 모듈 비공개. 같은 모듈 안에서만.     |
| `pub(shy)` | 외부에서 명시 import 만 (glob 차단). |
| `pub(sub)` | 같은 모듈 트리의 하위 모듈에서만.    |
| `pub(pkg)` | 같은 패키지 안에서만.                |
| `pub`      | 외부 패키지에서도.                   |

가장 좁은 것이 기본 (private) 이고, 가장 넓은 것이 `pub` 이다.

## 5.6.2. 기본: 가시성 키워드 없음

가시성 키워드를 적지 않으면 그 항목은 모듈 비공개로 같은 모듈 안에서만 보인다.

```nok
mod m {
    fn helper() -> i32 { 42i32 }   // 모듈 비공개 (기본)
    pub fn api() -> i32 { helper() }
}

let r = m::api();                   // OK
let h = m::helper();                // ERROR — 모듈 비공개
```

## 5.6.3. `pub`: 가장 넓은 공개

```nok
pub fn calculate(x: i32) -> i32 { ... }
pub struct Point { pub x: f64; pub y: f64; }
pub trait Display { ... }
```

다른 패키지에서도 `use` 로 가져와 사용할 수 있다.

## 5.6.4. `pub(pkg)`: 패키지 내부

```nok
pub(pkg) fn internal_helper() -> i32 { ... }
```

같은 패키지 안의 어느 모듈에서나 가시화된다. 외부 패키지에서는 보이지 않는다.

패키지 내부 라이브러리 도구를 위한 단계다. 공개 API 는 아니지만 모듈을 가로질러 공유해야 하는 함수와 타입에 쓴다.

## 5.6.5. `pub(sub)`: 하위 모듈

```nok
mod parent {
    pub(sub) fn shared_within_subtree() -> i32 { ... }

    pub mod child {
        pub fn use_it() -> i32 { super::shared_within_subtree() }   // OK
    }
}
```

`parent` 와 `parent` 의 모든 자손 모듈에서만 가시화된다. 부모와 형제 모듈에서는 보이지 않는다.

모듈 그룹 내부 도구를 위한 단계다. 모듈 트리의 한 가지 안에서만 공유한다.

## 5.6.6. `pub(shy)`: glob 차단

```nok
mod inner {
    pub(shy) fn rare() -> i32 { ... }
}

use inner::rare;                    // OK — 명시 import
use inner::*;                        // ERROR — glob 으로는 차단
```

- 외부에서 명시적 `use 경로::이름` 으로만 가져올 수 있다.
- `use 경로::*` 와 `use 경로::**` 같은 glob import 는 차단된다.
- 명시 import 후에는 일반 `pub` 처럼 쓸 수 있다.

드물게 쓰지만 외부 노출은 가능한 항목을 위한 단계다. 일반 사용자가 glob 으로 모듈을 가져왔을 때 자동으로 등장하지 않게 하여 잡음과 충돌이 줄어든다.

## 5.6.7. 단계의 포함 관계

가시성은 집합 포함 관계를 가진다.

```
pub  ⊇  pub(pkg)  ⊇  pub(sub) (관련 모듈에 대해)  ⊇  pub(shy) (관련 임포트에 대해)  ⊇  private
```

- `pub` 는 모든 호출자에게 보인다.
- `pub(shy)` 는 명시 import 에만 보이며, glob 으로는 보이지 않는다.
- `pub(sub)` 는 같은 모듈 트리 안에서만 보이고, 그 안에서는 (`pub(shy)` 와 무관하게) 가시화된다.
- `pub(pkg)` 는 같은 패키지 안에서만 보인다.

이 단계들은 외부 패키지에서 보이는지, 내부에서 보이는지, 명시 / glob 으로 보이는지 세 차원의 조합이다.

## 5.6.8. `struct` 필드의 가시성

```nok
struct Account {
    pub id: i64;                    // 외부 읽기 가능
    pub balance: f64;               // 외부 읽기 가능
        secret: str;                // 모듈 비공개
}
```

각 필드에 별도의 가시성을 부여한다. `struct` 자체의 가시성과는 별개다.

```nok
pub struct Counter {                // struct 자체는 외부 가시
        count: i32 = 0i32;          // 필드는 모듈 비공개 (기본)
}

let c = Counter { count: 0i32 };    // ERROR — 필드 비공개라 외부에서 직접 만들 수 없음
let c = Counter::new();             // OK — 팩토리 메서드 (있다면)
```

타입은 외부에 노출하되 직접 인스턴스화는 막는 표준 패턴이다 (newtype, opaque type).

## 5.6.9. `enum` 변형의 가시성

```nok
pub enum Result {
    pub Ok(i32),
    pub Err(str),
        Pending,                    // 모듈 비공개
}
```

각 변형도 별도의 가시성을 가진다. 변형 자체가 비공개면 외부에서 매칭이나 생성이 불가능하다.

## 5.6.10. `impl` 메서드의 가시성

[`impl`](../002-types/209-impl.md) 의 본질 `impl` 메서드는 기본적으로 모듈 비공개다. 외부 노출은 `pub` 으로 명시한다.

```nok
impl Account {
    pub fn balance(self) -> f64 { self.amount }
        fn _internal_check(self) { ... }
}
```

트레잇 `impl` 의 메서드는 그 트레잇의 가시성을 따른다. 별도로 표시할 수 없다.

## 5.6.11. `use` 의 가시성

[`use`](505-use.md) 의 import 도 가시성을 가진다.

```nok
use foo::bar;                       // 모듈 비공개 (기본)
pub use foo::bar;                   // 재내보내기 — 외부에서도 보임
pub(pkg) use foo::bar;              // 패키지 내부만
```

재내보내기는 [`use`](505-use.md) 의 가시성 절과 결합해 외부 노출 표면을 정한다.

## 5.6.12. 가시성의 위치

가시성 키워드는 항목 선언의 첫 토큰으로 등장한다.

```nok
pub fn f() { ... }
pub(pkg) fn g() { ... }
pub struct S { ... }
pub mod m { ... }
pub const X: i32 = 0;
pub static Y: i32 = 0;
pub use foo::bar;
pub trait T { ... }
```

`fn`, `struct`, `enum`, `union`, `trait`, `impl` (`impl` 자체에는 없음. 메서드별로), `mod`, `const`, `static`, `use`, `type` (별칭) 의 모든 선언 형태에 부착할 수 있다.

## 5.6.13. 가시성과 이름 해소

[`use`](505-use.md) 와 결합해 한 항목이 현재 모듈에서 보이려면 다음 두 조건을 모두 만족해야 한다.

1. 항목의 가시성이 현재 모듈에서 도달 가능해야 한다.
2. 항목이 직접 가시 (자식 / 가져온 / 자동 prelude) 해야 한다.

두 조건을 모두 만족하면 그 항목을 짧은 이름으로 쓸 수 있다.

가시성이 부족한 항목은 컴파일러 진단이 보이지 않는 이유를 알려준다.

```
ERROR: 'foo::secret' is not visible from this module.
       It is declared as private — only visible within 'foo'.
       To use it externally, add 'pub' or other visibility keyword.
```
