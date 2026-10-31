# 9.1. `rokc` CLI 인터페이스

이 절은 컴파일러 `rokc`의 명령줄 인터페이스를 정의합니다. toolchain은 세 바이너리로 이루어집니다.

| 바이너리 | 역할                                                                                  |
|----------|---------------------------------------------------------------------------------------|
| `rokc`   | 컴파일러(rustc에 해당). 빌드, 실행, 테스트, 포매터, 린터, REPL, LSP를 담음(이 절) |
| `rok`    | cargo 식 프런트엔드. 매니페스트, 의존성, 환경, toolchain 고정을 맡고 `rokc`를 부름([9.10](910-rok-frontend.md)) |
| `rokls`  | 에디터용 LSP 서버 실행 파일. `rok lsp`와 같음                                         |

런타임은 `librokr.a`(`rokr`)입니다. 별도 도구(`rok-fmt` 등)는 두지 않습니다. `rokc`는 의존성을 해소하지 않습니다. 의존 패키지는 `rok`이 `--extern`으로 넘깁니다([9.2.3](902-aot-pipeline.md#923-의존성-해소)).

## 9.1.1. 일반 형태

`rokc`는 다음 형태로 호출합니다.

```
rokc <subcommand> [options] [args]
```

문법 규칙으로 쓰면 다음과 같습니다.

```
cli_invocation  ::= "rokc" subcommand option* arg*
subcommand      ::= "build" | "emit" | "run" | "test" | "bench" | "check"
                  | "fmt" | "lint" | "doc" | "repl" | "jupyter" | "lsp"
                  | "clean" | "profdata" | "bindgen" | "demangle" | "rokhir-text" | "inspect" | "completions"
                  | "version" | "help"
option          ::= "--" identifier ("=" value)?
                  | "-" letter
```

`--name value`는 `--name=value`와 같습니다. 전역 옵션(9.1.5)은 서브명령 앞뒤 어디에나 둘 수 있습니다.

| 영역            | 서브명령                                              |
|-----------------|-------------------------------------------------------|
| 빌드 / 실행     | `build`, `emit`, `run`, `test`, `bench`, `check`      |
| 패키지          | `clean`                                               |
| 도구            | `fmt`, `lint`, `doc`, `lsp`, `profdata`, `bindgen`                |
| 인터랙티브      | `repl`, `jupyter`                                     |
| 디버그 / 진단   | `demangle`, `rokhir-text`, `inspect`                    |
| 그 밖           | `completions`, `version`, `help`                      |

### 패키지 지정(`path`)

`build` / `run` / `test` / `check` / `clean`의 `[path]`는 다음 가운데 하나입니다. 생략하면 현재 디렉터리입니다.

| `path`                    | 패키지                                                                                                  |
|---------------------------|---------------------------------------------------------------------------------------------------------|
| 패키지 디렉터리           | 그 디렉터리. `src/main.rok`이나 `src/lib.rok`이 없으면 위로 찾음(9.2.2)                                  |
| 패키지 `src/` 안의 파일   | 그 패키지. 파일에서 위로 올라가며 처음 만나는, `src/main.rok` / `src/lib.rok`이 있는 디렉터리                    |
| 그 밖의 파일              | 그 파일 하나로 된 익명 실행 패키지([4.16.8](../004-functions/416-entry-point.md#4168-단일-파일-실행)). 패키지 디렉터리 안이지만 `src/` 밖에 있는 `.rok` 파일도 포함 |

디렉터리를 주었는데 위로 올라가도 패키지가 없으면 에러입니다. 익명 패키지의 파일 서브모듈(`mod foo;`)은 그 파일 옆에 있습니다([4.16.8](../004-functions/416-entry-point.md#4168-단일-파일-실행)).

패키지 이름(산출물 이름)은 패키지 디렉터리의 이름입니다. 익명 패키지는 파일 이름에서 `.rok`을 뗀 것이 이름입니다.

## 9.1.2. 핵심 서브명령

### `rokc build`: [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 빌드

`rokc build`는 패키지를 AOT로 빌드합니다.

```
rokc build [path]
```

`[path]`는 위의 패키지 지정 규칙을 따릅니다. 산출물은 [9.2.15](902-aot-pipeline.md#9215-산출물)의 프로필별 `target/`에 놓입니다. `src/main.rok`은 `target/<profile>/<name>`을 만들고, `src/lib.rok`은 `target/<profile>/lib<name>.a`와 인터페이스 `target/<profile>/<name>.rokhir`([9.4](904-rokhir-bytecode.md))을 만듭니다. 둘 다 있는 패키지는 둘 다 만듭니다. 프로필은 `-O2` / `-O3` / `-Os`(`--release` 포함)이면 `release`이고, 그 밖에는 `debug`입니다. `--target=<target>`을 주면 `target/<name>/<profile>/`입니다. `--target-dir=<dir>`나 `ROKC_TARGET_DIR`이 있으면 `target/` 대신 그 디렉터리를 씁니다.

단독 파일([4.16.8](../004-functions/416-entry-point.md#4168-단일-파일-실행))은 `rokc run`처럼 빌드합니다. 실행 파일은 파일 옆에 "파일 이름의 stem + 타깃의 실행 파일 접미사"라는 이름으로 복사합니다(`hello.rok`이면 `hello`, Windows에서는 `hello.exe`). `-o`로 다른 파일을 지정하면 그곳에 둡니다. 그 경로에 이 스크립트의 이전 빌드가 쓰지 않은 파일이 이미 있으면 그 파일은 건드리지 않습니다. 이때 빌드는 실패하고 `-o <file>`을 안내합니다.

옵션은 다음과 같습니다.

- `--release`는 `release` 프로필로 최적화 빌드(`-O2`와 같음)를 합니다. 기본값은 `debug` 프로필의 디버그 빌드(`-O0`)입니다. 두 프로필의 차이는 [9.2.15](902-aot-pipeline.md#9215-산출물)가 설명합니다.
- `--target-dir=<dir>`는 `target/`의 위치를 바꿉니다(`ROKC_TARGET_DIR`과 같음).
- `--extern <name>=<dir | src/lib.rok | <name>.rokhir | lib<name>.a>`는 의존 패키지 하나를 최상위 경로 `name`에 마운트합니다([9.2.3](902-aot-pipeline.md#923-의존성-해소)). 소스(`dir` / `src/lib.rok`)는 같은 빌드에서 함께 컴파일합니다. 미리 빌드한 패키지(`.rokhir` / `.a`, 하나를 주면 옆의 짝을 찾음)는 ROKHIR을 읽고 아카이브를 링크합니다. 여러 번 쓸 수 있고, 보통 `rok`이 채웁니다. `name`은 식별자이고, 패키지 이름의 `-`를 `_`로 바꾼 것입니다([9.10.3](910-rok-frontend.md#9103-rokrokon--roklock)). `rok`은 표준 라이브러리용으로 예약되어 있습니다. `--extern rok=..`은 사용법 에러이고, `rok`이라는 라이브러리 패키지는 거부합니다. 키워드와 경로 시작점(`pkg`, `sup`)도 `name`으로 쓸 수 없고, 라이브러리나 라이브러리가 있는 패키지의 이름으로도 쓸 수 없습니다([9.10.3](910-rok-frontend.md#9103-rokrokon--roklock)).
- `--extern-feature <name>=<f>`는 소스 의존 패키지 `name`을 `cfg(feature = "<f>")`로 빌드합니다. feature마다 하나씩, 여러 번 줄 수 있습니다. 의존 패키지가 보는 feature는 이것뿐입니다. `--cfg`의 `feature` 키는 지금 컴파일하는 패키지에만 적용됩니다. 보통 `rok`이 채웁니다.
- `--name <n>`은 패키지 이름([9.10.3](910-rok-frontend.md#9103-rokrokon--roklock)의 이름 규칙)을 정합니다. 산출물은 `lib<n>.a` / `<n>.rokhir`이고, 마운트 이름은 `-`를 `_`로 바꾼 것입니다. 기본값은 디렉터리 이름입니다. `--extern a-b=..`도 `a_b`로 묶습니다.
- `--bootstrap <file>`은 미리 읽을 표준 라이브러리 루트이고(여러 번 가능), `--rokr <lib>`는 링크할 런타임입니다. 기본값은 sysroot의 `lib/rok/{core,std,bare}/src/lib.rok`과 `lib/librokr.a`입니다([9.2.19](902-aot-pipeline.md#9219-설치-레이아웃)). bootstrap 루트의 계층은 그 `src/` 위 디렉터리의 이름입니다. `bare` 루트 없이 `core/src/lib.rok`을 주면 그 옆의 `bare/src/lib.rok`도 읽습니다. 찾지 못하면 에러이고, 에러 메시지에 찾아본 경로를 나열합니다.
- `-O0` / `-O1` / `-O2` / `-O3` / `-Os`는 최적화 단계를 지정합니다. `-O0`은 native 백엔드를, 나머지는 MLIR → LLVM 백엔드를 씁니다([9.2.10](902-aot-pipeline.md#9210-백엔드-선택)).
- `--target=<target>`은 타깃 플랫폼을 지정합니다. [플랫폼 타깃](../000-meta/006-platform-targets.md)의 Tier 1 15개 같은 내장 이름이나, `<name>.rokon` 타깃 사양 파일([0.6.5](../000-meta/006-platform-targets.md#065-타깃-사양))을 받습니다. 어느 호스트에서나 어느 타깃이든 빌드할 수 있습니다([0.2.4](../000-meta/002-pipeline.md#024-백엔드)).
- `-L <dir>`은 `#[link]`의 라이브러리 검색 디렉터리를 더합니다([5.9.9](../005-modules/509-extern-block.md#599-linklibname-라이브러리-명시)). `LIBRARY_PATH`보다 먼저 찾습니다. 여러 번 쓸 수 있습니다.
- `-l [static=|dylib=]<name>`은 라이브러리 `<name>`을 링크합니다(`#[link]`처럼 찾음). `--native-runtime <file>`은 실행 파일 옆에 둘 공유 라이브러리를 지정합니다([9.2.14](902-aot-pipeline.md#9214-링킹실행-파일의-경우)). 둘 다 여러 번 쓸 수 있고, native 패키지가 있으면 `rok`이 넘깁니다([9.10.4b](910-rok-frontend.md#9104b-네이티브-패키지)).
- `-o` / `--out=<path>`(`--output`도 같음)는 출력 위치를 지정합니다.
- `--keep`은 빌드의 임시 디렉터리(오브젝트와 그 밖의 중간 파일)를 지우지 않고 남깁니다. 그 경로는 표준 에러에 알립니다(`rokc: kept temporaries in <dir>`).
- `--cfg <flag>` / `--cfg <key>=<value>`는 [조건부 컴파일](../006-meta-system/603-cfg.md)의 cfg 플래그를 설정합니다. 여러 번 쓸 수 있습니다.
- `--emit=<kind>[,<kind>..]`는 중간 산출물(`ast`, `hir`, `mir`, `mlir`, `llvmir`, `asm`, `obj`, `expanded`)을 `target/<profile>/build/<name>.<ext>`에 씁니다([9.9.1](909-debug-tools.md#991-중간-산출물-출력---emit)). 쉼표로 여러 종류를 한 번에 지정할 수 있고, 이때는 `-o`를 함께 쓸 수 없습니다. `--emit`이 있으면 실행 파일이나 아카이브는 만들지 않습니다.
- `--load-mlir-plugin <path|name>`은 MLIR 다이얼렉트 plugin을 로드하고, `--load-mlir-pass-plugin <path|name>`은 패스 plugin을 로드합니다. 둘 다 `rokc`의 MLIR에 로드합니다(plugin은 같은 고정 LLVM으로 빌드하고, Windows에서는 `rokc.exe`의 import library에 링크한 것입니다). `--mlir-pass <pass>`는 LLVM 다이얼렉트로 변환하기 전에 그 패스를 실행합니다([stdlib 1.10](../../refs-standard-libraries/100-bare/110-mlir-plugin.md#1103-plugin-로드)). 모두 여러 번 쓸 수 있고, `rok`도 그대로 넘깁니다.
- `-g`는 `release` 프로필 빌드에 디버그 정보를 더합니다. 디버그 정보는 DWARF이고, Windows 타깃에서는 PDB의 CodeView입니다([9.9.6](909-debug-tools.md#996-디버거-통합-gdb--lldb)). `debug` 프로필 빌드에는 항상 있습니다.
- `-C <option>`은 `--codegen <option>`으로도 쓸 수 있습니다.
- `-C overflow-checks[=yes|no]`는 고정 폭 정수 산술의 오버플로 검사(panic)를 켜거나 끕니다. 기본으로는 `-O0`에서만 켭니다. 값을 생략하면 `yes`입니다. 오버플로가 일어날 수 없는 상수 피연산자 연산은 검사하지 않습니다.
- `-C backend=native|llvm`은 최적화 수준이 고른 백엔드를 바꿉니다([9.2.10](902-aot-pipeline.md#9210-백엔드-선택)).
- `-C target-cpu=<cpu>|native`는 트리플의 baseline 대신 그 CPU를 대상으로 합니다([0.6.4](../000-meta/006-platform-targets.md#064-simd)). `native`는 빌드하는 기계의 CPU입니다. `-C target-feature=+f1,-f2,..`는 개별 기능(`+avx2` 등)을 켜거나 끕니다. 그렇게 만든 바이너리는 그 기능이 없는 CPU에서 돌지 않을 수 있습니다. native 백엔드(`-O0`)에서는 이 옵션을 경고로 알리고 baseline용으로 빌드합니다. `native`는 타깃이 호스트 플랫폼일 때만 받습니다.
- `-C subsystem=console|windows`는 Windows subsystem을 고릅니다. `console`(기본값, `mainCRTStartup`)이나 `windows`(GUI, `WinMainCRTStartup`, [9.5.9](905-symbol-mangling.md#959-main-함수의-심볼))를 받습니다.
- `-C profile-generate[=<dir>]`는 프로필 기반 최적화를 위해 프로그램과 소스 의존 패키지를 계측합니다. 계측한 프로그램을 실행하면 raw profile `<dir>/default_%m.profraw`를 씁니다. 기본 `<dir>`는 실행 시점의 작업 디렉터리이고, `LLVM_PROFILE_FILE`로 경로를 바꿀 수 있습니다. 계측한 실행 파일은 toolchain이 타깃용으로 제공하는 프로필 런타임을 링크합니다([9.2.19](902-aot-pipeline.md#9219-설치-레이아웃)). wasm32-unknown-unknown에서는 지원하지 않습니다(파일 시스템 없음).
- `-C cs-profile-generate[=<dir>]`은 `-C profile-use`와 함께 쓰며, 인라인한 뒤에 다시 계측합니다(context-sensitive). 그렇게 실행해 얻은 결과를 원래 프로필과 합치면 `-C profile-use`가 읽는 프로필 하나가 됩니다.
- `-C profile-use=<file>`은 `rokc profdata merge`로 만든 indexed profile로 최적화합니다. 파일이 없으면 에러입니다. 프로필을 기록한 뒤 제어 흐름이 바뀐 함수는 프로필 없이 최적화하고, 그런 함수의 수를 경고 하나로 알립니다.
- 두 옵션은 어느 수준에서나 MLIR → LLVM 백엔드를 고르며, 함께 쓰는 `-C backend=native`는 에러입니다. 표준 라이브러리는 캐시에서 가져오지 않고 프로그램과 함께 컴파일하므로, 함께 계측되고 최적화됩니다. REPL은 두 옵션을 모두 거부합니다.

`rokc build`는 패키지의 린트 진단을 `rokc check`처럼 보고합니다([6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)). 경고는 출력하고 빌드를 이어 가지만, `deny` 린트는 빌드를 실패시킵니다. 이전 빌드 뒤로 바뀐 것이 없으면 `rokc build` / `rokc run`은 `Fresh`를 출력하고, 그 빌드의 경고를 다시 출력한 뒤 빌드를 건너뜁니다([9.2.16](902-aot-pipeline.md#9216-증분-빌드)).

### `rokc profdata merge`: 프로필 병합

```
rokc profdata merge [-o <file>] <file | dir>...
```

raw(`.profraw`), indexed(`.profdata`), text(`.proftext`, LLVM의 텍스트 프로필 형식) 프로필을 `-C profile-use`용 indexed profile 하나(기본값 `default.profdata`)로 합칩니다. 디렉터리를 주면 그 안의 프로필 파일을 모두 뜻하고, 빈 raw profile은 건너뜁니다. 병합은 프로세스 안에서 실행합니다. 병합 경고(counter overflow, counter 불일치)는 보고하되, 병합 결과는 그대로 씁니다.

### `rokc profdata show`: 프로필 요약

```
rokc profdata show [--all-functions] [--topn N] <file>
```

프로필의 요약을 `llvm-profdata show` 형태로 출력합니다. 계측 수준, 보인 함수 수, 전체 함수 수가 나옵니다. `--all-functions`는 함수마다 해시와 counter를 더하고, `--topn N`은 내부 block count가 가장 큰 함수 N개를 나열합니다.

### `rokc emit`: 중간 산출물

```
rokc emit [path] [--emit=<kind>[,<kind>..]]
```

`rokc build --emit=..`와 같습니다. `--emit`을 생략하면 `mlir`입니다.

### `rokc run`: 빌드 후 실행

`rokc run`은 빌드한 산출물을 바로 실행합니다.

```
rokc run [path] [-- args...]
```

실행 패키지를 빌드한 뒤 곧바로 실행합니다. `--` 뒤에 적은 인자는 실행 파일에 그대로 전달됩니다. 옵션은 `rokc build`와 같습니다. 라이브러리 패키지에는 실행할 것이 없으므로 에러입니다.

패키지는 `target/<profile>/<name>`에 빌드합니다. 익명 패키지(단일 파일)는 임시 디렉터리에 빌드하므로 파일 옆에 `target/`을 만들지 않습니다. `rokc build <file>.rok`은 같은 방식으로 빌드하고 실행 파일을 파일 옆에 복사합니다(위의 `rokc build`).

```
rokc run hello.rok -- arg1 arg2
```

`rokc run`의 종료 코드는 프로그램의 종료 코드입니다. 프로그램이 시그널 `n`으로 끝나면 `128 + n`입니다.

### `rokc test`: 테스트

`rokc test`는 테스트 함수를 모아 테스트 하네스를 빌드합니다. 실행은 `rok test`([9.10](910-rok-frontend.md))가 합니다.

```
rokc test [path] [filter]
```

`#[test]`가 붙은 함수를 모아 테스트 실행 파일(하네스)을 빌드합니다. `#[test]`는 `#[cfg(test)]`를 함의하므로([6.2.5](../006-meta-system/602-built-in-attributes.md#625-test-테스트-함수-마커)) 테스트 함수는 일반 빌드에 들어가지 않습니다. 테스트는 `src/`(`#[cfg(test)]` 아래)와 `tests/` 디렉터리([0.3.5](../000-meta/003-source-units.md#035-테스트와-벤치마크-디렉터리)) 양쪽에서 모읍니다. 테스트 빌드는 패키지의 진입점 대신 테스트를 고르는 진입점을 씁니다.

`rokc`는 하네스를 실행하지 않습니다. `--emit-harness <file>`은 빌드한 실행 파일과 그 안의 테스트(`rok test`가 실행하는 매니페스트)를 `<file>`에 씁니다. 이 옵션을 주지 않으면 빌드한 실행 파일을 표준 에러에 알립니다. 하네스 실행(테스트 선택, 아래의 프로세스 격리, 병렬 실행, 보고)은 `rok test` 러너([9.10](910-rok-frontend.md))가 맡습니다.

인자가 하나이고 그 인자가 존재하는 경로가 아니면 `filter`로 봅니다. `filter`는 모듈 경로를 포함한 테스트 이름(`math::adds`)과 맞춰 봅니다.

- `^` `$` `.` `*` `+` `?`와 `\` 이스케이프를 쓰는 작은 정규식입니다. 고정하지 않은 검색이므로 `^` / `$`가 없으면 이름의 어느 부분과 맞아도 됩니다.
- 이 문자들이 없으면 단순 부분 문자열 검색과 같습니다.

**각 테스트는 자기 프로세스에서 실행됩니다.** 테스트가 panic으로 끝나거나(종료 코드 101) 잡히지 않은 에러로 끝나면(종료 코드 1) 그 테스트만 실패하고, 나머지는 계속 실행됩니다. 시그널로 끝나도 실패입니다.

옵션은 `rokc build`와 같고, 다음 두 가지가 더 있습니다.

- `--threads=N`은 동시에 실행할 테스트 프로세스 수를 지정합니다. 기본값은 `ROKC_NUM_JOBS`이고, 그것도 없으면 CPU 수입니다.
- `--show-output`은 성공한 테스트의 stdout / stderr도 표시합니다. 실패한 테스트의 출력은 늘 표시합니다.

출력은 stdout에 다음 형태로 나옵니다.

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

실패한 테스트가 하나라도 있으면 종료 코드는 `1`입니다.

루트 파일이 둘 다 있는 패키지에서는 라이브러리의 테스트를 먼저 실행합니다. 그다음 실행 파일의 테스트를 실행하며, 이 테스트는 테스트를 뺀 라이브러리에 대해 빌드합니다. 라이브러리 실행이 실패하면 명령이 끝납니다. `run`과 `bench`도 같은 방식입니다.

### `rokc bench`: 벤치마크

```
rokc bench [path] [filter]
```

`#[bench]`가 붙은 함수를 모아 최적화 빌드(`-O`를 주지 않으면 `-O2`)로 빌드합니다. 하네스는 `rok bench`([9.10](910-rok-frontend.md))가 실행해 측정합니다([9.9.11](909-debug-tools.md#9911-벤치마크)). `#[bench]`는 `#[test]`처럼 `#[cfg(test)]`를 함의합니다. 벤치는 `src/`와 `benches/` 디렉터리([0.3.5](../000-meta/003-source-units.md#035-테스트와-벤치마크-디렉터리)) 양쪽에서 모읍니다. `path`, `filter`, `--emit-harness` 규칙은 `rokc test`와 같습니다.

벤치마크는 `Bench` 핸들을 받아 클로저를 측정합니다. 그래서 준비 코드는 측정 구간 밖에 둡니다.

```rok
#[bench]
fn parse_1kb(b: Bench) {
    let input = sample(1024);               // setup: 한 번만, 측정 안 함
    b.bytes(1024);                          // 선택: 처리량 보고
    b.iter(|| parse(black_box(input)));     // 측정 구간
}
```

`b.iter(f)`가 측정 전체(보정, 예열, 표본 수집)를 맡고, 결과는 black_box를 거칩니다. `black_box`([`rok::core`](../../refs-standard-libraries/README.md))는 옵티마이저가 꿰뚫어 볼 수 없는 항등 함수입니다. 측정 구간의 입력과 결과에 적용하면 작업이 구간 밖으로 끌려 나가거나 상수로 접히지 않습니다. `b.bytes(n)` / `b.items(n)`은 처리량 열을 더합니다. `Bench`, `black_box`, 통계는 라이브러리 API이고, 측정 방법은 [9.9.11](909-debug-tools.md#9911-벤치마크)에서 정의합니다.

- `--iters=N`은 보정 대신 표본 하나의 호출 횟수를 고정합니다.
- `--samples=N`은 측정할 표본 수입니다(기본값 10).
- `--format=human|rokbench`는 출력 형식을 고릅니다(기본값 `human`).

벤치는 하나씩 차례로 실행됩니다. 각 벤치는 터미널이든 아니든 사람이 읽는 단위로 정렬한 한 줄을 출력합니다.

```
  json::parse_1kb     1.84 us +/- 0.03 us   543 MiB/s   [2 outliers, 0 gc]
```

`--format=rokbench`를 주면 그 대신 회귀 도구가 읽는 안정된 기계용 형식을 벤치마다 한 줄씩 출력합니다. 형식은 `#rokbench name=<name> median_ns=<f> mad_ns=<f> samples=<n> outliers=<n> gc=<n> bytes=<n> items=<n>`입니다.

`b.iter`를 한 번도 부르지 않은 벤치는 아무것도 측정하지 않은 것입니다. 실패는 아닙니다. 그 줄은 `` <name>  warning: measured nothing: time the code with `b.iter(|| ..)` ``이고, 요약은 그 벤치를 실행한 수가 아니라 `<n> measured nothing`으로 셉니다.

### `rokc repl`: 터미널 REPL

`rokc repl`은 터미널에서 인터랙티브 셸을 띄웁니다.

```
rokc repl [path]
```

셀의 의미는 [9.7.0](907-jit-cells.md#970-셀-세션의-규칙)의 규칙을 따릅니다. 각 셀은 한 번만 실행됩니다. 최상위 `let` 바인딩과 항목은 이후 셀로 이어지고, 실패한 셀은 바인딩에 흔적을 남기지 않습니다. 식 셀은 값을 출력합니다. `Display`로 출력하고, 그것이 없으면 `Debug`로, 둘 다 없으면 타입 이름을 출력합니다.

- 기본 모드는 각 셀을 native 백엔드로 컴파일해, 세션 동안 사는 JIT 호스트 프로세스에 넘깁니다. 호스트는 셀을 로드해 한 번 실행합니다([9.6](906-jit-engine.md)). 바인딩, `static` 상태, `any` 값이 셀 사이에 이어집니다. 최상위에서 이름을 적을 수 없는 타입(클로저, 비공개 타입 등)의 바인딩은 만들 때 거부합니다.
- 진단은 셀 자신의 텍스트를 기준으로 `<repl>:line:col`로 표시합니다. 앞 셀의 항목과 실행 중 위치는 `<repl cell N>:line:col`로 표시합니다([9.7.5](907-jit-cells.md#975-에러-모드)).
- 터미널에서는 줄 편집, Tab 자동 완성, 색, 셀 단위 기록을 지원합니다([9.7.8](907-jit-cells.md#줄-편집)). 기록은 `$ROK_HOME/repl_history`에 남습니다.
- 터미널에서는 세션이 toolchain 줄과 안내 한 줄로 시작합니다([9.7.8](907-jit-cells.md#줄-편집)). `-q`를 주면 이 줄을 생략합니다.
- 경로가 없으면 빈 세션으로 시작합니다. 프로그램 경로(단독 `.rok` 파일이나 `src/main.rok`이 있는 패키지)를 주면 로드해 실행하고 세션을 이어 갑니다. 이때 최상위 문은 셀로, `fn main`은 진입점으로 실행합니다([9.7.10](907-jit-cells.md#9710-프로그램-로드repl-path)). 라이브러리 패키지 경로(`src/lib.rok`)는 의존성으로 붙이고, `use <name>::*;`를 첫 셀로 넣습니다. `:reset` 뒤에도 다시 넣습니다.
- 패키지 안에서 경로 없이 시작하면 그 패키지를 로드하지 않습니다. 찾지 못한 이름을 그 패키지가 정의하고 있으면 ``load this package: `rok repl .` `` 힌트를 붙입니다.
- 경로가 둘 이상이면 사용법 에러입니다. `-`는 받지 않습니다. 표준 입력은 셀 입력으로 씁니다.

루트 파일이 둘 다 있는 패키지는 프로그램 경로로 다루고, 그 라이브러리는 세션의 의존 패키지가 됩니다. `rok`과 `rok repl`([9.10](910-rok-frontend.md))은 의존성을 붙여 같은 REPL을 실행합니다.

REPL 명령(`:type`, `:doc`, `:time`, `:vars`, `:load`, `:reload`, `:edit`, `:save`, `:opt`, `:reset`, `:add`, `:help`, `:quit` / `:exit`, 그리고 단독 `exit` / `quit`)과 디버그 수단은 [9.9.9](909-debug-tools.md#999-repl--jupyter-디버그)에 있습니다.

### `rokc jupyter`: Jupyter 커널

`rokc jupyter`는 Jupyter가 호출하는 커널 모드입니다.

```
rokc jupyter --connection-file <path>
```

사용자가 직접 실행하는 명령이 아닙니다. `--connection-file`은 필수이고, Jupyter가 제공하는 ZMQ 연결 정보 파일의 경로를 받습니다. 이 옵션이 없으면 사용법 에러입니다. `rok`을 거치면 `rok jupyter install`을 알려 주는 에러가 납니다. `--extern` 같은 빌드 옵션도 함께 받습니다. 커널은 REPL과 같은 셀 세션([9.7.0](907-jit-cells.md#970-셀-세션의-규칙)) 위에서 동작하고, 시작한 곳의 패키지를 `rokc repl .`처럼 로드합니다([9.8.2](908-jupyter-integration.md#982-로컬-네이티브-커널-동작)). 의존성을 해소해 실행하려면 `rok jupyter --connection-file <path>`([9.10](910-rok-frontend.md))를 쓰세요. 자세한 통합 내용은 [Jupyter 통합](908-jupyter-integration.md)에서 정의합니다.

커널 등록은 `rok jupyter install [--env <name>]`이 맡습니다([9.8.3](908-jupyter-integration.md#983-kernelspec-설치)).

### `rokc fmt`: 포매터

`rokc fmt`는 소스에 표준 포맷을 적용합니다.

```
rokc fmt [path]
```

`[path]`는 파일이나 디렉터리를 받고(여러 개 가능), [표준 포맷](#9111-표준-형식rokc-fmt)에 맞게 그 자리에서 고쳐 씁니다. 패키지인 디렉터리(9.2.2)는 `src/`, `tests/`, `benches/`, `examples/` 디렉터리와 `rok.rokon`을 포맷하고, 그 안의 다른 파일은 건드리지 않습니다. 경로가 없으면 현재 디렉터리가 속한 패키지를 포맷합니다. 그 패키지는 현재 디렉터리나 그 위에서 `src/main.rok`, `src/lib.rok`, `rok.rokon` 중 하나가 있는 가장 가까운 디렉터리입니다. 패키지 밖에서 경로 없이 실행하면 에러(종료 코드 `1`)이고, 메시지는 `` not inside a package; pass a file or directory: `rokc fmt <path>` ``입니다(`rok`을 거치면 `rok fmt`). 디렉터리는 재귀로 `.rok`과 `.rokon` 파일을 찾습니다. 이때 `target/`, `node_modules/`, `.`으로 시작하는 디렉터리, 그리고 찾는 디렉터리나 그 패키지 루트의 `.gitignore`가 무시하는 것은 건너뜁니다. `.gitignore`에서는 `name`(아래 어디서든 그 이름), `name/`(디렉터리만), `/name`(`.gitignore`가 있는 디렉터리 바로 아래), `*suffix` 형태의 줄만 읽습니다. 주석, `!` 줄, 다른 와일드카드, 중간에 `/`가 있는 경로는 건너뜁니다. 터미널에서는 찾는 데 1초가 넘게 걸리면 표준 에러에 `Scanning <n> files…`를 보여 줍니다. `.rokon` 파일([ROKON](../../refs-standard-libraries/300-std/309-rokon.md#394-정규-형식))은 ROKON인지 검사하고 들여쓰기만 다시 맞춥니다. 정규 형식으로 다시 쓰지 않으므로 주석이 남습니다.

`-q`가 아니면 마지막에 한 일을 표준 에러로 알립니다. 메시지는 `Formatted <n> files`(이미 포맷된 파일이 있으면 `, <m> already formatted`를 덧붙임), `note: <n> files already formatted` 가운데 하나입니다. `--check` / `--diff`이면 `` note: <n> files need formatting; `rokc fmt` without `--check` rewrites them ``입니다. 포맷할 것이 없는 디렉터리는 `note: no .rok or .rokon files under <dir>`입니다(종료 코드 `0`).

옵션은 다음과 같습니다.

- `--check`는 파일을 바꾸지 않고, 바뀔 파일과 그 diff를 출력합니다. 바뀔 파일이 있으면 종료 코드는 `1`이므로 CI에서 쓰기 좋습니다.
- `--diff`는 파일을 바꾸지 않고 diff만 출력합니다.

어휘 에러가 있는 파일은 건드리지 않고 에러를 보고합니다. 종료 코드는 `1`입니다.

### `rokc check`: 진단

```
rokc check [path] [--check-library]
```

프런트엔드를 타입 검사와 MIR 생성까지 돌리고, 린트와 함께 진단을 냅니다. 코드 생성(MLIR 포함)은 하지 않습니다. 옵션은 `rokc build`와 같습니다. `--check-library`는 도달 여부와 상관없이 라이브러리의 모든 본문을 낮추고 검사하며, 모든 제네릭 본체를 그 바운드에 맞춰 검사합니다([2.10.9](../002-types/210-generics.md#2109-본체-검사)). 루트 파일이 둘 다 있는 패키지는 먼저 라이브러리를 검사합니다. 라이브러리에 에러가 없으면 라이브러리를 함께 컴파일한 실행 파일을 검사합니다. 각 경고는 한 번만 보고합니다.

### `rokc lint`: 린터

```
rokc lint [path] [--list]
```

`rokc check`에서 코드 생성을 뺀 것입니다. 컴파일하는 모든 명령이 같은 린트를 보고합니다([6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)). `lint`는 린트와 프런트엔드만 돌립니다. 진단 형식은 9.1.8을 따르고, 린트 진단의 코드는 `E503`입니다. 린트 목록과 그룹은 [6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)에 있습니다. `--list`는 린트와 그룹, 기본 수준을 출력합니다.

린트 수준은 전역 옵션 `-A` / `-W` / `-D`(9.1.5)와 코드의 속성이 정합니다. `deny` 수준의 린트가 하나라도 걸리면 종료 코드는 `1`입니다.

### `rokc doc`: 문서 생성

```
rokc doc [path] [--std] [--open]
```

패키지의 라이브러리(없으면 실행 파일)와 `--extern` 의존 패키지의 공개 항목과 문서(`///`, `//!`, `#[doc]`, [6.2.8](../006-meta-system/602-built-in-attributes.md#628-doc---문서-주석-형태))로 HTML API 문서를 `target/doc/`에 만듭니다.

```
target/doc/
├── index.html                 # 패키지 목록
├── <pkg>/index.html           # 패키지의 루트 모듈
├── <pkg>/<mod>/index.html     # 모듈마다 한 쪽
└── search-index.js            # 검색용 색인
```

`<pkg>`는 패키지 이름(매니페스트의 `name`)이고, 표준 라이브러리 계층은 `rok-core` 등입니다. 공개 모듈마다 모듈 경로를 따라 `<pkg>/<mod>/.../index.html` 한 쪽이 있습니다. 그 쪽의 공개 항목마다 앵커 `#<kind>.<name>`이 붙고, `<kind>`는 `struct`, `enum`, `union`, `trait`, `type`, `fn`, `const`, `static` 중 하나입니다(`app/index.html#struct.Point`). 멤버의 앵커는 `#structfield.<Type>.<field>`(`union`은 `#unionfield.`), `#variant.<Type>.<variant>`, `#method.<Type>.<method>`, `#tymethod.<Trait>.<method>`, `#associatedtype.<Trait>.<name>`입니다. 이 URL 체계는 안정적입니다. docs.rok-lang.org 등 생성된 문서를 가리키는 링크는 툴체인 버전이 바뀌어도 계속 동작합니다.

`--std`를 주면 표준 라이브러리 계층도 `rok-core/`, `rok-std/`, `rok-bare/`로 문서화합니다. 경로가 없으면 표준 라이브러리 문서만 `./target/doc/`에 만듭니다.

`--open`은 문서를 만든 뒤 `index.html`을 엽니다. `$BROWSER`가 있으면 그것으로, 없으면 시스템의 opener(macOS는 `open`, Linux는 `xdg-open`, Windows는 `start`)로 엽니다. opener를 시작할 수 없으면 직접 열 파일을 경고로 알립니다. 종료 코드는 바뀌지 않습니다.

문서 텍스트는 markdown의 부분집합(제목, fenced 코드 블록, 인라인 코드, 링크, `**굵게**`, 목록, 문단)으로 해석합니다.

### `rokc lsp`: LSP 서버

`rokc lsp`는 Language Server Protocol 서버 모드로 동작합니다.

```
rokc lsp
```

이 모드는 VS Code, Vim, Emacs 같은 LSP 클라이언트가 호출하는 용도입니다. 에디터용 독립 바이너리 `rokls`는 `rok lsp`(곧 `rokc lsp`)와 같습니다. `.rokon` 버퍼는 구문만 검사합니다. 루트 파일이 둘 다 있는 패키지에서는 라이브러리의 모듈 트리가 닿는 파일을 라이브러리의 일부로 검사하고, 그 밖의 파일은 실행 파일의 일부로 검사합니다. 실행 파일은 라이브러리의 열린 버퍼를 봅니다. 기능 목록은 [9.9.13](909-debug-tools.md#9913-lsp-서버)에 있습니다.

## 9.1.3. 디버그 / 진단 서브명령

### `rokc bindgen`: C 헤더 바인딩

```
rokc bindgen <header> [--target <t>] [-o <file>] [-- <clang args>]
```

`<header>`의 선언을 [std 1.7.6](../../refs-standard-libraries/100-bare/107-ffi.md#176-자동-바인딩-생성) 형식으로 표준 출력(또는 `-o <file>`)에 씁니다. libclang은 먼저 toolchain의 `lib/`([9.2.19](902-aot-pipeline.md#9219-설치-레이아웃))에서 찾고, 다음으로 LLVM 캐시의 `libclang-<version>/lib/`에서 찾습니다. LLVM 캐시는 `ROK_LLVM_CACHE`가 있으면 그 경로이고, 없으면 `$XDG_CACHE_HOME/rok`이나 `~/.cache/rok`(Windows에서는 `%LOCALAPPDATA%\rok`)입니다. 둘 다에 없으면 에러(exit 1)이고, 메시지에 찾아본 곳을 알립니다.

### `rokc demangle`: 심볼 디맹글링

`rokc demangle`은 맹글링된 심볼을 사람이 읽을 수 있는 형식으로 풀어 출력합니다.

```
rokc demangle <symbol>...
```

심볼마다 [9.5.4](905-symbol-mangling.md) 형식의 블록을 빈 줄로 나눠 출력합니다([9.9.3](909-debug-tools.md#993-심볼-디맹글링)). Rok 심볼이 아닌 인자가 있으면 종료 코드는 `1`입니다.

### `rokc rokhir-text`: [ROKHIR](../000-meta/007-glossary.md#rokhir-rok-hir-bytecode) 텍스트 출력

`rokc rokhir-text`는 미리 빌드한 라이브러리의 ROKHIR을 텍스트로 출력합니다.

```
rokc rokhir-text path/to/foo.rokhir      # 또는 libfoo.a. 옆의 foo.rokhir 를 읽음
```

헤더 필드, producer, meta와 interface 파일을 출력합니다. 포맷은 [ROKHIR 포맷](904-rokhir-bytecode.md)에서 정의합니다.

### `rokc inspect`: 패키지 검사

`rokc inspect`는 아카이브의 내용을 보여 줍니다.

```
rokc inspect path/to/libfoo.a        # 또는 foo.rokhir
```

패키지 이름과 해시, rokc 버전, ROKHIR 버전, GC type id 구간, 내보낸 항목, [제네릭](../000-meta/007-glossary.md#generic--제네릭) 항목, 아카이브 멤버와 크기를 한 번에 표시합니다([9.9.14](909-debug-tools.md#9914-패키지-검사)).

## 9.1.4. 보조 서브명령

### `rokc clean`: 빌드 산출물 제거

`rokc clean`은 빌드 산출물을 지웁니다.

```
rokc clean [path]
```

패키지의 `target/` 디렉터리 전체를 지웁니다. 모든 프로필과 교차 빌드가 포함되고, `--target-dir` / `ROKC_TARGET_DIR`이 있으면 그 디렉터리를 지웁니다. 디렉터리가 없으면 아무 일도 하지 않습니다.

### `rokc version`: 버전

```
rokc version
rokc --version            # = -V
rokc -V -v                # host 줄 추가
```

stdout에 `rokc <버전> (<commit> <date>)` 한 줄을 출력합니다. `-v`를 더하면 `host: <triple>`과 `llvm: <version>` 줄을 덧붙입니다.

### `rokc --print cfg`: 활성 cfg

```
rokc --print cfg [--target=<target>] [-O<n>] [--cfg ..]
```

옵션이 가리키는 빌드의 cfg([6.3](../006-meta-system/603-cfg.md))를 한 줄에 하나씩 출력합니다. 플래그는 `flag`, 키는 `key="value"` 형태입니다(rustc와 같음). 키에는 `target_env`가 있고, `debug_assertions`는 `-O0`에서 켜집니다.

```
debug_assertions
target_arch="aarch64"
target_endian="little"
target_family="unix"
target_os="macos"
target_pointer_width="64"
```

### `rokc --print target-list` / `target-spec`: 타깃

```
rokc --print target-list
rokc --print target-spec [--target=<target>]
```

`target-list`는 내장 타깃 이름을 한 줄에 하나씩 출력합니다. `target-spec`은 타깃의 사양을 ROKON으로 출력합니다([0.6.5](../000-meta/006-platform-targets.md#065-타깃-사양)). `--target`이 없으면 호스트의 사양을 출력합니다.

### `rokc --print lldbinit`: lldb 포매터

이 toolchain의 lldb formatter([9.9.6](909-debug-tools.md#996-디버거-통합-gdb--lldb))를 로드하는 한 줄 `command script import '<path>'`을 출력합니다. 경로는 `<sysroot>/lib/rok/lldb/rok_lldb.py`입니다.

### `rokc completions`: 셸 자동완성

```
rokc completions bash|zsh|fish|powershell
```

그 셸의 자동 완성 스크립트를 stdout에 출력합니다. 스크립트는 `rokc help`와 같은 명령 / 옵션 텍스트에서 만들므로 둘은 어긋나지 않습니다. 서브명령과 그 옵션을 완성하고, 설치된 toolchain으로 `+<version>`(`rok`에서, [9.10.6](910-rok-frontend.md#9106-toolchain))을, `--print target-list`로 `--target`의 값을 완성합니다. 그 밖의 셸이거나 셸을 주지 않으면 사용법 에러(종료 코드 2)입니다.

### `rokc help`: 도움말

```
rokc help [subcommand]
rokc <subcommand> --help
```

서브명령 없이 실행하면 전체 참조(모든 서브명령, 옵션, 환경 변수)를 출력합니다. 서브명령을 주면 그 서브명령만의 도움말을 출력합니다. 도움말에는 사용 형태, 짧은 설명, 예제 하나 이상, 받는 옵션이 들어갑니다. 어느 쪽도 다른 서브명령의 텍스트는 출력하지 않습니다.

## 9.1.5. 전역 옵션

전역 옵션은 다음과 같습니다. 서브명령 앞뒤 어디에나 둘 수 있습니다.

| 옵션                             | 의미                                                        |
|----------------------------------|-------------------------------------------------------------|
| `--verbose` / `-v`               | 단계별 시간 표시([9.9.5](909-debug-tools.md#995-로그--진행-표시)) |
| `-vv`                            | 더 자세한 로그. `Reading <file>`과 `Specialized <path>(sig)` 줄([9.9.5](909-debug-tools.md#995-로그--진행-표시)) |
| `--quiet` / `-q`                 | 진행 줄을 출력하지 않음(진단은 출력)                       |
| `--color=auto\|always\|never`    | stdout과 stderr의 색(9.1.8)                              |
| `--no-color`                     | `--color=never`와 같음                                     |
| `--cwd=<path>`                   | 작업 디렉터리를 바꾼 뒤 실행                                |
| `--print cfg\|target-list\|target-spec\|lldbinit` | 활성 cfg, 내장 타깃 이름, 타깃 사양, 또는 Rok의 formatter를 로드하는 lldb 명령을 출력(9.1.4) |
| `--message-format=human\|json`   | 진단 형식. `json`은 stdout에 한 줄씩(9.1.8)              |
| `-A` / `-W` / `-D <lint \| group>` | 린트 수준(`--allow` / `--warn` / `--deny`). 패키지를 컴파일하는 모든 명령(`build`, `run`, `test`, `bench`, `emit`, `check`, `lint`, `lsp`)에 적용. 여러 번 쓸 수 있고 뒤의 것이 이김 |

린트 수준은 린트의 기본 수준 < `-A` / `-W` / `-D` < 코드의 `#[allow(..)]` / `#[warn(..)]` / `#[deny(..)]`(가장 안쪽이 이김) 순으로 정합니다. `rok`은 매니페스트의 `lints`를 이 플래그로 바꿔 앞에 붙여 넘깁니다([9.10.1](910-rok-frontend.md#9101-서브명령과-전역-옵션)).

색은 stdout과 stderr에 따로 다음 순서로 정합니다. `--color=never`면 끄고 `--color=always`면 켭니다. 그 밖(`--color=auto`, 기본값)에는 `NO_COLOR`가 비어 있지 않으면 끄고, `CLICOLOR_FORCE`가 비어 있지 않고 `0`도 아니면 켭니다. `TERM=dumb`면 끄고, 아니면 그 스트림이 터미널일 때 색을 씁니다. Windows에서 escape sequence를 받지 않는 콘솔에는 색 없는 글자를 보냅니다.

사용 예는 다음과 같습니다.

```
rokc --verbose build
rokc test -q
rokc --cwd=examples/hello run
```

## 9.1.6. 종료 코드

종료 코드는 다음과 같습니다.

| 코드  | 의미                                            |
|-------|-------------------------------------------------|
| `0`   | 성공                                            |
| `1`   | 사용자 코드 에러(컴파일 에러 / 테스트 실패 등) |
| `2`   | 사용법 에러(잘못된 옵션, 모르는 서브명령 등)   |
| `101` | rokc 내부 에러(panic)                          |

사용법 에러는 `rokc: <메시지>` 다음에 ``run `rokc help <subcommand>` for usage`` 줄을 출력합니다(알려진 서브명령이 없으면 ``run `rokc help` for usage``). 메시지는 대신 쓸 것을 알려 줍니다. 모르는 서브명령이나 옵션은 가장 가까운 것이 있으면 그 이름을 댑니다(``unknown option `--relase`; did you mean `--release`?``). 파일을 가리키는(`/`를 담거나 `.rok`으로 끝나는) 모르는 서브명령에는 `rokc run <file>`을 제안합니다. `ROK_CLI`가 `rok`이면(9.1.7) 두 줄 모두 `rokc` 대신 `rok`이라고 씁니다.

`rokc`는 `ROK_HOME`을 읽지 않습니다(sysroot는 실행 파일 기준). rokc 자신이 panic으로 끝나면(internal compiler error) 메시지를 출력하고 `101`로 끝납니다. `rokc lsp`, `rokc repl`, `rokc jupyter`는 이 panic을 견딥니다. language server는 이를 분석한 버퍼의 에러 진단으로 보고하고, REPL / 커널은 그 셀을 실패로 처리합니다([9.7.5](907-jit-cells.md#975-에러-모드)). 둘 다 계속 서비스합니다. `rokc run`은 실행한 프로그램의 종료 코드로 끝납니다(9.1.2).

표준 출력이나 표준 에러가 읽는 쪽이 사라진 파이프이면 `rokc`는 메시지 없이 멈춥니다. Unix에서는 `SIGPIPE`로 종료되고(셸은 `141`을 보고), Windows에서는 `1`로 끝납니다. 이 경우는 내부 에러가 아닙니다.

## 9.1.7. 환경 변수

`rokc`의 동작에 영향을 주는 환경 변수는 다음과 같습니다.

| 변수              | 의미                                                                                     |
|-------------------|------------------------------------------------------------------------------------------|
| `ROK_CC`          | 내장 lld 대신 마지막 링크를 하는 선택적 C 컴파일러 드라이버([9.2.14](902-aot-pipeline.md#9214-링킹실행-파일의-경우)) |
| `ROK_BACKEND`     | `native` 또는 `llvm`. `-C backend=`와 같음 |
| `ROKC_VERBOSE`    | `--verbose`와 같음                                                                      |
| `ROK_TIME_PASSES` | `1`이면 패스별 실제 시간을 stderr에 출력([9.9.5](909-debug-tools.md#rok_time_passes)) |
| `LLVM_PROFILE_FILE` | 계측한 프로그램(`-C profile-generate`)이 읽는 raw profile 경로 |
| `ROK_MLIR_PLUGIN_PATH` | 이름만 준 `--load-mlir-plugin` / `--load-mlir-pass-plugin`을 찾는 디렉터리 목록(`PATH`처럼 구분) |
| `ROK_LLVM_CACHE` | `rokc bindgen`이 libclang을 찾는 캐시 디렉터리(9.1.3) |
| `ROK_ENV_LIBS`  | `rok`이 환경에서 스크립트를 실행할 때 설정하는, 그 환경에 설치된 라이브러리 목록(쉼표로 구분). 스크립트가 그중 하나를 선언 없이 쓰면 그 이름을 알리는 힌트를 받음 |
| `ROK_CLI`         | `rok`이 `rokc`를 실행할 때 `rok`으로 설정(내부용). 사용법 에러(9.1.6), `--help`, 에러와 경고 줄의 `rokc:` 접두사에서 `rokc` 대신 `rok`이라고 씀. `rok run`이 시작한 프로그램도 물려받음 |
| `ROKC_TARGET_DIR` | `target/`의 위치(기본값은 패키지 안)                                                     |
| `ROKC_INCREMENTAL` | 프런트엔드가 패키지의 파싱 결과와 낮춘 본체를 두는 곳(기본값 `target/<profile>/incremental/`, `0`이면 두지 않음). 라이브러리에서 모은 항목은 `lib/rok-cache/library/`에 둠([9.2.16](902-aot-pipeline.md#9216-증분-빌드)) |
| `ROKC_INCR_VERIFY` | `1`이면 모든 본체를 다시 낮추고, 결과가 다르면 실패([9.2.16](902-aot-pipeline.md#9216-증분-빌드)) |
| `ROKC_NUM_JOBS`   | `rokc test --threads`의 기본값(없으면 CPU 코어 수)                                     |
| `NO_COLOR`        | 비어 있지 않으면 `--color=auto`가 색을 쓰지 않음(9.1.5)                                |
| `CLICOLOR_FORCE`  | 비어 있지 않고 `0`이 아니면 스트림이 터미널이 아니어도 `--color=auto`가 색을 씀. `NO_COLOR`가 있으면 쓰지 않음(9.1.5) |
| `COLORTERM`       | `truecolor`나 `24bit`이면 24비트 색 값을 씀(9.1.8)                                        |
| `TERM`            | `dumb`이면 `--color=auto`가 색을 쓰지 않음. `256color`를 포함하면 256색 팔레트를 씀(9.1.8) |

설정하지 않았거나 비어 있는 변수는 기본값을 씁니다. 표에 없는 값을 주면 stderr에 경고 한 줄을 출력하고, 그 변수는 기본값을 씁니다. 경고에는 변수, 값, 받는 값이 나옵니다.

## 9.1.8. 출력 형식

### 진단 메시지

진단 메시지는 다음 형식으로 출력합니다.

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

진단은 [소스 파일](../001-lexical/101-source-files.md)에서 정의한 파일 + 줄 + 컬럼 단위를 그대로 씁니다.

머리줄은 `<severity>[<code>]: <message>`이고, `--> path:line:col`이 주 위치입니다. 주 span은 `^`로, 보조 span은 `-`로 밑줄을 긋습니다. 다른 파일의 보조 span은 `::: path:line:col`로 시작합니다. 끝의 `= note:` 줄은 맥락을, `= help:` 줄은 고치는 방법을 덧붙입니다. 메시지는 이 사양의 절을 인용하지 않습니다. 관련 절은 JSON 형식이 나열합니다([9.9.4](909-debug-tools.md#994-진단--에러-메시지)).

### 빌드 진행

진행 상황은 stderr에 오른쪽으로 정렬한 동사로 표시합니다. `-q`면 출력하지 않습니다.

```
   Compiling std (-Os, cached for next builds)
   Compiling json v0.1.0
   Compiling foo (path/to/foo)
    Building target/debug/foo
    Finished in 1.23s
     Running `target/debug/foo`
```

| 동사        | 시점                                      |
|-------------|-------------------------------------------|
| `Compiling` | 프런트엔드 시작(패키지 이름과 디렉터리). `check`와 `lint`는 `Checking`, `doc`은 `Documenting`. `rok`이 가져온 의존 패키지(`$ROK_HOME/cache/` 아래)는 대신 `<name> v<version>`을 보임. 옵션 집합마다 한 번 캐시용으로 빌드하는 표준 라이브러리([9.2.16](902-aot-pipeline.md#9216-증분-빌드))는 `std (<-O 수준>, cached for next builds)`를 보임 |
| `Building`  | 실행 파일 / 아카이브 생성(`build`)        |
| `Built`     | 단독 파일의 실행 파일을 그 옆에 복사함(`build`). `Built hello (use -o to put it elsewhere)`. 테스트 / 벤치 harness를 `--emit-harness` 없이 빌드함(`test`, `bench`). 뒤에 그것을 실행할 `rok test`나 `rok bench`를 알리는 note가 따라옴 |
| `Wrote`     | `--emit` 산출물 하나를 씀                 |
| `Std`       | `-v`일 때 쓰는 표준 라이브러리. `prebuilt (<path>)`나 `from source` |
| `Finished`  | 빌드 끝(명령 시작부터 걸린 시간, 표준 라이브러리 빌드 포함) |
| `Running`   | 프로그램 실행(`run`). 현재 디렉터리 기준의 실행 파일. 단독 파일은 파일 이름(`Running hello.rok`), `-v`면 캐시된 실행 파일 |
| `Created`   | 패키지 생성(`new`). 그 아래에 이어서 실행할 명령을 보임([9.10.2](910-rok-frontend.md#9102-rok-new--rok-init-패키지-생성)) |
| `Removed`   | `target/` 삭제(`clean`)                  |
| `Fresh`     | 최신. 그 빌드의 경고가 뒤따름             |
| `Checked`   | `check` 끝(에러와 경고 수)               |
| `Generated` | 문서를 씀(`doc`)                         |
| `Formatted` | 파일을 고쳐 씀(`fmt`). 끝에 그 수를 알림(위 `rokc fmt`) |

프런트엔드에서 실패한 빌드(`build`, `run`, `test`, `bench`, `doc`)는 진단 뒤에 `error: could not build <name> (<n> error(s), <m> warning(s))`로 끝납니다. 수는 `check`가 세는 방식과 같습니다. `-q`와 `--message-format=json`에서는 이 줄을 생략합니다.

`rok`도 자신의 진행을 같은 방식으로 출력합니다. `Added`, `Installed`, `Updated`, `Downloading` 등이 있습니다([9.10](910-rok-frontend.md)).

### 색

색이 켜지면(9.1.5) 출력은 이 팔레트를 씁니다. 색 깊이는 터미널을 따릅니다. `COLORTERM`이 `truecolor`나 `24bit`면 24비트 값을 씁니다. 아니고 `TERM`이 `256color`를 포함하면 256색 팔레트를 쓰고, 그 밖에는 기본 16색을 씁니다.

| 부분                                  | 색                       | 24비트      | 256   | 16             |
|---------------------------------------|--------------------------|-------------|-------|----------------|
| 상태 동사(`Compiling`), 목록 표시 `*` | 하늘색, 굵게            | `#38bdf8`   | `81`  | bright cyan    |
| `error`                               | 빨강, 굵게               | `#ff5f5f`   | `203` | bright red     |
| `warning`                             | 호박색, 굵게             | `#e0b057`   | `179` | yellow         |
| `note`                                | 하늘색                   | `#38bdf8`   | `81`  | bright cyan    |
| `help`                                | 연분홍                   | `#f9a8d4`   | `218` | bright magenta |
| 진단의 거터(`-->`, `\|`, 줄 번호)     | 하늘색, 굵게             | `#38bdf8`   | `81`  | bright cyan    |

색만으로 뜻을 나타내지 않습니다. 라벨(`error:`, `warning:`, `note:`, `help:`)과 상태 동사는 색이 있든 없든 출력합니다. 항목 하나를 표시하는 목록(`rok toolchain list`, `rok env list`)은 그 항목 앞에 `*`를, 뒤에 `(default)`나 `(active)`를 붙입니다. 파이프로 보낸 출력, `--message-format=json`, `-q`는 영향을 받지 않습니다.

### JSON 출력(도구 통합용)

도구와 통합할 때는 JSON 형식으로도 출력할 수 있습니다.

```
rokc build --message-format=json
```

이 모드에서는 진단 하나가 stdout에 한 줄짜리 JSON 객체로 나옵니다. 그래서 IDE와 CI 도구가 바로 파싱할 수 있습니다. 객체의 모양은 [9.9.4](909-debug-tools.md#994-진단--에러-메시지)에서 정의합니다.

## 9.1.9. 표준 입력

`rokc check -`처럼 인자 자리에 `-`를 적으면 표준 입력을 뜻합니다. 파이프나 스크립트와 함께 쓸 때 씁니다. 표준 입력으로 받은 소스는 익명 실행 패키지의 `main.rok`으로 검사합니다.

## 9.1.10. 외부 도구와 호환

`rokc`는 저수준 컴파일러이고, 의존성 해소는 프런트엔드 `rok`([9.10](910-rok-frontend.md))이 맡습니다. `rok`은 해소한 의존 패키지의 루트를 `--extern`([9.2.3](902-aot-pipeline.md#923-의존성-해소))으로 넘겨 `rokc`를 부릅니다. 다른 빌드 시스템도 같은 방식으로 `rokc`를 직접 호출해 통합할 수 있습니다. CLI 호환성을 깨는 변경은 SemVer 규칙을 따릅니다.

## 9.1.11. 표준 형식(`rokc fmt`)

`rokc fmt`의 표준 포맷은 **토큰을 보존하는 배치 규칙**입니다. 줄 안의 토큰과 주석은 바꾸지 않고, 줄의 들여쓰기와 빈 줄만 정합니다. `rokc fmt`에는 일부러 설정이 없습니다. 모든 Rok 코드의 표준 포맷은 하나입니다.

1. 들여쓰기는 괄호(`(` `[` `{` `#[`) 깊이마다 공백 4칸입니다. 닫는 괄호로 시작하는 줄은 여는 줄의 깊이를 따릅니다. 탭은 공백으로 바뀝니다.
2. 이어지는 줄은 원래 파일에서 블록 들여쓰기보다 더 들여쓴 만큼을 유지합니다. 이어지는 줄은 앞 토큰이 `;` `,`나 괄호가 아닌 줄입니다(긴 식을 나눈 줄 등).
3. 줄 끝 공백을 지웁니다.
4. 연속된 빈 줄은 하나로 줄이고, 파일 처음의 빈 줄은 지웁니다.
5. 파일은 줄바꿈 하나로 끝납니다.

포매터는 토큰을 줄 사이로 옮기지 않고, 줄을 나누거나 합치지 않으며, 줄 안의 간격을 바꾸지 않습니다. 따라서 주석은 적힌 그대로 남고 토큰 순서는 바뀌지 않습니다. 여러 줄에 걸친 문자열과 블록 주석의 안쪽도 그대로 둡니다.

1.0 전까지는 이 배치 규칙이 표준 포맷입니다. 그때까지 코드를 다시 짜는 pretty-printer(토큰을 옮기거나 줄을 나누고 합치는 포매터)로 바꾸지 않습니다. 따라서 `rokc fmt --check`를 통과한 파일은 이후 0.x 툴체인에서도 계속 통과합니다.
