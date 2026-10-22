# 5.4. 모듈

`mod` 키워드 ([키워드](../001-lexical/105-keywords.md)) 로 하위 모듈을 선언하며, 이 문서에서는 모듈 선언과 파일 시스템 레이아웃 사이의 매핑 규칙을 정의합니다.

```
mod_decl        ::= "mod" identifier ";"
                  | "mod" identifier "{" item* "}"      // inline module
```

## 5.4.1. 모듈 선언

```rok
mod foo;
mod parsing;
```

모듈 선언은 `mod` 키워드 뒤에 모듈 이름을 적고 `;` 로 마무리하는 형태로 이루어집니다. 모듈 이름은 [식별자](../001-lexical/104-identifiers.md) 형식을 따르며, 선언만으로 같은 이름의 파일이나 폴더에 자동으로 매핑됩니다.

## 5.4.2. 매핑 형태

`mod foo;` 가 등장하면 컴파일러는 부모 모듈 본체와 같은 디렉토리에서 다음 두 가지 중 하나를 찾습니다. 이는 Rust 2018+ 와 같은 규칙입니다.

### 형태 1: 단일 파일 (`foo.rok`)

서브모듈이 없는 모듈은 단일 파일만으로도 충분합니다.

```
parent/
├── parent.rok                <- where `mod foo;` appears
└── foo.rok                   <- foo module body, no submodules
```

이 경우 `foo.rok` 한 파일의 내용이 곧 foo 모듈의 본체가 됩니다.

### 형태 2: 폴더 (`foo.rok` + `foo/`)

서브모듈을 가지는 모듈에서는 본체 파일 (`foo.rok`) 과 같은 이름의 폴더 (`foo/`) 가 함께 존재해야 합니다. 본체 파일은 모듈의 본체 역할을 하고, 같은 이름의 폴더는 서브모듈들을 담는 자리입니다.

```
parent/
├── parent.rok                <- where `mod foo;` appears
├── foo.rok                   <- foo module body
└── foo/                      <- foo's submodules
    ├── bar.rok                   declared as `mod bar;` inside foo.rok
    └── baz.rok                   declared as `mod baz;` inside foo.rok
```

여기서 `foo.rok` 는 foo 모듈의 본체이며, 일반 항목과 `mod bar;` 같은 서브모듈 선언, 그리고 가시성 등을 포함합니다. `foo/` 디렉토리에 들어 있는 각 `.rok` 파일은 foo 의 서브모듈이 되며, 본체 안에 `mod 이름;` 으로 명시 선언되어 있어야 합니다. 만약 `foo/` 만 있고 `foo.rok` 가 없다면 ERROR 입니다. 본체 파일은 반드시 있어야 합니다.

이 모델은 Rust 2018+ 의 `foo.rs` + `foo/` 구조와 동일하며, `mod.rok` 같은 별도의 본체 파일 이름은 두지 않습니다.

### 같은 이름의 파일과 폴더의 의미

같은 이름의 `foo.rok` 와 `foo/` 가 부모 디렉토리에 함께 놓여 있으면 자동으로 폴더 형태로 인식되며, 충돌로 처리되지 않습니다.

```
parent/
├── parent.rok
├── foo.rok                   <- foo body
└── foo/                      <- foo's submodules
    └── ...
```

정리하면, `foo.rok` 만 있으면 단일 파일 모듈이고, `foo.rok` 와 `foo/` 가 함께 있으면 폴더 모듈이며, `foo/` 만 있으면 ERROR 입니다.

## 5.4.3. 서브모듈

폴더 형태 (`foo.rok` + `foo/`) 의 모듈만 서브모듈을 가질 수 있습니다. 본체 (`foo.rok`) 안에 `mod 이름;` 을 적으면 `foo/이름.rok` 또는 `foo/이름.rok` + `foo/이름/` 이 그 서브모듈로 매핑됩니다.

```rok
// foo.rok 의 내용
mod bar;
mod baz;

pub fn foo_helper() { ... }
```

위 본체와 같은 디렉토리에는 다음과 같은 파일들이 있어야 합니다.

```
foo.rok                       <- module body (the code above)
foo/
├── bar.rok                   <- foo::bar submodule
└── baz.rok                   <- foo::baz submodule
```

