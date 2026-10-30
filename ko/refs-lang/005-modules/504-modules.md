# 5.4. 모듈

하위 모듈은 `mod` 키워드([키워드](../001-lexical/105-keywords.md))로 선언합니다. 이 문서는 모듈 선언과 파일 시스템 레이아웃 사이의 매핑 규칙을 정의합니다.

```
mod_decl        ::= "mod" identifier ";"
                  | "mod" identifier "{" item* "}"      // inline module
```

## 5.4.1. 모듈 선언

```rok
mod foo;
mod parsing;
```

모듈 선언은 `mod` 키워드, 모듈 이름, `;`로 이루어집니다. 모듈 이름은 [식별자](../001-lexical/104-identifiers.md) 형식을 따릅니다. 선언만 하면 같은 이름의 파일이나 폴더에 자동으로 매핑됩니다.

## 5.4.2. 매핑 형태

`mod foo;`가 나오면 컴파일러는 부모 모듈 본체와 같은 디렉터리에서 다음 두 형태 중 하나를 찾습니다. Rust 2018+와 같은 규칙입니다.

### 형태 1: 단일 파일(`foo.rok`)

서브모듈이 없는 모듈은 파일 하나면 충분합니다.

```
parent/
├── parent.rok                <- where `mod foo;` appears
└── foo.rok                   <- foo module body, no submodules
```

`foo.rok` 파일의 내용이 foo 모듈의 본체입니다.

### 형태 2: 폴더(`foo.rok` + `foo/`)

서브모듈이 있는 모듈에는 본체 파일(`foo.rok`)과 같은 이름의 폴더(`foo/`)가 함께 있어야 합니다. 본체 파일은 모듈의 본체이고, 폴더는 서브모듈을 담습니다.

```
parent/
├── parent.rok                <- where `mod foo;` appears
├── foo.rok                   <- foo module body
└── foo/                      <- foo's submodules
    ├── bar.rok                   declared as `mod bar;` inside foo.rok
    └── baz.rok                   declared as `mod baz;` inside foo.rok
```

여기서 `foo.rok`는 foo 모듈의 본체입니다. 일반 항목, `mod bar;` 같은 서브모듈 선언, 가시성 등을 담습니다. `foo/` 디렉터리의 각 `.rok` 파일은 foo의 서브모듈입니다. 서브모듈은 본체 안에 `mod 이름;`으로 명시적으로 선언해야 합니다. `foo/`만 있고 `foo.rok`가 없으면 에러입니다. 본체 파일은 꼭 있어야 합니다.

이 모델은 Rust 2018+의 `foo.rs` + `foo/` 구조와 같습니다. `mod.rok` 같은 별도의 본체 파일 이름은 없습니다.

### 같은 이름의 파일과 폴더의 의미

같은 이름의 `foo.rok`와 `foo/`가 부모 디렉터리에 함께 있으면 컴파일러는 이를 폴더 형태로 인식합니다. 충돌이 아닙니다.

```
parent/
├── parent.rok
├── foo.rok                   <- foo body
└── foo/                      <- foo's submodules
    └── ...
```

정리하면 `foo.rok`만 있으면 단일 파일 모듈, `foo.rok`와 `foo/`가 함께 있으면 폴더 모듈, `foo/`만 있으면 에러입니다.

## 5.4.3. 서브모듈

폴더 형태(`foo.rok` + `foo/`)의 모듈만 서브모듈을 둘 수 있습니다. 본체(`foo.rok`) 안에 `mod 이름;`을 적으면 `foo/이름.rok` 또는 `foo/이름.rok` + `foo/이름/`이 그 서브모듈로 매핑됩니다.

```rok
// foo.rok 의 내용
mod bar;
mod baz;

pub fn foo_helper() { ... }
```

위 본체와 함께 다음 파일이 있어야 합니다.

```
foo.rok                       <- module body (the code above)
foo/
├── bar.rok                   <- foo::bar submodule
└── baz.rok                   <- foo::baz submodule
```

서브모듈도 단일 파일 형태와 폴더 형태 중 어느 쪽이든 될 수 있습니다.

```
foo.rok
foo/
├── bar.rok                   <- foo::bar (single-file)
├── baz.rok                   <- foo::baz body
└── baz/                      <- foo::baz submodules
    ├── x.rok                     foo::baz::x
    └── y.rok                     foo::baz::y
```

이 패턴은 깊이에 제한 없이 재귀적으로 적용됩니다. 각 모듈은 단일 파일 형태와 폴더 형태 중 하나를 독립적으로 고릅니다.

### 단일 파일 모듈에는 파일 매핑 서브모듈이 없습니다

```rok
// foo.rok (단일 파일 모듈, foo/ 없음)
mod bar;            // ERROR: foo/ 가 없으므로 매핑할 자리가 없다
```

파일 매핑 서브모듈이 필요하면 같은 이름의 폴더(`foo/`)를 만들고 그 안에 서브모듈 파일을 두세요. 단일 파일 모듈을 폴더 모듈로 바꾸려면 폴더를 만들고 서브모듈 파일을 옮기기만 하면 됩니다.

