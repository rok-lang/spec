# 9.10. `rok` — 패키지 / 빌드 front-end

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
| `build`, `check`, `run`, `test`, `bench`, `emit`                  | 의존성을 붙여 `rokc` 에 위임            |
| `doc`, `lint`, `repl [path]`, `lsp`                               | 의존성을 붙여 `rokc` 에 위임            |
| `fmt`, `clean`, `demangle`, `profdata`                            | 의존성 없이 `rokc` 에 위임              |
| `jupyter`                                                         | 커널 실행 (`--connection-file`) 은 위임, `jupyter install [--env <name>]` 은 kernelspec 설치 ([9.8.3](908-jupyter-integration.md#983-kernelspec-설치)) |
| `version`, `help`                                                 |                                         |
| (없음)                                                            | `rok repl` 과 같음 — REPL 시작          |

위임하는 서브명령은 같은 이름의 `rokc` 서브명령을 실행합니다. `rok` 는 자기 전역 옵션 (`--env` / `--offline` / `--locked` / `--cwd`, `--` 앞 어디든) 만 소비하고, 나머지 인자는 그대로 넘기며 다음을 더합니다.

- 의존 패키지마다 `--extern name=<소스 디렉토리>`. `rok.lock` 을 따라 준비하며, 의존되는 쪽이 의존하는 쪽보다 앞입니다 (`rok deps` 가 출력하는 순서). 미리 빌드는 `rokc` 가 합니다 ([9.2.3](902-aot-pipeline.md#미리-빌드한-의존-패키지)).
- 매니페스트의 `lints` 를 `-A` / `-W` / `-D` 로 (인자 앞에). 매니페스트는 경로가 고르는 프로젝트의 것입니다: `.rok` 파일이면 그 파일 위치나 그 위에서 `main.rok`, `lib.rok`, `rok.rokon` 중 하나가 있는 가장 가까운 디렉토리. `main.rok` / `lib.rok` 패키지 안의 파일은 그 패키지를 고릅니다 ([9.1.1](901-rokc-cli.md#911-일반-형태)). `rok.rokon` 만 있는 디렉토리의 단독 파일은 그 매니페스트의 `lints` (와 `lints_from`) 와 의존성을 가진 단일 파일 패키지로 컴파일합니다.
- 패키지 밖 (환경, 스크립트) 이면 `--target-dir` 에 그 환경이나 스크립트의 `target/`.

`rok repl <path>` 는 `run` 처럼 그 경로의 프로젝트에서 의존성과 lint 를 정하고, 매니페스트 이름을 넘깁니다 (`--name`).

옵션과 출력, 종료 코드는 `rokc` 의 것 ([9.1](901-rokc-cli.md)) 을 따릅니다. `rokls` 는 `rok lsp` 와 같은 독립 실행 파일입니다.

서브명령 없이 `rok [옵션]` 만 실행하면 `rok repl` 과 같이 REPL 을 시작합니다.

```
rok                 # = rok repl
rok --env sci       # = rok repl --env sci
```

`rok` 의 전역 옵션은 다음과 같습니다.

| 옵션             | 의미                                                              |
|------------------|-------------------------------------------------------------------|
| `--env <name>`   | 프로젝트의 환경 대신 그 이름 있는 환경을 씀 (9.10.5)              |
| `--offline`      | 네트워크를 쓰지 않음. git 의존은 캐시에 있어야 함                 |
| `--locked`       | `rok.lock` 을 바꿔야 하면 바꾸지 않고 에러 (CI 용)                |
| `--cwd <dir>`    | 디렉토리를 바꾼 뒤 실행                                           |
| `-q`, `--quiet`  | 출력을 줄임                                                       |

환경 변수는 `ROK_HOME` (기본 `~/.rok`), `ROK_ENV` (9.10.5), `ROK_TOOLCHAIN` (9.10.6), `ROK_REGISTRY` 와 `ROK_REGISTRY_TOKEN` (9.10.4a) 입니다.

## 9.10.2. `rok new` / `rok init`: 패키지 생성

```
rok new [--lib | --bin] <name>
rok init [--lib | --bin] [dir]
```

`rok new` 는 `<name>` 디렉토리를 만들고 기본 파일을 채웁니다. `--bin` (기본값) 은 인사말을 출력하는 `main.rok` 를, `--lib` 는 예시 함수와 `#[test]` 하나를 담은 `lib.rok` 를 만듭니다. `<name>` 이 이미 있으면 에러입니다.

`rok init` 은 이미 있는 디렉토리 (생략하면 현재 디렉토리) 를 패키지로 만듭니다. 그 디렉토리에 `main.rok` 나 `lib.rok` 가 이미 있으면 에러입니다.

두 명령은 정규 형식 ([ROKON 3.9.4](../../refs-standard-libraries/300-std/309-rokon.md#394-정규-형식)) 의 `rok.rokon` 과 `target/` 을 담은 `.gitignore` 도 쓰며, git 저장소 안이 아니면 `git init` 을 합니다. `init` 은 디렉토리 이름을 이름 규칙 (9.10.3) 에 맞게 바꿔 패키지 이름으로 씁니다.

## 9.10.3. `rok.rokon` / `rok.lock`

패키지 루트의 `rok.rokon` 이 매니페스트이고, 같은 자리의 `rok.lock` 이 lockfile 입니다. 둘 다 [ROKON](../../refs-standard-libraries/300-std/309-rokon.md) 형식입니다. 이름은 이 철자 그대로 (모두 소문자) 입니다. 대소문자만 다른 파일 (`Rok.rokon` 등) 은 경고와 함께 무시하며, 읽지 않습니다.

매니페스트는 ROKON 값 하나, 즉 `Package { .. }` struct 입니다 (루트 이름 `Package` 는 생략 가능). 읽기만 하며 컴파일하거나 실행하지 않습니다.

```rokon
Package {
    name: "geom",                    // 문자열, 선택
    version: "0.3.1",                // 문자열, 선택
    toolchain: "0.4",                // 문자열, 선택 — toolchain 고정 (9.10.6)
    description: "2D geometry",      // 문자열, 선택
    license: "MIT OR Apache-2.0",    // 문자열, 선택 (SPDX 식)
    repository: "https://example.com/geom.git",   // 문자열, 선택
    deps: {                          // 패키지 이름 (문자열) -> Dep
        "util": Dep::path("../util"),
        "parser": Dep::git("https://example.com/parser.git", tag: "v1.2"),
        "json": "1.4",                                   // stone (= Dep::version("1.4"))
        "csv": Dep::version("^0.3", registry: "https://example.com/index"),
    },
    dev_deps: {                      // deps 와 같은 형태. test / bench 에서만
        "fixtures": Dep::path("tests/fixtures"),
    },
    native: { "zlib": "^1.3", "png": Dep::native(url: "vendor/png.rokon", sha256: "<64 hex>") },
    lints_from: "..",                // 문자열, 선택 — ../rok.rokon 의 lints 에서 시작
    lints: { "dead_code": "allow", "nonstandard_style": "deny" },
}
```

- 값은 ROKON 의 리터럴 (문자열, `{ "k": v }` dict, 빈 dict `{}`) 과 `Dep::path("..")` / `Dep::git("url", rev | tag | branch: "..")` / `Dep::version("req", registry: "url")` 입니다. `Dep::git` 의 `rev` / `tag` / `branch` 는 많아야 하나이고, `Dep::version` 의 `registry` 는 선택입니다. 의존성 값이 문자열 하나면 `Dep::version` 의 줄임입니다.
- `Dep::registry` 는 `rok.lock` 전용이며 `rok.rokon` 에 쓰면 에러입니다.
- 패키지 이름 (`deps` 의 키, `name`) 은 `[a-z0-9_-]+` 의 소문자 ASCII 이며 글자로 시작합니다 (숫자, `-`, `_` 로 시작하면 에러). 키는 문자열입니다 (`"rok-syntax": Dep::version("^0.1")`).
- 코드에서는 `-` 를 `_` 로 바꾼 이름이 그 패키지를 가리키는 최상위 경로입니다 (`rok-syntax` → `use rok_syntax::..`, `--extern rok_syntax=..`).
- 레지스트리와 한 의존 그래프 안에서 이름은 `-` 와 `_` 를 같게, 대소문자를 구분하지 않고 비교합니다. 그래서 `rok-syntax` 와 `rok_syntax` 는 함께 있을 수 없습니다.
- 의존 패키지는 `lib.rok` 를 가진 라이브러리 패키지여야 합니다. 전이 의존은 의존 패키지의 `deps` 입니다 (`dev_deps` 는 전이되지 않음).
- `native` 는 native 패키지 이름 (자기 이름 공간: 패키지와 native 패키지는 같은 이름을 쓸 수 있음) 을 `Dep::native("req"[, registry: "url"])`, 요구 문자열, 또는 `Dep::native(url: "<url 또는 매니페스트 기준 경로>", sha256: "<64 자리 hex>")` 에 대응시킵니다 (9.10.4b). `deps` / `dev_deps` 의 `Dep::native` 는 에러입니다. 그래프의 모든 패키지의 native 의존이 실행 파일에 링크됩니다 (`dev_deps` 로만 닿는 것은 test / bench 에서만).
- `lints` 는 lint 수준을 정합니다 (lint 나 그룹 이름 → `"allow"` | `"warn"` | `"deny"`, [6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)). `rokc lint` 의 CLI 플래그 앞에 붙은 것처럼 다뤄지므로, CLI `-A/-W/-D` 와 코드의 속성이 이를 덮습니다.
- `lints_from` 은 시작 lint 수준을 주는 `rok.rokon` 이 있는 디렉토리를 가리킵니다 (이 매니페스트의 디렉토리 기준 상대 경로, 또는 절대 경로). 그 매니페스트의 `lints` 를 그 자신의 `lints_from` 사슬 뒤에 먼저 적용하고, 이 매니페스트의 `lints` 가 그 뒤에 오므로, 여기 항목이 사슬의 같은 lint 나 그룹을 덮습니다 (모든 항목은 이 순서로 `-A`/`-W`/`-D` 로 넘어가며 뒤의 플래그가 이깁니다). 빌드하거나 검사하는 패키지 (와 환경의 매니페스트) 에만 적용되며, 의존 패키지의 매니페스트는 lint 에 쓰지 않습니다. 디렉토리가 없거나, `rok.rokon` 이 없거나, 사슬이 이미 지난 매니페스트로 돌아오면 (순환) 에러입니다. `rok env import` 는 상대 `lints_from` 을 절대 경로로 바꿔 씁니다. 스크립트의 `//! ```rokon` 블록에서는 쓸 수 없습니다.
- 에러: 모르는 필드 / `Dep` 종류 / 이름 있는 인자, 중복, 규칙에 맞지 않는 패키지 이름, ROKON 이 아닌 문법, `Package { .. }` 하나가 아닌 것, 뒤에 남은 토큰. 진단은 `file:line:col` 과 caret 으로 위치를 보여 줍니다.

| 의존성 출처 | 형태                                                         | `rok.lock` 에 고정  |
|-------------|--------------------------------------------------------------|---------------------|
| path        | `Dep::path("..")` (매니페스트 기준 상대 경로)                | 없음 (늘 현재 내용) |
| git         | `Dep::git("url")`, 선택으로 `rev:` / `tag:` / `branch:` 중 하나 | 커밋과 tree hash    |
| stone       | `Dep::version("req"[, registry: "url"])` 또는 문자열 `"req"` ([9.10.4a](#9104a-stone-과-rokstonesio)) | 버전, 커밋, tree hash |
| native      | `Dep::native("req")` / `Dep::native(url:, sha256:)` ([9.10.4b](#9104b-native-패키지)) | 버전과 항목 SHA-256 (index), 없음 (해시가 고정) |

rok 에는 build script 가 없습니다: 빌드 시점에 패키지의 코드는 실행되지 않습니다. C / C++ 라이브러리는 미리 빌드된 native 패키지로 들어옵니다 (9.10.4b).

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
- index 의 native 패키지는 선택 필드 `natives: [LockedNative { name, vers, source: Dep::registry("<index>"), sha256 }]` 에 고정하며, 이름 순으로 정렬하고 비어 있지 않을 때만 씁니다. `sha256` 은 `yanked: false` 로 둔 항목의 정규 index 줄의 SHA-256 이며, 항목이 바뀌면 "run `rok update`" 에러입니다.
- `version` 이 `1` 이 아니면 "unsupported lockfile version" 에러입니다.
- lock 을 읽다 난 모든 에러는 "(delete rok.lock and run `rok update` to make it again)" 를 덧붙입니다.

## 9.10.4. 의존성: `rok add` / `remove` / `update` / `deps`

```
rok add <name> --path <dir> [--dev] [--env <env>]
rok add <name> --git <url> [--rev <rev> | --tag <tag> | --branch <branch>] [--dev] [--env <env>]
rok add <name>[@<req>] [--registry <url>] [--dev] [--env <env>]      # stone
rok remove <name>
rok update [name...]
rok deps [--dev]
```

- `add` / `remove` 는 `rok.rokon` 을 고치며, 주석과 배치를 보존합니다. `--dev` 는 `dev_deps` 에 넣고, `--env` 는 그 이름 있는 환경의 `rok.rokon` 을 고칩니다. stone 을 요구 없이 더하면 `^<가장 새 버전>` 을 적습니다.
- 빌드는 `rok.lock` 의 커밋과 버전을 씁니다. 새로 선언된 의존은 lock 에 더하고 지운 의존은 정리하지만, 이미 고정된 것을 옮기는 것은 `update` 뿐입니다. `update` 는 이름을 주면 그 패키지만, 없으면 모든 git 의존과 stone 을 다시 해소합니다.
- `deps` 는 해소한 의존 패키지를 `<코드 이름>=<lib.rok 경로>` 줄로 (코드 이름은 `-` 를 `_` 로 바꾼 것), 의존되는 쪽이 먼저 오는 순서로 출력합니다 (`--dev` 는 dev 의존 포함). native 패키지마다 (의존하는 쪽이 먼저, host 용) `# native <name> <vers>` 와 그 `rokc` 인자를 한 줄에 하나씩 출력합니다.
- `rok.lock` 은 버전 관리에 함께 넣습니다.

### 캐시

가져온 git 소스는 `$ROK_HOME/cache/` 에 두고 모든 프로젝트가 공유합니다.

| 경로                                       | 내용                                                    |
|--------------------------------------------|---------------------------------------------------------|
| `cache/git/db/<repo>-<hash>/`              | 저장소의 bare clone (url 마다 하나)                     |
| `cache/registry/db/<repo>-<hash>/`         | stone index 의 bare clone (index 마다 하나)             |
| `cache/git/checkouts/<commit>/`            | 커밋의 checkout. 쓰기 전에 tree hash 로 검증하고, 다르면 다시 풀어 놓음 |
| `cache/scripts/<hash>/rok.lock`            | 단일 파일 스크립트의 lockfile (9.10.7)                  |
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

index 저장소의 루트에 `config.rokon` (`Registry { name, dl, api }`) 이 있습니다. stone 하나는 publish 된 소문자 이름 (`-` 는 그대로) 의 shard 경로에 있는 파일 하나입니다.

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
- `native/<shard path>` 는 native 패키지를 한 줄에 `Native { .. }` 하나씩 담습니다 ([9.10.4b](#9104b-native-패키지)).
- 매니페스트의 `description`, `license`, `repository` 는 선택인 뒤쪽 필드로 들어갑니다.
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

- `search` 는 index 에서 이름으로 찾습니다. index 조회는 `-` / `_` 어느 쪽 철자든 받습니다.
- `publish` 는 기본이 dry run 입니다. 깨끗한 work tree, 저장소 루트의 패키지, `lib.rok` 를 요구합니다. path / git 의 normal 의존은 거부하고, path / git 의 dev 의존은 뺍니다. 같은 버전과, `-` / `_` 와 대소문자를 같게 본 이름의 충돌을 검사합니다. 다른 철자가 이미 가진 이름은 거부합니다. `--index` 없는 dry run 은 항목만 출력하고, `--index` 가 있으면 그 checkout 에 대해서도 검사합니다.
- `--commit` 이면 `publish` 는 "Publish <name> <vers>" 커밋을 만듭니다: `--index` 가 있으면 그 checkout 에 (push 하지 않음), 없으면 설정된 index ("Which index") 를 새로 clone 한 곳에 만들고 index 로 push 합니다.
- `yank` 는 그 버전의 `yanked` 를 켜고 (`--undo` 면 끄고) 커밋합니다: `--index` 로 준 checkout 에, 없으면 설정된 index 를 새로 clone 한 곳에 커밋하고 push 합니다.
- index 로 push 하려면 토큰이 필요하며, `--index` checkout 에는 필요 없습니다. 토큰은 `ROK_REGISTRY_TOKEN` 이 설정되어 비어 있지 않으면 그것, 아니면 `rok login` 이 저장한 것입니다. 둘 다 없으면 "run `rok login` first" 에러입니다.
- `rok login` 은 GitHub 토큰을 받습니다 (생략하면 표준 입력에서 읽으며 화면에 보이지 않습니다). publish 할 수 있다는 것은 index 저장소에 쓰기 권한이 있다는 뜻입니다: `login` 은 토큰으로 index 의 git receive-pack 에 물어보고 (dry-run push; 아무것도 보내지 않음), push 할 수 없는 토큰은 거부합니다 ("publishing is limited to the index's maintainers"). 그런 다음 `$ROK_HOME/credentials.rokon` 에 `Credentials { token: ".." }` 를 저장합니다 (Unix 에서는 mode 0600). `rok logout` 은 그 파일을 지웁니다.
- 토큰은 git 의 환경으로만 전달되며 (HTTPS `Authorization` 헤더로, credential helper 는 끔), 명령줄, url, clone 의 git 설정에 들어가지 않고 출력되지도 않습니다.

## 9.10.4b. Native 패키지

rok 은 C 를 컴파일하지 않습니다. native 패키지는 target triple 마다의 미리 빌드된 archive 와 다음 값입니다:

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

`publish --native` 는 모든 target 을 받아 검사하며, 절대 url, 라이선스 파일, index 에서 온 의존성만을 요구합니다. `--commit` 과 `yank --native` 는 stone 과 같은 규칙을 따릅니다: `--index` checkout, 없으면 설정된 index 에 토큰으로 push 합니다.

- ELF 대상에서 runtime 라이브러리의 soname 은 파일 이름과 같아야 합니다. `-musl` 대상의 `dylib` link 는 에러입니다.
- `repl` / `jupyter` 는 native 패키지를 로드합니다: 공유 라이브러리는 시스템 loader 로, static archive 는 세션에 링크합니다. `check` / `lint` / `doc` 는 아무것도 링크하지 않습니다.
- 실행 파일 옆의 사본은 빌드 산출물입니다.

## 9.10.5. 환경

uv / conda 식 환경을 둡니다. 활성화 단계는 없습니다.

- `rok.rokon` 이 있는 모든 디렉토리 (프로젝트) 가 자기 환경입니다. 그 안에서의 `rok` 명령은 그 의존성을 씁니다.
- 이름 있는 환경은 `$ROK_HOME/envs/<name>/` 에 `rok.rokon` / `rok.lock` 을 둡니다. 그 안의 path 의존은 절대 경로로 씁니다.
- 프로젝트 밖에서 쓸 환경은 `--env <name>` > 환경 변수 `ROK_ENV` > `rok env use` 가 정한 것 (`$ROK_HOME/envs/CURRENT`) > `default` 순으로 정합니다.

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

REPL 의 `:add <dep> ..` 는 현재 환경에 의존성을 추가합니다 (`rok add` 와 같은 인자). 추가한 뒤 의존성을 다시 읽어 세션에 붙입니다.

## 9.10.6. toolchain

`$ROK_HOME/bin/` 의 `rok` / `rokc` / `rokls` 는 활성 toolchain 의 같은 실행 파일을 실행하는 proxy 입니다 (rustup 식). 활성 toolchain 은 다음 순서로 정합니다.

1. 명령줄의 `+<version>` (`rok +0.4 build`)
2. 환경 변수 `ROK_TOOLCHAIN`
3. 매니페스트의 `toolchain: "X"` (가장 가까운 `rok.rokon`)
4. 기본 (`$ROK_HOME/config.rokon` 의 `Config { registry, toolchain }`)

버전 접두는 그 아래 버전과 맞습니다 (`0.3` 은 `0.3.*` 의 가장 높은 것). 맞는 toolchain 이 없으면 `rok toolchain install` 을 안내하는 에러입니다.

```
rok toolchain list
rok toolchain install <version> --from <dir | archive | url>
rok toolchain default [<version>]          # 인자 없으면 지금의 기본을 출력
rok toolchain remove <version>
```

- 설치본은 `$ROK_HOME/toolchains/<version>-<triple>/` 이며 자기 완결입니다 ([9.2.19](902-aot-pipeline.md#9219-설치-레이아웃)). `install` 은 `--from` 이 필수입니다.
- `install` 은 `bin/rokc --version --verbose` 로 검증하며, 버전과 host 가 맞아야 합니다.
- 처음 설치한 toolchain 이 기본이 되고, 이때 `bin/` proxy 를 만듭니다.

### 배포

- 릴리스 archive 는 `rok-<version>-<triple>.tar.gz` (와 `.sha256`) 이며, 안은 toolchain 레이아웃입니다.
- `install.sh` (<https://rok-lang.org/install.sh>) 는 archive 나 `rok-lang` 조직의 GitHub release 에서 `$ROK_HOME` 에 설치합니다.

## 9.10.7. 단일 파일 스크립트의 inline 매니페스트

익명 패키지 ([4.16.8](../004-functions/416-entry-point.md#4168-단일-파일-실행)) 는 파일 맨 앞의 `//!` 줄 안에 ```` ```rokon ```` 블록으로 `Package { deps: { .. } }` 를 담아 의존성을 선언합니다 (uv 의 inline script metadata 와 같음). 블록에는 `deps` 만 둘 수 있습니다. 다른 언어의 코드 블록은 무시합니다. path 는 스크립트 디렉토리 기준입니다. 스크립트의 lockfile 은 캐시 (`cache/scripts/`) 에 둡니다.

````rok
//! ```rokon
//! Package {
//!     deps: { "util": Dep::git("https://example.com/util.git", tag: "v1") },
//! }
//! ```

use util::greet;
greet();
````
