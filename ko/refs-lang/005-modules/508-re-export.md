# 5.8. 재내보내기 (`pub use`)

`use` 에 가시성 키워드를 결합하면 가져온 항목을 현재 모듈에서 다시 외부로 노출하는 재내보내기가 됩니다. 문법은 다음과 같습니다.

```
re_export       ::= visibility "use" use_tree ";"
```

## 5.8.1. 형태

다음과 같이 가시성 키워드와 `use`, 경로, 세미콜론을 결합하여 작성합니다.

```rok
pub use foo::bar;
pub(pkg) use pkg::internal::helper;
pub(sub) use sup::shared;
```

`pub` 또는 다른 가시성 키워드를 `use`, 경로, 그리고 `;` 와 함께 쓰면 가져온 항목이 현재 모듈에서 해당 가시성으로 다시 노출됩니다.

## 5.8.2. 의미

```rok
// pkg::deep::Item
mod deep {
    pub struct Item { ... }
}

// 루트
pub use deep::Item;             // Item 이 루트에서도 pub
```

이렇게 하면 외부 패키지에서 두 가지 경로로 같은 항목에 접근할 수 있습니다.

```rok
use my_pkg::deep::Item;          // 원래 경로 — OK
use my_pkg::Item;                // 재내보내기 경로 — OK
```

같은 항목이 두 경로로 보이게 됩니다.

## 5.8.3. 사용처

### facade 패턴

내부 구조를 감추고 외부에는 평탄한 API 를 제공하고 싶을 때 활용합니다.

```rok
// pkg::core::geom::Point
mod core {
    pub mod geom {
        pub struct Point { ... }
    }
}

// 루트 — 외부에 평탄하게
pub use core::geom::Point;
pub use core::geom::distance;
```

이 경우 외부 사용자는 `my_pkg::Point` 로 접근하면 되고, 내부 구조 (`core::geom::`) 는 알 필요가 없습니다.

### prelude 만들기

자주 쓰는 항목을 한 곳에 모아두는 prelude 모듈도 같은 방식으로 만듭니다.

```rok
// pkg::prelude
pub use pkg::types::{Point, Circle, Polygon};
pub use pkg::ops::{distance, area};
```

사용자는 `use my_pkg::prelude::*;` 한 줄로 자주 쓰는 항목을 한꺼번에 도입할 수 있습니다.

### 버전 호환

내부 항목의 위치가 바뀌어도 재내보내기 경로를 유지하면 외부 사용자에게는 이전 경로가 계속 작동합니다.

```rok
// v1: pkg::Item
pub struct Item { ... }

// v2: 위치 변경
pub mod core { pub struct Item { ... } }
pub use core::Item;             // v1 의 경로 유지
```

## 5.8.4. 가시성 단계의 결합

가시성 키워드와 결합한 재내보내기는 다음 단계로 정리됩니다.

```
pub use         → 외부 패키지에 노출
pub(pkg) use    → 패키지 내부에 노출
pub(sub) use    → 하위 모듈에 노출
use             → 모듈 비공개 (재내보내기 아님)
```

이때 원본 항목의 가시성 ≥ 재내보내기 가시성이어야 합니다. 이름이 명시된 `pub use` (단일 항목 재내보내기) 가 원본보다 더 넓게 노출하려고 하면 ERROR 가 됩니다.

```rok
mod inner {
    fn private_fn() { ... }      // private
}

pub use inner::private_fn;       // ERROR — private 을 pub 으로 못 노출
```

