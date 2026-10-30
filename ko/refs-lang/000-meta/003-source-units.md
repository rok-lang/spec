# 0.3. 소스 단위

Rok의 컴파일 단위는 **패키지**입니다. 패키지는 파일 시스템의 정해진 규약(`src/main.rok` / `src/lib.rok`)으로 인식됩니다. 매니페스트 `rok.rokon`은 의존성 등을 선언할 때만 두며, 의존성이 없는 패키지에는 필요하지 않습니다 (0.3.4).

## 0.3.1. 실행 파일과 라이브러리

패키지는 실행 파일, 라이브러리, 또는 둘 다를 가집니다. 무엇을 가지는지는 소스 디렉토리 `src/`의 루트 파일만으로 정해집니다. `src/main.rok`은 실행 파일의 루트이고, `src/lib.rok`은 라이브러리의 루트입니다.

패키지 디렉토리는 패키지 소스를 `src/`에 두고, 나머지는 모두 그 옆에 둡니다. 매니페스트 `rok.rokon`과 lockfile `rok.lock` (0.3.4), `target/` (빌드 산출물, [9.2.15](../009-tools/902-aot-pipeline.md#9215-산출물)), `README`, `LICENSE`, 테스트 데이터 같은 파일이 여기에 해당합니다. `src/` 아래의 파일만 패키지 소스이며, `src/` 밖의 `.rok` 파일은 패키지에 속하지 않습니다(함께 컴파일되지 않고, 식별자(0.3.3)에 들어가지 않으며, 따로 실행하면 익명 패키지가 됩니다, [4.16.8](../004-functions/416-entry-point.md#4168-단일-파일-실행)). 유일한 예외는 `tests/`와 `benches/` 디렉토리이며, 이 둘은 `rokc test` / `rokc bench`에서만 컴파일됩니다 ([0.3.5](#035-테스트와-벤치마크-디렉토리)).

### 실행 패키지: `src/main.rok`

`src/main.rok`이 있는 디렉토리는 실행 패키지로 인식됩니다.

```
my_app/
├── rok.rokon         <- manifest (optional, 0.3.4)
├── src/
│   ├── main.rok      <- entry point
│   └── ...           <- other module files
└── target/           <- build output
```

`main.rok`은 진입 모듈이며, 명시적인 `fn main`이나 top-level 문 중 한 형태로 진입점을 둡니다. 진입점의 정확한 형태와 top-level 문의 의미는 [4.16 진입점](../004-functions/416-entry-point.md)에서 정의합니다. 패키지에 속하지 않은 단일 파일을 실행하면 그 파일을 익명 패키지의 `main.rok`으로 취급합니다 ([4.16.8](../004-functions/416-entry-point.md#4168-단일-파일-실행)).

### 라이브러리 패키지: `src/lib.rok`

`src/lib.rok`이 있는 디렉토리는 라이브러리 패키지로 인식됩니다.

```
my_lib/
├── rok.rokon
└── src/
    ├── lib.rok       <- library root
    └── ...
```

라이브러리 패키지에는 진입점이 없고, 외부에서 가져다 쓰는 정의 묶음 역할을 합니다. `main.rok`을 제외한 모든 모듈 파일(`lib.rok` 포함)은 항목만 담을 수 있고 top-level 문을 둘 수 없습니다. 그래서 모듈을 가져와도 아무 코드도 실행되지 않습니다 ([4.16.6](../004-functions/416-entry-point.md#4166-모듈은-정의-전용)).

### 한 패키지의 라이브러리와 실행 파일

패키지는 `src/main.rok`과 `src/lib.rok`을 둘 다 가질 수 있습니다. 둘은 같은 `src/`에서 빌드하는 두 target, 곧 라이브러리와 그 라이브러리를 쓰는 실행 파일입니다. 실행 파일은 라이브러리를 패키지 이름을 딴 의존 패키지로 봅니다 (`use my_app::parse;`; [9.10.3](../009-tools/910-rok-frontend.md#9103-rokrokon--roklock)의 코드 이름, `-`를 `_`로 바꾼 것). 라이브러리는 실행 파일과 별개의 패키지입니다. 실행 파일에서는 라이브러리의 `pub` 항목만 보이고 `pub(pkg)`, `pub(sub)`, 비공개 항목은 보이지 않으며 ([5.7.14](../005-modules/507-paths.md#5714-패키지-사이-경로)), 실행 파일이 `use`하기 전에는 어떤 항목도 범위에 없습니다. 어느 target에서든 `pkg::`는 그 target 자신의 루트입니다. target마다 모듈 트리가 따로 있으므로 둘 다 같은 `src/util.rok`을 가리켜 `mod util;`을 선언할 수 있고, 각자 자기 사본을 얻습니다. 이 패키지에 의존하는 패키지는 라이브러리만 얻습니다.

## 0.3.2. 파일과 모듈

패키지 안의 추가 모듈은 `src/` 아래 파일로 두며, 두 형태 중 하나를 고릅니다.

```
foo.rok                <- single-file module (no submodules)
```

또는 다음처럼 폴더를 함께 두는 형태도 있습니다.

```
foo.rok                <- module body
foo/                   <- foo's submodules
├── bar.rok                foo::bar submodule
└── baz.rok                foo::baz submodule
```

폴더 형태에서는 본체 파일 (`foo.rok`)과 같은 이름의 폴더 (`foo/`)가 함께 놓입니다. 본체 안에서 `mod 이름;`으로 선언한 서브모듈만 폴더 안의 파일과 매핑됩니다. Rust 2018+와 같은 규칙이며, 자세한 매핑 규칙은 [모듈](../005-modules/504-modules.md)에서 정의합니다.

별도 파일 없이 본체 안에 `mod 이름 { ... }` 또는 `pub mod 이름 { ... }` 형태로 인라인 모듈을 둘 수도 있습니다. Rust의 인라인 모듈과 같은 동작이며, 자세한 형태와 매핑 역시 [모듈](../005-modules/504-modules.md)에서 정의합니다.

## 0.3.3. 패키지 식별자

각 패키지에는 내용 기반 해시가 붙습니다. 해시 입력은 패키지의 모든 소스 파일을 `src/` 기준 상대 경로(`/` 구분자)의 바이트 순서로 정렬해 이어 붙인 것이며, 각 파일은 `경로 + \0 + 내용 + \0`으로 직렬화합니다. 그 SHA-256의 앞 8글자를 식별자로 씁니다.

내용이 같은 패키지는 어디서 빌드해도 해시가 같고, 한 글자라도 바뀌면 새 해시가 나옵니다.

이 해시는 산출물의 심볼 이름과 캐시 키에 쓰이며, 구체적인 사용처는 뒤의 문서에서 정의합니다.

## 0.3.4. 매니페스트 (`rok.rokon`) 는 선택

패키지인지는 파일이 있는지만으로 정해집니다. 디렉토리에 `src/main.rok`이나 `src/lib.rok`이 있으면 그 디렉토리가 패키지이고, 어느 파일이 있느냐에 따라 실행 파일, 라이브러리, 또는 둘 다를 가집니다. 매니페스트는 이 규약을 바꾸지 않습니다.

의존성, dev 의존성, toolchain 고정 같은 정보가 필요할 때만 패키지 루트의 `src/` 옆에 매니페스트 `rok.rokon` ([9.10.3](../009-tools/910-rok-frontend.md#9103-rokrokon--roklock))을 두고, 도구가 해소한 결과는 lockfile `rok.lock`에 남습니다. 두 파일 이름은 이 철자 그대로(모두 소문자) 씁니다. 대소문자가 다른 이름(`Rok.rokon` 등)은 경고입니다. 두 파일은 [ROKON](../../refs-standard-libraries/300-std/309-rokon.md) 형식이며, 매니페스트는 `Package { .. }` 값 하나로 읽기만 할 뿐 컴파일하거나 실행하지 않습니다.

이렇게 정하면 두 가지가 가능합니다. 첫째, 의존성이 없는 패키지는 컴파일러가 파일만 보고 처리할 수 있습니다. 둘째, 매니페스트는 덧붙는 것일 뿐이어서 기존 패키지 정의를 깨지 않습니다.

## 0.3.5. 테스트와 벤치마크 디렉토리

패키지는 `src/` 옆에 `tests/`와 `benches/` 디렉토리를 둘 수 있습니다. `src/` 밖의 다른 파일(0.3.1)과 달리 이 둘은 컴파일되지만, 해당 명령에서만 컴파일됩니다. `tests/`는 `rokc test`, `benches/`는 `rokc bench`에서입니다. 일반 빌드, 라이브러리 인터페이스(`.rokhir`), 패키지 식별자(0.3.3)에는 들어가지 않으며, 이 셋은 모두 `src/`만으로 정해집니다.

각 디렉토리는 합성 루트 모듈로 붙으며(`tests/`는 `tests` 모듈, `benches/`는 `benches` 모듈 아래), [모듈](../005-modules/504-modules.md#544-패키지-루트)의 일반 레이아웃 규칙을 따릅니다. `tests/parse.rok`은 `tests::parse`이고, 옆의 `tests/parse/` 폴더가 그 모듈의 하위 모듈을 담습니다. `mod.rok`은 없습니다([5.4.6](../005-modules/504-modules.md#546-사용자-정의-경로-path)). 이름 규칙은 `src/`와 같습니다.

이 코드는 in-crate입니다. 패키지와 함께 컴파일되고, `pkg::`로 패키지에 접근하며, `src/` 아래 모듈과 같은 패키지 내부 가시성을 가집니다. 의존성처럼 `pub`만 보는 것이 아니므로 `pub(pkg)` 항목과 패키지 자신의 비공개 모듈까지 다룰 수 있습니다. 매니페스트의 `dev_deps` ([9.10.3](../009-tools/910-rok-frontend.md#9103-rokrokon--roklock))를 쓸 수 있습니다. `src/lib.rok`과 `src/main.rok`을 둘 다 가진 패키지에서는 두 디렉토리를 라이브러리에 대해 빌드하며, 실행 파일 자신의 테스트는 `src/` 안에 인라인으로 둡니다.

이 디렉토리의 파일은 암묵적으로 test/bench 전용이므로 `#[cfg(test)]`가 필요 없습니다. 측정 함수는 `#[test]`나 `#[bench]`로 표시하고 ([6.2.5](../006-meta-system/602-built-in-attributes.md#625-test-테스트-함수-마커)), 디렉토리의 다른 항목은 공용 헬퍼입니다.

```
my_lib/
├── rok.rokon
├── src/
│   └── lib.rok
├── tests/                <- rokc test 전용
│   ├── parse.rok             tests::parse   (pkg:: 사용)
│   └── common.rok            tests::common  (공용 헬퍼, #[test] 없음)
└── benches/              <- rokc bench 전용
    └── parse.rok             benches::parse
```
