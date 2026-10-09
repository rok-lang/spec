# 9.1. `rokc` CLI 인터페이스

이 절에서는 `rokc` 의 명령줄 인터페이스를 정의하며, 모든 rok 작업을 이 단일 바이너리 하나로 처리하도록 합니다.

## 9.1.1. 일반 형태

`rokc` 는 다음과 같은 일반 형태로 호출됩니다.

```
rokc <subcommand> [options] [args]
```

문법 규칙으로 정리하면 아래와 같습니다.

```
cli_invocation  ::= "rokc" subcommand option* arg*
subcommand      ::= "build" | "run" | "test" | "repl" | "jupyter" | "fmt"
                  | "check" | "lsp" | "demangle" | "rhir-text" | "inspect"
                  | "new" | "clean" | "version" | "help"
option          ::= "--" identifier ("=" value)?
                  | "-" letter
```

## 9.1.2. 핵심 서브명령

### `rokc build`: [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 빌드

`rokc build` 는 패키지를 AOT 로 빌드해 주는 서브명령입니다.

```
rokc build [path]
```

`[path]` 는 패키지 디렉토리를 가리키며, 생략하면 현재 디렉토리를 사용합니다. 산출물은 `<path>/target/libname.a` 또는 `main.rok` 가 있는 경우 실행 파일로 생성됩니다.

옵션은 다음과 같습니다.

- `--release` 는 최적화 빌드를 수행하며, 기본값은 디버그 빌드입니다.
- `-O0` / `-O1` / `-O2` / `-O3` / `-Os` 는 최적화 단계를 명시적으로 지정합니다.
- `--target=<triple>` 은 타깃 플랫폼을 지정하며, [플랫폼 타깃](../000-meta/006-platform-targets.md) 의 13 개 Tier 1 중 하나를 사용합니다.
- `--out=<path>` 는 출력 위치를 지정합니다.
- `--cfg <key>=<value>` 는 [조건부 컴파일](../006-meta-system/603-cfg.md) 의 cfg flag 를 설정합니다.
- `--emit=<kind>` 는 중간 산출물 (`hir`, `mir`, `mlir`, `llvmir`, `asm`, `obj`, `expanded`) 을 추출합니다.
- `--load-mlir-plugin=<path|name>` 은 MLIR 다이얼렉트 plugin 을 동적으로 로드합니다 ([rok::bare::mlir::plugin](../../refs-standard-libraries/100-bare/110-mlir-plugin.md)). 여러 번 사용 가능하며, 빌드 시점에 plugin 의 다이얼렉트 op 이 `__mlir_op` 으로 호출 가능해집니다.

### `rokc run`: 빌드 후 실행

`rokc run` 은 빌드 직후 산출물을 바로 실행해 주는 명령입니다.

```
rokc run [path] [-- args...]
```

`path` 가 `main.rok` 를 포함한 패키지라면 빌드 후 곧바로 실행하며, `--` 뒤에 적은 인자는 실행 파일에 그대로 전달됩니다. 옵션은 `rokc build` 와 동일합니다.

