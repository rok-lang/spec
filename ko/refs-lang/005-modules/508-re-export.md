# 5.8. 재내보내기 (`pub use`)

`use`에 가시성 키워드를 붙이면 재내보내기가 됩니다. 가져온 항목을 현재 모듈에서 다시 외부로 노출합니다. 문법은 다음과 같습니다.

```
re_export       ::= visibility "use" use_tree ";"
```

## 5.8.1. 형태

재내보내기는 가시성 키워드, `use`, 경로, 세미콜론으로 이루어집니다.

```rok
pub use foo::bar;
pub(pkg) use pkg::internal::helper;
pub(sub) use sup::shared;
```

`pub`이나 다른 가시성 키워드를 `use`, 경로, `;`와 함께 쓰면 가져온 항목이 현재 모듈에서 그 가시성으로 다시 노출됩니다.

## 5.8.2. 의미

```rok
// pkg::deep::Item
mod deep {
    pub struct Item { ... }
}

// 루트
pub use deep::Item;             // Item 이 루트에서도 pub
```

그러면 외부 패키지는 두 경로로 같은 항목에 접근할 수 있습니다.

```rok
use my_pkg::deep::Item;          // 원래 경로: OK
use my_pkg::Item;                // 재내보내기 경로: OK
```

같은 항목이 두 경로로 보입니다.

## 5.8.3. 사용처

### facade 패턴

내부 구조를 감추고 외부에는 평탄한 API를 내놓을 때 씁니다.

```rok
// pkg::core::geom::Point
mod core {
    pub mod geom {
        pub struct Point { ... }
    }
}

// 루트: 외부에 평탄하게
pub use core::geom::Point;
pub use core::geom::distance;
```

외부 사용자는 `my_pkg::Point`로 접근하며, 내부 구조(`core::geom::`)를 알 필요가 없습니다.

### prelude 만들기

자주 쓰는 항목을 한곳에 모은 prelude 모듈도 같은 방식으로 만듭니다.

```rok
// pkg::prelude
pub use pkg::types::{Point, Circle, Polygon};
pub use pkg::ops::{distance, area};
```

사용자는 `use my_pkg::prelude::*;` 한 줄로 자주 쓰는 항목을 한꺼번에 가져올 수 있습니다.

### 버전 호환

내부 항목의 위치가 바뀌어도 재내보내기 경로를 유지하면 외부 사용자는 이전 경로를 계속 쓸 수 있습니다.

```rok
// v1: pkg::Item
pub struct Item { ... }

// v2: 위치 변경
pub mod core { pub struct Item { ... } }
pub use core::Item;             // v1 의 경로 유지
```

## 5.8.4. 가시성 단계의 결합

가시성 키워드별 재내보내기는 다음과 같습니다.

```
pub use         → 외부 패키지에 노출
pub(pkg) use    → 패키지 내부에 노출
pub(sub) use    → 하위 모듈에 노출
use             → 모듈 비공개 (재내보내기 아님)
```

가시성의 순서는 비공개 < `pub(sub)` < `pub(pkg)` < `pub`입니다. 원본 항목의 가시성은 재내보내기 가시성 이상이어야 하며, 원본보다 넓은 `pub use`는 에러입니다.

```rok
mod inner {
    fn private_fn() { ... }      // private
}

pub use inner::private_fn;       // ERROR: private 을 pub 으로 못 노출
```

단, glob 재내보내기(`pub use mod::*`, `pub use mod::**`)는 다르게 동작합니다. 원본 가시성이 재내보내기 가시성보다 좁은 항목은 조용히 걸러지며, ERROR가 아닙니다([5.8.12 glob 재내보내기](#5812-glob-재내보내기와-가시성)).

## 5.8.5. 다중 재내보내기

여러 항목을 한 번에 재내보낼 수 있고, 별칭도 함께 쓸 수 있습니다.

```rok
pub use foo::{a, b, c};
pub use foo::{
    Point,
    Circle,
    distance as dist,
};
```

[`use`](505-use.md)의 다중 import와 별칭 형태가 그대로 적용됩니다.

## 5.8.6. glob 재내보내기

모듈 전체나 서브트리를 한꺼번에 재내보내려면 다음과 같이 씁니다.

```rok
pub use foo::*;
pub use foo::**;
```

모듈 전체나 그 아래 서브트리가 한 번에 외부로 노출됩니다.

> [!WARNING]
> 이름이 충돌하면 그 재내보내기에서 에러입니다.

## 5.8.7. 별칭과 결합

별칭으로 외부에 보이는 이름을 짧게 만들 수 있습니다.

```rok
pub use foo::very_long_name as short_name;
```

외부에서는 짧은 이름(`short_name`)만 보이며, 원래 이름(`very_long_name`)을 쓰려면 별도로 import해야 합니다.

## 5.8.8. 모듈 자체의 재내보내기

항목뿐 아니라 모듈 자체도 재내보낼 수 있습니다.

```rok
pub use foo::geom;               // 모듈 자체 재내보내기
```

외부 사용자는 `my_pkg::geom::Point`로 접근할 수 있습니다. 재내보낸 모듈을 거치는 경로는 원래 모듈을 거치는 경로와 똑같이 해소됩니다.

## 5.8.9. `prelude` 패턴

라이브러리 작성자는 관례적으로 자주 쓰는 항목을 모은 `prelude` 모듈을 만들어 외부 노출 표면을 다듬습니다. 컴파일러가 자동으로 가져오지 않으므로 사용자가 명시적으로 가져와야 합니다.

```rok
// pkg::prelude
pub use pkg::collections::{List, Dict, Set};
pub use pkg::ops::{Add, Sub, Mul};
pub use pkg::iter::Iterator;
```

사용자는 한 줄로 가져옵니다.

```rok
// 사용자 측
use my_pkg::prelude::*;
```

컴파일러는 `prelude`라는 이름에 특별한 의미를 두지 않으며, 일반 모듈로 다룹니다.

## 5.8.10. 순환 재내보내기 (없음)

재내보내기가 서로를 가리키는 순환은 허용하지 않습니다.

```rok
mod a { pub use pkg::b::Item; }
mod b { pub use pkg::a::Item; }
// ERROR: 순환 의존
```

이런 순환은 ERROR이며, 컴파일러가 검출합니다.

## 5.8.11. 구현 측면

재내보내기는 이름 표에 항목 하나를 추가할 뿐입니다. 실제 코드와 메모리는 원본 항목에만 있고, 별도의 wrapper 함수나 복사가 없으므로 비용은 0입니다. 재내보내기 사슬은 길이와 관계없이 원본 항목으로 해소되며, 단계 제한은 없습니다.

## 5.8.12. glob 재내보내기와 가시성

glob 재내보내기가 항목을 노출하는지는 원본 항목의 가시성에 따라 정해집니다.

```rok
mod source {
    pub fn a() { ... }
    pub(pkg) fn b() { ... }
}

pub use source::*;               // a 는 외부에 노출, b 는 pub(pkg) 라 패키지 안에서만 보임
```

`pub use ...::*`는 재내보내기 가시성이 호출자에 닿고, 원본 가시성이 재내보내기 가시성을 허용하는 항목만 가져옵니다. reach가 좁은 항목은 재내보내기로 넓힐 수 없습니다([5.8.4](#584-가시성-단계의-결합)).

## 5.8.13. 정리

`pub use`는 내부 구조와 외부 API를 분리하는 주요 도구입니다. 사용자는 공개 위치만 알면 되고, 라이브러리 작성자는 내부 위치를 자유롭게 재구성할 수 있습니다.
