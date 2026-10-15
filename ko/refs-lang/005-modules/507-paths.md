# 5.7. 경로

`::` 토큰 ([구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)) 으로 이어진 식별자의 연속은 모듈 트리 안의 위치를 가리키는 경로입니다. 이 문서에서는 모듈과 항목 참조 측면의 경로를 정의합니다.

[타입 경로](../002-types/213-type-paths.md) 가 타입 위치의 경로를 다루었기에, 이 문서는 값과 모듈 참조의 경로를 다룹니다. 두 영역의 형태는 동일합니다.

```
path            ::= path_root ("::" path_segment)*
path_root       ::= "pkg" | "sup" | "rok" | identifier
path_segment    ::= identifier
                  | identifier "::" "<" type_arg_list ">"      // turbofish
type_arg_list   ::= type_expr ("," type_expr)* ","?
```

위 문법은 값과 모듈 위치 경로의 turbofish (`name::<T>`) 만을 정의합니다. 타입 위치에서 `name<T>` 형태로 적는 제네릭은 [타입 경로](../002-types/213-type-paths.md) 의 `type_path` 가 정의합니다.

## 5.7.1. 경로의 시작 식별자

경로의 첫 식별자는 다음 중 하나입니다.

| 시작 토큰   | 의미                                      |
|-------------|-------------------------------------------|
| `pkg`       | 현재 패키지의 루트부터                    |
| `sup`       | 부모 모듈부터                             |
| `rok`       | 표준 라이브러리 가상 네임스페이스         |
| 일반 식별자 | 현재 가시 이름 (자식 모듈, `use` 한 이름) |

`pkg` 는 가시성 키워드 `pub(pkg)` 와 같은 어간이고, `sup` 은 가시성 `pub(sub)` 의 **대칭** 방향입니다 (sub 은 자손, sup 은 조상을 뜻합니다). `rok` 은 컴파일러가 관리하는 표준 라이브러리 패키지 묶음을 한 이름으로 묶어 주는 가상 루트입니다. 세 단어 모두 `pub(...)` 한정자와 마찬가지로 컨텍스트 키워드라서 일반 식별자로도 자유롭게 쓸 수 있습니다 ([1.5.4 컨텍스트 키워드](../001-lexical/105-keywords.md)).

`self::` 는 경로의 시작점으로 두지 않습니다. 같은 모듈의 자식이나 항목은 일반 식별자 (`child::item`) 로 충분합니다.