다만 glob 재내보내기 (`pub use mod::*`, `pub use mod::`) 는 동작이 다릅니다. 원본 가시성이 재내보내기 가시성보다 좁은 항목은 조용히 걸러지며 ERROR 가 아닙니다 ([5.8.12 glob 재내보내기](#5812-glob-재내보내기와-가시성)).

## 5.8.5. 다중 재내보내기

여러 항목을 한 번에 재내보내거나 별칭과 함께 묶어 표현할 수도 있습니다.

```rok
pub use foo::{a, b, c};
pub use foo::{
    Point,
    Circle,
    distance as dist,
};
```

[`use`](505-use.md) 의 다중 import 와 별칭 형태를 그대로 적용할 수 있습니다.

## 5.8.6. glob 재내보내기

전체 모듈이나 서브트리를 한꺼번에 재내보낼 때는 다음과 같이 작성합니다.

```rok
pub use foo::*;
pub use foo::**;
```

이렇게 하면 모듈 전체나 그 아래 서브트리가 한 번에 외부로 노출됩니다.

> [!WARNING]
> 같은 이름이 충돌하면 재내보내기 단계에서 에러가 납니다.

## 5.8.7. 별칭과 결합

별칭을 결합하면 외부에 보이는 이름을 짧게 다듬을 수 있습니다.

```rok
pub use foo::very_long_name as short_name;
```

외부에서는 짧은 이름 (`short_name`) 으로만 보이며, 원래 이름 (`very_long_name`) 을 쓰려면 별도의 import 가 필요합니다.

## 5.8.8. 모듈 자체의 재내보내기

항목뿐만 아니라 모듈 자체를 재내보낼 수도 있습니다.

```rok
pub use foo::geom;               // 모듈 자체 재내보내기
```

이 경우 외부 사용자는 `my_pkg::geom::Point` 로 접근할 수 있습니다.

## 5.8.9. `prelude` 패턴

라이브러리 작성자는 자주 쓰는 항목을 모은 `prelude` 모듈을 만들어 외부 노출 표면을 큐레이션하는 관습을 씁니다. 컴파일러가 자동으로 끌어오지는 않으며, 사용자가 명시적으로 가져와야 합니다.

```rok
// pkg::prelude
pub use pkg::collections::{List, Dict, Set};
pub use pkg::ops::{Add, Sub, Mul};
pub use pkg::iter::Iterator;
```

사용자 측에서는 다음과 같이 한 줄로 가져옵니다.

```rok
// 사용자 측
use my_pkg::prelude::*;
```

`prelude` 라는 이름에 컴파일러가 부여하는 특수 의미는 없으며, 일반 모듈로 취급됩니다.

## 5.8.10. 순환 재내보내기 (없음)

재내보내기가 서로를 가리키는 순환은 허용되지 않습니다.

```rok
mod a { pub use pkg::b::Item; }
mod b { pub use pkg::a::Item; }
// ERROR — 순환 의존
```

이러한 순환은 ERROR 이며, 컴파일러가 검출합니다.

## 5.8.11. 구현 측면

재내보내기는 이름 표에 항목으로 추가될 뿐입니다. 실제 코드와 메모리는 원본 항목 한 곳에만 존재하며, 별도의 wrapper 함수나 복사가 없으므로 비용이 0 입니다.

## 5.8.12. glob 재내보내기와 가시성

glob 재내보내기는 원본 항목의 가시성에 따라 노출 여부가 달라집니다.

```rok
mod source {
    pub fn a() { ... }
    pub(pkg) fn b() { ... }
}

pub use source::*;               // a 는 외부에 노출, b 는 pub(pkg) 라 패키지 안에서만 보임
```

`pub use ...::*` 가 끌어오는 항목은 재내보내기 가시성이 호출자에 도달하는 동시에 원본 가시성이 재내보내기 가시성을 허락하는 항목뿐입니다. 즉, reach 가 좁은 항목은 재내보내기로 더 넓힐 수 없습니다 ([5.8.4](#584-가시성-단계의-결합)).

## 5.8.13. 정리

`pub use` 는 내부 구조와 외부 API 를 분리하는 핵심 도구입니다. 사용자는 공개 위치만 알면 되고, 라이브러리 작성자는 내부 위치를 자유롭게 재구성할 수 있습니다.
