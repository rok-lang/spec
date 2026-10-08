# 5.7. 경로

`::` 토큰 ([구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)) 으로 이어진 식별자의 연속은 모듈 트리 안의 위치를 가리키는 경로다. 이 문서는 모듈 / 항목 참조 측면의 경로를 정의한다.

[타입 경로](../002-types/213-type-paths.md) 가 타입 위치의 경로를 다뤘고, 이 문서는 값과 모듈 참조의 경로를 다룬다. 두 영역의 형태는 동일하다.

```
path            ::= path_root ("::" path_segment)*
path_root       ::= "crate" | "super" | "self" | identifier
path_segment    ::= identifier
                  | identifier "::" "<" type_arg_list ">"      // turbofish
type_arg_list   ::= type_expr ("," type_expr)* ","?
```

## 5.7.1. 경로의 시작 식별자

경로의 첫 식별자는 다음 중 하나다.

| 시작 토큰   | 의미                      |
| ----------- | ------------------------- |
| `crate`     | 현재 패키지의 루트부터    |
| `super`     | 부모 모듈부터             |
| `self`      | 현재 모듈부터 (보통 생략) |
| 일반 식별자 | 현재 가시 이름            |

## 5.7.2. `crate::`: 패키지 루트부터

```nok
crate::geom::Point
crate::config::DEFAULT_TIMEOUT
crate::main_module::run
```

- 패키지의 루트 모듈 (`main.nok` 또는 `lib.nok`) 부터 시작한다.
- 자기 패키지 어디서든 같은 형태로 적을 수 있다.
- 큰 코드베이스에서 절대 경로로 안전하게 참조하는 표준 형태다.

## 5.7.3. `super::`: 부모 모듈부터

```nok
super::sibling_function
super::parent_const
```

- 현재 모듈의 부모 모듈부터 시작한다.
- 형제와 부모 모듈의 항목을 참조할 때 자연스럽다.
- 여러 단계 거슬러 올라가려면 `super::super::...` 를 연쇄한다.

```nok
super::super::ancestor_item
```

## 5.7.4. `self::`: 현재 모듈

```nok
self::child_module::item
self::local_item
```

- 현재 모듈부터 시작한다.
- 자식 모듈이나 같은 모듈의 항목을 명시적으로 가리킬 때 쓴다.
- `self::` 는 보통 생략할 수 있다. 일반 식별자 (`child_module::item`) 로 충분하다.

명시가 필요한 경우는 같은 이름의 외부 import 와 충돌을 피해야 하는 등의 드문 사례다.

## 5.7.5. 일반 식별자 시작

```nok
foo::bar::baz                   // foo 가 현재 가시 이름이면 OK
geom::Point                      // geom 모듈이 자식이거나 import 되어 있으면
Point                            // 그냥 단일 이름 — 이름 해소가 자동
```

- 첫 식별자가 현재 모듈에서 가시한 이름이면 OK.
- 가시한 이름이란 자식 모듈, `use` 한 항목, 자동 prelude 등이다.

## 5.7.6. 경로의 단계별 의미

```nok
crate::geom::shapes::Circle
└──┘  └──┘  └────┘   └────┘
시작  모듈  모듈     항목
```

각 단계는 모듈이거나 항목 (마지막) 이다. 마지막 식별자가 가리키려는 최종 항목이고, 그 앞은 경로상의 모듈 또는 타입이다.

## 5.7.7. 항목 종류별 경로

| 가리키는 것                                  | 형태                  |
| -------------------------------------------- | --------------------- |
| 함수                                         | `path::function_name` |
| 타입 (`struct` / `enum` / `union` / `trait`) | `path::TypeName`      |
| `const`                                      | `path::CONST_NAME`    |
| `static`                                     | `path::STATIC_NAME`   |
| 모듈 자체                                    | `path::module_name`   |
| 정적 메서드                                  | `Type::method_name`   |
| 연관 `const`                                 | `Type::CONST`         |
| 연관 타입                                    | `T::AssocType`        |

[타입 경로](../002-types/213-type-paths.md) 의 연관 항목 접근 형태와 동일하다.

## 5.7.8. 표현식 위치와 타입 위치

같은 경로 형태를 표현식 위치 (값을 가리킬 때) 와 타입 위치 (타입을 가리킬 때) 모두에서 쓸 수 있다.

```nok
// 표현식 위치
let f = crate::geom::distance;          // 함수 값
let p = crate::geom::Point::origin();   // 정적 메서드 호출
let v = crate::config::DEFAULT;         // const 값

// 타입 위치
let p: crate::geom::Point = ...;
fn make(): crate::geom::Point { ... }
```

두 위치의 차이는 제네릭 인자 표기 시 turbofish (`::<...>`) 의 사용 여부다. 표현식 위치는 turbofish 를 쓰고, 타입 위치는 일반 `<...>` 을 쓴다. 자세한 내용은 [타입 경로](../002-types/213-type-paths.md) 에서 다룬다.

## 5.7.9. 경로의 결합과 글로브

[`use`](505-use.md) 의 import 형태와 결합하면 다양한 import 표기를 쓸 수 있다.

```nok
use crate::geom::{Point, Circle, distance};
use super::{a, b as alias};
use foo::*;
```

`*` (glob) 과 `**` (재귀 glob) 도 [`use` (가져오기)](505-use.md) 에서 정의한 대로 동작한다.

## 5.7.10. 빈 경로 / 단독 `::`

빈 경로 (`::` 만 있고 식별자가 없음) 는 컴파일 에러다.

`::A` 형태 (선두 `::`) 도 ERROR. 절대 경로는 `crate::A` 로 명시한다.

## 5.7.11. `use` 와 결합한 단축

`use` 로 가져온 이름은 현재 모듈에서 짧은 이름으로 쓸 수 있다.

```nok
use crate::geom::Point;

fn f() {
    let p = Point { ... };          // 짧은 이름
    let q: Point = ...;
}
```

자주 쓰는 항목을 모듈 상단에 `use` 로 모아두면 코드 본체가 간결해진다.

## 5.7.12. 경로의 해석 시점

경로 해석은 컴파일 타임의 이름 해소 단계에서 일어난다. 런타임 동작에는 영향이 없다 (단, 동적 슬롯의 메서드 디스패치는 별개로 런타임. [`any` 슬롯](../002-types/202-any-slot.md) 참조).

## 5.7.13. 모듈을 동적으로 참조 (없음)

```nok
let m = "foo";
m::bar();                            // ERROR — 동적 모듈 참조 안 됨
```

경로의 모듈 부분은 항상 정적이며 컴파일 타임에 결정된다. 런타임에 모듈을 변수로 받아 그 안의 함수를 호출하는 형태는 없다.

모듈 트리가 정적 구조라는 [모듈](504-modules.md) 의 원칙과 일관된 결정이다.

## 5.7.14. 패키지 사이 경로

다른 패키지의 항목은 그 패키지 이름으로 시작하는 경로로 참조한다.

```nok
external_pkg::module::item
```

`external_pkg` 가 현재 가시한 이름이 되려면 패키지 매니저나 빌드 도구가 그 이름을 등록해야 한다. 정확한 메커니즘은 본 사양의 범위 밖이며, 패키지 매니저 사양에서 정의한다.

본 사양에서는 경로 형태가 같다는 점만 보장한다. 패키지 경계가 경로 표기에 특별히 드러나지 않는다 (Rust 의 `extern crate` 같은 별도 선언이 필요 없다).