서브모듈 자체도 단일 파일 또는 폴더 형태 중 어느 쪽이든 자유롭게 선택할 수 있습니다.

```
foo.rok
foo/
├── bar.rok                   <- foo::bar (single-file)
├── baz.rok                   <- foo::baz body
└── baz/                      <- foo::baz submodules
    ├── x.rok                     foo::baz::x
    └── y.rok                     foo::baz::y
```

이 패턴은 임의의 깊이로 재귀적으로 적용되며, 각 모듈은 독립적으로 단일 파일 형태와 폴더 형태 중 하나를 선택할 수 있습니다.

### 단일 파일 모듈에는 **파일 매핑** 서브모듈이 없습니다

```rok
// foo.rok (단일 파일 모듈, foo/ 없음)
mod bar;            // ERROR — foo/ 가 없으므로 매핑할 자리가 없다
```

파일 매핑 서브모듈이 필요해진다면 같은 이름의 폴더 (`foo/`) 를 만들고 그 안에 서브모듈 파일을 두면 됩니다. 단일 파일 모듈을 폴더 모듈로 승격하는 비용은 폴더를 만들고 서브모듈 파일을 옮기는 정도뿐입니다.

한편 인라인 서브모듈 (`mod bar { ... }`) 은 단일 파일 모듈 안에서도 자유롭게 사용할 수 있는데, 별도의 파일 매핑이 필요하지 않기 때문입니다 ([5.4.5 인라인 모듈](#545-인라인-모듈-본체-안에-정의)).

## 5.4.4. 패키지 루트

[소스 단위](../000-meta/003-source-units.md) 에서 정의한 `src/main.rok` 또는 `src/lib.rok` 이 패키지 루트 모듈의 본체 파일이며, 패키지의 소스 디렉토리 `src/` 가 루트 모듈의 폴더 역할을 맡습니다.

루트 모듈은 별도의 `mod` 선언 없이 자동으로 인식되고, 다른 모든 모듈은 루트에서 시작하는 트리의 노드가 됩니다.

```
my_pkg/
├── rok.rokon
└── src/
    ├── lib.rok                   <- root module body (declares `mod geom; mod parsing;`)
    ├── geom.rok                  <- geom module    (single-file: geom.rok)
    ├── parsing.rok               <- parsing module body
    └── parsing/                  <- parsing's submodules (declared inside parsing.rok)
        ├── lexer.rok                 parsing::lexer
        └── parser.rok                parsing::parser
```

루트 모듈은 항상 폴더 형태이며, `src/` 가 그 폴더가 되고 `lib.rok` (라이브러리) 또는 `main.rok` (실행) 이 본체 파일이 됩니다. 루트의 `mod foo;` 는 `src/foo.rok` 에 대응하며, `src/` 옆의 `foo.rok` 는 결코 모듈이 아닙니다. `#[path(..)]` (5.4.6) 는 여전히 선언한 파일 기준입니다.

## 5.4.5. 인라인 모듈: 본체 안에 정의

`mod` 선언 바로 뒤에 블록 본체를 두면 별도의 파일 없이 인라인 모듈을 정의할 수 있습니다.

```rok
mod inner {
    pub fn helper() -> i32 { 42i32 }
    const X: i32 = 100i32;
}

// 사용
let r = inner::helper();
```

인라인 모듈은 `mod 이름 { ... }` 형태로 작성하며, 본체에는 함수, 타입, 트레잇, `const`, `static`, 그리고 다시 `mod` 등 임의의 항목을 둘 수 있습니다. 다만 같은 이름의 파일이나 폴더가 함께 존재해서는 안 되며, 충돌이 발생하면 ERROR 입니다.

## 5.4.6. 사용자 정의 경로: `#[path("...")]`

```
path_attr       ::= "#" "[" "path" "(" string_lit ")" "]"
```

기본 매핑을 재정의하고 싶다면 `#[path("...")]` 속성을 사용하면 됩니다.

```rok
#[path("custom_geom.rok")]
mod geom;

#[path("../shared/shared.rok")]
mod shared;
```

이 경우 주어진 경로의 파일이 모듈 본체로 매핑됩니다. 경로는 `name.rok` 를 보통 찾을 디렉토리 기준입니다. 그 모듈의 하위 모듈은 파일 옆의, 파일 이름 stem 의 폴더에 둡니다: `#[path("f.rok")]` 이면 `f/` 입니다. `#[path(..)]` 형태만 있습니다. 일반적인 사용에서는 거의 등장하지 않으며, 빌드 시스템이나 자동 생성 코드 같은 특수한 사례에서 활용됩니다.

## 5.4.7. 모듈 안의 항목

각 모듈은 [개요](README.md) 에서 소개한 항목들을 포함할 수 있습니다.

```rok
mod geom {
    struct Point { ... }
    fn distance(a: Point, b: Point) -> f64 { ... }
    const PI: f64 = 3.14159f64;
    static CACHE: Mutex<Dict<str, f64>> = ...;
    impl Point { ... }
    trait Shape { ... }
    mod inner { ... }            // 다시 중첩 모듈
}
```

모듈에는 항목만 둘 수 있으며, 실행되는 문 (top-level `let`, 식 문장) 은 진입 모듈 `main.rok` 에서만 허용됩니다 ([4.16.6](../004-functions/416-entry-point.md#4166-모듈은-정의-전용)). 이러한 모든 항목은 해당 모듈의 일부가 되며, 외부에서 참조하려면 [가시성](506-visibility.md) 의 키워드와 [경로](507-paths.md) 의 경로 표기가 함께 필요합니다. 함수 본체에 선언한 항목 ([3.9.1](../003-expressions/309-block-expressions.md#391-형태)) 은 coherence 에서는 그 모듈에 속하지만, 이름은 자기 블록 안에서만 부를 수 있습니다.

## 5.4.8. 모듈의 가시성

모듈 자체에도 가시성을 부여할 수 있습니다.

```rok
pub mod public_api;            // 외부 패키지에서 사용 가능
pub(pkg) mod internal;          // 같은 패키지 내부만
mod _private;                   // 모듈 비공개 — 같은 부모 모듈에서만
```

자세한 의미는 [가시성](506-visibility.md) 에서 정의합니다.

## 5.4.9. 모듈 안에서 부모 / 형제 / 자식 참조

다른 모듈의 항목을 참조할 때 쓰이는 경로 표기는 [경로](507-paths.md) 에서 정의합니다.

```rok
pkg::geom::Point             // 패키지 루트부터
sup::sibling_mod::item        // 부모 모듈 거쳐 형제로
child_mod::item                // 같은 모듈 안 (식별자로 충분)
```

## 5.4.10. 모듈 선언의 위치

`mod` 선언은 모듈 수준, 즉 또 다른 `mod` 의 본체나 패키지 루트에서만 가능합니다. 함수 본체 안에서 `mod` 를 선언하면 ERROR 입니다 (블록이 선언할 수 있는 항목은 [3.9.1](../003-expressions/309-block-expressions.md#391-형태) 에 있음).

```rok
fn f() {
    mod inner { ... }          // ERROR — 함수 안에서 모듈 선언 안 됨
}
```

이는 모듈 트리가 컴파일 타임에 결정되는 정적 구조라는 원칙에서 비롯된 제약입니다.

## 5.4.11. 같은 모듈을 여러 파일로 쪼개기

본체를 여러 파일로 쪼개고자 한다면 폴더 형태를 사용하고, 본체 (`foo.rok`) 에서 서브모듈을 통해 항목을 재내보내면 됩니다.

```rok
// foo.rok
mod core;
mod ext;

pub use core::*;
pub use ext::*;
```

```
foo.rok                       <- module body (re-exports only)
foo/
├── core.rok                  <- core items
└── ext.rok                   <- extension items
```

이 패턴을 사용하면 사용자는 `foo::item` 형태로 모든 항목에 접근할 수 있고, 본체 파일은 작게 유지되며 실제 코드는 서브모듈에 분산됩니다. Rust 의 `pub use` 재내보내기 패턴과 같은 방식입니다.

## 5.4.12. 동일 부모 안 중복 모듈 금지

같은 부모 모듈 안에서 같은 이름의 모듈을 두 번 선언하면 ERROR 입니다.

```rok
mod foo;
mod foo;                        // ERROR — 중복
```
