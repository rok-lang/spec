# 9.10. `rok`: 패키지 / 빌드 프런트엔드

`rok`은 cargo 같은 프런트엔드입니다. 매니페스트 `rok.rokon`, lockfile `rok.lock`, 의존성, 환경, toolchain 고정, 스크립트 의존성을 맡습니다. 컴파일은 컴파일러 `rokc`([9.1](901-rokc-cli.md))를 불러 처리합니다. 런타임은 `rokr`(`librokr.a`)입니다.

의존성이 없는 패키지는 매니페스트 없이 파일 규약만으로 빌드됩니다([0.3.4](../000-meta/003-source-units.md#034-매니페스트rokrokon는-선택-사항)). 이때 `rok build`는 `rokc build`와 같습니다.

## 9.10.1. 서브명령과 전역 옵션

| 서브명령                                                          | 동작                                    |
|-------------------------------------------------------------------|-----------------------------------------|
| `new`, `init`                                                     | 패키지 생성(9.10.2)                     |
| `add`, `remove`, `update`, `deps`                                 | 의존성(9.10.4)                          |
| `search`, `publish`, `yank`, `login`, `logout`                    | stone(9.10.4a)                          |
| `env`                                                             | 환경(9.10.5)                            |
| `toolchain`                                                       | toolchain(9.10.6)                       |
| `install`, `uninstall`                                            | REPL / Jupyter용 라이브러리와 실행 파일(9.10.5a) |
| `build`, `check`, `run`, `emit`                                   | 의존성을 붙여 `rokc`에 위임            |
| `test`, `bench`                                                   | `rokc`가 의존성을 붙여 하네스를 빌드하고, `rok`이 실행([9.1](901-rokc-cli.md)) |
| `doc`, `lint`, `repl [path]`, `lsp`                               | 의존성을 붙여 `rokc`에 위임            |
| `fmt`, `clean`, `demangle`, `profdata`                            | 의존성 없이 `rokc`에 위임              |
| `jupyter`                                                         | 커널 실행(`--connection-file`)은 위임, `jupyter install [--env <name>]`은 kernelspec 설치([9.8.3](908-jupyter-integration.md#983-kernelspec-설치)) |
| `version`, `help`                                                 |                                         |
| (없음)                                                            | `rok repl`과 같음. REPL 시작           |

위임하는 서브명령은 같은 이름의 `rokc` 서브명령을 실행합니다. `rok`은 자기 전역 옵션(`--env` / `--offline` / `--locked` / `--cwd`, `--` 앞 어디든)만 소비합니다. 나머지 인자는 그대로 넘기고 다음을 더합니다. `test`와 `bench`에서 `rok`은 `rokc`로 하네스를 빌드하게만 하고(`--emit-harness`) 실행은 직접 합니다([9.1](901-rokc-cli.md)). 그래서 러너는 컴파일러에 속하지 않습니다.

- 의존 패키지마다 `--extern name=<소스 디렉토리>`. `rok.lock`을 따라 준비하고, 의존되는 쪽이 의존하는 쪽보다 앞에 옵니다(`rok deps`가 출력하는 순서). 미리 빌드는 `rokc`가 합니다([9.2.3](902-aot-pipeline.md#미리-빌드한-의존-패키지)).
- 패키지 자신의 feature는 `--cfg feature=<f>`로, 각 의존 패키지의 feature는 `--extern-feature <name>=<f>`로 넘깁니다(9.10.4c). build / run / test / bench / check / lint / doc / emit / repl / lsp / jupyter / deps의 feature 옵션은 `rok`이 소비합니다(`--` 앞). 이런 옵션에는 `--features <f,..>`(여러 번 가능, `<dep>/<f>`도), `--all-features`, `--no-default-features`가 있습니다.
- 매니페스트의 `lints`를 `-A` / `-W` / `-D`로 넘깁니다(인자 앞에). 매니페스트는 경로가 고르는 프로젝트의 것입니다. `.rok` 파일이면 그 파일 위치나 그 위에서 `src/main.rok`, `src/lib.rok`, `rok.rokon` 중 하나가 있는 가장 가까운 디렉터리입니다. 패키지 `src/` 안의 파일은 그 패키지를 고릅니다([9.1.1](901-rokc-cli.md#911-일반-형태)). `rok.rokon`이 있는 디렉터리의 그 밖의 단독 파일이 있습니다. 패키지 `src/` 옆의 스크립트나 `rok.rokon`만 있는 디렉터리의 파일이 그렇습니다. 이런 파일은 그 매니페스트의 `lints`(와 `lints_from`)와 의존성을 쓰는 단일 파일 패키지로 컴파일합니다.
- 패키지 밖(환경, 스크립트)이면 `--target-dir`에 그 환경이나 스크립트의 `target/`을 넘깁니다.

`rok repl <path>`는 `run`처럼 그 경로의 프로젝트에서 의존성과 린트를 정하고, 매니페스트 이름을 넘깁니다(`--name`).

옵션과 출력, 종료 코드는 `rokc`의 것([9.1](901-rokc-cli.md))을 따릅니다. `rok`은 닫힌 표준 출력이나 표준 에러를 `rokc`와 같은 방식으로 다룹니다([9.1.6](901-rokc-cli.md#916-종료-코드)). `rokls`는 `rok lsp`와 같은 독립 실행 파일입니다.

서브명령 없이 `rok [옵션]`만 실행하면 `rok repl`과 같이 REPL을 시작합니다.

```
rok                 # = rok repl
rok --env sci       # = rok repl --env sci
```

`rok [options] <file> [args..]`은 스크립트를 실행합니다. `rok run -q <file> -- args..`와 같습니다. 파일 뒤의 모든 인자는 그대로 프로그램에 넘기고, 종료 코드는 프로그램의 것입니다. 그 단어가 `/`(Windows에서는 `\`도)를 담거나 확장자가 `.rok`이면 파일로 봅니다. 그런 서브명령은 없으므로 서브명령을 파일로 잘못 아는 일은 없습니다. shebang 줄([1.2.3](../001-lexical/102-lexical-structure.md#shebang-줄))과 함께 쓰면 `./script.rok args`가 스크립트를 실행합니다.

위임하는 명령의 경로는 옵션도 옵션의 값도 아닌 첫 인자입니다. 그래서 옵션이 그 앞에 와도 됩니다(`rok run --release t.rok`).

`rok`의 전역 옵션은 다음과 같습니다.

| 옵션             | 의미                                                              |
|------------------|-------------------------------------------------------------------|
| `--env <name>`   | 그 이름의 환경에서 작업함(REPL, Jupyter, `rok install`). 패키지에 작용하는 명령에서는 에러([9.10.5](#9105-환경)) |
| `--offline`      | 네트워크를 쓰지 않음. git 의존은 캐시에 있어야 함                 |
| `--locked`       | `rok.lock`을 바꿔야 하면 바꾸지 않고 에러(CI용)                 |
| `--cwd <dir>`    | 디렉터리를 바꾼 뒤 실행                                           |
| `-q`, `--quiet`  | 출력을 줄임                                                       |

환경 변수는 `ROK_HOME`(기본 `~/.rok`), `ROK_ENV`(9.10.5), `ROK_TOOLCHAIN`(9.10.6), `ROK_REGISTRY`(index)와 `ROK_REGISTRY_TOKEN`(레지스트리 API 토큰, 9.10.4a), `ROK_DIST`(릴리스 서버, 9.10.6)입니다.

## 9.10.2. `rok new` / `rok init`: 패키지 생성

```
rok new [--lib] [--bin] <name>
rok init [--lib] [--bin] [dir]
rok init --script <file.rok>
```

`rok new`는 `<name>` 디렉터리를 만들고 기본 파일을 채웁니다. `--bin`(기본값)은 인사말을 출력하는 `src/main.rok`을 만듭니다. `--lib`은 예시 함수와 `#[test]` 하나를 담은 `src/lib.rok`을 만듭니다. 둘을 함께 주면 둘 다 만들고, 이때 `src/main.rok`은 `use <name>::add;`로 라이브러리의 함수를 부릅니다([0.3.1](../000-meta/003-source-units.md#031-실행-파일과-라이브러리)). `<name>`이 이미 있으면 에러입니다.

`rok init`은 이미 있는 디렉터리(생략하면 현재 디렉터리)를 패키지로 만듭니다. 그 디렉터리에 `src/main.rok`이나 `src/lib.rok`이 이미 있으면 에러입니다.

`rok init --script <file.rok>`은 스크립트([9.10.7](#9107-단일-파일-스크립트의-인라인-매니페스트))를 패키지로 바꿉니다. 패키지는 그 스크립트의 디렉터리에 두고, 이름은 파일 이름에서 `.rok`을 뗀 것입니다. `rok.rokon`은 스크립트 블록을 합친 `deps`(와 `native`)를 받고, 경로는 그 파일 기준으로 다시 씁니다. 스크립트는 `src/main.rok`이 됩니다. 그 `mod`가 읽는 모든 파일은 같은 상대 경로로 `src/` 아래로 옮기고, 각 파일의 블록은 뺍니다. 다음 경우는 에러입니다. 스크립트가 패키지 안에 있을 때, 옮길 파일이나 `rok.rokon`이 이미 있을 때, 서브모듈 파일이 스크립트의 디렉터리 밖에 있을 때입니다.

두 명령은 정규 형식([ROKON 3.9.4](../../refs-standard-libraries/300-std/309-rokon.md#394-정규-형식))의 `rok.rokon`과 `target/`을 담은 `.gitignore`도 씁니다. git 저장소 안이 아니면 `git init`을 실행합니다. `init`은 디렉터리 이름을 이름 규칙(9.10.3)에 맞게 바꿔 패키지 이름으로 씁니다.

```
hello/
├── .gitignore        <- target/
├── rok.rokon
└── src/
    └── main.rok
```

## 9.10.3. `rok.rokon` / `rok.lock`

패키지 루트의 `rok.rokon`이 매니페스트이고, 같은 자리의 `rok.lock`이 lockfile입니다. 둘 다 [ROKON](../../refs-standard-libraries/300-std/309-rokon.md) 형식입니다. 이름은 이 철자 그대로(모두 소문자)입니다. 대소문자만 다른 파일(`Rok.rokon` 등)은 경고를 내고 무시하며, 읽지 않습니다.

매니페스트는 ROKON 값 하나, 곧 `Package { .. }` struct입니다(루트 이름 `Package`는 생략 가능). 매니페스트는 읽기만 하고 컴파일하거나 실행하지 않습니다.

```rokon
Package {
    name: "geom",                    // 문자열, 선택
    version: "0.3.1",                // 문자열, 선택
    toolchain: "0.4",                // 문자열, 선택 (toolchain 고정, 9.10.6)
    description: "2D geometry",      // 문자열, 선택
    license: "MIT OR Apache-2.0",    // 문자열, 선택 (SPDX 식)
    repository: "https://example.com/geom.git",   // 문자열, 선택
    authors: ["Ann Lee <ann@example.com>"],       // 문자열 목록, 선택
    homepage: "https://example.com/geom",         // 문자열, 선택
    documentation: "https://example.com/geom/doc",   // 문자열, 선택
    readme: "README.md",             // 문자열, 선택 (rok.rokon 기준 경로)
    keywords: ["geometry", "math"],  // 문자열 목록, 선택 (많아야 5 개)
    publish: true,                   // bool, 선택 (false: `rok publish` 가 거부)
    deps: {                          // 패키지 이름 (문자열) -> Dep
        "util": Dep::path("../util"),
        "parser": Dep::git("https://example.com/parser.git", tag: "v1.2"),
        "http": "0.9",                                   // stone (= Dep::version("0.9"))
        "json": Dep::version("^1.4", optional: true),    // feature 가 켤 때만 (9.10.4c)
        "csv": Dep::version("^0.3", registry: "https://example.com/index"),
    },
    dev_deps: {                      // deps 와 같은 형태. test / bench 에서만
        "fixtures": Dep::path("tests/fixtures"),
    },
    features: { "default": ["json"], "json": ["dep:json"], "tls": ["http/tls"], "full": ["json", "tls"] },
    native: { "zlib": "^1.3", "png": Dep::native(url: "vendor/png.rokon", sha256: "<64 hex>") },
    lints_from: "..",                // 문자열, 선택 (../rok.rokon 의 lints 에서 시작)
    lints: { "dead_code": "allow", "nonstandard_style": "deny" },
}
```

- 값은 ROKON의 리터럴(문자열, `{ "k": v }` dict, 빈 dict `{}`)과 `Dep::path("..")` / `Dep::git("url", rev | tag | branch: "..")` / `Dep::version("req", registry: "url")`입니다. `Dep::git`에는 `rev` / `tag` / `branch` 중 많아야 하나만 씁니다. `Dep::version`의 `registry`는 선택입니다. 의존성 값이 문자열 하나면 `Dep::version`을 줄여 쓴 것입니다.
- `Dep::path` / `Dep::git` / `Dep::version`은 `optional: true`, `features: ["<f>", ..]`, `default_features: false`도 받습니다(9.10.4c). optional인 dev 의존은 에러입니다.
- `Dep::registry`는 `rok.lock` 전용입니다. `rok.rokon`에 쓰면 에러입니다.
- 패키지 이름(`deps`의 키, `name`)은 `[a-z0-9_-]+`의 소문자 ASCII이고 글자로 시작해야 합니다. 숫자, `-`, `_`로 시작하면 에러입니다.
- 이름 `rok`은 표준 라이브러리용으로 예약되어 있습니다. 패키지, 의존 패키지, native 패키지, 레지스트리 이름이 `rok`이면 에러입니다(`` `rok` is reserved for the standard library; name the package something else, e.g. `rok_util` ``). 그 이름으로 `rok new` / `rok init` / `rok add`를 실행해도 마찬가지입니다.
- 코드가 패키지를 이름으로 부를 수 있어야 하므로 코드 이름(아래)은 키워드(1.5: `fn`, `match`, `self`, ..)나 경로 시작점(5.5.7: `pkg`, `sup`)일 수 없습니다. 실행 파일의 `use pkg::add;`는 언제나 자기 루트를 가리킵니다(`` `pkg` already names a package's own root (`pkg::`), so code can't name a package called that; name the package something else, e.g. `pkg_util` ``). 키는 문자열입니다(`"rok-syntax": Dep::version("^0.1")`).
- 코드에서는 `-`를 `_`로 바꾼 이름이 그 패키지를 가리키는 최상위 경로입니다(`rok-syntax` → `use rok_syntax::..`, `--extern rok_syntax=..`).
- 레지스트리와 한 의존 그래프 안에서 이름은 `-`와 `_`를 같게, 대소문자를 구분하지 않고 비교합니다. 그래서 `rok-syntax`와 `rok_syntax`는 함께 있을 수 없습니다.
- 의존 패키지에는 `src/lib.rok`이 있어야 합니다. 그 패키지에 `src/main.rok`이 있든 없든, 의존하는 쪽이 얻는 것은 그 라이브러리입니다. `src/main.rok`만 있는 의존 패키지는 에러이고, 에러는 그 패키지가 `rok install`용 프로그램이라고 알립니다. 전이 의존은 의존 패키지의 `deps`입니다(`dev_deps`는 전이되지 않음).
- `native`는 native 패키지 이름을 `Dep::native("req"[, registry: "url"])`, 요구 문자열, 또는 `Dep::native(url: "<url 또는 매니페스트 기준 경로>", sha256: "<64 자리 hex>")`에 연결합니다(9.10.4b). native 패키지 이름은 따로 이름 공간을 쓰므로 패키지와 native 패키지는 같은 이름을 쓸 수 있습니다. `deps` / `dev_deps`의 `Dep::native`는 에러입니다. 그래프에 있는 모든 패키지의 native 의존이 실행 파일에 링크됩니다(`dev_deps`로만 닿는 것은 test / bench에서만).
- `lints`는 린트 수준을 정합니다(린트나 그룹 이름 → `"allow"` | `"warn"` | `"deny"`, [6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)). 이 수준은 `rok`이 실행하는 모든 `rokc` 명령의 CLI 플래그 앞에 붙은 것처럼 다룹니다. 그래서 CLI `-A/-W/-D`와 코드의 속성이 이를 덮습니다.
- `lints_from`은 시작 린트 수준을 주는 `rok.rokon`이 있는 디렉터리를 가리킵니다(이 매니페스트의 디렉터리 기준 상대 경로, 또는 절대 경로). 먼저 그 매니페스트의 `lints`를 그 자신의 `lints_from` 사슬 뒤에 적용합니다. 이 매니페스트의 `lints`는 그 뒤에 옵니다. 그래서 여기 항목이 사슬의 같은 린트나 그룹을 덮습니다. 모든 항목은 이 순서로 `-A`/`-W`/`-D`로 넘어가고, 뒤의 플래그가 이깁니다. 이 설정은 빌드하거나 검사하는 패키지(와 환경의 매니페스트)에만 적용합니다. 의존 패키지의 매니페스트는 린트에 쓰지 않습니다. 디렉터리가 없거나, `rok.rokon`이 없거나, 사슬이 이미 지난 매니페스트로 돌아오면(순환) 에러입니다. `rok env import`는 상대 `lints_from`을 절대 경로로 바꿔 씁니다. 스크립트의 `//! ```rokon` 블록에서는 쓸 수 없습니다.
- 패키지 메타데이터
  - `authors`는 `"Name <email>"` 문자열입니다. `rok new` / `rok init`은 채우지 않습니다.
  - `keywords`는 많아야 5개입니다. `authors`나 `keywords`의 빈 항목은 에러입니다.
  - `readme`는 매니페스트의 디렉터리 기준으로 패키지 안의 경로입니다. 절대 경로나 `..`로 패키지를 벗어나는 경로는 에러입니다. 이 필드가 없으면 `README.md`가 있을 때 그 파일이 readme입니다.
  - `rok publish`는 readme 파일이 있는지 검사하고, `publish: false`인 패키지는 거부합니다(9.10.4a).
- 다음은 에러입니다. 모르는 필드 / `Dep` 종류 / 이름 있는 인자, 중복, 9.10.4c의 규칙을 어기는 `features`, 규칙에 맞지 않는 패키지 이름, ROKON이 아닌 문법, `Package { .. }` 하나가 아닌 값, 뒤에 남은 토큰. 진단은 `file:line:col`과 캐럿(^)으로 위치를 보여 줍니다.

| 의존성 출처 | 형태                                                         | `rok.lock`에 고정  |
|-------------|--------------------------------------------------------------|---------------------|
| path        | `Dep::path("..")`(매니페스트 기준 상대 경로)                 | 없음(늘 현재 내용)  |
| git         | `Dep::git("url")`, 선택으로 `rev:` / `tag:` / `branch:` 중 하나 | 커밋과 tree 해시    |
| stone       | `Dep::version("req"[, registry: "url"])` 또는 문자열 `"req"`([9.10.4a](#9104a-stone과-rokstonesio)) | 버전, 커밋, tree 해시 |
| native      | `Dep::native("req")` / `Dep::native(url:, sha256:)`([9.10.4b](#9104b-네이티브-패키지)) | 버전과 항목 SHA-256(index), 없음(해시가 고정) |

Rok에는 빌드 스크립트가 없습니다. 빌드할 때 패키지의 코드는 실행되지 않습니다. C / C++ 라이브러리는 미리 빌드된 native 패키지로 들어옵니다(9.10.4b).

한 빌드 안에서 한 이름은 한 패키지입니다. 같은 이름이 서로 다른 출처(다른 경로, 또는 다른 url / 커밋)로 요구되면 에러입니다. 에러는 두 요구처를 보여 줍니다.

### `rok.lock` 형식

`rok.lock`은 `rok`이 결정적으로 쓰는 ROKON(정규 형식)입니다. 머리 주석 뒤에 git 패키지와 stone마다 `Locked`를 하나씩 이름순으로 둡니다. path 의존은 고정하지 않으므로 나오지 않습니다.

```rokon
// rok.lock: generated by `rok`; do not edit. Commit it with your sources.
Lock {
    version: 1,
    packages: [
        Locked {
            name: "parser",
            source: Dep::git("https://example.com/parser.git", tag: "v1.2"),
            commit: "4f0c2a9e1b7d3c58a6e09f21d4b8c7a3e5f60123",
            tree: "9a1e7c3b5d2f4a6081c9e3b7d5f2a4c6e8b0d1f3",
        },
        Locked {
            name: "json",
            vers: "1.4.2",
            source: Dep::registry("https://github.com/rok-lang/rokstones.io-index"),
            commit: "0b6d..",
            tree: "77e1..",
        },
    ],
}
```

| 필드      | 뜻                                                                   |
|-----------|----------------------------------------------------------------------|
| `version` | lock 형식 버전(`1`)                                                  |
| `name`    | `rok.rokon`의 `deps` 키                                             |
| `vers`    | stone만. 해소된 버전                                                |
| `source`  | git은 `rok.rokon`에 적은 `Dep::git(..)` 그대로, stone은 `Dep::registry("<index url>")` |
| `commit`  | 해소된 커밋(40자리 16진수)                                         |
| `tree`    | 그 커밋의 git tree 해시. 캐시의 checkout을 검증하는 데 씀           |

- `Locked`의 `source`는 `Dep::git(..)` 또는 `Dep::registry(..)`입니다.
- `packages` 필드가 없으면 빈 리스트로 읽습니다.
- feature는 기록하지 않습니다. lock은 optional인 것을 포함해 그래프의 모든 패키지를 고정하므로 빌드가 켜는 feature에 의존하지 않습니다.
- index의 native 패키지는 선택 필드 `natives: [LockedNative { name, vers, source: Dep::registry("<index>"), sha256 }]`에 고정합니다. 이름순으로 정렬하고 비어 있지 않을 때만 씁니다. `sha256`은 `yanked: false`로 둔 항목의 정규 index 줄의 SHA-256입니다. 항목이 바뀌면 "run `rok update`" 에러입니다.
- `version`이 `1`이 아니면 "unsupported lockfile version" 에러입니다.
- lock을 읽다 난 모든 에러는 "(delete rok.lock and run `rok update` to make it again)"를 덧붙입니다.

## 9.10.4. 의존성: `rok add` / `remove` / `update` / `deps`

```
rok add <name> --path <dir> [--dev]
rok add <name> --git <url> [--rev <rev> | --tag <tag> | --branch <branch>] [--dev]
rok add <name>[@<req>] [--registry <url>] [--dev]      # stone
rok add <name> .. [--features <f,..>] [--no-default-features] [--optional]   # 위의 어느 것과도
rok remove <name>
rok add .. <file.rok>                    # 스크립트의 //! rokon 블록에
rok remove [--native] <name> <file.rok>
rok update [name...]
rok deps [--dev] [--features <f,..>] [--all-features] [--no-default-features]
```

- `add` / `remove`는 패키지의 `rok.rokon`을 고치고, 주석과 배치를 보존합니다. `--dev`는 `dev_deps`에 넣습니다.
- 마지막 인자로 스크립트 파일을 주면 `add` / `remove`는 패키지의 `rok.rokon` 대신 그 파일의 `//! rokon` 블록을 고칩니다(9.10.7). 블록이 없으면 shebang 줄과 맨 앞의 `//!` 줄 뒤에 만듭니다. 새 블록은 정규 형식으로 쓰고, 이미 있는 블록은 배치와 주석을 유지합니다. 비게 된 블록은 지웁니다. path는 스크립트 기준으로 쓰고, `--dev` / `--optional`은 에러입니다. 패키지 안의 스크립트는 에러입니다(패키지의 `rok.rokon`을 씀).
- `add` / `remove`를 스크립트 없이 패키지 밖에서 쓰거나 `--env`와 함께 쓰면 에러입니다. 에러는 `rok add <name> <file.rok>`과 `rok install` / `rok uninstall`을 알립니다([9.10.5a](#9105a-rok-install--rok-uninstall)).
- stone을 요구 없이 더하면 `^<가장 새 버전>`을 적습니다. `--features`, `--no-default-features`, `--optional`은 `Dep::..`에 적습니다. 패키지에 같은 이름의 feature가 없으면 `--optional`은 feature `"<name>": ["dep:<name>"]`도 더합니다.
- 빌드는 `rok.lock`의 커밋과 버전을 씁니다. 새로 선언된 의존은 lock에 더하고 지운 의존은 정리합니다. 하지만 이미 고정된 것을 옮기는 것은 `update`뿐입니다. `update`는 이름을 주면 그 패키지만, 없으면 모든 git 의존과 stone을 다시 해소합니다.
- `deps`는 해소한 의존 패키지를 `<코드 이름>=<그 src/lib.rok 경로>` 줄로 출력합니다. 코드 이름은 `-`를 `_`로 바꾼 것이고, 의존되는 쪽이 먼저 옵니다(`--dev`면 dev 의존 포함). 각 줄 뒤에는 그 `--extern-feature=<코드 이름>=<f>` 줄이 옵니다. `deps`는 feature 옵션과 `--dev`를 받습니다. native 패키지마다(의존하는 쪽이 먼저, 호스트용) `# native <name> <vers>`와 그 `rokc` 인자를 한 줄에 하나씩 출력합니다.
- `rok.lock`은 버전 관리에 함께 넣습니다.
- `rok update`는 `rok.lock`을 다시 해소하고, `rok toolchain update`는 toolchain을 교체합니다(9.10.6).

### 캐시

가져온 git 소스는 `$ROK_HOME/cache/`에 두고 모든 프로젝트가 공유합니다.

| 경로                                       | 내용                                                    |
|--------------------------------------------|---------------------------------------------------------|
| `cache/git/db/<repo>-<hash>/`              | 저장소의 bare clone(url마다 하나)                      |
| `cache/registry/db/<repo>-<hash>/`         | stone index의 bare clone(index마다 하나)              |
| `cache/git/checkouts/<commit>/`            | 커밋의 checkout. 쓰기 전에 tree 해시로 검증하고, 다르면 다시 풀어 놓음 |
| `cache/scripts/<hash>/`                    | 스크립트의 `rok.lock`과 `target/`. `<hash>`는 스크립트의 경로와, 스크립트가 닿는 모든 파일의 경로와 블록을 반영함([9.10.7](#9107-단일-파일-스크립트의-인라인-매니페스트)) |
| `cache/native/<sha256>/`                   | SHA-256이 맞은 뒤 푼 native 아카이브. 다시 받지 않음(9.10.4b) |
| `cache/native/meta/<sha256>.rokon`         | `Dep::native(url:)` 파일                                |

## 9.10.4a. stone과 rokstones.io

레지스트리에 올린 패키지를 **stone**이라 합니다. 기본 레지스트리는 **rokstones.io**이고, index는 별도 git 저장소 `https://github.com/rok-lang/rokstones.io-index`입니다.

### 버전 요구

Cargo와 같습니다.

| 요구                     | 뜻                                              |
|--------------------------|-------------------------------------------------|
| `1.2.3`, `^1.2.3`        | `>=1.2.3, <2.0.0`(`^0.2.3`은 `<0.3.0`, `^0.0.3`은 `<0.0.4`) |
| `~1.2.3`                 | `>=1.2.3, <1.3.0`                               |
| `=1.2.3`                 | 정확히                                          |
| `>=` `>` `<` `<=`        | 비교                                            |
| `*`, `1.*`, `1.2.x`      | 와일드카드                                      |
| `>=1.2, <1.5`            | 쉼표로 이은 조건을 모두 만족                    |

pre-release 버전(`1.3.0-beta.1`)은 요구에 같은 major.minor.patch의 pre-release를 적었을 때만 맞습니다.

### 사용하는 index

`ROK_REGISTRY` > `$ROK_HOME/config.rokon`의 `Config { registry: "<url>" }` > 기본값(rokstones.io index) 순입니다. `Dep::version`의 `registry:`와 `rok add --registry`는 그 의존 하나의 index를 정합니다.

### index 형식

index 저장소의 루트에는 `config.rokon`, `Config { api: "<url>" }`가 있습니다. 이 값은 레지스트리 API의 기준 URL입니다(rokstones.io는 `https://rokstones.io`). URL은 `https://`여야 하고, `http://127.0.0.1`과 `http://localhost`만 예외입니다. stone 하나는 파일 하나이고, 파일은 공개한 소문자 이름(`-`는 그대로)의 shard 경로에 있습니다.

| 이름 길이 | 경로                     |
|-----------|--------------------------|
| 1         | `1/a`                    |
| 2         | `2/ab`                   |
| 3         | `3/a/abc`                |
| 4 이상    | `ab/cd/abcd..`           |

파일의 각 줄은 버전 하나이고, 정규 형식의 한 줄 ROKON입니다. 오래된 버전이 먼저 옵니다.

```rokon
Stone { name: "json", vers: "1.4.2", deps: [{ name: "util", req: "^0.3", kind: "normal" }], source: Dep::git("https://example.com/json.git", rev: "0b6d.."), cksum: "77e1..", yanked: false }
```

- `deps`의 `kind`는 `"normal"`, `"dev"`, `"native"` 중 하나이고, `registry`는 선택입니다.
- feature가 있는 stone은 `deps` 뒤에 `features: { "<name>": ["<entry>", ..] }`를 두고, 9.10.4c처럼 검사합니다. `normal` 의존에는 `optional: true`, `features: [..]`, `default_features: false`를 둘 수 있습니다(기본값이 아닐 때만 적음). `dev` / `native` 의존은 optional일 수 없고, native 의존에는 feature가 없습니다. `rok publish`가 `rok.rokon`에서 이 값을 옮겨 적고, 해소할 때는 항목에서 읽습니다. checkout한 뒤 항목의 `features`는 그 커밋에 있는 stone의 `rok.rokon`과 같아야 합니다(순서 무관). 다르면 에러입니다.
- `native/<shard path>`는 native 패키지를 한 줄에 `Native { .. }` 하나씩 담습니다([9.10.4b](#9104b-네이티브-패키지)).
- 매니페스트의 `description`, `license`, `repository`는 뒤쪽의 선택 필드로 들어갑니다. 다른 패키지 메타데이터(`authors`, `homepage`, `documentation`, `keywords`)는 아직 항목에 들어가지 않습니다.
- 읽는 쪽은 index 줄(`Stone`, 그 `deps` 항목, `Native`, `Artifact`)의 모르는 필드를 건너뜁니다. 그래서 새 `rok`이 필드를 더해도 옛 `rok`에서 패키지가 가려지지 않습니다. 직접 만든 `Native { .. }` 파일(`Dep::native(url:)`, `rok publish --native`)의 모르는 필드는 여전히 거부합니다.
- 패키지는 `source` 저장소의 루트에 있습니다. `cksum`은 그 커밋의 git tree 해시이고, checkout한 뒤에 검증합니다.

### 해소

- 맞는 버전 가운데 철회(yank)되지 않은 가장 높은 버전을 고릅니다. lock에 고정된 버전은 여전히 맞으면 그대로 둡니다. 그 뒤에 철회되었어도 마찬가지입니다.
- 한 의존 그래프에서 이름 하나는 버전 하나입니다. 전이 의존은 index 항목의 `deps`를 따릅니다.
- 두 요구가 겹치지 않으면 에러이고, 에러는 두 요구를 밝힙니다. 백트래킹은 하지 않습니다. 둘 다 받아들이는 요구를 루트에 적으면 풀립니다.
- lock은 index와 대조합니다. 고정된 버전의 `rev`가 바뀌었으면 "run `rok update`" 에러입니다.
- `--offline`은 캐시된 index를 쓰고, `--locked`를 지킵니다.

### `rok search` / `publish` / `yank` / `login` / `logout`

```
rok search [--native] <query>
rok login [<token>]
rok logout
rok publish [--index <checkout>] [--commit]
rok yank [--native] --vers <version> [--undo] [--index <checkout>] [name]   # --native 면 name 필수
```

- `search`는 index에서 이름으로 찾고, 찾은 것마다 `<name> <newest version> - <description>`을 한 줄씩 출력합니다. 항목에 description이 없으면 그 부분은 생략합니다. index 조회는 `-` / `_` 어느 쪽 철자든 받습니다.
- `publish`는 기본이 dry run입니다.
  - `publish: false`인 패키지는 거부합니다. 에러는 지울 줄을 알려 줍니다.
  - readme(9.10.3)가 있어야 합니다. 깨끗한 작업 트리, 저장소 루트의 패키지, 그리고 `src/lib.rok`, `src/main.rok`, 또는 둘 다가 필요합니다.
  - `src/main.rok`만 있는 stone은 프로그램입니다. `rok install`이 그것을 빌드하고, 의존 패키지가 될 수 없습니다.
  - path / git의 normal 의존은 거부하고, path / git의 dev 의존은 뺍니다.
  - 같은 버전이 있는지, 그리고 `-` / `_`와 대소문자를 같게 보았을 때 이름이 충돌하는지 검사합니다. 다른 철자가 이미 차지한 이름은 거부합니다.
  - `--index` 없는 dry run은 항목만 출력합니다. `--index`가 있으면 그 checkout에 대해서도 검사합니다.
  - 항목의 source는 설정된 그대로(`insteadOf` 재작성 전)의 git remote `origin` url입니다. 이 url은 `https://`여야 하고, `http://127.0.0.1`과 `http://localhost`만 예외입니다.
- `--commit`이면 `publish`는 항목을 더합니다. `--index`가 있으면 그 checkout에 "Publish <name> <vers>" 커밋으로 더합니다(push하지 않음). 없으면 설정된 index("Which index")의 레지스트리 API로 보내고, API가 항목을 index에 넣습니다. 클라이언트는 index 저장소에 직접 쓰지 않습니다.
- `yank`는 그 버전의 `yanked`를 켭니다(`--undo`면 끕니다). `--index`가 있으면 그 checkout의 커밋으로, 없으면 레지스트리 API로 처리합니다.
- 레지스트리 API의 주소는 index의 `config.rokon`의 `api`에 있습니다. 요청은 `Authorization: Bearer <token>`을 싣고, 본문은 한 줄 ROKON(`application/rokon; charset=utf-8`)입니다.
  - `GET <api>/api/v1/me` → `Me { login: "..", publish: <bool> }`
  - `POST <api>/api/v1/publish`, 본문은 index 줄(`Stone { .. }` / `Native { .. }`) → `Published { name, vers, file }`
  - `POST <api>/api/v1/yank`, 본문 `Yank { name, vers, undo: <bool>, native: <bool> }` → `Yanked { name, vers, yanked }`

  실패는 4xx / 5xx 상태와 `Error { message: ".." }`로 옵니다. `rok`은 그 message를 보고합니다. 401은 토큰이 유효하지 않다는 뜻입니다.
- 레지스트리 API에는 토큰이 필요하고, `--index` checkout에는 필요 없습니다. `ROK_REGISTRY_TOKEN`이 설정되어 있고 비어 있지 않으면 그 값을 토큰으로 씁니다. 아니면 `rok login`이 저장한 토큰을 씁니다. 둘 다 없으면 "run `rok login` first" 에러입니다.
- `rok login`은 웹사이트에서 만든 rokstones.io 계정 토큰을 받습니다(`<api>/me`, GitHub로 로그인. 지금은 레지스트리 maintainer만 만들 수 있음). 토큰을 생략하면 표준 입력에서 화면에 보이지 않게 읽습니다(Ctrl-C에도 에코를 되돌림). `login`은 `/api/v1/me`에 물어봅니다. API가 거부하는 토큰과 `publish`가 false인 계정("this account cannot publish yet: publishing is open to rokstones.io maintainers only")은 아무것도 저장하지 않고 거절합니다. 그 밖에는 `$ROK_HOME/credentials.rokon`에 `Credentials { token: ".." }`를 저장하고(Unix에서는 mode 0600) "Logged in as <login>"을 출력합니다. `rok logout`은 그 파일을 지웁니다.
- 토큰은 `curl`의 표준 입력으로만 전달합니다. 명령줄이나 환경에 넣지 않고 출력하지도 않습니다(에러에서는 `***`로 보임).

## 9.10.4b. 네이티브 패키지

Rok은 C를 컴파일하지 않습니다. native 패키지는 타깃 트리플마다 미리 빌드한 아카이브와 다음과 같은 값으로 이루어집니다.

```rokon
Native {
    name: "png", vers: "1.6.43",
    deps: { "zlib": Dep::native("^1.3") },
    targets: {
        "x86_64-unknown-linux-gnu": Artifact {
            url: "https://example.com/png-1.6.43-x86_64-linux.tar.gz",
            sha256: "<64 hex>",
            links: { "png16": "static" },
            runtime: [],
        },
    },
    yanked: false, description: "PNG codec", license: "Libpng", repository: "https://example.com/png",
}
```

이 값은 index 한 줄이거나 `Dep::native(url:, sha256:)`의 파일입니다. 파일 안의 artifact url은 그 파일 기준 상대 경로여도 됩니다.

- 아카이브(tar로 읽을 수 있는 어떤 형식이든)에는 `lib/`가 들어갑니다. 그 안에는 `lib<n>.a`, `lib<n>.dylib` / `lib<n>.so`, Windows `<n>.lib`(static 또는 import library)가 있습니다. `runtime` 파일과 라이선스 파일도 들어갑니다. 그 안의 어떤 것도 실행하지 않습니다.
- 해시는 필수입니다. 아카이브는 풀기 전에 검사합니다. 절대 경로나 `..` 경로, 밖으로 나가는 링크가 있거나, 이름 붙은 라이브러리나 runtime 파일이 없으면 거부합니다.
- Apple 타깃에서 runtime dylib에는 `@rpath/` install name이 있어야 합니다.
- 빌드는 타깃 트리플의 artifact를 씁니다. 없으면 에러이고, 에러는 artifact가 있는 트리플을 나열합니다.
- `rok`은 `rokc`에 `-L <archive>/lib`, link마다 `-l kind=name`, runtime 파일마다 `--native-runtime`을 넘깁니다. 의존하는 쪽이 의존되는 쪽보다 앞에 옵니다.

```
rok add --native <name>[@<req>]
rok remove --native <name>
rok search --native <query>
rok publish --native <file> [--index <dir>] [--commit]
rok yank --native --vers <version> [--undo] [--index <dir>] <name>
```

`publish --native`는 모든 타깃을 받아 검사합니다. 절대 url, 라이선스 파일, index에서 온 의존성만 허용합니다. `--commit`과 `yank --native`는 stone과 같은 규칙을 따릅니다. `--index` checkout을 쓰고, 없으면 토큰으로 레지스트리 API를 씁니다(`Yank`의 `native: true`).

- ELF 타깃에서 runtime 라이브러리의 soname은 파일 이름과 같아야 합니다. `-musl` 타깃의 `dylib` link는 에러입니다.
- `repl` / `jupyter`는 native 패키지를 로드합니다. 공유 라이브러리는 시스템 로더로 로드하고, static 아카이브는 세션에 링크합니다. `check` / `lint` / `doc`은 아무것도 링크하지 않습니다.
- 실행 파일 옆의 사본은 빌드 산출물입니다.

## 9.10.4c. feature

패키지는 `features`에 feature를 선언합니다. 이 dict는 feature 이름마다 그 feature가 켜는 항목의 리스트를 적습니다. 이름은 `[a-z0-9][a-z0-9_-]*`이고, 무엇을 켜는지 말해 주는 단어로 짓습니다(`http1`, `http2`, `tls`). 각 항목은 다음 중 하나입니다.

| 항목        | 켜는 것                                                              |
|-------------|----------------------------------------------------------------------|
| `<f>`       | 이 패키지의 feature `f`                                              |
| `dep:<d>`   | optional 의존 `d`                                                    |
| `<d>/<f>`   | 의존 `d`의 feature `f`, 그리고 `d`가 optional이면 `d` 자체        |
| `<d>?/<f>`  | optional 의존 `d`의 feature `f`. 다른 무언가가 `d`를 켤 때만       |

- `default`는 끄지 않는 한 켜져 있는 보통의 feature입니다. 빌드하는 패키지에서는 `--no-default-features`로, 의존에서는 `default_features: false`로 끕니다.
- optional 의존(`optional: true`)은 켜진 feature의 `dep:<d>`나 `<d>/<f>` 항목이 켤 때만 빌드에 들어갑니다. 그 이름의 암묵 feature는 없습니다. 켜는 항목이 하나도 없으면 에러입니다.
- 다음은 에러입니다. 항목이 패키지에 없는 feature나 (`deps`의) 의존을 가리킬 때, `dep:`나 `?`가 optional이 아닌 의존을 가리킬 때, feature가 순환할 때입니다. 진단은 `file:line:col`을 보여 줍니다. 의존 패키지에 없는 feature를 요구해도 에러입니다.
- 해소: lock은 그래프 전체를 덮습니다. 빌드는 루트 패키지의 feature에서 시작합니다. 끄지 않았으면 `default`, `--features`로 준 것, `--all-features`이면 모든 feature입니다. 켜진 의존마다 그 의존에 요구한 feature와 그 의존의 `default`를 가지고 따라갑니다. feature는 더해지기만 하고 합쳐집니다. 그래서 그래프 전체에서 패키지마다 feature 집합은 하나입니다. 루트의 `dev_deps`와 그 의존이 요구하는 feature는 `test` / `bench`에서만 셉니다. 결과는 어떤 것의 순서에도 의존하지 않습니다.
- `--features`는 루트의 feature 이름과, 직접 의존의 `<dep>/<f>`를 받습니다.
- 각 패키지는 켜진 feature에 대해서만 `cfg(feature = "<f>")`가 참인 상태로 컴파일됩니다([6.3.6](../006-meta-system/603-cfg.md#636-cfg-값의-출처)). native 패키지에는 feature가 없습니다.
- 스크립트의 ```` ```rokon ```` 블록에는 `features`가 없습니다(9.10.7).

## 9.10.5. 환경

환경은 conda / venv를 대신합니다. 활성화 단계는 없습니다.

- 패키지(`src/main.rok`, `src/lib.rok`, `rok.rokon` 중 하나가 있는 디렉터리)는 자기 `rok.rokon`의 의존성만 씁니다. 환경에 설치한 라이브러리는 보지 않습니다. 스크립트도 마찬가지입니다. 패키지에 작용하는 명령을 `--env`와 함께 쓰면 에러입니다.
- 이름 있는 환경은 `$ROK_HOME/envs/<name>/`에 `rok.rokon`, `rok.lock`, `target/`을 둡니다. 그 `deps`는 REPL과 Jupyter를 위해 `rok install`([9.10.5a](#9105a-rok-install--rok-uninstall))로 설치한 라이브러리입니다. 이 목록을 바꾸는 것은 `rok install` / `rok uninstall`과 `rok env import`뿐입니다. 그 안의 path 의존은 절대 경로입니다. 설치한 라이브러리는 그 `target/`에 미리 빌드합니다.
- 패키지 밖(REPL, Jupyter, 언어 서버, `rok install`)에서 쓸 환경은 `--env <name>` > `ROK_ENV` > `rok env use`(`$ROK_HOME/envs/CURRENT`) > `default` 순으로 정합니다. `default` 환경은 그 환경으로 처음 `rok install`을 실행할 때 만들어집니다. 패키지 안에서 경로 없는 `rok repl` / `jupyter` / `lsp`는 그 패키지를 쓰고, `--env`가 있으면 그 환경을 씁니다.

### 의존성의 출처

| 컴파일 대상 | 의존성 | Lock | 미리 빌드하는 곳 |
|---|---|---|---|
| 패키지 | 그 `rok.rokon`의 `deps`(test / bench는 `dev_deps`) | `<package>/rok.lock` | `<package>/target/<profile>/deps/` |
| 스크립트 | 스크립트와 그 `mod`가 읽는 파일의 `//! rokon` 블록을 합친 `deps` | `cache/scripts/<hash>/rok.lock` | `cache/scripts/<hash>/target/` |
| 패키지 밖의 REPL / Jupyter | 활성 환경에 설치한 라이브러리 | 환경의 것 | 환경의 `target/` |

```
rok env create <name>
rok env list                      # 현재 환경에 * 표시
rok env remove <name>
rok env use <name>                # 프로젝트 밖의 기본 환경을 바꿈
rok env export <name> <dir>       # rok.rokon / rok.lock 을 <dir> 에 씀
rok env import <name> <dir | rok.rokon>    # 프로젝트나 export 에서 만듦 (상대 path 는 절대로)
rok env shell [name]              # ROK_ENV=<name> 인 $SHELL
rok repl --env <name>
```

REPL의 `:add <dep> ..`는 현재 환경에서는 `rok install <dep> ..`을, 패키지의 세션에서는 `rok add`를 실행합니다. 그 뒤 의존성을 다시 읽어 세션에 붙입니다.

## 9.10.5a. `rok install` / `rok uninstall`

```
rok install <name>[@<req>] [--registry <url>]
rok install [<name>] --path <dir | file.rok>
rok install [<name>] --git <url> [--rev <rev> | --tag <tag> | --branch <branch>]
rok install .. [--features <f,..>] [--no-default-features] [--lib] [--force]
rok install --native <name>[@<req>] [--registry <url>]
rok install --list
rok uninstall [--native] <name>..
```

- 소스의 패키지는 먼저 캐시로 가져옵니다([9.10.4](#9104-의존성-rok-add--remove--update--deps)).
- `src/main.rok`이 있는 패키지나 스크립트 파일은 실행 파일입니다. `cargo install`처럼 `--release`로 빌드해 그 실행 파일을 `$ROK_HOME/bin/<name>`에 둡니다. git이나 stone 패키지는 checkout의 임시 사본에서, path 패키지와 스크립트 파일은 제자리에서 빌드합니다. `$ROK_HOME/installs.rokon`(`Package { deps: { "<name>": <source> } }`)이 각 실행 파일과 그 정확한 출처를 기록합니다. stone은 `=<version>`으로, git 패키지는 그 `rev:`와 함께 기록합니다. 패키지에 `src/lib.rok`도 있으면 `rok install`은 그 사실을 알리고, `--lib`를 주면 라이브러리를 대신 설치한다고 알립니다.
- `src/lib.rok`만 있는 패키지, 또는 `--lib`를 준 `src/lib.rok`이 있는 패키지는 라이브러리입니다. 라이브러리는 `pip install`처럼 `rok repl`, `rok jupyter`, `:add`를 위해 활성 환경([9.10.5](#9105-환경))에 들어갑니다. 환경의 `rok.rokon`에 적고(요구 없는 stone은 `^<가장 새 버전>`, path는 절대 경로), 그 `rok.lock`으로 해소하고, 그 `target/`에 미리 빌드합니다. 스크립트는 이 라이브러리를 보지 않습니다. 스크립트는 `rok add <name> <file.rok>`으로 라이브러리를 얻고, 출력 줄이 이를 알립니다. 빌드되지 않으면 아무것도 설치하지 않습니다. `src/lib.rok`이 없는 패키지나 스크립트에 `--lib`를 주면 에러입니다.
- `<name>`은 라이브러리의 최상위 이름, 또는 실행 파일의 파일 이름입니다. 생략하면 패키지의 `name`을 씁니다. 그것도 없으면 디렉터리 이름(path), `.git`을 뗀 저장소 이름(git), 파일 stem(스크립트)을 씁니다. [9.10.3](#9103-rokrokon--roklock)의 이름 규칙을 따릅니다.
- `--features` / `--no-default-features`는 라이브러리의 `Dep`에, 또는 실행 파일의 빌드에 들어갑니다. `--dev` / `--optional`은 에러입니다. `--native`는 환경의 `native`에 native 패키지를 더합니다([9.10.4b](#9104b-네이티브-패키지)).
- 프록시 `rok`, `rokc`, `rokls`([9.10.6](#9106-toolchain))는 절대 교체하지 않습니다. `rok install`이 두지 않은 `$ROK_HOME/bin`의 파일도 마찬가지입니다(에러). 실행 파일을 다시 설치할 때는 다음과 같습니다.
  - 같은 stone 버전이나 git 커밋에 feature도 같으면 다시 빌드하지 않고 유지합니다(`` `tool` 0.1.0 is installed already (<path>); to rebuild it, run `rok install tool --force` ``). `--force`를 주면 다시 빌드합니다(`Reinstalled`).
  - 버전이 다르면 교체합니다(`Upgraded tool from 0.1.0 to 0.2.0`, `Downgraded ...`). git 커밋이 달라도 마찬가지입니다(`Replaced <url> at <commit> with ...`).
  - path나 스크립트 파일에서 온 것은 `cargo install --path`처럼 언제나 다시 빌드합니다.
  - 라이브러리 설치는 그대로입니다.
- `--list`는 ``libraries (environment `<env>`):``와 `executables (<bin dir>):`를 출력합니다. 각각 뒤에 설치 순서대로 `  <name> <source>` 줄이 옵니다.
- `rok uninstall <name>..`은 각 이름을 활성 환경의 라이브러리에서 지웁니다(lock을 다시 해소). 라이브러리가 아니면 `rok install`이 `$ROK_HOME/bin`에 둔 실행 파일을 지웁니다. `--native`는 환경에서 native 패키지를 지웁니다. 둘 다 아닌 이름은 에러입니다.
- `rok install` 말고 `$ROK_HOME` 아래에 쓰는 것은 `rok toolchain`(toolchain, 프록시, `config.rokon`), `rok login`(`credentials.rokon`), `rok env`(환경), 그리고 캐시뿐입니다.

## 9.10.6. toolchain

`$ROK_HOME/bin/`의 `rok` / `rokc` / `rokls`는 활성 toolchain의 같은 실행 파일을 실행하는 프록시입니다(rustup 방식). 활성 toolchain은 다음 순서로 정합니다.

1. 명령줄의 `+<version>`(`rok +0.4 build`)
2. 환경 변수 `ROK_TOOLCHAIN`
3. 매니페스트의 `toolchain: "X"`(가장 가까운 `rok.rokon`). 프록시는 이 필드만 읽으므로 모르는 필드가 있어도 고정은 유지됩니다.
4. 기본값(`$ROK_HOME/config.rokon`의 `Config { registry, toolchain }`)

버전 접두사는 그 아래 버전과 맞습니다(`0.3`은 `0.3.*` 가운데 가장 높은 버전). 맞는 toolchain이 없으면 에러이고, 에러는 `rok toolchain install`을 안내합니다.

```
rok toolchain list
rok toolchain install <version> --from <dir | archive | url>
rok toolchain default [<version>]          # 인자 없으면 지금의 기본을 출력
rok toolchain remove <version>
rok toolchain update [<version>]
rok toolchain clean                        # 기본값을 제외한 설치된 모든 toolchain 제거
```

- 설치본은 `$ROK_HOME/toolchains/<version>-<triple>/`에 있고, 그 안에서 완결됩니다([9.2.19](902-aot-pipeline.md#9219-설치-레이아웃)). `install`은 `--from`이 필수입니다.
- `install`은 `bin/rokc --version --verbose`로 검증합니다. 버전과 호스트가 맞아야 합니다.
- 처음 설치한 toolchain이 기본값이 되고, 이때 `bin/` 프록시를 만듭니다.
- `rok toolchain update`는 `$ROK_HOME` 아래의 기본 toolchain을 교체합니다. 인자가 없으면 최신 릴리스를, 버전을 주면 그 버전을 설치합니다. 설치한 toolchain이 기본값이 되고 `bin/` 프록시를 다시 씁니다. Windows에서는 실행 중인 프록시의 이름을 바꾼 뒤 그 사본으로 교체합니다. 릴리스 아카이브를 받아 SHA-256을 검증하고(아래 배포), 따로 풀어 제자리로 옮깁니다. 그런 다음 프록시가 새 toolchain을 실행하는지 확인합니다. 실패하면 이전 기본값과 프록시를 되돌리고 새 toolchain을 지웁니다. 이전 toolchain은 설치된 채로 남습니다. 이미 설치된 버전은 내려받지 않고 기본값이 됩니다. `rok update`는 `rok.lock`을 다시 해소하고, `rok toolchain update`는 toolchain을 교체합니다(9.10.4).
- `rok toolchain clean`은 지금의 기본값을 제외한 설치된 모든 toolchain을 지웁니다. `update`가 남겨 둔 이전 버전을 정리하는 선택(opt-in) 명령입니다. 프로젝트가 고정한 버전도 함께 지워지므로 빌드에 필요하면 다시 설치해야 합니다. 기본 toolchain이 없으면 에러입니다.

### 배포

- 릴리스 아카이브는 `rok-<version>-<triple>.tar.gz`(Windows는 `.zip`)이고, 안은 toolchain 레이아웃입니다. Windows가 아니면 같은 파일을 담은 `rok-<version>-<triple>.tar.xz`도 옆에 있습니다. 각 아카이브 옆에는 `<archive>.sha256`이 있습니다. 이 파일에는 아카이브의 SHA-256을 hex로 적고, 그 뒤에 파일 이름이 올 수 있습니다(sha256sum 형식).
- `install.sh`, `install.ps1`, `rok toolchain install --from <url>`은 같은 곳에서 `<url>.sha256`을 받고, SHA-256이 다른 아카이브는 거부합니다. `.sha256`이 없으면 에러입니다. 경로로 준 아카이브는 `<path>.sha256`이 있을 때 그것과 대조합니다. 서명은 정의하지 않습니다. 출처는 HTTPS가 보증합니다.
- `install.sh`(<https://rok-lang.org/install.sh>)는 아카이브나 `rok-lang` 조직의 GitHub release에서 `$ROK_HOME`에 설치합니다. tar가 읽을 수 있고(macOS, 또는 `PATH`에 `xz`) 릴리스에 있으면 `.tar.xz`를, 아니면 `.tar.gz`를 받습니다. `rok toolchain install`은 `.tar.gz`(Windows에서는 `.zip`)를 받습니다.
- 릴리스는 `$ROK_DIST`(기본값 `https://github.com/rok-lang/rok/releases`)에서 내려받습니다. `<dist>/download/v<version>/<archive>`와 그 `.sha256`, 그리고 최신 릴리스의 버전을 담은 `<dist>/latest/download/rok-version`이 있습니다.

## 9.10.7. 단일 파일 스크립트의 인라인 매니페스트

익명 패키지([4.16.8](../004-functions/416-entry-point.md#4168-단일-파일-실행))는 의존성을 ```` ```rokon ```` 블록에서만 얻습니다. 이 블록은 그 파일과 그 `mod`가 읽는 각 파일의 맨 앞(shebang 줄 뒤) `//!` 줄 안에 있고, `Package { deps: { .. } }`(와 `native`)를 담습니다(uv의 inline script metadata와 같음). 환경에 설치한 라이브러리는 쓰지 않습니다. 블록에는 `deps`와 `native`만 둘 수 있습니다(`features`가 없으므로 optional 의존도 없음). 다른 언어의 코드 블록은 무시합니다. 블록 안의 path는 그 블록을 담은 파일 기준입니다.

블록은 합쳐집니다. 두 파일이 한 라이브러리에 서로 다른 항목을 주면 에러이고, 에러는 두 파일과 두 항목을 알립니다. 서브모듈 파일을 직접 실행하면 그 실행이 닿는 파일의 블록만 씁니다. `rok add <name> .. <file.rok>` / `rok remove <name> <file.rok>`이 블록을 씁니다([9.10.4](#9104-의존성-rok-add--remove--update--deps)). `rok init --script`는 스크립트를 패키지로 바꿉니다([9.10.2](#9102-rok-new--rok-init-패키지-생성)).

스크립트는 lockfile과 빌드를 `cache/scripts/<hash>/`에 둡니다. 키는 스크립트 경로와, 스크립트가 닿는 모든 파일의 경로와 블록입니다. 스크립트의 실행 파일은 빌드를 이루는 모든 것을 키로 캐시합니다. 여기에는 스크립트가 포함하는 모든 파일과 그 라이브러리가 들어갑니다.

스크립트에 없는 패키지를 `use x::..`로 쓰면 에러입니다. 에러는 실행할 명령 ``run `rok add x <file>` ``을 help로 주고, 활성 환경에 `x`가 있으면 note를 덧붙입니다. 패키지 안에서는 `rok add x`를 제안합니다. `import x`(와 `include` / `require`)는 에러입니다. help는 파일 옆에 `x.rok`이 있으면 `mod x;`, `x`가 경로 루트이거나 파일이 선언한 모듈이면 `use`, 그 밖에는 `rok add x <file>`입니다.

````rok
//! ```rokon
//! Package {
//!     deps: { "util": Dep::git("https://example.com/util.git", tag: "v1") },
//! }
//! ```

use util::greet;
greet();
````

단일 파일 스크립트를 다시 실행하면 이전 실행 파일을 재사용합니다. 익명 패키지의 `rokc run`은 실행 파일을 `<target>/<profile>/scripts/<key>/`에 둡니다. `<target>`은 스크립트의 `target/`(`cache/scripts/<hash>/` 아래, 그 `rok.lock` 옆)입니다. `<key>`는 빌드를 이루는 모든 것의 해시입니다. 여기에는 다음이 들어갑니다.

- toolchain, 빌드 옵션, 린트 수준과 타깃
- 소스와 소스가 포함하는 모든 파일
- toolchain이 그 타깃용으로 배포하는 런타임과 C 라이브러리
- 미리 빌드한 라이브러리와 함께 넘긴 의존 패키지

키가 맞으면 아무것도 컴파일하지 않고, 그 실행 파일을 만든 빌드의 경고를 다시 출력합니다. 새 실행 파일은 임시 이름으로 쓴 뒤 제자리로 이름을 바꿉니다. 그래서 한 스크립트를 동시에 실행해도 안전합니다. 하루 동안 쓰지 않은 항목은 다른 항목을 빌드할 때 지웁니다.
