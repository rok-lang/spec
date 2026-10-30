# 5. 변수 / 모듈 / 가시성 / 패키지 개요

이 블록은 Rok의 명명 공간 (name space) 을 다룹니다. 변수가 어디서 보이는지, 모듈이 어떻게 묶이는지, 가시성 단계가 어떻게 작동하는지, 그리고 패키지 사이의 경계가 어디에 놓이는지를 정의하는 것이 이 블록의 목표입니다.

## 목차

- [5.1. 변수 선언](501-variable-declaration.md)
- [5.2. `const`](502-const.md)
- [5.3. `static`](503-static.md)
- [5.4. 모듈](504-modules.md)
- [5.5. `use` (가져오기)](505-use.md)
- [5.6. 가시성](506-visibility.md)
- [5.7. 경로](507-paths.md)
- [5.8. 재내보내기 (`pub use`)](508-re-export.md)
- [5.9. `extern` 블록 (FFI 선언)](509-extern-block.md)

## 명명 공간의 네 단계

명명 공간은 다음과 같이 네 단계로 구성됩니다.

```
package
  └── module
        └── item
              └── local variable
```

위에서 아래로 내려가는 포함 관계입니다. 하나의 패키지는 여러 개의 모듈을 가지며, 하나의 모듈은 여러 개의 항목 (함수, 타입, 트레잇, `const`, `static`) 을 가지고, 하나의 항목 (특히 함수 본체) 은 여러 개의 지역 변수를 가지게 됩니다.

각 단계의 경계 규칙은 이어지는 문서들에서 차례로 정의됩니다.

## 패키지: 가장 큰 단위

[소스 단위](../000-meta/003-source-units.md) 에서 정의한 패키지가 명명 공간의 가장 바깥쪽 단계입니다. 패키지마다 내용 기반 해시가 식별자로 부여됩니다.

같은 패키지 안의 모든 항목은 동일한 코드 권한 영역에 속합니다. 이러한 점이 [`impl`](../002-types/209-impl.md) 의 orphan 규칙이 패키지 단위로 적용되는 근거가 됩니다.

## 모듈: 패키지 안의 묶음 단위

하나의 패키지는 여러 모듈을 가집니다. 각 모듈은 단일 파일 (`foo.rok`) 의 형태를 취하거나, 본체 파일과 같은 이름의 폴더 (`foo.rok` + `foo/`) 의 형태를 취합니다. 이는 Rust 2018+ 와 동일한 모델입니다. 자세한 규칙은 [모듈](504-modules.md) 에서 정의됩니다.

```
my_pkg/
└── src/
    ├── lib.rok                  <- root module body
    ├── geom.rok                 <- geom module    (single-file: geom.rok)
    ├── parsing.rok              <- parsing module body
    └── parsing/                 <- parsing's submodules
        ├── lexer.rok                parsing::lexer
        └── parser.rok               parsing::parser
```

## 항목: 모듈 안의 명명된 정의

각 모듈은 항목을 가집니다. 함수, 타입, 트레잇, `impl` 블록, `const`, `static`, 그리고 다른 모듈 선언이 모두 여기에 해당합니다.

[함수 선언](../004-functions/401-function-declaration.md), [`struct`](../002-types/203-struct.md), [`enum`](../002-types/204-enum.md), [`trait`](../002-types/208-trait.md), [`impl`](../002-types/209-impl.md) 의 모든 정의가 모듈 안의 항목으로 자리 잡게 됩니다.

## 지역 변수: 함수 / 블록 안의 변수

함수, 블록, 그리고 다른 식 안에서 지역 변수는 `let` 키워드 ([키워드](../001-lexical/105-keywords.md)) 를 통해 선언됩니다.

```rok
fn area(r: f64) -> f64 {
    let pi = 3.14159f64;          // 지역 변수
    pi * r * r
}
```

지역 변수의 가시 범위는 [블록 표현식](../003-expressions/309-block-expressions.md) 의 블록 단위로 결정됩니다. 자세한 규칙은 [변수 선언](501-variable-declaration.md) 에서 정의됩니다.

## 가시성: 항목의 외부 노출

각 항목에는 가시성이 부여됩니다. 가시성 단계 ([가시성](506-visibility.md)) 가 해당 항목이 어디까지 보이는지를 결정합니다.

```
private (기본)        같은 모듈 안에서만
pub(sub)             하위 모듈에서만
pub(pkg)             같은 패키지 안에서만
pub                  외부 패키지에서도 가능
```

## 이름 해소 규칙

코드 안의 한 이름이 어떤 항목을 가리키는지 결정하는 과정을 이름 해소 (name resolution) 라고 합니다.

이 과정에서는 다음과 같은 정보들이 종합적으로 고려됩니다. 첫째, 현재 모듈에서 지역적으로 보이는 항목들이 후보가 됩니다. 둘째, `use` 로 가져온 외부 항목 ([`use`](505-use.md)) 또한 후보에 포함됩니다. 셋째, 자동으로 import 되는 `rok::**` ([표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md)) 도 함께 참조됩니다. 마지막으로 가시성 검사가 수행되어, 볼 수 있는 항목만이 최종 후보로 남게 됩니다.

이름 해소는 컴파일 타임에 일어나는 작업입니다. 동적 슬롯의 메서드 디스패치는 런타임에 이루어지지만, 이름 자체의 해소는 어디까지나 컴파일 타임에 완결됩니다.
