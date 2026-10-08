# 9.1. `nokc` CLI 인터페이스

`nokc` 의 명령줄 인터페이스를 정의한다. 모든 nok 작업을 이 단일 바이너리로 처리한다.

## 9.1.1. 일반 형태

```
nokc <subcommand> [options] [args]
```

```
cli_invocation  ::= "nokc" subcommand option* arg*
subcommand      ::= "build" | "run" | "test" | "repl" | "jupyter" | "fmt"
                  | "check" | "lsp" | "demangle" | "nhir-text" | "inspect"
                  | "new" | "clean" | "version" | "help"
option          ::= "--" identifier ("=" value)?
                  | "-" letter
```

## 9.1.2. 핵심 서브명령

### `nokc build`: [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 빌드

```
nokc build [path]
```

- `[path]`: 패키지 디렉토리 (생략 시 현재 디렉토리).
- 산출: `<path>/target/libname.a` 또는 실행 파일 (`main.nok` 의 경우).

옵션:

- `--release`: 최적화 빌드 (기본은 디버그).
- `-O0` / `-O1` / `-O2` / `-O3` / `-Os`: 최적화 단계 명시.
- `--target=<triple>`: 타깃 플랫폼 ([플랫폼 타깃](../000-meta/006-platform-targets.md) 의 13 개 Tier 1 중 하나).
- `--out=<path>`: 출력 위치.
- `--cfg <key>=<value>`: [조건부 컴파일](../006-meta-system/603-cfg.md) 의 cfg flag 설정.
- `--emit=<kind>`: 중간 산출물 (`hir`, `mir`, `mlir`, `llvmir`, `asm`, `obj`, `expanded`).

### `nokc run`: 빌드 후 실행

```
nokc run [path] [-- args...]
```

- `path` 가 `main.nok` 가 있는 패키지면 빌드 후 바로 실행한다.
- `--` 뒤의 인자는 실행 파일에 그대로 전달한다.

옵션은 `nokc build` 와 동일.

### `nokc test`: 테스트

```
nokc test [path] [filter]
```

- `[filter]`: 테스트 이름 패턴 (정규식 또는 단순 부분 문자열).
- `#[test]` 가 부착된 함수들을 모아 빌드하고 실행한다.

옵션은 `nokc build` 와 동일하며 추가로 다음을 지원한다.

- `--threads=N`: 병렬 테스트 수.
- `--show-output`: 성공 테스트도 stdout / stderr 표시.

### `nokc repl`: 터미널 REPL

```
nokc repl [--package <path>]
```

- 터미널 인터랙티브 셸. [JIT 엔진](906-jit-engine.md) 을 사용한다.
- `--package`: 시작 시 그 패키지를 로드한다 (`use` 자동).

REPL 키바인드와 명령은 본 사양 범위 외이며 구현이 결정한다.

### `nokc jupyter`: Jupyter 커널

```
nokc jupyter [--connection-file <path>]
```

- Jupyter 가 호출하는 커널 모드. 사용자가 직접 실행하지 않는다.
- `--connection-file`: Jupyter 가 제공하는 ZMQ 연결 정보 파일 경로.

자세한 통합은 [Jupyter 통합](908-jupyter-integration.md) 에서 정의한다.

#### 커널 등록

```
nokc jupyter --install-kernel
```

- Jupyter 의 kernelspec 을 등록한다. `~/.local/share/jupyter/kernels/nok/kernel.json` 을 생성한다.
- 한 번 실행하면 이후 Jupyter 가 자동으로 인식한다.

### `nokc fmt`: 포매터

```
nokc fmt [path]
```

- `[path]`: 파일 또는 디렉토리 (재귀).
- 표준 포맷으로 in-place 수정한다.

옵션:

- `--check`: 변경 없이 불일치 검출만 (CI 용).
- `--diff`: diff 출력만.

### `nokc check`: 타입 / 진단

```
nokc check [path]
```

- 빌드까지 가지 않고 타입 검사 / 진단만 수행한다.
- LSP 서버가 내부적으로 사용한다.

옵션은 `nokc build` 와 거의 동일.

### `nokc lsp`: LSP 서버

```
nokc lsp
```

- Language Server Protocol 서버 모드. VS Code / Vim / Emacs 등 LSP 클라이언트가 호출한다.

## 9.1.3. 디버그 / 진단 서브명령

