# 9.10. `rok` - 패키지 / 빌드 front-end

`rok` 는 cargo 식 front-end 입니다. 매니페스트 `rok.rokon` 과 lockfile `rok.lock`, 의존성, 환경, toolchain 고정, 스크립트 의존성을 맡고, 컴파일은 컴파일러 `rokc` ([9.1](901-rokc-cli.md)) 를 불러 처리합니다. 런타임은 `rokr` (`librokr.a`) 입니다.

의존성이 없는 패키지는 매니페스트 없이 파일 규약만으로 빌드됩니다 ([0.3.4](../000-meta/003-source-units.md#034-매니페스트-rokrokon-는-선택)). 이때 `rok build` 는 `rokc build` 와 같습니다.

## 9.10.1. 서브명령과 전역 옵션

| 서브명령                                                          | 동작                                    |
|-------------------------------------------------------------------|-----------------------------------------|
| `new`, `init`                                                     | 패키지 생성 (9.10.2)                    |
| `add`, `remove`, `update`, `deps`                                 | 의존성 (9.10.4)                         |
| `search`, `publish`, `yank`, `login`, `logout`                    | stone (9.10.4a)                         |
| `env`                                                             | 환경 (9.10.5)                           |
| `toolchain`                                                       | toolchain (9.10.6)                      |
| `completions <shell>`                                             | 셸 자동완성 스크립트 (아래)             |
| `install`, `uninstall`                                            | REPL / Jupyter 용 라이브러리와 실행 파일 (9.10.5a) |
| `build`, `check`, `run`, `emit`                                   | 의존성을 붙여 `rokc` 에 위임            |
| `test`, `bench`                                                   | `rokc` 가 의존성을 붙여 하네스를 빌드하고, `rok` 가 실행 ([9.1](901-rokc-cli.md)) |
| `doc`, `lint`, `repl [path]`, `lsp`                               | 의존성을 붙여 `rokc` 에 위임            |
| `fmt`, `clean`, `demangle`, `profdata`                            | 의존성 없이 `rokc` 에 위임              |
| `jupyter`                                                         | 커널 실행 (`--connection-file`) 은 위임, `jupyter install [--env <name>]` 은 kernelspec 설치 ([9.8.3](908-jupyter-integration.md#983-kernelspec-설치)), 인자 없는 `rok jupyter` 는 `rok jupyter install` 을 알려 주는 사용법 에러 |
| `version`, `help`                                                 | 버전, 도움말 (아래)                     |
| (없음)                                                            | `rok repl` 과 같음: REPL 시작           |

위임하는 서브명령은 같은 이름의 `rokc` 서브명령을 실행합니다. `rok` 는 자기 전역 옵션 (`--env` / `--offline` / `--locked` / `--cwd` / `--color`, `--` 앞 어디든) 만 소비하고, 나머지 인자는 그대로 넘기며 다음을 더합니다. `test` 와 `bench` 는 `rok` 가 `rokc` 로 하네스를 빌드하게만 하고 (`--emit-harness`) 실행은 직접 하므로 ([9.1](901-rokc-cli.md)), 러너는 컴파일러에 속하지 않습니다.

- 의존 패키지마다 `--extern name=<소스 디렉토리>`. `rok.lock` 을 따라 준비하며, 의존되는 쪽이 의존하는 쪽보다 앞입니다 (`rok deps` 가 출력하는 순서). 미리 빌드는 `rokc` 가 합니다 ([9.2.3](902-aot-pipeline.md#미리-빌드한-의존-패키지)).
- 패키지 자신의 feature 는 `--cfg feature=<f>` 로, 각 의존 패키지의 feature 는 `--extern-feature <name>=<f>` 로 (9.10.4c). build / run / test / bench / check / lint / doc / emit / repl / lsp / jupyter / deps 의 feature 옵션 `--features <f,..>` (여러 번 가능, `<dep>/<f>` 도), `--all-features`, `--no-default-features` 는 `rok` 이 소비합니다 (`--` 앞).
- 매니페스트의 `lints` 를 `-A` / `-W` / `-D` 로 (인자 앞에). 매니페스트는 경로가 고르는 프로젝트의 것입니다: `.rok` 파일이면 그 파일 위치나 그 위에서 `src/main.rok`, `src/lib.rok`, `rok.rokon` 중 하나가 있는 가장 가까운 디렉토리. 패키지 `src/` 안의 파일은 그 패키지를 고릅니다 ([9.1.1](901-rokc-cli.md#911-일반-형태)). `rok.rokon` 이 있는 디렉토리의 그 밖의 단독 파일 (패키지 `src/` 옆의 스크립트, 또는 `rok.rokon` 만 있는 디렉토리) 은 그 매니페스트의 `lints` (와 `lints_from`) 와 의존성을 가진 단일 파일 패키지로 컴파일합니다.
- 패키지 밖 (환경, 스크립트) 이면 `--target-dir` 에 그 환경이나 스크립트의 `target/`.

`rok repl <path>` 는 `run` 처럼 그 경로의 프로젝트에서 의존성과 lint 를 정하고, 매니페스트 이름을 넘깁니다 (`--name`).

옵션과 출력, 종료 코드는 `rokc` 의 것 ([9.1](901-rokc-cli.md)) 을 따릅니다. `rok` 는 닫힌 표준 출력이나 표준 에러를 `rokc` 와 같은 방식으로 다룹니다 ([9.1.6](901-rokc-cli.md#916-종료-코드)). `rokls` 는 `rok lsp` 와 같은 독립 실행 파일입니다.

`rok help` (`rok --help` 도) 는 짧은 페이지를 출력합니다: 묶음별로 한 줄씩의 서브명령과 전역 옵션. `rok help <subcommand>` (`rok <subcommand> --help` 도) 는 그 서브명령만의 도움말을 출력합니다: 사용 형태, 설명, 예제, 옵션. 위임하는 서브명령이면 `rok` 으로 쓴 `rokc` 의 도움말에 `rok` 의 feature 옵션을 더한 것입니다.

모르는 서브명령이나 옵션은 사용법 에러 (종료 코드 2) 입니다. 메시지는 가장 가까운 서브명령이나 옵션이 있으면 그 이름을 대고 (흔히 쓰는 다른 이름도: `build` 대신 `compile`, `remove` 대신 `rm`), 이어서 ``run `rok help <subcommand>` for usage`` 를 출력합니다. index 에 없는 stone 이나 native 패키지는 그 index 에서 가장 가까운 이름과 실행할 `rok search` 를 알려 줍니다. `rok` 는 자신이 실행하는 `rokc` 에 `ROK_CLI=rok` 를 설정하므로 `rokc` 의 사용법 에러, 도움말, 에러와 경고 줄의 접두사도 `rok` 이라고 씁니다 ([9.1.7](901-rokc-cli.md#917-환경-변수)).

서브명령 없이 `rok [옵션]` 만 실행하면 `rok repl` 과 같이 REPL 을 시작합니다.

```
rok                 # = rok repl
rok --env sci       # = rok repl --env sci
```

`rok [options] <file> [args..]` 은 스크립트를 실행합니다: `rok run -q <file> -- args..` 와 같습니다. 파일 뒤의 모든 인자는 그대로 프로그램에 넘기며, 종료 코드는 프로그램의 것입니다. 그 단어는 `/` (Windows 에서는 `\` 도) 를 담거나 확장자가 `.rok` 이면 파일입니다. 그런 서브명령은 없으므로 서브명령을 파일로 오인하는 일은 없습니다. shebang 줄 ([1.2.3](../001-lexical/102-lexical-structure.md#shebang-줄)) 과 함께 쓰면 `./script.rok args` 가 스크립트를 실행합니다.

위임하는 명령의 경로는 옵션도 옵션의 값도 아닌 첫 인자이므로, 옵션이 그 앞에 와도 됩니다 (`rok run --release t.rok`). 없는 경로는 다른 무엇보다 먼저 `` `<path>` does not exist `` 에러입니다 ("not inside a package" 가 아님). 첫 단어가 필터일 수 있는 `test` 와 `bench` 는 `.rok` 로 끝나는 단어만 경로로 봅니다.

`rok completions <shell>` 은 `bash`, `zsh`, `fish`, `powershell` 의 자동완성 스크립트를 표준 출력에 냅니다. `rok help` (위임하는 서브명령은 `rokc` 의 도움말) 와 같은 명령 / 옵션 텍스트에서 만들므로 둘은 어긋나지 않습니다. 서브명령과 그 옵션, 설치된 toolchain 으로 `+<version>` 과 `toolchain default` / `toolchain remove` 의 인자, `env use` / `remove` / `shell` / `export` 와 `--env` 의 환경 이름, `--target` 의 값을 완성합니다. 이름은 완성할 때 `rok __complete toolchains` / `envs` (한 줄에 하나) 를 실행해 읽으므로 `$ROK_HOME` 을 따릅니다. `install.sh` 와 `install.ps1` 이 스크립트를 설치하고, `rok toolchain default` / `update` 가 다시 씁니다 (9.10.6).

`rok` 의 전역 옵션은 다음과 같습니다.

| 옵션             | 의미                                                              |
|------------------|-------------------------------------------------------------------|
| `--env <name>`   | 그 이름 있는 환경에서 작업함 (REPL, Jupyter, `rok install`). 패키지에 작용하는 명령에서는 에러 ([9.10.5](#9105-환경)) |
| `--offline`      | 네트워크를 쓰지 않음. git 의존은 캐시에 있어야 함                 |
| `--locked`       | `rok.lock` 을 바꿔야 하면 바꾸지 않고 에러 (CI 용)                |
| `--cwd <dir>`    | 디렉토리를 바꾼 뒤 실행                                           |
| `-q`, `--quiet`  | 출력을 줄임                                                       |
| `-v`, `--verbose` | 출력을 늘림 (`rokc` 에도 넘김)                                   |
| `--color <when>`, `--no-color` | `auto`, `always`, `never`: 색. `rokc` 와 같이 정함 ([9.1.5](901-rokc-cli.md#915-전역-옵션)). 실행하는 `rokc` 에 `--color=<when>` 으로 넘김 |

전역 옵션은 서브명령 뒤에 와도 됩니다: `rok new app -q` 는 `rok -q new app` 과 같습니다.

환경 변수는 `ROK_HOME` (기본 `~/.rok`), `ROK_ENV` (9.10.5), `ROK_TOOLCHAIN` (9.10.6), `ROK_REGISTRY` (index) 와 `ROK_REGISTRY_TOKEN` (registry API 토큰) (9.10.4a), 그리고 `ROK_DIST` (릴리스 서버, 9.10.6) 입니다. `NO_COLOR`, `CLICOLOR_FORCE`, `COLORTERM`, `TERM` 은 `rokc` 와 같이 색을 정합니다 ([9.1.7](901-rokc-cli.md#917-환경-변수)).

## 9.10.2. `rok new` / `rok init`: 패키지 생성

```
rok new [--lib] [--bin] <name>
rok init [--lib] [--bin] [dir]
rok init --script <file.rok>
```

`rok new` 는 `<name>` 디렉토리를 만들고 기본 파일을 채웁니다. `--bin` (기본값) 은 인사말을 출력하는 `src/main.rok` 를, `--lib` 는 예시 함수와 `#[test]` 하나를 담은 `src/lib.rok` 를 만듭니다. 둘을 함께 주면 둘 다 만들며, 그러면 `src/main.rok` 는 `use <name>::add;` 로 라이브러리의 함수를 부릅니다 ([0.3.1](../000-meta/003-source-units.md#031-실행-파일과-라이브러리)). `<name>` 이 이미 있으면 에러입니다.

`rok init` 은 이미 있는 디렉토리 (생략하면 현재 디렉토리) 를 패키지로 만듭니다. 그 디렉토리에 `src/main.rok` 나 `src/lib.rok` 가 이미 있으면 에러입니다.

`rok init --script <file.rok>` 은 스크립트 ([9.10.7](#9107-단일-파일-스크립트의-inline-매니페스트)) 를 그 스크립트의 디렉토리에서 파일 이름 (`.rok` 를 뗀 것) 을 딴 패키지로 바꿉니다. `rok.rokon` 은 스크립트 블록들을 합친 `deps` (와 `native`) 를 받으며, 경로는 그것 기준으로 다시 씁니다. 스크립트는 `src/main.rok` 가 되고, 그 `mod` 가 읽는 모든 파일은 같은 상대 경로로 `src/` 아래로 옮기며, 각각 블록은 뺍니다. 스크립트가 패키지 안에 있거나, 옮길 파일이나 `rok.rokon` 이 이미 있거나, 서브모듈 파일이 스크립트의 디렉토리 밖에 있으면 에러입니다.

두 명령은 정규 형식 ([ROKON 3.9.4](../../refs-standard-libraries/300-std/309-rokon.md#394-정규-형식)) 의 `rok.rokon` 과 `target/` 을 담은 `.gitignore` 도 쓰며, git 저장소 안이 아니면 `git init` 을 합니다. `init` 은 디렉토리 이름을 이름 규칙 (9.10.3) 에 맞게 바꿔 패키지 이름으로 씁니다.

```
hello/
├── .gitignore        <- target/
├── rok.rokon
└── src/
    └── main.rok
```

## 9.10.3. `rok.rokon` / `rok.lock`

패키지 루트의 `rok.rokon` 이 매니페스트이고, 같은 자리의 `rok.lock` 이 lockfile 입니다. 둘 다 [ROKON](../../refs-standard-libraries/300-std/309-rokon.md) 형식입니다. 이름은 이 철자 그대로 (모두 소문자) 입니다. 대소문자만 다른 파일 (`Rok.rokon` 등) 은 경고와 함께 무시하며, 읽지 않습니다.

매니페스트는 ROKON 값 하나, 즉 `Package { .. }` struct 입니다 (루트 이름 `Package` 는 생략 가능). 읽기만 하며 컴파일하거나 실행하지 않습니다.

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

- 값은 ROKON 의 리터럴 (문자열, `{ "k": v }` dict, 빈 dict `{}`) 과 `Dep::path("..")` / `Dep::git("url", rev | tag | branch: "..")` / `Dep::version("req", registry: "url")` 입니다. `Dep::git` 의 `rev` / `tag` / `branch` 는 많아야 하나이고, `Dep::version` 의 `registry` 는 선택입니다. 의존성 값이 문자열 하나면 `Dep::version` 의 줄임입니다.
- `Dep::path` / `Dep::git` / `Dep::version` 은 `optional: true`, `features: ["<f>", ..]`, `default_features: false` 도 받습니다 (9.10.4c). optional 인 dev 의존은 에러입니다.
- `Dep::registry` 는 `rok.lock` 전용이며 `rok.rokon` 에 쓰면 에러입니다.
- 패키지 이름 (`deps` 의 키, `name`) 은 `[a-z0-9_-]+` 의 소문자 ASCII 이며 글자로 시작합니다 (숫자, `-`, `_` 로 시작하면 에러). 이름 `rok` 은 표준 라이브러리용으로 예약되어 있습니다: 패키지, 의존 패키지, native 패키지, registry 이름이 `rok` 이면 에러이며 (`` `rok` is reserved for the standard library; name the package something else, e.g. `rok_util` ``), 그것으로 `rok new` / `rok init` / `rok add` 해도 마찬가지입니다. 코드가 패키지를 이름으로 부를 수 있어야 하므로, 코드 이름 (아래) 은 키워드 (1.5: `fn`, `match`, `self`, ..) 나 경로 시작점 (5.5.7: `pkg`, `sup`) 일 수 없습니다. 실행 파일의 `use pkg::add;` 는 언제나 자기 루트를 가리킵니다 (`` `pkg` already names a package's own root (`pkg::`), so code can't name a package called that; name the package something else, e.g. `pkg_util` ``). 키는 문자열입니다 (`"rok-syntax": Dep::version("^0.1")`).
- 코드에서는 `-` 를 `_` 로 바꾼 이름이 그 패키지를 가리키는 최상위 경로입니다 (`rok-syntax` → `use rok_syntax::..`, `--extern rok_syntax=..`).
- 레지스트리와 한 의존 그래프 안에서 이름은 `-` 와 `_` 를 같게, 대소문자를 구분하지 않고 비교합니다. 그래서 `rok-syntax` 와 `rok_syntax` 는 함께 있을 수 없습니다.
- 의존 패키지는 `src/lib.rok` 를 가져야 하며, 그 패키지에 `src/main.rok` 도 있든 없든 그 라이브러리가 의존하는 쪽이 얻는 것입니다. `src/main.rok` 만 있는 의존 패키지는 그것이 `rok install` 용 프로그램이라고 알리는 에러입니다. 전이 의존은 의존 패키지의 `deps` 입니다 (`dev_deps` 는 전이되지 않음).
- `native` 는 native 패키지 이름 (자기 이름 공간: 패키지와 native 패키지는 같은 이름을 쓸 수 있음) 을 `Dep::native("req"[, registry: "url"])`, 요구 문자열, 또는 `Dep::native(url: "<url 또는 매니페스트 기준 경로>", sha256: "<64 자리 hex>")` 에 대응시킵니다 (9.10.4b). `deps` / `dev_deps` 의 `Dep::native` 는 에러입니다. 그래프의 모든 패키지의 native 의존이 실행 파일에 링크됩니다 (`dev_deps` 로만 닿는 것은 test / bench 에서만).
- `lints` 는 lint 수준을 정합니다 (lint 나 그룹 이름 → `"allow"` | `"warn"` | `"deny"`, [6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)). `rok` 이 실행하는 모든 `rokc` 명령의 CLI 플래그 앞에 붙은 것처럼 다뤄지므로, CLI `-A/-W/-D` 와 코드의 속성이 이를 덮습니다.
- `lints_from` 은 시작 lint 수준을 주는 `rok.rokon` 이 있는 디렉토리를 가리킵니다 (이 매니페스트의 디렉토리 기준 상대 경로, 또는 절대 경로). 그 매니페스트의 `lints` 를 그 자신의 `lints_from` 사슬 뒤에 먼저 적용하고, 이 매니페스트의 `lints` 가 그 뒤에 오므로, 여기 항목이 사슬의 같은 lint 나 그룹을 덮습니다 (모든 항목은 이 순서로 `-A`/`-W`/`-D` 로 넘어가며 뒤의 플래그가 이깁니다). 빌드하거나 검사하는 패키지 (와 환경의 매니페스트) 에만 적용되며, 의존 패키지의 매니페스트는 lint 에 쓰지 않습니다. 디렉토리가 없거나, `rok.rokon` 이 없거나, 사슬이 이미 지난 매니페스트로 돌아오면 (순환) 에러입니다. `rok env import` 는 상대 `lints_from` 을 절대 경로로 바꿔 씁니다. 스크립트의 `//! ```rokon` 블록에서는 쓸 수 없습니다.
- 패키지 메타데이터: `authors` 는 `"Name <email>"` 문자열이며, `rok new` / `rok init` 은 채우지 않습니다. `keywords` 는 많아야 5 개입니다. `authors` 나 `keywords` 의 빈 항목은 에러입니다. `readme` 는 매니페스트의 디렉토리 기준의, 패키지 안의 경로이며 (절대 경로나 `..` 로 패키지를 벗어나는 경로는 에러), 없으면 `README.md` 가 있을 때 그것이 readme 입니다. `rok publish` 는 readme 파일이 있는지 검사하고, `publish: false` 인 패키지는 거부합니다 (9.10.4a).
- 에러: 모르는 필드 / `Dep` 종류 / 이름 있는 인자, 중복, 9.10.4c 의 규칙을 어기는 `features`, 규칙에 맞지 않는 패키지 이름, ROKON 이 아닌 문법, `Package { .. }` 하나가 아닌 것, 뒤에 남은 토큰. 진단은 `file:line:col` 과 caret 으로 위치를 보여 줍니다.

| 의존성 출처 | 형태                                                         | `rok.lock` 에 고정  |
|-------------|--------------------------------------------------------------|---------------------|
| path        | `Dep::path("..")` (매니페스트 기준 상대 경로)                | 없음 (늘 현재 내용) |
| git         | `Dep::git("url")`, 선택으로 `rev:` / `tag:` / `branch:` 중 하나 | 커밋과 tree hash    |
| stone       | `Dep::version("req"[, registry: "url"])` 또는 문자열 `"req"` ([9.10.4a](#9104a-stone-과-rokstonesio)) | 버전, 커밋, tree hash |
| native      | `Dep::native("req")` / `Dep::native(url:, sha256:)` ([9.10.4b](#9104b-native-패키지)) | 버전과 항목 SHA-256 (index), 없음 (해시가 고정) |

Rok 에는 build script 가 없습니다: 빌드 시점에 패키지의 코드는 실행되지 않습니다. C / C++ 라이브러리는 미리 빌드된 native 패키지로 들어옵니다 (9.10.4b).

한 빌드 안에서 한 이름은 한 패키지입니다. 같은 이름이 서로 다른 출처 (다른 경로, 또는 다른 url / 커밋) 로 요구되면 두 요구처를 보여 주는 에러입니다.

### `rok.lock` 형식

`rok.lock` 은 `rok` 가 결정적으로 쓰는 ROKON (정규 형식) 입니다. 머리 주석 뒤에 git 패키지와 stone 마다 `Locked` 하나를 이름 순으로 둡니다. path 의존은 고정하지 않으므로 나오지 않습니다.

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
| `version` | lock 형식 버전 (`1`)                                                 |
| `name`    | `rok.rokon` 의 `deps` 키                                             |
| `vers`    | stone 만. 해소된 버전                                                |
| `source`  | git 은 `rok.rokon` 에 적은 `Dep::git(..)` 그대로, stone 은 `Dep::registry("<index url>")` |
| `commit`  | 해소된 커밋 (40 자리 16 진수)                                        |
| `tree`    | 그 커밋의 git tree hash. 캐시의 checkout 을 검증하는 데 씀           |

- `Locked` 의 `source` 는 `Dep::git(..)` 또는 `Dep::registry(..)` 입니다.
- `packages` 필드가 없으면 빈 리스트로 읽습니다.
- feature 는 기록하지 않습니다: lock 은 optional 인 것을 포함해 그래프의 모든 패키지를 고정하므로, 빌드가 켜는 feature 에 의존하지 않습니다.
- index 의 native 패키지는 선택 필드 `natives: [LockedNative { name, vers, source: Dep::registry("<index>"), sha256 }]` 에 고정하며, 이름 순으로 정렬하고 비어 있지 않을 때만 씁니다. `sha256` 은 `yanked: false` 로 둔 항목의 정규 index 줄의 SHA-256 이며, 항목이 바뀌면 "run `rok update`" 에러입니다.
- `version` 이 `1` 이 아니면 "unsupported lockfile version" 에러입니다.
- lock 을 읽다 난 모든 에러는 "(delete rok.lock and run `rok update` to make it again)" 를 덧붙입니다.

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

- `add` / `remove` 는 `rok.rokon` 을 고치며, 주석과 배치를 보존합니다. `--dev` 는 `dev_deps` 에 넣습니다. `add` / `remove` 는 패키지의 `rok.rokon` 을 고칩니다. 마지막 인자로 스크립트 파일을 주면 `add` / `remove` 는 대신 그 파일의 `//! rokon` 블록을 고칩니다 (9.10.7): 블록이 없으면 shebang 줄과 맨 앞의 `//!` 줄 뒤에 만들고 (정규 형식으로 쓰며, 이미 있는 블록은 배치와 주석을 유지함), 비게 된 블록은 지우며, path 는 스크립트 기준으로 쓰고, `--dev` / `--optional` 은 에러입니다. 패키지 안의 스크립트는 에러입니다 (패키지의 `rok.rokon` 을 씀). 스크립트 없이 패키지 밖에서, 또는 `--env` 와 함께 쓰면 `rok add <name> <file.rok>` 와 `rok install` / `rok uninstall` 을 알리는 에러입니다 ([9.10.5a](#9105a-rok-install--rok-uninstall)). stone 을 요구 없이 더하면 `^<가장 새 버전>` 을 적습니다. `--features`, `--no-default-features`, `--optional` 은 `Dep::..` 에 적습니다. `--optional` 은 패키지에 같은 이름의 feature 가 없으면 feature `"<name>": ["dep:<name>"]` 도 더합니다.
- 빌드는 `rok.lock` 의 커밋과 버전을 씁니다. 새로 선언된 의존은 lock 에 더하고 지운 의존은 정리하지만, 이미 고정된 것을 옮기는 것은 `update` 뿐입니다. `update` 는 이름을 주면 그 패키지만, 없으면 모든 git 의존과 stone 을 다시 해소합니다.
- `deps` 는 해소한 의존 패키지를 `<코드 이름>=<그 src/lib.rok 경로>` 줄로 (코드 이름은 `-` 를 `_` 로 바꾼 것), 의존되는 쪽이 먼저 오는 순서로 출력합니다 (`--dev` 는 dev 의존 포함). 각 줄 뒤에는 그 `--extern-feature=<코드 이름>=<f>` 줄들이 옵니다. `deps` 는 feature 옵션과 `--dev` 를 받습니다. native 패키지마다 (의존하는 쪽이 먼저, host 용) `# native <name> <vers>` 와 그 `rokc` 인자를 한 줄에 하나씩 출력합니다.
- `rok.lock` 은 버전 관리에 함께 넣습니다.
- `update` 는 lock 에서 바뀐 것마다 `Updated <name> v0.1.0 -> v0.1.1` 줄을 출력합니다 (git 의존성은 짧은 커밋으로, 새로 고정된 것은 `Locked`, 빠진 것은 `Removed`). 바뀐 것이 없으면 `note: all dependencies are up to date`, 의존성이 없으면 `` note: no dependencies to update; add one: `rok add <name>` `` 입니다. 패키지 밖에서 `--env` 없이 실행하면, 패키지 밖에서 쓰는 환경 (9.10.5) 에 라이브러리가 있을 때 그 환경을 업데이트하고 (무언가 바뀌었으면 끝에 `` Updated environment `<name>` ``), 아니면 에러입니다: `` not inside a package (..): make one with `rok init`; for an environment's libraries: `rok update --env <name>` ``.
- `rok update` 는 `rok.lock` 을 다시 해소하고, `rok toolchain update` 는 toolchain 을 교체합니다 (9.10.6).

### 캐시

가져온 git 소스는 `$ROK_HOME/cache/` 에 두고 모든 프로젝트가 공유합니다.

| 경로                                       | 내용                                                    |
|--------------------------------------------|---------------------------------------------------------|
| `cache/git/db/<repo>-<hash>/`              | 저장소의 bare clone (url 마다 하나)                     |
| `cache/registry/db/<repo>-<hash>/`         | stone index 의 bare clone (index 마다 하나)             |
| `cache/git/checkouts/<commit>/`            | 커밋의 checkout. 쓰기 전에 tree hash 로 검증하고, 다르면 다시 풀어 놓음 |
| `cache/scripts/<hash>/`                    | 스크립트의 `rok.lock` 과 `target/`. `<hash>` 는 스크립트의 경로와, 그것이 닿는 모든 파일의 경로와 블록을 덮음 ([9.10.7](#9107-단일-파일-스크립트의-inline-매니페스트)) |
| `cache/native/<sha256>/`                   | SHA-256 이 맞은 뒤 푼 native archive. 다시 받지 않음 (9.10.4b) |
| `cache/native/meta/<sha256>.rokon`         | `Dep::native(url:)` 파일                                |

## 9.10.4a. stone 과 rokstones.io

레지스트리의 패키지를 **stone** 이라 합니다. 기본 레지스트리는 **rokstones.io** 이며, index 는 별도 git 저장소 `https://github.com/rok-lang/rokstones.io-index` 입니다.

### 버전 요구

Cargo 와 같습니다.

| 요구                     | 뜻                                              |
|--------------------------|-------------------------------------------------|
| `1.2.3`, `^1.2.3`        | `>=1.2.3, <2.0.0` (`^0.2.3` 은 `<0.3.0`, `^0.0.3` 은 `<0.0.4`) |
| `~1.2.3`                 | `>=1.2.3, <1.3.0`                               |
| `=1.2.3`                 | 정확히                                          |
| `>=` `>` `<` `<=`        | 비교                                            |
| `*`, `1.*`, `1.2.x`      | 와일드카드                                      |
| `>=1.2, <1.5`            | 쉼표는 모두 만족                                |

pre-release 버전 (`1.3.0-beta.1`) 은 요구가 같은 major.minor.patch 의 pre-release 를 적었을 때만 맞습니다.

### 어느 index 인가

`ROK_REGISTRY` > `$ROK_HOME/config.rokon` 의 `Config { registry: "<url>" }` > 기본 (rokstones.io index) 순입니다. `Dep::version` 의 `registry:` 와 `rok add --registry` 는 그 의존 하나의 index 를 정합니다.

### index 형식

index 저장소의 루트에는 `config.rokon`, `Config { api: "<url>" }` 가 있습니다: registry API 의 기준 URL 입니다 (rokstones.io 는 `https://rokstones.io`). `https://` 여야 하며, `http://127.0.0.1` 과 `http://localhost` 만 예외입니다. stone 하나는 publish 된 소문자 이름 (`-` 는 그대로) 의 shard 경로에 있는 파일 하나입니다.

| 이름 길이 | 경로                     |
|-----------|--------------------------|
| 1         | `1/a`                    |
| 2         | `2/ab`                   |
| 3         | `3/a/abc`                |
| 4 이상    | `ab/cd/abcd..`           |

파일의 각 줄은 버전 하나이며, 정규 형식의 한 줄 ROKON 입니다. 오래된 버전이 먼저입니다.

```rokon
Stone { name: "json", vers: "1.4.2", deps: [{ name: "util", req: "^0.3", kind: "normal" }], source: Dep::git("https://example.com/json.git", rev: "0b6d.."), cksum: "77e1..", yanked: false }
```

- `deps` 의 `kind` 는 `"normal"`, `"dev"`, `"native"` 중 하나이며, `registry` 는 선택입니다.
- feature 가 있는 stone 은 `deps` 뒤에 `features: { "<name>": ["<entry>", ..] }` 를 가지며, 9.10.4c 처럼 검사합니다. `normal` 의존은 `optional: true`, `features: [..]`, `default_features: false` 를 가질 수 있습니다 (기본값이 아닐 때만 적음). `dev` / `native` 의존은 optional 일 수 없고, native 의존에는 feature 가 없습니다. `rok publish` 가 `rok.rokon` 에서 이것들을 적고, 해소는 항목에서 읽습니다. checkout 한 뒤 항목의 `features` 는 그 commit 의 stone 의 `rok.rokon` 의 것과 같아야 하며 (순서 무관), 그렇지 않으면 에러입니다.
- `native/<shard path>` 는 native 패키지를 한 줄에 `Native { .. }` 하나씩 담습니다 ([9.10.4b](#9104b-native-패키지)).
- 매니페스트의 `description`, `license`, `repository` 는 선택인 뒤쪽 필드로 들어갑니다. 다른 패키지 메타데이터 (`authors`, `homepage`, `documentation`, `keywords`) 는 아직 항목에 들어가지 않습니다.
- 읽는 쪽은 index 줄 (`Stone`, 그 `deps` 항목, `Native`, `Artifact`) 의 모르는 필드를 건너뜁니다. 그래서 새 `rok` 이 필드를 더해도 옛 `rok` 에서 패키지가 가려지지 않습니다. 자기 `Native { .. }` 파일 (`Dep::native(url:)`, `rok publish --native`) 은 여전히 거부합니다.
- 패키지는 `source` 저장소의 루트에 있습니다. `cksum` 은 그 커밋의 git tree hash 이며, checkout 뒤에 검증합니다.

### 해소

- 맞는 버전 가운데 yank 되지 않은 가장 높은 것을 고릅니다. lock 에 고정된 버전은 여전히 맞으면 (그 뒤 yank 되었더라도) 그대로 둡니다.
- 한 의존 그래프에서 이름 하나는 버전 하나입니다. 전이 의존은 index 항목의 `deps` 를 따릅니다.
- 두 요구가 겹치지 않으면 두 요구를 밝히는 에러입니다. backtracking 은 없으며, 둘 다 받아들이는 요구를 루트에 적으면 풀립니다.
- lock 은 index 와 대조합니다. 고정된 버전의 `rev` 가 바뀌었으면 "run `rok update`" 에러입니다.
- `--offline` 은 캐시된 index 를 쓰고, `--locked` 를 지킵니다.

### `rok search` / `publish` / `yank` / `login` / `logout`

```
rok search [--native] <query>
rok login [<token>]
rok logout
rok publish [--index <checkout>] [--commit]
rok yank [--native] --vers <version> [--undo] [--index <checkout>] [name]   # --native 면 name 필수
```

- `search` 는 index 에서 이름으로 찾아, 찾은 것마다 한 줄 `<name> <newest version> - <description>` 을 출력합니다 (항목에 description 이 없으면 그 부분은 생략). index 조회는 `-` / `_` 어느 쪽 철자든 받습니다.
- `publish` 는 기본이 dry run 입니다. `publish: false` 인 패키지는 지울 줄을 알려 주는 에러로 거부합니다. readme (9.10.3) 는 있어야 합니다. 깨끗한 work tree, 저장소 루트의 패키지, 그리고 `src/lib.rok`, `src/main.rok`, 또는 둘 다를 요구합니다. `src/main.rok` 만 있는 stone 은 프로그램입니다: `rok install` 이 그것을 빌드하며, 의존 패키지가 될 수 없습니다. path / git 의 normal 의존은 거부하고, path / git 의 dev 의존은 뺍니다. 같은 버전과, `-` / `_` 와 대소문자를 같게 본 이름의 충돌을 검사합니다. 다른 철자가 이미 가진 이름은 거부합니다. `--index` 없는 dry run 은 항목만 출력하고, `--index` 가 있으면 그 checkout 에 대해서도 검사합니다. 항목의 source 는 설정된 그대로의 (`insteadOf` 재작성 전) git remote `origin` url 이며, `https://` 여야 합니다. `http://127.0.0.1` 과 `http://localhost` 만 예외입니다.
- `--commit` 이면 `publish` 는 항목을 더합니다: `--index` 가 있으면 그 checkout 에 "Publish <name> <vers>" 커밋으로 (push 하지 않음), 없으면 설정된 index ("Which index") 의 registry API 로 더하며, API 가 그것을 index 에 넣습니다. 클라이언트는 index 저장소에 직접 쓰지 않습니다.
- `yank` 는 그 버전의 `yanked` 를 켭니다 (`--undo` 면 끕니다): `--index` 가 있으면 그 checkout 의 커밋으로, 없으면 registry API 로.
- registry API 는 index 의 `config.rokon` 의 `api` 에 있습니다. 요청은 `Authorization: Bearer <token>` 을 실으며, body 는 한 줄 ROKON (`application/rokon; charset=utf-8`) 입니다:
  - `GET <api>/api/v1/me` → `Me { login: "..", publish: <bool> }`
  - `POST <api>/api/v1/publish`, body 는 index 줄 (`Stone { .. }` / `Native { .. }`) → `Published { name, vers, file }`
  - `POST <api>/api/v1/yank`, body `Yank { name, vers, undo: <bool>, native: <bool> }` → `Yanked { name, vers, yanked }`

  실패는 4xx / 5xx 상태와 `Error { message: ".." }` 이며, `rok` 는 그 message 를 보고합니다. 401 은 토큰이 유효하지 않다는 뜻입니다.
- registry API 에는 토큰이 필요하며, `--index` checkout 에는 필요 없습니다. 토큰은 `ROK_REGISTRY_TOKEN` 이 설정되어 비어 있지 않으면 그것, 아니면 `rok login` 이 저장한 것입니다. 둘 다 없으면 "run `rok login` first" 에러입니다.
- `rok login` 은 웹사이트에서 만든 rokstones.io 계정 토큰을 받습니다 (`<api>/me`, GitHub 로 로그인. 지금은 registry 의 maintainer 만 만들 수 있음). 생략하면 표준 입력에서 화면에 보이지 않게 읽습니다 (Ctrl-C 에도 echo 를 되돌림). `login` 은 `/api/v1/me` 에 물어, API 가 거부하는 토큰과 `publish` 가 false 인 계정 ("this account cannot publish yet: publishing is open to rokstones.io maintainers only") 은 아무것도 저장하지 않고 거절합니다. 그 밖이면 `$ROK_HOME/credentials.rokon` 에 `Credentials { token: ".." }` 를 저장하고 (Unix 에서는 mode 0600) "Logged in as <login>" 을 출력합니다. `rok logout` 은 그 파일을 지웁니다.
- 토큰은 `curl` 의 표준 입력으로만 전달되며, 명령줄이나 환경에 들어가지 않고 출력되지도 않습니다 (에러는 `***` 로 보임).

## 9.10.4b. Native 패키지

Rok 은 C 를 컴파일하지 않습니다. native 패키지는 target triple 마다의 미리 빌드된 archive 와 다음 값입니다:

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

이 값은 index 한 줄이거나, `Dep::native(url:, sha256:)` 의 파일이며 파일의 artifact url 은 그 파일 기준 상대 경로일 수 있습니다.

- archive (tar 로 읽을 수 있는 어떤 형식이든) 는 `lib/` 를 담습니다: `lib<n>.a`, `lib<n>.dylib` / `lib<n>.so`, Windows `<n>.lib` (static 또는 import library). `runtime` 파일과 라이선스 파일도 담습니다. 그 안의 어떤 것도 실행하지 않습니다.
- 해시는 필수입니다. archive 는 풀기 전에 검사하며, 절대 경로나 `..` 경로, 밖으로 나가는 링크가 있거나, 이름 붙은 라이브러리나 runtime 파일이 없으면 거부합니다.
- Apple 대상에서 runtime dylib 은 `@rpath/` install name 을 가져야 합니다.
- 빌드는 target triple 의 artifact 를 씁니다. 없으면 가진 triple 들을 나열하는 에러입니다.
- `rok` 는 `rokc` 에 `-L <archive>/lib`, link 마다 `-l kind=name`, runtime 파일마다 `--native-runtime` 을 넘기며, 의존하는 쪽이 의존되는 쪽보다 앞입니다.

```
rok add --native <name>[@<req>]
rok remove --native <name>
rok search --native <query>
rok publish --native <file> [--index <dir>] [--commit]
rok yank --native --vers <version> [--undo] [--index <dir>] <name>
```

`publish --native` 는 모든 target 을 받아 검사하며, 절대 url, 라이선스 파일, index 에서 온 의존성만을 요구합니다. `--commit` 과 `yank --native` 는 stone 과 같은 규칙을 따릅니다: `--index` checkout, 없으면 토큰으로 registry API (`Yank` 의 `native: true`).

- ELF 대상에서 runtime 라이브러리의 soname 은 파일 이름과 같아야 합니다. `-musl` 대상의 `dylib` link 는 에러입니다.
- `repl` / `jupyter` 는 native 패키지를 로드합니다: 공유 라이브러리는 시스템 loader 로, static archive 는 세션에 링크합니다. `check` / `lint` / `doc` 는 아무것도 링크하지 않습니다.
- 실행 파일 옆의 사본은 빌드 산출물입니다.

## 9.10.4c. Features

패키지는 `features` 에 feature 를 선언합니다. feature 이름에서 그 feature 가 켜는 것들의 리스트로 가는 dict 입니다. 이름은 `[a-z0-9][a-z0-9_-]*` 이며, 무엇인지 말해 주는 단어입니다 (`http1`, `http2`, `tls`). 각 항목은 다음 중 하나입니다.

| 항목        | 켜는 것                                                              |
|-------------|----------------------------------------------------------------------|
| `<f>`       | 이 패키지의 feature `f`                                              |
| `dep:<d>`   | optional 의존 `d`                                                    |
| `<d>/<f>`   | 의존 `d` 의 feature `f`, 그리고 `d` 가 optional 이면 `d` 자체        |
| `<d>?/<f>`  | optional 의존 `d` 의 feature `f`. 다른 무언가가 `d` 를 켤 때만       |

- `default` 는 끄지 않는 한 켜져 있는 보통의 feature 입니다. 빌드하는 패키지는 `--no-default-features` 로, 의존에서는 `default_features: false` 로 끕니다.
- optional 의존 (`optional: true`) 은 켜진 feature 의 `dep:<d>` 나 `<d>/<f>` 항목이 켤 때만 빌드에 들어갑니다. 그 이름의 묵시 feature 는 없으며, 켜는 항목이 하나도 없으면 에러입니다.
- 항목이 패키지에 없는 feature 나 (`deps` 의) 의존을 가리키거나, `dep:` 나 `?` 가 optional 이 아닌 의존을 가리키거나, feature 가 순환하면 에러입니다. 진단은 `file:line:col` 을 보여 줍니다. 의존 패키지에 없는 feature 를 요구해도 에러입니다.
- 해소: lock 은 그래프 전체를 덮습니다. 빌드는 루트 패키지의 feature (끄지 않으면 `default`, `--features`, `--all-features` 이면 모든 feature) 에서 시작해, 켜진 의존마다 그 의존에 요구한 feature 와 그 의존의 `default` 를 가지고 따라갑니다. feature 는 더해지기만 하고 합쳐집니다: 패키지마다 그래프 전체에서 feature 집합은 하나입니다. 루트의 `dev_deps` 와 그것들이 요구하는 feature 는 `test` / `bench` 에서만 셉니다. 결과는 어떤 것의 순서에도 의존하지 않습니다.
- `--features` 는 루트의 feature 이름과, 직접 의존에 대한 `<dep>/<f>` 를 받습니다.
- 각 패키지는 켜진 feature 에 대해서만 정확히 `cfg(feature = "<f>")` 로 컴파일됩니다 ([6.3.6](../006-meta-system/603-cfg.md#636-매개변수의-출처)). native 패키지에는 feature 가 없습니다.
- 스크립트의 ```` ```rokon ```` 블록에는 `features` 가 없습니다 (9.10.7).

## 9.10.5. 환경

환경은 conda / venv 를 대신합니다. 활성화 단계는 없습니다.

- 패키지 (`src/main.rok`, `src/lib.rok`, `rok.rokon` 중 하나가 있는 디렉토리) 는 자기 `rok.rokon` 의 의존성만 씁니다. 환경에 설치한 라이브러리는 결코 보지 않습니다. 스크립트도 마찬가지입니다. 패키지에 작용하는 명령을 `--env` 와 함께 쓰면 에러입니다.
- 이름 있는 환경은 `$ROK_HOME/envs/<name>/` 에 `rok.rokon`, `rok.lock`, `target/` 을 둡니다. 그 `deps` 는 REPL 과 Jupyter 를 위해 `rok install` ([9.10.5a](#9105a-rok-install--rok-uninstall)) 로 설치한 라이브러리이며, 그것을 바꾸는 것은 `rok install` / `rok uninstall` 과 `rok env import` 뿐입니다. 그 안의 path 의존은 절대 경로입니다. 설치한 라이브러리는 그 `target/` 에 미리 빌드합니다.
- 패키지 밖 (REPL, Jupyter, language server, `rok install`) 에서 쓸 환경은 `--env <name>` > `ROK_ENV` > `rok env use` (`$ROK_HOME/envs/CURRENT`) > `default` 순으로 정합니다. `default` 는 거기로 처음 `rok install` 할 때 만들어집니다. 패키지 안에서 경로 없는 `rok repl` / `jupyter` / `lsp` 는 그 패키지를 쓰며, `--env` 가 있으면 그 환경을 씁니다.

### 의존성의 출처

| 컴파일 대상 | 의존성 | Lock | 미리 빌드하는 곳 |
|---|---|---|---|
| 패키지 | 그 `rok.rokon` 의 `deps` (test / bench 는 `dev_deps`) | `<package>/rok.lock` | `<package>/target/<profile>/deps/` |
| 스크립트 | 그것과 그 `mod` 가 읽는 파일들의 `//! rokon` 블록을 합친 `deps` | `cache/scripts/<hash>/rok.lock` | `cache/scripts/<hash>/target/` |
| 패키지 밖의 REPL / Jupyter | 활성 환경에 설치한 라이브러리 | 환경의 것 | 환경의 `target/` |

```
rok env create <name>
rok env list                      # 현재 환경에 * 와 (active) 표시, 없으면 만드는 법을 알려 줌
rok env remove <name>
rok env use <name>                # 프로젝트 밖의 기본 환경을 바꿈
rok env export <name> <dir>       # rok.rokon / rok.lock 을 <dir> 에 씀
rok env import <name> <dir | rok.rokon>    # 프로젝트나 export 에서 만듦 (상대 path 는 절대로)
rok env shell [name]              # ROK_ENV=<name> 인 $SHELL
rok repl --env <name>
```

REPL 의 `:add <dep> ..` 는 현재 환경으로 `rok install <dep> ..` 을, 패키지의 세션에서는 `rok add` 를 실행합니다. 그 뒤 의존성을 다시 읽어 세션에 붙입니다.

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

- 소스의 패키지는 먼저 캐시로 가져옵니다 ([9.10.4](#9104-의존성-rok-add--remove--update--deps)). `src/main.rok` 가 있는 패키지나 스크립트 파일은 실행 파일입니다: `cargo install` 처럼 `--release` 로 빌드해 그 실행 파일을 `$ROK_HOME/bin/<name>` 에 둡니다. git 이나 stone 패키지는 checkout 의 임시 사본에서, path 패키지와 스크립트 파일은 제자리에서 빌드합니다. `$ROK_HOME/installs.rokon` (`Package { deps: { "<name>": <source> } }`) 이 각 실행 파일과 그 정확한 출처 (stone 은 `=<version>`, git 패키지는 그 `rev:` 와 함께) 를 기록합니다. 패키지에 `src/lib.rok` 도 있으면 `rok install` 은 그것을 알리고, `--lib` 가 대신 라이브러리를 설치한다고 알립니다.
- `src/lib.rok` 만 있는 패키지, 또는 `--lib` 를 준 `src/lib.rok` 가 있는 패키지는 라이브러리이며, `pip install` 처럼 `rok repl`, `rok jupyter`, `:add` 를 위해 활성 환경 ([9.10.5](#9105-환경)) 에 들어갑니다: 환경의 `rok.rokon` 에 적고 (요구 없는 stone 은 `^<가장 새 버전>`, path 는 절대 경로), 그 `rok.lock` 으로 해소하며, 그 `target/` 에 미리 빌드합니다. 스크립트는 그것을 보지 않습니다: 스크립트는 `rok add <name> <file.rok>` 로 라이브러리를 얻으며, 출력 줄이 이것을 알립니다. 빌드되지 않으면 아무것도 설치하지 않습니다. `src/lib.rok` 가 없는 패키지나 스크립트에 `--lib` 를 주면 에러입니다.
- `<name>` 은 라이브러리의 최상위 이름, 또는 실행 파일의 파일 이름입니다. 생략하면 패키지의 `name`, 없으면 디렉토리 이름 (path), `.git` 을 뗀 저장소 이름 (git), 파일 stem (스크립트) 입니다. [9.10.3](#9103-rokrokon--roklock) 의 이름 규칙을 따릅니다.
- `--features` / `--no-default-features` 는 라이브러리의 `Dep` 에, 또는 실행 파일의 빌드에 들어가며, `--dev` / `--optional` 은 에러입니다. `--native` 는 환경의 `native` 에 native 패키지를 더합니다 ([9.10.4b](#9104b-native-패키지)).
- proxy `rok`, `rokc`, `rokls` ([9.10.6](#9106-toolchain)) 는 결코 교체하지 않으며, `rok install` 이 두지 않은 `$ROK_HOME/bin` 의 파일도 마찬가지입니다 (에러). 실행 파일을 다시 설치할 때: 같은 stone 버전이나 git commit 에 같은 feature 인 것은 다시 빌드하지 않고 유지하며 (`` `tool` 0.1.0 is installed already (<path>); to rebuild it, run `rok install tool --force` ``), `--force` 가 다시 빌드합니다 (`Reinstalled`). 다른 버전은 그것을 교체하며 (`Upgraded tool from 0.1.0 to 0.2.0`, `Downgraded ...`), 다른 git commit 도 마찬가지입니다 (`Replaced <url> at <commit> with ...`). path 나 스크립트 파일에서 온 것은 `cargo install --path` 처럼 언제나 다시 빌드합니다. 라이브러리 설치는 그대로입니다.
- `--list` 는 ``libraries (environment `<env>`):`` 와 `executables (<bin dir>):` 를 출력하며, 각각 뒤에 설치 순서대로 `  <name> <source>` 줄이 옵니다.
- `rok uninstall <name>..` 은 각 이름을 활성 환경의 라이브러리에서 지우거나 (lock 을 다시 해소), 아니면 `rok install` 이 `$ROK_HOME/bin` 에 둔 실행 파일을 지웁니다. `--native` 는 환경에서 native 패키지를 지웁니다. 둘 다 아닌 이름은 에러입니다.
- `rok install` 말고 `$ROK_HOME` 아래에 쓰는 것은 `rok toolchain` (toolchain, proxy, `config.rokon`), `rok login` (`credentials.rokon`), `rok env` (환경), 그리고 캐시뿐입니다.

## 9.10.6. toolchain

`$ROK_HOME/bin/` 의 `rok` / `rokc` / `rokls` 는 활성 toolchain 의 같은 실행 파일을 실행하는 proxy 입니다 (rustup 식). 활성 toolchain 은 다음 순서로 정합니다.

1. 명령줄의 `+<version>` (`rok +0.4 build`)
2. 환경 변수 `ROK_TOOLCHAIN`
3. 매니페스트의 `toolchain: "X"` (가장 가까운 `rok.rokon`. proxy 는 이 필드만 읽으므로, 모르는 필드가 있어도 고정은 유지됨)
4. 기본 (`$ROK_HOME/config.rokon` 의 `Config { registry, toolchain }`)

버전 접두는 그 아래 버전과 맞습니다 (`0.3` 은 `0.3.*` 의 가장 높은 것). 맞는 toolchain 이 없으면 `rok toolchain install` 을 안내하는 에러입니다.

```
rok toolchain list                         # 기본 toolchain 에 * 와 (default) 표시
rok toolchain install <version> --from <dir | archive | url>
rok toolchain default [<version>]          # 인자 없으면 지금의 기본을 출력
rok toolchain remove <version>
rok toolchain update [<version>]
rok toolchain clean                        # 기본값을 제외한 설치된 모든 toolchain 제거
```

- 설치본은 `$ROK_HOME/toolchains/<version>-<triple>/` 이며 자기 완결입니다 ([9.2.19](902-aot-pipeline.md#9219-설치-레이아웃)). `install` 은 `--from` 이 필수입니다.
- `install` 은 `bin/rokc --version --verbose` 로 검증하며, 버전과 host 가 맞아야 합니다.
- 처음 설치한 toolchain 이 기본이 되고, 이때 `bin/` proxy 를 만듭니다.
- `rok toolchain update` 는 `$ROK_HOME` 아래의 기본 toolchain 을 교체합니다. 인자가 없으면 최신 릴리스를, 버전을 주면 그 버전을 설치합니다. 설치한 toolchain 이 기본값이 되고 `bin/` proxy 를 다시 씁니다. Windows 에서는 실행 중인 proxy 의 이름을 바꾼 뒤 그 사본으로 교체합니다. 릴리스 archive 를 받아 SHA-256 을 검증하고 (아래 배포), 따로 풀어 제자리로 옮긴 뒤 proxy 가 그것을 실행하는지 확인합니다. 실패하면 이전 기본값과 proxy 를 되돌리고 새 toolchain 을 지웁니다. 이전 toolchain 들은 설치된 채로 남습니다. 이미 설치된 버전은 다운로드 없이 기본값이 됩니다. `rok update` 는 `rok.lock` 을 다시 해소하고, `rok toolchain update` 는 toolchain 을 교체합니다 (9.10.4).
- `rok toolchain default <version>` 이나 `rok toolchain update` 뒤에는 설치 스크립트가 쓴 자동완성 스크립트 (아래 배포) 를 새 기본 toolchain 의 `rok completions` / `rokc completions` 로 다시 씁니다. 있는 파일만 다시 쓰고 새로 만들지는 않으며, `completions` 가 없는 toolchain 이면 그대로 둡니다. 아무것도 출력하지 않고, `-v` 이면 다시 쓴 파일을 알려 줍니다.
- `rok toolchain default <version>` 이나 `rok toolchain update` 뒤에 `rok` 은 `PATH` 에서 `rok` 을 찾습니다 (Windows 에서는 `PATHEXT` 의 각 확장자로). 처음 찾은 것이 `$ROK_HOME/bin/rok` 이 아니면 (실제 경로로 비교) 그 파일과 그 `--version` 이 출력하는 버전으로 경고합니다: `warning: your shell runs <path> (rok X) before ~/.rok/bin/rok; put ~/.rok/bin first on PATH or remove the other one`. `-q` 이면 경고하지 않습니다 (설치 스크립트가 그렇게 실행함).
- `rok --version -v` (또는 `rok version -v`) 는 버전 줄 다음에 `binary` (이 실행 파일), `toolchain` (활성 toolchain 과 그 이유: 명령줄의 `+`, `ROK_TOOLCHAIN`, 고정한 `rok.rokon`, 또는 기본값. 없으면 `none`), `rokc` (실행할 `rokc`), `host` 를 출력합니다.
- `rok toolchain clean` 은 지금의 기본값을 제외한 설치된 모든 toolchain 을 지웁니다. `update` 가 남겨 둔 이전 버전을 정리하는 선택적(opt-in) 명령입니다. 프로젝트가 고정한 버전도 함께 지워지므로 빌드에 필요하면 다시 설치해야 하며, 기본 toolchain 이 없으면 에러입니다.

### 배포

- 릴리스 archive 는 `rok-<version>-<triple>.tar.gz` (Windows 는 `.zip`) 이며, 안은 toolchain 레이아웃입니다. Windows 가 아니면 같은 파일을 담은 `rok-<version>-<triple>.tar.xz` 도 옆에 있습니다. 각 archive 옆에 `<archive>.sha256` 이 있습니다: archive 의 SHA-256 을 hex 로 적고, 그 뒤에 파일 이름이 올 수 있습니다 (sha256sum 형식).
- `install.sh`, `install.ps1`, `rok toolchain install --from <url>` 은 같은 곳에서 `<url>.sha256` 을 받아, SHA-256 이 다른 archive 를 거부합니다. `.sha256` 이 없으면 에러입니다. 경로로 준 archive 는 `<path>.sha256` 이 있을 때 그것과 대조합니다. 서명은 정의하지 않으며, 출처는 HTTPS 가 보증합니다.
- `install.sh` (<https://rok-lang.org/install.sh>) 는 archive 나 `rok-lang` 조직의 GitHub release 에서 `$ROK_HOME` 에 설치합니다. tar 가 읽을 수 있고 (macOS, 또는 `PATH` 에 `xz`) 릴리스에 있으면 `.tar.xz` 를, 아니면 `.tar.gz` 를 받습니다. `rok toolchain install` 은 `.tar.gz` (Windows 에서는 `.zip`) 를 받습니다.
- 이어서 `install.sh` 는 rustup 처럼 셸을 설정합니다. `$ROK_HOME/bin` 이 `PATH` 에 아직 없으면 맨 앞에 두는 `$ROK_HOME/env` (sh) 와 `$ROK_HOME/env.fish` 를 쓰고, 가드한 줄 `[ -f "$HOME/.rok/env" ] && . "$HOME/.rok/env"` (그 `$ROK_HOME` 의 것) 을 `~/.profile`, `~/.bashrc`, `~/.bash_profile`, `~/.zshenv` 중 있는 파일마다, 그리고 로그인 셸의 파일 (없으면 만듦; bash: `~/.bashrc`, 그리고 `~/.bash_profile` 이나 `~/.bash_login` 이 없으면 `~/.profile`; zsh: `$ZDOTDIR/.zshenv`; 그 밖의 셸: `~/.profile`) 에 한 번 더합니다. fish 가 로그인 셸이거나 그 설정 디렉토리가 있으면 `~/.config/fish/conf.d/rok.fish` (`$XDG_CONFIG_HOME` 이 있으면 그 아래) 를 씁니다. `rok` 과 `rokc` 의 자동완성 스크립트는 `${XDG_DATA_HOME:-~/.local/share}/bash-completion/completions/`, `$ROK_HOME/zsh/` (`env` 가 zsh 의 `fpath` 에 더함), fish 는 `~/.config/fish/completions/` 에 둡니다. 바꾼 파일을 알려 주며, 새 셸이 그 변경을 읽습니다. 다시 실행해도 바뀌는 것은 없습니다. `$ROK_HOME` 이 없으면 가드한 줄은 아무 일도 하지 않습니다.
- `install.ps1` 은 `$ROK_HOME\bin` 을 사용자 `Path` 맨 앞에 두고, `$ROK_HOME\completions\rok.ps1` 을 써서 PowerShell 프로필의 가드한 줄 (한 번 더함) 로 읽게 합니다.
- `install.sh` 와 `install.ps1` 은 `rok` 처럼 stderr 에 상태 줄을 출력하며, 같은 팔레트와 규칙 ([9.1.5](901-rokc-cli.md#915-전역-옵션), [9.1.8](901-rokc-cli.md#색)) 을 `tput` 없이 따릅니다. `install.sh` 는 stderr 를 보므로 터미널에서 `curl .. | sh` 도 색이 납니다.
- `--no-modify-path` (`install.ps1 -NoModifyPath`) 나 `ROK_NO_MODIFY_PATH=1` 은 셸 파일, `Path`, 프로필을 건드리지 않고 `PATH` 에 더할 것을 출력합니다.
- 릴리스는 `$ROK_DIST` (기본값 `https://github.com/rok-lang/rok/releases`) 에서 제공됩니다: `<dist>/download/v<version>/<archive>` 와 그 `.sha256`, 그리고 최신 릴리스의 버전을 담은 `<dist>/latest/download/rok-version`.

## 9.10.7. 단일 파일 스크립트의 inline 매니페스트

익명 패키지 ([4.16.8](../004-functions/416-entry-point.md#4168-단일-파일-실행)) 는 의존성을 그 파일과 그 `mod` 가 읽는 각 파일의 맨 앞 (shebang 줄 뒤) `//!` 줄 안에 `Package { deps: { .. } }` (와 `native`) 를 담은 ```` ```rokon ```` 블록에서만 얻습니다 (uv 의 inline script metadata 와 같음). 환경에 설치한 라이브러리는 결코 쓰지 않습니다. 블록에는 `deps` 와 `native` 만 둘 수 있습니다 (`features` 가 없으므로 optional 의존도 없음). 다른 언어의 코드 블록은 무시합니다. 블록 안의 path 는 그 블록을 담은 파일 기준입니다. 블록들은 합쳐지며, 두 파일이 한 라이브러리에 서로 다른 항목을 주면 두 파일과 두 항목을 알리는 에러입니다. 서브모듈 파일을 직접 실행하면 그 실행이 닿는 파일들의 블록만 씁니다. `rok add <name> .. <file.rok>` / `rok remove <name> <file.rok>` 이 블록을 쓰며 ([9.10.4](#9104-의존성-rok-add--remove--update--deps)), `rok init --script` 는 스크립트를 패키지로 바꿉니다 ([9.10.2](#9102-rok-new--rok-init-패키지-생성)). 스크립트는 그 lockfile 과 빌드를 `cache/scripts/<hash>/` 에 두며, key 는 그 경로와 그것이 닿는 모든 파일의 경로와 블록입니다. 스크립트의 실행 파일은 그것이 포함하는 모든 파일과 그 라이브러리를 포함해 빌드를 이루는 모든 것으로 캐시합니다. 스크립트에 없는 패키지의 `use x::..` 는 실행할 명령 ``run `rok add x <file>` `` 을 help 로 주는 에러이며, 활성 환경에 `x` 가 있으면 note 를 덧붙입니다. 패키지 안에서는 `rok add x` 를 제안합니다. `import x` (와 `include` / `require`) 는 에러이며, help 는 파일 옆에 `x.rok` 가 있으면 `mod x;`, `x` 가 경로 루트이거나 파일이 선언한 모듈이면 `use`, 그 밖에는 `rok add x <file>` 입니다.

````rok
//! ```rokon
//! Package {
//!     deps: { "util": Dep::git("https://example.com/util.git", tag: "v1") },
//! }
//! ```

use util::greet;
greet();
````

단일 파일 스크립트의 실행은 그 실행 파일을 재사용합니다. 익명 패키지의 `rokc run` 은 실행 파일을 `<target>/<profile>/scripts/<key>/` 에 두며, `<target>` 은 스크립트의 `target/` (`cache/scripts/<hash>/` 아래, 그 `rok.lock` 옆) 입니다. `<key>` 는 빌드를 이루는 모든 것의 해시입니다: toolchain, 빌드 옵션, lint 수준과 target, 소스와 그것이 포함하는 모든 파일, toolchain 이 그 target 용으로 배포하는 런타임과 C 라이브러리, 그리고 미리 빌드한 라이브러리와 함께 넘긴 의존 패키지. key 가 맞으면 아무것도 컴파일하지 않으며, 그 실행 파일을 만든 빌드의 경고를 다시 출력합니다. 새 실행 파일은 임시 이름으로 쓴 뒤 제자리로 이름을 바꾸므로, 한 스크립트를 동시에 실행해도 안전합니다. 하루 동안 쓰지 않은 항목은 다른 항목을 빌드할 때 지웁니다.
