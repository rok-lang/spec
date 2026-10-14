# 9.10. `rok` — 패키지 / 빌드 front-end

`rok` 는 cargo 식 front-end 입니다. 매니페스트 `Rok.toml` 과 lockfile `Rok.lock`, 의존성, 환경, toolchain 고정, 스크립트 의존성을 맡고, 컴파일은 컴파일러 `rokc` ([9.1](901-rokc-cli.md)) 를 불러 처리합니다. 런타임은 `rokr` (`librokr.a`) 입니다.

의존성이 없는 패키지는 매니페스트 없이 파일 규약만으로 빌드됩니다 ([0.3.4](../000-meta/003-source-units.md#034-매니페스트-roktoml-는-선택)). 이때 `rok build` 는 `rokc build` 와 같습니다.

## 9.10.1. 서브명령과 전역 옵션

| 서브명령                                                          | 동작                                    |
|-------------------------------------------------------------------|-----------------------------------------|
| `new`, `init`                                                     | 패키지 생성 (9.10.2)                    |
| `add`, `remove`, `update`, `deps`                                 | 의존성 (9.10.4)                         |
| `env`                                                             | 환경 (9.10.5)                           |
| `toolchain`                                                       | toolchain (9.10.6)                      |
| `build`, `check`, `run`, `test`, `emit`, `clean`                  | 의존성을 해소한 뒤 `rokc` 에 위임       |
| `fmt`, `repl`, `lsp`, `demangle`                                  | 의존성을 해소한 뒤 `rokc` 에 위임       |
| `jupyter`                                                         | 커널 실행 (`--connection-file`) 은 위임, `jupyter install [--user]` 은 kernelspec 설치 ([9.8.3](908-jupyter-integration.md#983-kernelspec-설치)) |
| `bench`, `doc`, `lint`                                            | 의존성을 해소한 뒤 `rokc` 에 위임       |
| `version`, `help`                                                 |                                         |
| (없음)                                                            | `rok repl` 과 같음 — REPL 시작          |

위임하는 서브명령은 `Rok.lock` 을 따라 의존 패키지를 준비한 뒤, 의존 패키지마다 `--extern name=<lib.rok>` 를 붙여 같은 이름의 `rokc` 서브명령을 실행합니다 ([9.2.3](902-aot-pipeline.md#923-의존성-해소)). 그 밖의 인자는 `rokc` 에 그대로 넘어가며, 옵션과 출력, 종료 코드는 `rokc` 의 것 ([9.1](901-rokc-cli.md)) 을 따릅니다. `rokls` 는 `rok lsp` 와 같은 독립 실행 파일입니다.

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
| `--locked`       | `Rok.lock` 을 바꿔야 하면 바꾸지 않고 에러 (CI 용)                |
| `--cwd <dir>`    | 디렉토리를 바꾼 뒤 실행                                           |
| `-q`, `--quiet`  | 출력을 줄임                                                       |

환경 변수는 `ROK_HOME` (기본: `rok` 가 있는 `bin/` 의 상위) 과 `ROK_ENV` (9.10.5) 입니다.

## 9.10.2. `rok new` / `rok init`: 패키지 생성

```
rok new [--lib | --bin] <name>
rok init [--lib | --bin] [dir]
```

`rok new` 는 `<name>` 디렉토리를 만들고 그 안에 기본 파일을 채워 줍니다. `--bin` (기본값) 은 인사말을 출력하는 `main.rok` 를, `--lib` 는 예시 함수와 `#[test]` 하나를 담은 `lib.rok` 를 만듭니다. `<name>` 이 이미 있으면 에러입니다.

`rok init` 은 이미 있는 디렉토리 (생략하면 현재 디렉토리) 를 패키지로 만듭니다. 그 디렉토리에 `main.rok` 나 `lib.rok` 가 이미 있으면 에러입니다.

두 명령은 패키지 파일을 `rokc new` / `rokc init` ([9.1.4](901-rokc-cli.md#914-보조-서브명령)) 으로 만들고, 거기에 `Rok.toml` 을 더합니다.

## 9.10.3. `Rok.toml` / `Rok.lock`

패키지 루트의 `Rok.toml` (TOML 1.0) 이 매니페스트이고, 같은 자리의 `Rok.lock` 이 lockfile 입니다. 이름은 이 철자 그대로 (첫 글자만 대문자) 입니다. 대소문자만 다른 파일 (`rok.toml` 등) 은 경고와 함께 무시하며, 읽지 않습니다.

```toml
[package]
name = "geom"
version = "0.3.1"
toolchain = "0.4"            # 선택 — toolchain 고정 (9.10.6)

[deps]
util = { path = "../util" }
parser = { git = "https://example.com/parser.git", tag = "v1.2" }

[dev-deps]                   # test / bench 에서만
fixtures = { path = "tests/fixtures" }

[lints]
dead_code = "allow"
nonstandard_style = "deny"
```

- 의존성 이름 (키) 은 식별자여야 합니다 (`-` 불가). 이 이름이 코드에서 그 패키지를 가리키는 최상위 경로입니다 (`use parser::Token;`).
- 의존 패키지는 `lib.rok` 를 가진 라이브러리 패키지여야 합니다. 전이 의존은 의존 패키지의 `[deps]` 입니다 (`[dev-deps]` 는 제외).
- `[lints]` 는 lint 수준을 정합니다 (`name = "allow" | "warn" | "deny"`, name 은 lint 나 그룹, [6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)). `rokc lint` 의 CLI 플래그 앞에 붙은 것처럼 다뤄지므로, CLI `-A/-W/-D` 와 코드의 속성이 이를 덮습니다.
- 모르는 키는 에러입니다.

| 의존성 출처 | 형태                                                     | `Rok.lock` 에 고정 |
|-------------|----------------------------------------------------------|--------------------|
| path        | `{ path = ".." }` (매니페스트 기준 상대 경로)            | 없음 (늘 현재 내용) |
| git         | `{ git = "url" }`, 선택으로 `rev` / `tag` / `branch` 중 하나 | 커밋과 tree hash |

다음은 예약이며, 쓰면 "reserved, not supported yet" 에러입니다.

- 문자열 값의 의존성 (`util = "1.2"`, registry), 의존성 안의 `version` / `registry` 키
- 어디에 있든 `native` (conda 식 C 라이브러리)
- 최상위 `[registry]`

한 빌드 안에서 한 이름은 한 패키지입니다. 같은 이름이 서로 다른 출처 (다른 경로, 또는 다른 url / 커밋) 로 요구되면 두 요구처를 보여 주는 에러입니다.

### `Rok.lock` 형식

`Rok.lock` 은 결정적으로 쓰이는 TOML 입니다. 머리 주석, `version = 1`, 그다음 git 패키지마다 `[[package]]` 하나를 이름 순으로 둡니다. path 의존은 고정하지 않으므로 나오지 않습니다.

```toml
# Rok.lock: generated by `rok`; do not edit. Commit it with your sources.
version = 1

[[package]]
name = "parser"
git = "https://example.com/parser.git"
tag = "v1.2"
commit = "4f0c2a9e1b7d3c58a6e09f21d4b8c7a3e5f60123"
tree = "9a1e7c3b5d2f4a6081c9e3b7d5f2a4c6e8b0d1f3"
```

| 키                      | 뜻                                                          |
|-------------------------|-------------------------------------------------------------|
| `name`                  | `Rok.toml` 의 의존성 이름                                   |
| `git`                   | 저장소 url                                                  |
| `rev` / `tag` / `branch` | `Rok.toml` 에 적은 것이 있을 때만                          |
| `commit`                | 해소된 커밋 (40 자리 16 진수)                               |
| `tree`                  | 그 커밋의 git tree hash. 캐시의 checkout 을 검증하는 데 씀 |

## 9.10.4. 의존성: `rok add` / `remove` / `update` / `deps`

```
rok add <name> --path <dir> [--dev] [--env <env>]
rok add <name> --git <url> [--rev <rev> | --tag <tag> | --branch <branch>] [--dev] [--env <env>]
rok remove <name>
rok update [name...]
rok deps [--dev]
```

- `add` / `remove` 는 `Rok.toml` 을 고치며, 주석과 형식을 보존합니다. `--dev` 는 `[dev-deps]` 에 넣고, `--env` 는 그 이름 있는 환경의 `Rok.toml` 을 고칩니다.
- 빌드는 `Rok.lock` 의 커밋을 씁니다. 새로 선언된 의존은 lock 에 더하고 지운 의존은 정리하지만, 이미 고정된 커밋을 옮기는 것은 `update` 뿐입니다. `update` 는 이름을 주면 그 패키지만, 없으면 모든 git 의존을 다시 해소합니다.
- `deps` 는 해소한 의존 패키지를 `name=<lib.rok 경로>` 줄로 출력합니다 (`--dev` 는 dev 의존 포함). 도구 연동과 REPL 이 씁니다.
- `Rok.lock` 은 버전 관리에 함께 넣습니다.

### 캐시

가져온 git 소스는 `$ROK_HOME/cache/` 에 두고 모든 프로젝트가 공유합니다.

| 경로                                       | 내용                                                    |
|--------------------------------------------|---------------------------------------------------------|
| `cache/git/db/<repo>-<hash>/`              | 저장소의 bare clone (url 마다 하나)                     |
| `cache/git/checkouts/<commit>/`            | 커밋의 checkout. 쓰기 전에 tree hash 로 검증하고, 다르면 다시 풀어 놓음 |
| `cache/scripts/<hash>/Rok.lock`            | 단일 파일 스크립트의 lockfile (9.10.7)                  |

## 9.10.5. 환경

uv / conda 식 환경을 둡니다. 활성화 단계는 없습니다.

- `Rok.toml` 이 있는 모든 디렉토리 (프로젝트) 가 자기 환경입니다. 그 안에서의 `rok` 명령은 그 의존성을 씁니다.
- 이름 있는 환경은 `$ROK_HOME/envs/<name>/` 에 `Rok.toml` / `Rok.lock` 을 둡니다. 그 안의 path 의존은 절대 경로로 씁니다.
- 프로젝트 밖에서 쓸 환경은 `--env <name>` > 환경 변수 `ROK_ENV` > `rok env use` 가 정한 것 (`$ROK_HOME/envs/CURRENT`) > `default` 순으로 정합니다.

```
rok env create <name>
rok env list                      # 현재 환경에 * 표시
rok env remove <name>
rok env use <name>                # 프로젝트 밖의 기본 환경을 바꿈
rok env export <name> <dir>       # Rok.toml / Rok.lock 을 <dir> 에 씀
rok env import <name> <dir | Rok.toml>   # 프로젝트나 export 에서 만듦 (상대 path 는 절대로)
rok env shell [name]              # ROK_ENV=<name> 인 $SHELL
rok repl --env <name>
```

REPL 의 `:add <dep> ..` 는 현재 환경에 의존성을 추가합니다 (`rok add` 와 같은 인자). 추가한 뒤 의존성을 다시 읽어 세션에 붙입니다.

## 9.10.6. toolchain 고정

`[package] toolchain = "X"` 가 지금 `rok` 의 버전과 맞지 않으면, `rok` 는 `$ROK_HOME/toolchains/<ver>/bin/rokc` 를 `ROK_HOME` 을 그 toolchain 디렉토리로 바꿔 실행합니다. 매니페스트와 환경 처리는 지금의 `rok` 가 하며, `rok` 자신은 다시 실행하지 않습니다. 버전 접두는 그 아래 버전과 맞습니다 (`0.3` 은 `0.3.*`). 맞는 toolchain 이 없으면 `rok toolchain install` 을 안내하는 에러입니다.

```
rok toolchain list
rok toolchain install <version> --from <dir | .tar | .tar.gz>
rok toolchain remove <version>
```

설치본은 `$ROK_HOME/toolchains/<version>/` (`bin/rokc`, `lib/`) 입니다. 다운로드 서버는 아직 없으므로 `--from` 으로 디렉토리나 tarball 을 줍니다.

## 9.10.7. 단일 파일 스크립트의 inline 매니페스트

익명 패키지 ([4.16.8](../004-functions/416-entry-point.md#4168-단일-파일-실행)) 는 파일 맨 앞의 `//!` 줄 안에 `Rok.toml` 조각을 담아 의존성을 선언합니다 (uv 의 inline script metadata 와 같음). 블록에는 `[deps]` 만 둘 수 있고, path 는 스크립트 디렉토리 기준입니다. 스크립트의 lockfile 은 캐시 (`cache/scripts/`) 에 둡니다.

````rok
//! ```toml
//! [deps]
//! util = { git = "https://example.com/util.git", tag = "v1" }
//! ```

use util::greet;
greet();
````