`path` 가 어떤 패키지에도 속하지 않은 단일 `.rok` 파일이면, 그 파일 하나로 이루어진 익명 실행 패키지를 구성하여 파일을 `main.rok` 로 취급합니다 ([4.16.8 단일 파일 실행](../004-functions/416-entry-point.md#4168-단일-파일-실행)). `rokc build <file>.rok` 도 같은 규칙을 따릅니다.

```
rokc run hello.rok -- arg1 arg2
```

### `rokc test`: 테스트

`rokc test` 는 테스트 함수들을 모아 빌드하고 실행해 주는 명령입니다.

```
rokc test [path] [filter]
```

`[filter]` 는 테스트 이름 패턴이며 정규식 또는 단순 부분 문자열 모두 사용할 수 있습니다. `#[test]` 가 부착된 함수들을 모아 빌드하고 실행해 줍니다.

옵션은 `rokc build` 와 동일하며, 추가로 다음 두 가지를 지원합니다.

- `--threads=N` 은 병렬로 실행할 테스트 수를 지정합니다.
- `--show-output` 은 성공한 테스트의 stdout / stderr 까지도 표시해 줍니다.

### `rokc repl`: 터미널 REPL

`rokc repl` 은 터미널에서 인터랙티브 셸을 띄워 주는 명령입니다.

```
rokc repl [--package <path>]
```

이 모드는 터미널 인터랙티브 셸로 동작하며, 내부적으로 [JIT 엔진](906-jit-engine.md) 을 사용합니다. `--package` 를 지정하면 시작 시 해당 패키지를 자동으로 로드해 줍니다 (`use` 가 자동으로 적용됩니다).

REPL 의 키바인드와 명령은 본 사양의 범위 바깥이며, 구현 측에서 결정합니다.

### `rokc jupyter`: Jupyter 커널

`rokc jupyter` 는 Jupyter 가 호출하는 커널 모드를 제공합니다.

```
rokc jupyter [--connection-file <path>]
```

이 명령은 Jupyter 가 호출하는 커널 모드이므로 사용자가 직접 실행하지는 않습니다. `--connection-file` 은 Jupyter 가 제공하는 ZMQ 연결 정보 파일의 경로를 받습니다. 자세한 통합 내용은 [Jupyter 통합](908-jupyter-integration.md) 에서 정의합니다.

#### 커널 등록

커널 등록은 다음과 같이 수행합니다.

```
rokc jupyter --install-kernel
```

이 명령은 Jupyter 의 kernelspec 을 등록해 주며, `~/.local/share/jupyter/kernels/rok/kernel.json` 을 생성합니다. 한 번만 실행하면 그 이후로는 Jupyter 가 자동으로 인식해 줍니다.

### `rokc fmt`: 포매터

`rokc fmt` 는 표준 포맷을 적용해 주는 포매터 명령입니다.

```
rokc fmt [path]
```

`[path]` 는 파일 또는 디렉토리 (재귀 처리) 를 받으며, 표준 포맷에 맞게 in-place 로 수정해 줍니다.

옵션은 다음과 같습니다.

- `--check` 는 변경 없이 불일치만 검출하며, CI 용도로 적합합니다.
- `--diff` 는 diff 만 출력해 줍니다.

### `rokc check`: 타입 / 진단

`rokc check` 는 빌드까지 진행하지 않고 타입 검사와 진단만 수행해 주는 명령입니다.

```
rokc check [path]
```

빌드 단계까지 가지 않고 타입 검사와 진단만 수행하기 때문에, LSP 서버가 내부적으로 이 명령을 사용합니다. 옵션은 `rokc build` 와 거의 동일합니다.

### `rokc lsp`: LSP 서버

`rokc lsp` 는 Language Server Protocol 서버 모드로 동작합니다.

```
rokc lsp
```

이 모드는 VS Code, Vim, Emacs 등 LSP 클라이언트에서 호출하도록 설계되어 있습니다.

## 9.1.3. 디버그 / 진단 서브명령

### `rokc demangle`: 심볼 demangle

`rokc demangle` 은 mangled 심볼을 사람이 읽을 수 있는 형식으로 풀어 출력해 줍니다.

```
rokc demangle <symbol>
```

### `rokc rhir-text`: [RHIR](../000-meta/007-glossary.md#rhir-rok-hir-bytecode) 텍스트 출력

`rokc rhir-text` 는 archive 안에 들어 있는 RHIR bytecode 를 텍스트로 출력해 줍니다.

```
rokc rhir-text path/to/libfoo.a
```

[archive](../000-meta/007-glossary.md#archive) 안의 RHIR bytecode 를 텍스트로 출력해 주며, 자세한 포맷은 [RHIR bytecode 포맷](904-rhir-bytecode.md) 에 기술되어 있습니다.

### `rokc inspect`: 패키지 검사

`rokc inspect` 는 archive 의 내용을 한눈에 살펴볼 수 있도록 해 줍니다.

```
rokc inspect path/to/libfoo.a
```

archive 의 exports, [generic](../000-meta/007-glossary.md#generic--제네릭) items, 의존, 빌드 메타 등을 한 번에 표시해 줍니다.

## 9.1.4. 보조 서브명령

### `rokc new`: 새 패키지 생성

`rokc new` 는 새 패키지를 생성해 주는 명령입니다.

```
rokc new [--lib | --bin] <name>
```

`<name>` 디렉토리를 만들고 그 안에 기본 파일 (`lib.rok` 또는 `main.rok`) 을 채워 줍니다.

### `rokc clean`: 빌드 산출물 제거

`rokc clean` 은 빌드 산출물을 제거해 주는 명령입니다.

```
rokc clean [path]
```

`<path>/target/` 디렉토리를 삭제해 줍니다.

### `rokc version`: 버전

`rokc version` 은 rokc 의 버전 정보를 출력해 줍니다.

```
rokc version
rokc --version
```

이 명령은 rokc 버전, [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 해시 ([8. 런타임 ABI](../008-abi/README.md) 참조), Tier 1 타깃 목록을 함께 출력해 줍니다.

### `rokc help`: 도움말

`rokc help` 는 도움말을 보여 줍니다.

```
rokc help [subcommand]
```

## 9.1.5. 전역 옵션

서브명령 앞에 둘 수 있는 전역 옵션들은 다음 표와 같습니다.

| 옵션                          | 의미               |
|-------------------------------|--------------------|
| `--verbose` / `-v`            | 자세한 로그        |
| `--quiet` / `-q`              | 최소 로그          |
| `--color=<auto/always/never>` | 색상 출력          |
| `--cwd=<path>`                | 작업 디렉토리 변경 |

사용 예는 다음과 같습니다.

```
rokc --verbose build
rokc --quiet test
```

## 9.1.6. 종료 코드

종료 코드는 다음과 같이 정의되어 있습니다.

| 코드  | 의미                                            |
|-------|-------------------------------------------------|
| `0`   | 성공                                            |
| `1`   | 사용자 코드 에러 (컴파일 에러 / 테스트 실패 등) |
| `2`   | 사용 / CLI 에러 (잘못된 옵션 등)                |
| `101` | rokc 내부 에러 (panic)                          |

## 9.1.7. 환경 변수

`rokc` 의 동작에 영향을 주는 환경 변수는 다음 표와 같습니다.

| 변수              | 의미                                                                                     |
|-------------------|------------------------------------------------------------------------------------------|
| `ROK_HOME`        | toolchain 설치 위치 ([9.2.19](902-aot-pipeline.md#9219-설치-레이아웃)). 없으면 `rokc` 실행 파일의 `../` |
| `ROKC_VERBOSE`    | `--verbose` 와 동일                                                                      |
| `ROKC_TARGET_DIR` | `target/` 의 위치 (기본은 패키지 안)                                                     |
| `ROKC_CACHE_DIR`  | [JIT](../000-meta/007-glossary.md#jit-just-in-time) 캐시 디렉토리 (기본 `~/.cache/rok/`) |
| `ROKC_NUM_JOBS`   | 병렬 빌드 수 (기본은 CPU 코어 수)                                                        |

## 9.1.8. 출력 형식

### 진단 메시지

진단 메시지는 다음과 같은 형식으로 출력됩니다.

```
error: 'count' is not declared
  --> src/main.rok:5:10
   |
 5 |     let x = count + 1;
   |             ^^^^^ unknown identifier
   |
   = note: did you mean 'counter'?
```

[소스 파일](../001-lexical/101-source-files.md) 에서 정의한 파일 + 줄 + 컬럼 단위의 진단을 그대로 사용합니다.

### 빌드 진행

빌드 진행 상황은 다음과 같이 표시됩니다.

```
   Compiling foo v0.1.0
    Building target/libfoo.a
   Finished in 1.23s
```

### JSON 출력 (도구 통합용)

도구 통합을 위해서는 JSON 형식으로도 출력할 수 있습니다.

```
rokc build --message-format=json
```

이 모드에서는 각 줄이 하나의 JSON 객체로 구성되므로, IDE 와 CI 도구에서 쉽게 파싱할 수 있습니다.

## 9.1.9. 표준 입력

`rokc check -` 처럼 인자 자리에 `-` 를 적으면 표준 입력을 의미하며, 파이프 또는 스크립트 통합 용도로 활용할 수 있습니다.

## 9.1.10. 외부 도구와의 호환

`rokc` 는 상위에 별도의 패키지 매니저가 존재하는 경우, 그 도구가 호출해서 사용하는 형태로 설계되어 있습니다.

패키지 매니저는 `rokc` 의 CLI 를 낮은 수준 도구로 호출하는 위치에 자리잡습니다. CLI 호환성을 깨는 변경은 SemVer 규칙에 따라 관리됩니다.
