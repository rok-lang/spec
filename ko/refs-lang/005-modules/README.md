# 5. 변수 / 모듈 / 가시성 / 패키지 개요

이 장은 Rok의 명명 공간(name space)을 다룹니다. 변수가 어디서 보이는지, 모듈을 어떻게 묶는지, 가시성 단계가 어떻게 작동하는지, 패키지 사이의 경계가 어디에 있는지를 정의합니다.

## 목차

- [5.1. 변수 선언](501-variable-declaration.md)
- [5.2. `const`](502-const.md)
- [5.3. `static`](503-static.md)
- [5.4. 모듈](504-modules.md)
- [5.5. `use`(가져오기)](505-use.md)
- [5.6. 가시성](506-visibility.md)
- [5.7. 경로](507-paths.md)
- [5.8. 재내보내기(`pub use`)](508-re-export.md)
- [5.9. `extern` 블록 (FFI 선언)](509-extern-block.md)

## 명명 공간의 네 단계

명명 공간은 네 단계로 이루어집니다.

```
package
  └── module
        └── item
              └── local variable
```

위 단계가 아래 단계를 포함합니다. 패키지에는 모듈이 여럿 있습니다. 모듈에는 항목(함수, 타입, 트레잇, `const`, `static`)이 여럿 있고, 항목(특히 함수 본체)에는 지역 변수가 여럿 있습니다.

각 단계의 경계 규칙은 이어지는 문서에서 차례로 정의합니다.

## 패키지: 가장 큰 단위

[소스 단위](../000-meta/003-source-units.md)에서 정의한 패키지가 명명 공간의 가장 바깥 단계입니다. 패키지는 내용 기반 해시로 식별합니다.

같은 패키지 안의 모든 항목은 같은 코드 권한 영역에 속합니다. [`impl`](../002-types/209-impl.md)의 orphan 규칙을 패키지 단위로 적용하는 이유가 여기에 있습니다.

## 모듈: 패키지 안의 묶음 단위

패키지에는 모듈이 여럿 있습니다. 모듈은 단일 파일(`foo.rok`)이거나, 본체 파일과 같은 이름의 폴더(`foo.rok` + `foo/`)입니다. Rust 2018+와 같은 모델입니다. 자세한 규칙은 [모듈](504-modules.md)에서 정의합니다.

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

모듈에는 항목이 있습니다. 함수, 타입, 트레잇, `impl` 블록, `const`, `static`, 다른 모듈 선언이 모두 항목입니다.

[함수 선언](../004-functions/401-function-declaration.md), [`struct`](../002-types/203-struct.md), [`enum`](../002-types/204-enum.md), [`trait`](../002-types/208-trait.md), [`impl`](../002-types/209-impl.md)의 모든 정의는 모듈 안의 항목입니다.

## 지역 변수: 함수 / 블록 안의 변수

함수, 블록, 그 밖의 식 안에서는 `let` 키워드([키워드](../001-lexical/105-keywords.md))로 지역 변수를 선언합니다.

```rok
fn area(r: f64) -> f64 {
    let pi = 3.14159f64;          // 지역 변수
    pi * r * r
}
```

지역 변수가 보이는 범위는 [블록 표현식](../003-expressions/309-block-expressions.md)의 블록 단위로 정해집니다. 자세한 규칙은 [변수 선언](501-variable-declaration.md)에서 정의합니다.

## 가시성: 항목의 외부 노출

항목마다 가시성이 있습니다. 가시성 단계([가시성](506-visibility.md))가 그 항목이 어디까지 보이는지를 정합니다.

```
private (기본)        같은 모듈 안에서만
pub(sub)             하위 모듈에서만
pub(pkg)             같은 패키지 안에서만
pub                  외부 패키지에서도 가능
```

## 이름 해소 규칙

코드 안의 이름이 어떤 항목을 가리키는지 정하는 과정을 이름 해소(name resolution)라고 합니다.

이름 해소는 다음을 고려합니다. 첫째, 현재 모듈에서 지역적으로 보이는 항목이 후보입니다. 둘째, `use`로 가져온 외부 항목([`use`](505-use.md))도 후보입니다. 셋째, 자동으로 가져오는 `rok::**`([표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md))도 참조합니다. 마지막으로 가시성을 검사해 볼 수 있는 항목만 후보로 남깁니다.

이름 해소는 컴파일 타임에 일어납니다. 동적 슬롯의 메서드 디스패치는 런타임에 일어납니다. 하지만 이름 자체는 언제나 컴파일 타임에 해소됩니다.
