# 9.10. `rok` — 패키지 / 빌드 front-end

> [!NOTE]
> 이 장 전체는 계획된 인터페이스입니다. 아직 제공하지 않습니다.

`rok` 는 cargo 식 front-end 입니다. 매니페스트 `Rok.toml` 과 lockfile `Rok.lock`, 의존성, 환경, toolchain 고정, 스크립트 의존성을 맡고, 컴파일은 컴파일러 `rokc` ([9.1](901-rokc-cli.md)) 를 불러 처리합니다. 런타임은 `rokr` (`librokr.a`) 입니다.

의존성이 없는 패키지는 매니페스트 없이 파일 규약만으로 빌드됩니다 ([0.3.4](../000-meta/003-source-units.md#034-매니페스트-roktoml-는-선택)). 이때 `rok build` 는 `rokc build` 와 같습니다.

## 9.10.1. 서브명령

| 서브명령                                         | 동작                              |
|--------------------------------------------------|-----------------------------------|
| `new`, `init`                                    | `rok` 전용 — 패키지 생성 (9.10.2) |
| `add`, `remove`, `update`                        | `rok` 전용 — 의존성 (9.10.4)      |
| `env`                                            | `rok` 전용 — 환경 (9.10.5)        |
| `toolchain`                                      | `rok` 전용 — toolchain (9.10.6)   |
| `build`, `check`, `run`, `test`, `bench`, `clean` | `rokc` 에 위임                    |
| `fmt`, `lint`, `doc`, `repl`, `lsp`              | `rokc` 에 위임                    |

위임하는 서브명령은 의존성을 해소한 뒤 (`Rok.lock` 을 따라 가져오고 빌드), 의존 패키지의 루트를 `--extern` 으로 붙여 같은 이름의 `rokc` 서브명령을 부릅니다 ([9.2.3](902-aot-pipeline.md#923-의존성-해소)). 옵션과 출력, 종료 코드는 `rokc` 의 것 ([9.1](901-rokc-cli.md)) 을 따릅니다. `rokls` 는 `rok lsp` 와 같은 독립 실행 파일입니다.

## 9.10.2. `rok new` / `rok init`: 패키지 생성

```
rok new [--lib | --bin] <name>
rok init [--lib | --bin] [path]
```

`rok new` 는 `<name>` 디렉토리를 만들고 그 안에 기본 파일을 채워 줍니다. `--bin` (기본값) 은 인사말을 출력하는 `main.rok` 를, `--lib` 는 예시 함수와 `#[test]` 하나를 담은 `lib.rok` 를 만듭니다. `<name>` 이 이미 있으면 에러입니다.

`rok init` 은 이미 있는 디렉토리 (생략하면 현재 디렉토리) 를 패키지로 만듭니다. 그 디렉토리에 `main.rok` 나 `lib.rok` 가 이미 있으면 에러입니다.

## 9.10.3. `Rok.toml` / `Rok.lock`

패키지 루트의 `Rok.toml` (TOML 1.0) 이 매니페스트이고, 같은 자리의 `Rok.lock` 이 lockfile 입니다. 이름은 이 철자 그대로 (첫 글자만 대문자) 이며, 대소문자가 다르면 경고입니다.

```toml
[package]
name = "geom"
version = "0.3.1"
toolchain = "0.4.0"          # 선택 — toolchain 고정

[deps]
util = { path = "../util" }
parser = { git = "https://example.com/parser.git", tag = "v1.2" }

[dev-deps]                   # test / bench 에서만
fixtures = { path = "tests/fixtures" }
```

| 의존성 출처 | 형태                                                            | `Rok.lock` 에 고정  |
|-------------|-----------------------------------------------------------------|---------------------|
| path        | `{ path = ".." }`                                               | 없음 (늘 현재 내용) |
| git         | `{ git = "url", rev = ".." }` (또는 `tag` / `branch`)           | 커밋 해시           |
| registry    | 예약 — 쓰면 "not yet supported" 에러                            |                     |
| `native`    | 예약 (conda 식 C 라이브러리) — 쓰면 "not yet supported" 에러    |                     |

한 빌드 안에서 한 패키지는 한 버전만 쓸 수 있습니다. 의존 그래프가 같은 패키지의 두 버전을 요구하면 에러입니다.

## 9.10.4. `rok add` / `rok remove` / `rok update`

```
rok add <name> --path <dir> [--dev]
rok add <name> --git <url> [--rev <rev> | --tag <tag> | --branch <branch>] [--dev]
rok remove <name>
rok update [name]
```

- `add` / `remove` 는 `Rok.toml` 을 고치며, 주석과 형식을 보존합니다. `--dev` 는 `[dev-deps]` 에 넣습니다.
- `Rok.lock` 을 바꾸는 것은 `update` 뿐입니다. 빌드는 `Rok.lock` 을 따릅니다. `Rok.lock` 은 버전 관리에 함께 넣습니다.
- `--offline` 은 네트워크를 쓰지 않고 캐시만 씁니다. `--locked` 는 `Rok.lock` 을 바꿔야 하는 상황이면 에러로 끝냅니다 (CI 용).
- 가져온 소스는 전역 내용 주소 캐시 `$ROK_HOME/cache/` 에 둡니다.

의존 패키지는 같은 파이프라인으로 빌드되어 `.a` 로 링크됩니다 ([9.2.3](902-aot-pipeline.md#923-의존성-해소)).

## 9.10.5. 환경

uv / conda 식 환경을 둡니다. 활성화 단계는 없습니다.

- `Rok.toml` 이 있는 모든 디렉토리가 환경입니다. 그 안에서의 `rok` 명령은 그 환경의 의존성을 씁니다.
- 이름 있는 환경은 `$ROK_HOME/envs/<name>/` 에 `Rok.toml` / `Rok.lock` 을 둡니다. 프로젝트 밖에서는 `default` 환경을 씁니다.

```
rok env create <name>
rok env list
rok env remove <name>
rok env use <name>                # 프로젝트 밖의 기본 환경을 바꿈
rok env export <name> [file]      # Rok.toml + Rok.lock 내보내기
rok env import <name> <file>
rok env shell <name>              # 그 환경을 쓰는 셸
rok repl --env <name>
```

REPL 의 `:add <dep>` 는 현재 환경에 의존성을 추가합니다 (`rok add` 와 같음).

## 9.10.6. toolchain 고정

`[package] toolchain = "X"` 가 있으면 `rok` 는 `$ROK_HOME/toolchains/X/bin/rok` 를 다시 실행 (re-exec) 해 그 toolchain 의 `rok` / `rokc` 로 명령을 처리합니다. 설치되어 있지 않으면 `rok toolchain install X` 를 안내하는 에러입니다.

```
rok toolchain list
rok toolchain install <version>
rok toolchain remove <version>
```

## 9.10.7. 단일 파일 스크립트의 inline 매니페스트

익명 패키지 ([4.16.8](../004-functions/416-entry-point.md#4168-단일-파일-실행)) 는 파일 머리의 `//!` 주석 안에 `Rok.toml` 조각을 담아 의존성을 선언합니다 (uv 의 inline script metadata 와 같음).

````rok
//! ```toml
//! [deps]
//! util = { git = "https://example.com/util.git", tag = "v1" }
//! ```

use util::greet;
greet();
````