### `nokc demangle`: 심볼 demangle

```
nokc demangle <symbol>
```

- mangled 심볼을 사람이 읽을 수 있는 형식으로 출력한다.

### `nokc nhir-text`: [NHIR](../000-meta/007-glossary.md#nhir-nok-hir-bytecode) 텍스트 출력

```
nokc nhir-text path/to/libfoo.a
```

- [archive](../000-meta/007-glossary.md#archive) 안의 NHIR bytecode 를 텍스트로 출력한다 ([NHIR bytecode 포맷](904-nhir-bytecode.md)).

### `nokc inspect`: 패키지 검사

```
nokc inspect path/to/libfoo.a
```

- archive 의 exports, [generic](../000-meta/007-glossary.md#generic-/-제네릭) items, 의존, 빌드 메타 등을 한눈에 보여준다.

## 9.1.4. 보조 서브명령

### `nokc new`: 새 패키지 생성

```
nokc new [--lib | --bin] <name>
```

- `<name>` 디렉토리를 만들고 기본 파일 (`lib.nok` 또는 `main.nok`) 을 채운다.

### `nokc clean`: 빌드 산출물 제거

```
nokc clean [path]
```

- `<path>/target/` 을 삭제한다.

### `nokc version`: 버전

```
nokc version
nokc --version
```

- nokc 버전, [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 해시 ([8. 런타임 ABI](../008-abi/README.md) 참조), Tier 1 타깃 목록을 출력한다.

### `nokc help`: 도움말

```
nokc help [subcommand]
```

## 9.1.5. 전역 옵션

서브명령 앞에 둘 수 있는 옵션.

| 옵션                          | 의미               |
| ----------------------------- | ------------------ |
| `--verbose` / `-v`            | 자세한 로그        |
| `--quiet` / `-q`              | 최소 로그          |
| `--color=<auto/always/never>` | 색상 출력          |
| `--cwd=<path>`                | 작업 디렉토리 변경 |

```
nokc --verbose build
nokc --quiet test
```

## 9.1.6. 종료 코드

| 코드  | 의미                                            |
| ----- | ----------------------------------------------- |
| `0`   | 성공                                            |
| `1`   | 사용자 코드 에러 (컴파일 에러 / 테스트 실패 등) |
| `2`   | 사용 / CLI 에러 (잘못된 옵션 등)                |
| `101` | nokc 내부 에러 (panic)                          |

## 9.1.7. 환경 변수

| 변수              | 의미                                                                                     |
| ----------------- | ---------------------------------------------------------------------------------------- |
| `NOKC_VERBOSE`    | `--verbose` 와 동일                                                                      |
| `NOKC_TARGET_DIR` | `target/` 의 위치 (기본은 패키지 안)                                                     |
| `NOKC_CACHE_DIR`  | [JIT](../000-meta/007-glossary.md#jit-just-in-time) 캐시 디렉토리 (기본 `~/.cache/nok/`) |
| `NOKC_NUM_JOBS`   | 병렬 빌드 수 (기본은 CPU 코어 수)                                                        |

## 9.1.8. 출력 형식

### 진단 메시지

```
error: 'count' is not declared
  --> src/main.nok:5:10
   |
 5 |     let x = count + 1;
   |             ^^^^^ unknown identifier
   |
   = note: did you mean 'counter'?
```

[소스 파일](../001-lexical/101-source-files.md) 의 파일 + 줄 + 컬럼 진단 단위를 사용한다.

### 빌드 진행

```
   Compiling foo v0.1.0
    Building target/libfoo.a
   Finished in 1.23s
```

### JSON 출력 (도구 통합용)

```
nokc build --message-format=json
```

각 줄이 한 JSON 객체이며 IDE 와 CI 도구가 파싱할 수 있다.

## 9.1.9. 표준 입력

`nokc check -` 등에서 `-` 는 표준 입력을 의미한다. 파이프 / 스크립트 통합용.

## 9.1.10. 외부 도구와의 호환

`nokc` 는 상위 패키지 매니저가 따로 있을 경우 그 도구가 호출하는 형태로 설계된다.

향후 별도의 nok 패키지 매니저가 등장하면, `nokc` 의 CLI 는 그 도구가 호출하는 낮은 수준 도구가 된다. CLI 호환성을 깨는 변경은 SemVer 를 따른다.