이름 해소는 조상 모듈을 암묵적으로 찾아 올라가지 않습니다 (예외: `extern` 함수, [5.9.4](509-extern-block.md#extern-함수의-이름-공간)). 일반 식별자는 현재 모듈의 항목, 자식 모듈, `use` 로 가져온 이름 (암묵 `use rok::core::**` 포함) 에서만 찾습니다. 부모나 그 위 모듈의 항목은 `sup::` 로 명시하거나 `use` 로 가져와야 합니다.

찾는 순서는 다음과 같으며, 먼저 찾은 쪽이 이깁니다.

1. 현재 모듈의 항목과 자식 모듈
2. 현재 모듈의 `use`. 이름을 적은 import 가 glob (`*`, `**`) 보다 먼저입니다.
3. 암묵 `use rok::core::**`

함수는 제네릭이든 아니든 하나의 값 이름 공간을 씁니다. 그래서 가장 가까운 선언이 종류와 무관하게 이깁니다 (현재 모듈의 비제네릭 `max` 는 core 의 제네릭 `max` 를 가림).

```rok
// pkg::a::b 안에서
helper();              // ERROR — helper 가 pkg::a 에 있어도 암묵 탐색 없음
sup::helper();         // OK
```

## 5.7.2. `pkg::`: 패키지 루트부터

```rok
pkg::geom::Point
pkg::config::DEFAULT_TIMEOUT
pkg::main_module::run
```

`pkg::` 는 패키지의 루트 모듈 (`main.rok` 또는 `lib.rok`) 부터 시작하며, 자기 패키지 어디에서든 같은 형태로 적을 수 있습니다. 큰 코드베이스에서 절대 경로로 안전하게 참조할 수 있는 표준 형태입니다.

## 5.7.3. `sup::`: 부모 모듈부터

```rok
sup::sibling_function
sup::parent_const
```

`sup::` 는 현재 모듈의 부모 모듈부터 시작하기 때문에, 형제와 부모 모듈의 항목을 참조할 때 자연스럽게 어울립니다. 여러 단계를 거슬러 올라가려면 `sup::sup::...` 처럼 연쇄해서 적으시면 됩니다.

```rok
sup::sup::ancestor_item
```

## 5.7.4. `rok::`: 표준 라이브러리 가상 네임스페이스

```rok
rok::core::vec::Vec
rok::core::iter::Iterator
rok::bare::ptr::Ptr
rok::std::fs::File
```

`rok::` 는 컴파일러가 관리하는 표준 라이브러리 패키지 묶음을 가리킵니다. 그 아래 세 개의 tier 가 별도 패키지로 자리 잡습니다 ([0.5 표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md)).

| 경로          | 의미                          | 링크 메커니즘 |
|---------------|-------------------------------|---------------|
| `rok::bare::*` | raw / unsafe 영역             | `-NL bare`     |
| `rok::core::*` | foundation / 자주 쓰는 트레잇 | `-NL core`     |
| `rok::std::*` | OS 통합 / I/O 등              | `-NL std`     |

세 패키지는 빌드 시점에 컴파일러가 자동으로 링크해 주는 별개 archive 이며, 사용자 패키지 의존성과는 다른 `-NL` 메커니즘으로 처리됩니다. 본문 측에서 `rok` 은 이들을 한 path 루트로 묶어 보여 줄 뿐이며, 패키지 경계는 빌드 시스템이 알아서 분리합니다.

`rok::bare` 와 `rok::std` 는 자동 import 의 대상이 아니므로 명시 import 가 필요합니다. 반면 `rok::core` 영역은 [5.5 use](505-use.md) 에서 정의한 묵시 `use rok::core::**;` 를 통해 자동으로 가시화됩니다.

## 5.7.5. 일반 식별자 시작

```rok
foo::bar::baz                   // foo 가 현재 가시 이름이면 OK
geom::Point                      // geom 모듈이 자식이거나 import 되어 있으면
Point                            // 그냥 단일 이름 — 이름 해소가 자동
```

첫 식별자가 현재 모듈에서 가시한 이름이라면 그대로 쓸 수 있습니다. 여기서 가시한 이름이란 자식 모듈이거나 `use` 로 가져온 항목을 의미합니다.

## 5.7.6. 경로의 단계별 의미

```rok
pkg::geom::shapes::Circle
└──┘  └──┘  └────┘   └────┘
시작  모듈  모듈     항목
```

각 단계는 모듈이거나 (마지막의 경우) 항목입니다. 마지막 식별자가 가리키려는 최종 항목이며, 그 앞은 경로상의 모듈 또는 타입에 해당합니다.

## 5.7.7. 항목 종류별 경로

| 가리키는 것                                  | 형태                  |
|----------------------------------------------|-----------------------|
| 함수                                         | `path::function_name` |
| 타입 (`struct` / `enum` / `union` / `trait`) | `path::TypeName`      |
| `const`                                      | `path::CONST_NAME`    |
| `static`                                     | `path::STATIC_NAME`   |
| 모듈 자체                                    | `path::module_name`   |
| 정적 메서드                                  | `Type::method_name`   |
| 연관 `const`                                 | `Type::CONST`         |
| 연관 타입                                    | `T::AssocType`        |

[타입 경로](../002-types/213-type-paths.md) 의 연관 항목 접근 형태와 동일합니다.

## 5.7.8. 표현식 위치와 타입 위치

같은 경로 형태를 표현식 위치 (값을 가리킬 때) 와 타입 위치 (타입을 가리킬 때) 양쪽에서 모두 쓸 수 있습니다.

```rok
// 표현식 위치
let f = pkg::geom::distance;          // 함수 값
let p = pkg::geom::Point::origin();   // 정적 메서드 호출
let v = pkg::config::DEFAULT;           // const 값

// 타입 위치
let p: pkg::geom::Point = ...;
fn make() -> pkg::geom::Point { ... }
```

두 위치의 차이는 제네릭 인자를 표기할 때 turbofish (`::<...>`) 를 쓰느냐 여부입니다. 표현식 위치에서는 turbofish 를 쓰고, 타입 위치에서는 일반 `<...>` 를 씁니다. 자세한 내용은 [타입 경로](../002-types/213-type-paths.md) 에서 다루고 있습니다.

## 5.7.9. 경로의 결합과 글로브

[`use`](505-use.md) 의 import 형태와 결합하면 다양한 import 표기를 쓰실 수 있습니다.

```rok
use pkg::geom::{Point, Circle, distance};
use sup::{a, b as alias};
use foo::*;
```

`*` (glob) 과 `**` (재귀 glob) 도 [`use` (가져오기)](505-use.md) 에서 정의한 대로 동작합니다.

## 5.7.10. 빈 경로 / 단독 `::`

빈 경로 (`::` 만 있고 식별자가 없음) 는 컴파일 에러입니다.

`::A` 형태처럼 선두에 `::` 를 두는 것 역시 ERROR 입니다. 절대 경로는 `pkg::A` 로 명시해 주세요.

## 5.7.11. `use` 와 결합한 단축

`use` 로 가져온 이름은 현재 모듈에서 짧은 이름으로 쓸 수 있습니다.

```rok
use pkg::geom::Point;

fn f() {
    let p = Point { ... };          // 짧은 이름
    let q: Point = ...;
}
```

자주 쓰는 항목을 모듈 상단에 `use` 로 모아두시면 코드 본체가 한층 간결해집니다.

## 5.7.12. 경로의 해석 시점

경로 해석은 컴파일 타임의 이름 해소 단계에서 일어나며, 런타임 동작에는 영향이 없습니다 (다만 동적 슬롯의 메서드 디스패치는 별개로 런타임에 일어나니 [`any` 슬롯](../002-types/202-any-slot.md) 을 참조해 주세요).

## 5.7.13. 모듈을 동적으로 참조 (없음)

```rok
let m = "foo";
m::bar();                            // ERROR — 동적 모듈 참조 안 됨
```

경로의 모듈 부분은 항상 정적이며 컴파일 타임에 결정됩니다. 런타임에 모듈을 변수로 받아 그 안의 함수를 호출하는 형태는 제공되지 않습니다.

이는 모듈 트리가 정적 구조를 갖는다는 [모듈](504-modules.md) 의 원칙과 일관된 결정입니다.

## 5.7.14. 패키지 사이 경로

다른 패키지 (의존 패키지) 의 항목은 그 패키지 이름으로 시작하는 경로로 참조합니다.

```rok
use parser::Token;
let t = parser::lex(src);
```

- 의존 패키지의 이름은 최상위 경로입니다. 이름은 `rok.rokon` 의 `deps` 키이며, 빌드 도구가 `rokc --extern name=..` 로 등록합니다 ([9.2.3](../009-tools/902-aot-pipeline.md#923-의존성-해소), [9.10.3](../009-tools/910-rok-frontend.md#9103-rokrokon--roklock)). Rust 의 `extern crate` 같은 별도 선언은 필요하지 않습니다.
- 의존 패키지 안의 `pkg::` 는 그 의존 패키지 자신의 루트입니다.
- 사용자 패키지의 모듈이 의존 패키지와 같은 이름이면 에러입니다.

### 패키지 경계

다음은 각각 별개의 패키지입니다. 가시성 `pub(pkg)` / `pub(sub)` ([5.6](506-visibility.md)) 은 이 경계를 넘지 않습니다.

- 빌드 중인 패키지
- 의존 패키지 각각
- `rok::` 의 tier 각각 (`rok::bare`, `rok::core`, `rok::std`)

그래서 의존 패키지의 비공개 / `pub(sub)` / `pub(pkg)` 항목은 그 바깥에서 쓸 수 없고, `pub` 항목만 쓸 수 있습니다.
