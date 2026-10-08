# 5.8. 재내보내기 (`pub use`)

`use` 에 가시성 키워드를 결합하면, 가져온 항목을 현재 모듈에서 다시 외부로 노출하는 재내보내기가 된다.

```
re_export       ::= visibility "use" use_tree ";"
```

## 5.8.1. 형태

```nok
pub use foo::bar;
pub(pkg) use crate::internal::helper;
pub(sub) use super::shared;
pub(shy) use crate::deep::rare;
```

- `pub` (또는 다른 가시성 키워드) + `use` + 경로 + `;`.
- 가져온 항목이 현재 모듈에서 해당 가시성으로 다시 노출된다.

## 5.8.2. 의미

```nok
// crate::deep::Item
mod deep {
    pub struct Item { ... }
}

// 루트
pub use deep::Item;             // Item 이 루트에서도 pub
```

외부 패키지에서:

```nok
use my_pkg::deep::Item;          // 원래 경로 — OK
use my_pkg::Item;                // 재내보내기 경로 — OK
```

같은 항목이 두 경로로 보인다.

## 5.8.3. 사용처

### facade 패턴

내부 구조를 감추고 외부에는 평탄한 API 를 제공한다.

```nok
// crate::core::geom::Point
mod core {
    pub mod geom {
        pub struct Point { ... }
    }
}

// 루트 — 외부에 평탄하게
pub use core::geom::Point;
pub use core::geom::distance;
```

외부 사용자는 `my_pkg::Point` 로 접근한다. 내부 구조 (`core::geom::`) 는 몰라도 된다.

### prelude 만들기

```nok
// crate::prelude
pub use crate::types::{Point, Circle, Polygon};
pub use crate::ops::{distance, area};
```

사용자는 `use my_pkg::prelude::*;` 한 줄로 자주 쓰는 항목을 한꺼번에 도입한다.

### 버전 호환

내부 항목의 위치가 바뀌어도 재내보내기 경로를 유지하면 외부 사용자에게는 이전 경로가 계속 작동한다.

```nok
// v1: crate::Item
pub struct Item { ... }

// v2: 위치 변경
pub mod core { pub struct Item { ... } }
pub use core::Item;             // v1 의 경로 유지
```

## 5.8.4. 가시성 단계의 결합

```
pub use         → 외부 패키지에 노출
pub(pkg) use    → 패키지 내부에 노출
pub(sub) use    → 하위 모듈에 노출
pub(shy) use    → 명시 import 만 노출
use             → 모듈 비공개 (재내보내기 아님)
```

원본 항목의 가시성 ≥ 재내보내기 가시성이어야 한다. 원본보다 더 넓게 노출하려고 하면 ERROR.

```nok
mod inner {
    fn private_fn() { ... }      // private
}

pub use inner::private_fn;       // ERROR — private 을 pub 으로 못 노출
```

## 5.8.5. 다중 재내보내기

```nok
pub use foo::{a, b, c};
pub use foo::{
    Point,
    Circle,
    distance as dist,
};
```

[`use`](505-use.md) 의 다중 import 와 별칭 형태를 그대로 적용한다.

## 5.8.6. glob 재내보내기

```nok
pub use foo::*;
pub use foo::**;
```

전체 모듈이나 서브트리를 한꺼번에 재노출한다.

> [!WARNING]
> 같은 이름이 충돌하면 재내보내기 단계에서 에러가 난다.

## 5.8.7. 별칭과 결합

```nok
pub use foo::very_long_name as short_name;
```

외부에서는 짧은 이름 (`short_name`) 으로만 보인다. 원래 이름 (`very_long_name`) 을 쓰려면 별도 import 가 필요하다.

## 5.8.8. 모듈 자체의 재내보내기

```nok
pub use foo::geom;               // 모듈 자체 재노출
```

외부 사용자는 `my_pkg::geom::Point` 로 접근한다.

## 5.8.9. 재내보내기와 자동 prelude

[표준 라이브러리 계층](../000-meta/005-stdlib-tiers.md) 의 `core/prelude.nok` 는 재내보내기 모음이다. 표준 라이브러리에서 자주 쓰는 타입과 트레잇을 한 모듈에 모아 자동 가시 처리한다.

```nok
// core/prelude.nok 의 일부 (예시)
pub use core::collections::{List, Dict, Set};
pub use core::ops::{Add, Sub, Mul};
pub use core::iter::Iterator;
```

자동 가시는 사용자 코드에서 `use core::prelude::*;` 가 암묵적으로 적용된 것과 같은 효과다.

## 5.8.10. 순환 재내보내기 (없음)

```nok
mod a { pub use crate::b::Item; }
mod b { pub use crate::a::Item; }
// ERROR — 순환 의존
```

재내보내기가 서로를 가리키는 순환은 ERROR. 컴파일러가 검출한다.

## 5.8.11. 구현 측면

재내보내기는 이름 표에 항목으로 추가될 뿐이다. 실제 코드와 메모리는 원본 항목 한 곳에만 있다. 별도 wrapper 함수나 복사가 없으므로 비용이 0 이다.

## 5.8.12. glob import 의 가시성 전달

```nok
mod source {
    pub fn a() { ... }
    pub(shy) fn b() { ... }
}

pub use source::*;               // a 만 재노출 (b 는 pub(shy) 로 glob 차단)
```

[가시성](506-visibility.md) 의 `pub(shy)` 차단 규칙이 glob 재내보내기에도 그대로 적용된다.

## 5.8.13. 정리

`pub use` 는 내부 구조와 외부 API 를 분리하는 핵심 도구다. 사용자는 공개 위치만 알면 되고, 라이브러리 작성자는 내부 위치를 자유롭게 재구성한다.
