# 9.1. `rokc` CLI 인터페이스

이 절에서는 컴파일러 `rokc` 의 명령줄 인터페이스를 정의합니다. toolchain 은 세 바이너리로 이루어집니다.

| 바이너리 | 역할                                                                                  |
|----------|---------------------------------------------------------------------------------------|
| `rokc`   | 컴파일러 (rustc 에 해당). 빌드, 실행, 테스트, 포매터, 린터, REPL, LSP 를 담음 (이 절) |
| `rok`    | cargo 식 front-end. 매니페스트, 의존성, 환경, toolchain 고정을 맡고 `rokc` 를 부름 ([9.10](910-rok-frontend.md)) |
| `rokls`  | 편집기용 LSP 서버 실행 파일. `rok lsp` 와 같음                                         |

런타임은 `librokr.a` (`rokr`) 입니다. 별도 도구 (`rok-fmt` 등) 는 두지 않습니다. `rokc` 는 의존성을 해소하지 않으며, 의존 패키지는 `rok` 가 `--extern` 으로 넘깁니다 ([9.2.3](902-aot-pipeline.md#923-의존성-해소)).

## 9.1.1. 일반 형태

`rokc` 는 다음과 같은 일반 형태로 호출됩니다.

```
rokc <subcommand> [options] [args]
```

문법 규칙으로 정리하면 아래와 같습니다.

```
cli_invocation  ::= "rokc" subcommand option* arg*
subcommand      ::= "build" | "emit" | "run" | "test" | "bench" | "check"
                  | "fmt" | "lint" | "doc" | "repl" | "jupyter" | "lsp"
                  | "clean" | "demangle" | "rhir-text" | "inspect" | "version" | "help"
option          ::= "--" identifier ("=" value)?
                  | "-" letter
```

`--name value` 는 `--name=value` 와 같습니다. 전역 옵션 (9.1.5) 은 서브명령 앞뒤 어디에나 둘 수 있습니다.

| 영역            | 서브명령                                              |
|-----------------|-------------------------------------------------------|
| 빌드 / 실행     | `build`, `emit`, `run`, `test`, `bench`, `check`      |
| 패키지          | `clean`                                               |
| 도구            | `fmt`, `lint`, `doc`, `lsp`                           |
| 인터랙티브      | `repl`, `jupyter`                                     |
| 디버그 / 진단   | `demangle`, `rhir-text`, `inspect`                    |
| 그 밖           | `version`, `help`                                     |

### 패키지 지정 (`path`)

`build` / `run` / `test` / `check` / `clean` 의 `[path]` 는 다음 가운데 하나입니다. 생략하면 현재 디렉토리입니다.

| `path`                    | 패키지                                                                                                  |
|---------------------------|---------------------------------------------------------------------------------------------------------|
| 패키지 디렉토리           | 그 디렉토리. `main.rok` 나 `lib.rok` 가 없으면 위로 찾음 (9.2.2)                                          |
| 패키지 안의 파일          | 파일에서 위로 올라가며 처음 `main.rok` / `lib.rok` 가 있는 디렉토리                                       |
| 어떤 패키지에도 없는 파일 | 그 파일 하나로 된 익명 실행 패키지 ([4.16.8](../004-functions/416-entry-point.md#4168-단일-파일-실행)) |

한 디렉토리에 `main.rok` 와 `lib.rok` 가 함께 있으면 에러입니다 ([0.3](../000-meta/003-source-units.md)). 디렉토리를 주었는데 위로 올라가도 패키지가 없으면 에러입니다. 익명 패키지는 파일 서브모듈 (`mod foo;`) 을 가질 수 없습니다.

패키지 이름 (산출물 이름) 은 패키지 디렉토리의 이름이며, 익명 패키지는 파일 이름에서 `.rok` 를 뗀 것입니다.

## 9.1.2. 핵심 서브명령

### `rokc build`: [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 빌드

`rokc build` 는 패키지를 AOT 로 빌드하는 서브명령입니다.

```
rokc build [path]
```

`[path]` 는 위의 패키지 지정 규칙을 따릅니다. 산출물은 [9.2.15](902-aot-pipeline.md#9215-산출물) 의 profile 별 `target/` 에 놓입니다. 실행 패키지는 `target/<profile>/<name>`, 라이브러리 패키지는 `target/<profile>/lib<name>.a` (와 `<name>.o`) 와 인터페이스 `target/<profile>/<name>.rhir` ([9.4](904-rhir-bytecode.md)) 입니다. profile 은 `-O2` / `-O3` / `-Os` (`--release` 포함) 이면 `release`, 그 밖에는 `debug` 이며, `--target=<triple>` 이면 `target/<triple>/<profile>/` 입니다. `--target-dir=<dir>` 나 `ROKC_TARGET_DIR` 이 있으면 `target/` 대신 그 디렉토리를 씁니다.

옵션은 다음과 같습니다.

- `--release` 는 최적화 빌드 (`-O2` 와 같음) 를 `release` profile 로 수행하며, 기본값은 `debug` profile 의 디버그 빌드 (`-O0`) 입니다.
- `--target-dir=<dir>` 는 `target/` 의 위치를 바꿉니다 (`ROKC_TARGET_DIR` 과 같음).
- `--extern <name>=<dir | lib.rok | <name>.rhir | lib<name>.a>` 는 의존 패키지 하나를 최상위 경로 `name` 에 mount 합니다 ([9.2.3](902-aot-pipeline.md#923-의존성-해소)). 소스 (`dir` / `lib.rok`) 는 같은 빌드에서 함께 컴파일하고, 미리 빌드한 패키지 (`.rhir` / `.a`, 하나를 주면 옆의 짝을 찾음) 는 RHIR 을 읽고 archive 를 링크합니다. 여러 번 쓸 수 있고, 보통 `rok` 가 채웁니다. `name` 은 식별자이며, 패키지 이름의 `-` 를 `_` 로 바꾼 것입니다 ([9.10.3](910-rok-frontend.md#9103-rokrokon--roklock)).
- `--name <n>` 은 패키지 이름 ([9.10.3](910-rok-frontend.md#9103-rokrokon--roklock) 의 이름 규칙) 을 정합니다. 산출물은 `lib<n>.a` / `<n>.rhir` 이고 mount 이름은 `-` 를 `_` 로 바꾼 것입니다. 기본은 디렉토리 이름입니다. `--extern a-b=..` 도 `a_b` 로 묶습니다.
- `--bootstrap <file>` 은 미리 읽을 표준 라이브러리 루트 (여러 번 가능), `--rokr <lib>` 는 링크할 런타임입니다. 기본값은 sysroot 의 `lib/rok/{core,std,bare}/lib.rok` 와 `lib/librokr.a` 입니다 ([9.2.19](902-aot-pipeline.md#9219-설치-레이아웃)). 찾지 못하면 찾아본 경로를 나열하는 에러입니다.
- `-O0` / `-O1` / `-O2` / `-O3` / `-Os` 는 최적화 단계를 명시적으로 지정합니다. `-O0` 은 native backend 를, 나머지는 MLIR → LLVM backend 를 씁니다 ([9.2.10](902-aot-pipeline.md#9210-backend-선택)).
- `--target=<triple>` 은 타깃 플랫폼을 지정하며, [플랫폼 타깃](../000-meta/006-platform-targets.md) 의 13 개 Tier 1 중 하나를 사용합니다. 어느 host 에서나 어느 대상이든 빌드할 수 있습니다 ([0.2.4](../000-meta/002-pipeline.md#024-backend)).
- `-o` / `--out=<path>` 는 출력 위치를 지정합니다.
- `--cfg <flag>` / `--cfg <key>=<value>` 는 [조건부 컴파일](../006-meta-system/603-cfg.md) 의 cfg flag 를 설정합니다. 여러 번 쓸 수 있습니다.
- `--emit=<kind>[,<kind>..]` 는 중간 산출물 (`ast`, `hir`, `mir`, `mlir`, `llvmir`, `asm`, `obj`, `expanded`) 을 `target/<profile>/build/<name>.<ext>` 에 씁니다 ([9.9.1](909-debug-tools.md#991-중간-산출물-출력---emit)). 쉼표로 여러 종류를 한 번에 지정하며, 이때는 `-o` 를 함께 쓸 수 없습니다. `--emit` 이 있으면 실행 파일이나 archive 는 만들지 않습니다.
- `--load-mlir-plugin <path|name>` 은 MLIR 다이얼렉트 plugin 을, `--load-mlir-pass-plugin <path|name>` 은 패스 plugin 을 로드하고, `--mlir-pass <pass>` 는 LLVM 다이얼렉트로 변환하기 전에 그 패스를 실행합니다 ([rok::bare::mlir::plugin](../../refs-standard-libraries/100-bare/110-mlir-plugin.md#1103-plugin-로드)). 모두 여러 번 쓸 수 있으며, `rok` 도 그대로 넘깁니다.
- `-g` / `--debug` 는 DWARF 디버그 정보를 냅니다 ([9.9](909-debug-tools.md#dwarf-디버그-정보)).
- `-C overflow-checks[=yes|no]` 는 고정 폭 정수 산술의 overflow 검사 (panic) 를 켜거나 끕니다. 기본값은 `-O0` 에서만 켬입니다. 값을 생략하면 `yes` 입니다. overflow 가 일어날 수 없는 상수 피연산자 연산은 검사를 생략합니다.

이전 빌드 뒤로 바뀐 것이 없으면 `rokc build` / `rokc run` 은 `Fresh` 를 출력하고 빌드를 건너뜁니다 ([9.2.16](902-aot-pipeline.md#9216-증분-빌드)).

### `rokc emit`: 중간 산출물

```
rokc emit [path] [--emit=<kind>[,<kind>..]]
```

`rokc build --emit=..` 와 같습니다. `--emit` 을 생략하면 `mlir` 입니다.

### `rokc run`: 빌드 후 실행

`rokc run` 은 빌드 직후 산출물을 바로 실행하는 명령입니다.

```
rokc run [path] [-- args...]
rokc run -i [path]
```

실행 패키지를 빌드한 뒤 곧바로 실행하며, `--` 뒤에 적은 인자는 실행 파일에 그대로 전달됩니다. 옵션은 `rokc build` 와 동일합니다. 라이브러리 패키지에는 실행할 것이 없으므로 에러입니다.

패키지는 `target/<profile>/<name>` 에 빌드하고, 익명 패키지 (단일 파일) 는 임시 디렉토리에 빌드하므로 파일 옆에 `target/` 을 만들지 않습니다. `rokc build <file>.rok` 은 익명 패키지도 `target/` 에 빌드합니다.

```
rokc run hello.rok -- arg1 arg2
```

`rokc run` 의 종료 코드는 프로그램의 종료 코드입니다. 프로그램이 시그널 `n` 으로 끝나면 `128 + n` 입니다.

`-i` / `--interactive` 는 프로그램을 REPL 세션에 로드한 뒤 `python -i` 처럼 대화형으로 이어 갑니다 ([9.7.10](907-jit-cells.md#9710-프로그램-로드-run--i)). `-- args` 는 받지 않으며, 세션이 끝나면 `0` 으로 종료합니다.

### `rokc test`: 테스트

`rokc test` 는 테스트 함수들을 모아 빌드하고 실행하는 명령입니다.

```
rokc test [path] [filter]
```

`#[test]` 가 붙은 함수를 모아 테스트 실행 파일을 빌드하고 실행합니다. `#[test]` 는 `#[cfg(test)]` 를 함의하므로 ([6.2.5](../006-meta-system/602-built-in-attributes.md#625-test-테스트-함수-마커)) 테스트 함수는 일반 빌드에 들어가지 않습니다. 테스트 빌드에서는 패키지의 진입점 대신 테스트를 고르는 진입점이 쓰입니다.

인자가 하나이고 존재하는 경로가 아니면 그것을 `filter` 로 봅니다. `filter` 는 모듈 경로를 포함한 테스트 이름 (`math::adds`) 에 맞춥니다.

- `^` `$` `.` `*` `+` `?` 와 `\` 이스케이프를 쓰는 작은 정규식입니다. 고정하지 않은 검색이므로 `^` / `$` 가 없으면 이름의 어느 부분과 맞아도 됩니다.
- 이 문자들이 없으면 단순 부분 문자열 검색과 같습니다.

**각 테스트는 자기 프로세스에서 실행됩니다.** 테스트가 panic 하거나 (종료 코드 101) 잡히지 않은 에러로 끝나면 (종료 코드 1) 그 테스트만 실패하고 나머지는 계속 실행됩니다. 시그널로 끝나도 실패입니다.

옵션은 `rokc build` 와 동일하며, 추가로 다음 두 가지를 지원합니다.

- `--threads=N` 은 동시에 실행할 테스트 프로세스 수를 지정합니다. 기본값은 `ROKC_NUM_JOBS`, 그것도 없으면 CPU 수입니다.
- `--show-output` 은 성공한 테스트의 stdout / stderr 까지도 표시합니다. 실패한 테스트의 출력은 늘 표시합니다.

출력은 stdout 에 다음 형태로 나옵니다.

```
running 3 tests
test math::adds ... ok
test math::subs ... FAILED
test util::trims ... ok

failures:

---- math::subs ----
<그 테스트의 stdout / stderr>
(panicked (exit 101))

failures:
    math::subs

test result: FAILED. 2 passed; 1 failed; 0 filtered out; finished in 0.12s
```

실패한 테스트가 하나라도 있으면 종료 코드는 `1` 입니다.

### `rokc bench`: 벤치마크

```
rokc bench [path] [filter]
```

`#[bench]` 가 붙은 함수를 모아 최적화 빌드 (`-O` 를 주지 않으면 `-O2`) 로 실행하고 측정합니다 ([9.9.11](909-debug-tools.md#9911-벤치마크)). `#[bench]` 는 `#[test]` 처럼 `#[cfg(test)]` 를 함의합니다. `path` 와 `filter` 규칙은 `rokc test` 와 같습니다.

- `--iters=N` 은 표본 하나에서 함수를 부르는 횟수입니다. 생략하면 표본 하나가 약 10 ms 가 되도록 맞춥니다.
- `--samples=N` 은 측정할 표본 수입니다 (기본 10). 그 전에 버리는 warm-up 표본이 하나 있습니다.

벤치는 하나씩 차례로 실행되며, cargo 형태로 호출 한 번의 평균 ns 와 표본 사이의 표준편차를 출력합니다.

```
bench fib_20 ... bench:       41,503 ns/iter (+/- 312)
```

### `rokc repl`: 터미널 REPL

`rokc repl` 은 터미널에서 인터랙티브 셸을 띄워 주는 명령입니다.

```
rokc repl [--package <path>]
```

이 모드는 터미널 인터랙티브 셸로 동작합니다. 셀의 의미는 [9.7.0](907-jit-cells.md#970-셀-세션의-규칙) 의 규칙을 따릅니다. 각 셀은 한 번만 실행되고, top-level `let` 바인딩과 항목은 이후 셀로 이어지며, 실패한 셀은 바인딩에 흔적을 남기지 않습니다. 식 셀은 `타입: 값` 을 출력합니다.

- 기본 모드는 각 셀을 native backend 로 컴파일해 세션 동안 사는 JIT host 프로세스에 넘기고, host 가 그것을 로드해 한 번 실행합니다 ([9.6](906-jit-engine.md)). 바인딩, `static` 상태, `any` 값이 셀 사이에 이어집니다. 최상위에서 이름을 적을 수 없는 타입 (클로저, 비공개 타입 등) 의 바인딩은 만들 때 거부합니다.
- 진단과 panic 위치는 셀 자신의 텍스트 기준 `<repl>:line:col` 로 표시합니다.
- 입력은 줄 편집과 history 를 지원하며, history 는 `$ROK_HOME/repl_history` 에 남습니다.
- `--package <path>` 는 라이브러리 패키지를 `--extern` 으로 붙이고, 첫 셀 앞에 `use <name>::*;` 를 넣습니다. `:reset` 뒤에도 다시 넣습니다. `main.rok` 패키지 (실행 패키지) 는 거부합니다.

`rok` 와 `rok repl` ([9.10](910-rok-frontend.md)) 은 의존성을 붙여 같은 REPL 을 실행합니다.

REPL 명령 (`:type`, `:reset`, `:add`, `:help`, `:quit`) 과 디버그 수단은 [9.9.9](909-debug-tools.md#999-repl--jupyter-디버그) 에 있습니다.

### `rokc jupyter`: Jupyter 커널

`rokc jupyter` 는 Jupyter 가 호출하는 커널 모드를 제공합니다.

```
rokc jupyter [--connection-file <path>]
```

이 명령은 Jupyter 가 호출하는 커널 모드이므로 사용자가 직접 실행하지는 않습니다. `--connection-file` 은 Jupyter 가 제공하는 ZMQ 연결 정보 파일의 경로를 받으며, `--extern` 같은 빌드 옵션도 함께 받습니다. 커널은 REPL 과 같은 셀 세션 ([9.7.0](907-jit-cells.md#970-셀-세션의-규칙)) 위에서 동작합니다. 의존성을 해소해 실행하려면 `rok jupyter --connection-file <path>` ([9.10](910-rok-frontend.md)) 를 씁니다. 자세한 통합 내용은 [Jupyter 통합](908-jupyter-integration.md) 에서 정의합니다.

커널 등록은 `rok jupyter install [--env <name>]` 이 합니다 ([9.8.3](908-jupyter-integration.md#983-kernelspec-설치)).

### `rokc fmt`: 포매터

`rokc fmt` 는 표준 포맷을 적용하는 포매터 명령입니다.

```
rokc fmt [path]
```

`[path]` 는 파일 또는 디렉토리를 받으며 (여러 개 가능, 생략하면 현재 디렉토리), [표준 포맷](#9111-표준-포맷-rokc-fmt) 에 맞게 in-place 로 수정합니다. 디렉토리는 재귀로 `.rok` 파일을 찾으며 `target/` 과 `.` 으로 시작하는 디렉토리는 건너뜁니다. `.rokon` 파일 ([RokON](../../refs-standard-libraries/300-std/309-rokon.md#394-정규-형식)) 은 RokON 인지 검사하고 들여쓰기만 다시 맞춥니다 (정규 형식으로 다시 쓰지 않으므로 주석이 남음).

옵션은 다음과 같습니다.

- `--check` 는 파일을 바꾸지 않고, 바뀔 파일과 그 diff 를 출력합니다. 바뀔 파일이 있으면 종료 코드 `1` 이며, CI 용도로 적합합니다.
- `--diff` 는 파일을 바꾸지 않고 diff 만 출력합니다.

어휘 에러가 있는 파일은 건드리지 않고 에러를 보고하며, 종료 코드는 `1` 입니다.

### `rokc check`: 진단

```
rokc check [path] [--check-library]
```

front end 를 타입 검사와 MIR 생성까지 돌리고 lint 와 함께 진단을 냅니다. 코드 생성 (MLIR 포함) 은 하지 않습니다. 옵션은 `rokc build` 와 같습니다. `--check-library` 는 도달 여부와 무관하게 라이브러리의 모든 본문을 낮추고 검사합니다.

### `rokc lint`: 린터

```
rokc lint [path] [--list]
```

`rokc check` 에서 코드 생성을 뺀 것입니다. 진단 형식은 9.1.8, lint 진단의 코드는 `E503` 입니다. lint 목록과 그룹은 [6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어) 에 있습니다. `--list` 는 lint 와 그룹, 기본 수준을 출력합니다.

lint 수준은 전역 옵션 `-A` / `-W` / `-D` (9.1.5) 와 코드의 속성이 정합니다. `deny` 수준의 lint 가 하나라도 걸리면 종료 코드는 `1` 입니다.

### `rokc doc`: 문서 생성

```
rokc doc [path] [--std]
```

패키지 (와 `--extern` 의존 패키지) 의 공개 항목과 문서 (`///`, `//!`, `#[doc]`, [6.2.8](../006-meta-system/602-built-in-attributes.md#628-doc---문서-주석-형태)) 로 HTML API 문서를 `target/doc/` 에 만듭니다.

```
target/doc/
├── index.html                 # 패키지 목록
├── <pkg>/<mod>/index.html     # 모듈마다 한 쪽
└── search-index.js            # 검색용 색인
```

`--std` 는 표준 라이브러리 tier 도 `rok-core/`, `rok-std/`, `rok-bare/` 로 문서화합니다. 경로가 없으면 그것만 `./target/doc/` 에 만듭니다.

문서 텍스트는 markdown 의 부분집합 (제목, fenced 코드 블록, 인라인 코드, 링크, `**굵게**`, 목록, 문단) 으로 해석합니다.

### `rokc lsp`: LSP 서버

`rokc lsp` 는 Language Server Protocol 서버 모드로 동작합니다.

```
rokc lsp
```

이 모드는 VS Code, Vim, Emacs 등 LSP 클라이언트에서 호출하도록 설계됩니다. 편집기용 독립 바이너리 `rokls` 는 `rok lsp` (곧 `rokc lsp`) 와 같습니다. `.rokon` 버퍼는 구문만 검사합니다. 기능 목록은 [9.9.13](909-debug-tools.md#9913-lsp-서버) 에 있습니다.

## 9.1.3. 디버그 / 진단 서브명령

### `rokc demangle`: 심볼 demangle

`rokc demangle` 은 mangled 심볼을 사람이 읽을 수 있는 형식으로 풀어 출력합니다.

```
rokc demangle <symbol>...
```

심볼마다 [9.5.4](905-symbol-mangling.md) 형식의 블록을 빈 줄로 나눠 출력합니다 ([9.9.3](909-debug-tools.md#993-심볼-demangle)). rok 심볼이 아닌 인자가 있으면 종료 코드는 `1` 입니다.

### `rokc rhir-text`: [RHIR](../000-meta/007-glossary.md#rhir-rok-hir-bytecode) 텍스트 출력

`rokc rhir-text` 는 미리 빌드한 라이브러리의 RHIR 을 텍스트로 출력합니다.

```
rokc rhir-text path/to/foo.rhir      # 또는 libfoo.a — 옆의 foo.rhir 를 읽음
```

header 필드, producer, meta 와 interface 파일들을 출력하며, 포맷은 [RHIR 포맷](904-rhir-bytecode.md) 에 기술됩니다.

### `rokc inspect`: 패키지 검사

`rokc inspect` 는 archive 의 내용을 한눈에 살펴볼 수 있도록 합니다.

```
rokc inspect path/to/libfoo.a        # 또는 foo.rhir
```

패키지 이름과 해시, rokc 버전, RHIR 버전, GC type id 구간, exports, [generic](../000-meta/007-glossary.md#generic--제네릭) items, archive 멤버와 크기를 한 번에 표시합니다 ([9.9.14](909-debug-tools.md#9914-패키지-검사)).

## 9.1.4. 보조 서브명령

### `rokc clean`: 빌드 산출물 제거

`rokc clean` 은 빌드 산출물을 제거하는 명령입니다.

```
rokc clean [path]
```

패키지의 `target/` 디렉토리 전체 (모든 profile 과 교차 빌드 포함, `--target-dir` / `ROKC_TARGET_DIR` 이 있으면 그 디렉토리) 를 삭제합니다. 없으면 아무 일도 하지 않습니다.

### `rokc version`: 버전

```
rokc version
rokc --version            # = -V
rokc -V -v                # host 줄 추가
```

stdout 에 `rokc <버전> (<commit> <date>)` 한 줄을 출력합니다. `-v` 를 더하면 `host: <triple>` 줄을 덧붙입니다.

### `rokc help`: 도움말

```
rokc help [subcommand]
rokc <subcommand> --help
```

서브명령을 주면 그 명령의 도움말을 보여 줍니다.

## 9.1.5. 전역 옵션

전역 옵션들은 다음 표와 같습니다. 서브명령 앞뒤 어디에나 둘 수 있습니다.

| 옵션                             | 의미                                                        |
|----------------------------------|-------------------------------------------------------------|
| `--verbose` / `-v`               | 단계별 시간 표시 ([9.9.5](909-debug-tools.md#995-로그--진행-표시)) |
| `-vv`                            | 더 자세한 로그                                              |
| `--quiet` / `-q`                 | 진행 줄을 출력하지 않음 (진단은 출력)                       |
| `--color=auto\|always\|never`    | 진단의 색상 출력                                            |
| `--no-color`                     | `--color=never` 와 같음                                     |
| `--cwd=<path>`                   | 작업 디렉토리를 바꾼 뒤 실행                                |
| `--message-format=human\|json`   | 진단 형식. `json` 은 stdout 에 한 줄씩 (9.1.8)              |
| `-A` / `-W` / `-D <lint \| group>` | lint 수준 (`--allow` / `--warn` / `--deny`). `check` / `lint` / `lsp` 에 적용. 여러 번 쓸 수 있고 뒤의 것이 이김 |

lint 수준은 lint 의 기본 수준 < `-A` / `-W` / `-D` < 코드의 `#[allow(..)]` / `#[warn(..)]` / `#[deny(..)]` (가장 안쪽이 이김) 순으로 정합니다. `rok` 는 매니페스트의 `lints` 를 이 플래그로 앞에 붙여 넘깁니다 ([9.10.1](910-rok-frontend.md#9101-서브명령과-전역-옵션)).

`--color=auto` (기본값) 는 stderr 가 터미널이고 환경 변수 `NO_COLOR` 가 없을 때 색을 씁니다.

사용 예는 다음과 같습니다.

```
rokc --verbose build
rokc test -q
rokc --cwd=examples/hello run
```

## 9.1.6. 종료 코드

종료 코드는 다음과 같이 정의됩니다.

| 코드  | 의미                                            |
|-------|-------------------------------------------------|
| `0`   | 성공                                            |
| `1`   | 사용자 코드 에러 (컴파일 에러 / 테스트 실패 등) |
| `2`   | 사용법 에러 (잘못된 옵션, 모르는 서브명령 등)   |
| `101` | rokc 내부 에러 (panic)                          |

`rokc` 는 `ROK_HOME` 을 읽지 않습니다 (sysroot 는 실행 파일 기준). rokc 자신이 panic 하면 (internal compiler error) 메시지를 출력하고 `101` 로 끝납니다. `rokc run` 은 실행한 프로그램의 종료 코드로 끝납니다 (9.1.2).

## 9.1.7. 환경 변수

`rokc` 의 동작에 영향을 주는 환경 변수는 다음 표와 같습니다.

| 변수              | 의미                                                                                     |
|-------------------|------------------------------------------------------------------------------------------|
| `ROK_CC`          | 내장 lld 대신 마지막 링크를 하는 선택적 C 컴파일러 드라이버 ([9.2.14](902-aot-pipeline.md#9214-링킹-실행-파일의-경우)) |
| `ROKC_VERBOSE`    | `--verbose` 와 동일                                                                      |
| `ROK_TIME_PASSES` | `1` 이면 패스별 실제 시간을 stderr 에 출력 ([9.9.5](909-debug-tools.md#rok_time_passes)) |
| `ROKC_TARGET_DIR` | `target/` 의 위치 (기본은 패키지 안)                                                     |
| `ROKC_NUM_JOBS`   | `rokc test --threads` 의 기본값 (없으면 CPU 코어 수)                                     |
| `NO_COLOR`        | 있으면 `--color=auto` 가 색을 쓰지 않음                                                  |

## 9.1.8. 출력 형식

### 진단 메시지

진단 메시지는 다음과 같은 형식으로 출력됩니다.

```
error[E100]: cannot find value `count` in this scope
  --> src/main.rok:5:13
   |
 2 |     let counter = 0;
   |         ------- similar name defined here
...
 5 |     let x = count + 1;
   |             ^^^^^ not found in this scope
   |
   = help: did you mean `counter`?
```

[소스 파일](../001-lexical/101-source-files.md) 에서 정의한 파일 + 줄 + 컬럼 단위의 진단을 그대로 사용합니다.

머리줄은 `<severity>[<code>]: <message>` 이고 `--> path:line:col` 이 주 위치입니다. 주 span 은 `^`, 보조 span 은 `-` 로 밑줄을 긋고, 다른 파일의 보조 span 은 `::: path:line:col` 로 시작합니다. 끝의 `= note:` 줄은 맥락을, `= help:` 줄은 고치는 방법을 덧붙입니다.

### 빌드 진행

진행 상황은 stderr 에 오른쪽 정렬한 동사로 표시됩니다. `-q` 면 출력하지 않습니다.

```
   Compiling foo (path/to/foo)
    Building target/debug/foo
    Finished in 1.23s
     Running `target/debug/foo`
```

| 동사        | 시점                                      |
|-------------|-------------------------------------------|
| `Compiling` | 프론트엔드 시작 (패키지 이름과 디렉토리)  |
| `Building`  | 실행 파일 / archive 생성 (`build`)        |
| `Wrote`     | `--emit` 산출물 하나를 씀                 |
| `Finished`  | 빌드 끝 (걸린 시간)                       |
| `Running`   | 프로그램 실행 (`run`)                     |
| `Created`   | 패키지 생성 (`new`)                       |
| `Removed`   | `target/` 삭제 (`clean`)                  |

### JSON 출력 (도구 통합용)

도구 통합을 위해서는 JSON 형식으로도 출력할 수 있습니다.

```
rokc build --message-format=json
```

이 모드에서는 진단 하나가 stdout 의 한 줄 JSON 객체가 되므로, IDE 와 CI 도구에서 쉽게 파싱할 수 있습니다. 객체의 모양은 [9.9.4](909-debug-tools.md#994-진단--에러-메시지) 에서 정의합니다.

## 9.1.9. 표준 입력

`rokc check -` 처럼 인자 자리에 `-` 를 적으면 표준 입력을 의미하며, 파이프 또는 스크립트 통합 용도로 활용할 수 있습니다. 표준 입력의 소스는 익명 실행 패키지의 `main.rok` 로 검사합니다.

## 9.1.10. 외부 도구와의 호환

`rokc` 는 저수준 컴파일러이며, 의존성 해소는 front-end `rok` ([9.10](910-rok-frontend.md)) 가 맡습니다. `rok` 는 해소한 의존 패키지의 루트를 `--extern` ([9.2.3](902-aot-pipeline.md#923-의존성-해소)) 으로 넘겨 `rokc` 를 부릅니다. 다른 빌드 시스템도 같은 방식으로 `rokc` 를 직접 호출해 통합할 수 있습니다. CLI 호환성을 깨는 변경은 SemVer 규칙에 따라 관리됩니다.

## 9.1.11. 표준 포맷 (`rokc fmt`)

`rokc fmt` 의 표준 포맷은 **토큰을 보존하는 배치 규칙** 입니다. 줄 안의 토큰과 주석은 바꾸지 않고 줄의 들여쓰기와 공백 줄만 정합니다.

1. 들여쓰기는 괄호 (`(` `[` `{` `#[`) 깊이마다 공백 4 칸입니다. 닫는 괄호로 시작하는 줄은 여는 줄의 깊이를 따릅니다. 탭은 공백으로 바뀝니다.
2. 이어지는 줄, 곧 앞 토큰이 `;` `,` 나 괄호가 아닌 줄 (긴 식을 나눈 줄 등) 은 원래 파일에서 블록 들여쓰기보다 더 들여쓴 만큼을 유지합니다.
3. 줄 끝 공백을 지웁니다.
4. 연속된 빈 줄은 하나로 줄이고, 파일 처음의 빈 줄은 지웁니다.
5. 파일은 줄바꿈 하나로 끝납니다.

포매터는 토큰을 줄 사이로 옮기지 않고, 줄을 나누거나 합치지 않으며, 줄 안의 간격을 바꾸지 않습니다. 따라서 주석은 적힌 그대로 남고 토큰 열은 바뀌지 않습니다. 여러 줄에 걸친 문자열과 블록 주석의 안쪽도 그대로 둡니다.