반면 인라인 서브모듈(`mod bar { ... }`)은 파일 매핑이 필요 없으므로 단일 파일 모듈 안에서도 자유롭게 쓸 수 있습니다([5.4.5 인라인 모듈](#545-인라인-모듈-본체-안에-정의)).

## 5.4.4. 패키지 루트

[소스 단위](../000-meta/003-source-units.md)에서 정의한 `src/main.rok` 또는 `src/lib.rok`가 패키지 루트 모듈의 본체 파일입니다. 패키지의 소스 디렉터리 `src/`가 루트 모듈의 폴더입니다.

루트 모듈은 `mod` 선언 없이 자동으로 인식됩니다. 다른 모듈은 모두 루트에서 시작하는 트리의 노드입니다.

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

루트 모듈은 항상 폴더 형태입니다. `src/`가 그 폴더이고 `lib.rok`(라이브러리) 또는 `main.rok`(실행 파일)가 본체 파일입니다. 루트의 `mod foo;`는 `src/foo.rok`에 대응합니다. `src/` 옆의 `foo.rok`는 결코 모듈이 아닙니다. `#[path(..)]`(5.4.6)는 하위 모듈 파일이 원래 놓일 폴더를 기준으로 해소합니다. 루트의 `mod`는 `src/`, `parsing.rok` 안의 `mod`는 `src/parsing/`이 기준입니다.

## 5.4.5. 인라인 모듈: 본체 안에 정의

`mod` 선언 바로 뒤에 블록 본체를 두면 별도 파일 없이 인라인 모듈을 정의합니다.

```rok
mod inner {
    pub fn helper() -> i32 { 42i32 }
    const X: i32 = 100i32;
}

// 사용
let r = inner::helper();
```

인라인 모듈은 `mod 이름 { ... }` 형태로 씁니다. 본체에는 함수, 타입, 트레잇, `const`, `static`, 중첩 `mod` 등 어떤 항목이든 둘 수 있습니다. 같은 이름의 파일이나 폴더가 함께 있으면 안 됩니다. 충돌하면 에러입니다.

## 5.4.6. 사용자 정의 경로: `#[path("...")]`

```
path_attr       ::= "#" "[" "path" "(" string_lit ")" "]"
```

`#[path("...")]` 속성은 기본 매핑을 재정의합니다.

```rok
#[path("custom_geom.rok")]
mod geom;

#[path("../shared/shared.rok")]
mod shared;
```

주어진 경로의 파일이 모듈 본체로 매핑됩니다. 경로는 보통 `name.rok`를 찾는 디렉터리를 기준으로 합니다. 그 모듈의 하위 모듈은 파일 옆에 있는 폴더에 둡니다. 폴더 이름은 파일 이름에서 확장자를 뺀 부분(stem)입니다. `#[path("f.rok")]`이면 `f/`입니다. 대상 파일 이름은 `mod.rok`일 수 없습니다. Rok에는 `mod.rok` 본체 형식이 없으므로([5.4.4](#544-패키지-루트)) 에러입니다. 형태는 `#[path(..)]` 하나뿐입니다. 일반 코드에서는 거의 쓰지 않습니다. 빌드 시스템이나 자동 생성 코드 같은 특수한 경우에 씁니다.

## 5.4.7. 모듈 안의 항목

모듈은 [개요](README.md)에서 소개한 항목을 담을 수 있습니다.

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

모듈에는 항목만 둘 수 있습니다. 실행되는 문장(최상위 `let`, 식 문장)은 진입 모듈 `main.rok`에만 쓸 수 있습니다([4.16.6](../004-functions/416-entry-point.md#4166-모듈은-정의-전용)). 이 항목은 모두 그 모듈의 일부입니다. 외부에서 참조하려면 [가시성](506-visibility.md) 키워드와 [경로](507-paths.md) 표기가 모두 필요합니다. 함수 본체에 선언한 항목([3.9.1](../003-expressions/309-block-expressions.md#391-형태))은 coherence 검사에서는 그 모듈에 속합니다. 하지만 이름은 자기 블록 안에서만 부를 수 있습니다.

## 5.4.8. 모듈의 가시성

모듈 자체에도 가시성을 붙일 수 있습니다.

```rok
pub mod public_api;            // 외부 패키지에서 사용 가능
pub(pkg) mod internal;          // 같은 패키지 내부만
mod _private;                   // 모듈 비공개: 같은 부모 모듈에서만
```

자세한 의미는 [가시성](506-visibility.md)에서 정의합니다.

## 5.4.9. 모듈 안에서 부모 / 형제 / 자식 참조

다른 모듈의 항목을 참조하는 경로 표기는 [경로](507-paths.md)에서 정의합니다.

```rok
pkg::geom::Point             // 패키지 루트부터
sup::sibling_mod::item        // 부모 모듈 거쳐 형제로
child_mod::item                // 같은 모듈 안 (식별자로 충분)
```

## 5.4.10. 모듈 선언의 위치

`mod` 선언은 모듈 수준, 곧 다른 `mod`의 본체나 패키지 루트에만 올 수 있습니다. 함수 본체 안에서 `mod`를 선언하면 에러입니다. 블록이 선언할 수 있는 항목은 [3.9.1](../003-expressions/309-block-expressions.md#391-형태)에 있습니다.

```rok
fn f() {
    mod inner { ... }          // ERROR: 함수 안에서 모듈 선언 안 됨
}
```

이 제약은 모듈 트리가 컴파일 타임에 정해지는 정적 구조라는 원칙에서 나옵니다.

## 5.4.11. 같은 모듈을 여러 파일로 쪼개기

본체를 여러 파일로 나누려면 폴더 형태를 쓰고, 본체(`foo.rok`)에서 서브모듈의 항목을 재내보내세요.

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

이렇게 하면 사용하는 쪽은 모든 항목에 `foo::item` 형태로 접근합니다. 본체 파일은 작게 유지되고 실제 코드는 서브모듈에 나뉩니다. Rust의 `pub use` 재내보내기 패턴과 같습니다.

## 5.4.12. 동일 부모 안 중복 모듈 금지

같은 부모 모듈 안에서 같은 이름의 모듈을 두 번 선언하면 에러입니다.

```rok
mod foo;
mod foo;                        // ERROR: 중복
```
