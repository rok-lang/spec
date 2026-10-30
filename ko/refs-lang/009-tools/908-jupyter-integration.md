# 9.8. Jupyter 커널 통합

[Jupyter 표면](../000-meta/001-execution-modes.md)의 Rok 커널을 정의합니다.

## 9.8.1. 커널

Jupyter가 Rok 커널을 별도 프로세스로 실행하며, ZeroMQ 기반 Jupyter wire protocol로 통신합니다. 커널은 `protocol_version`을 `"5.3"`으로 알립니다 (`jupyter_client`가 기대하는 값, 5.3 이상의 클라이언트와 동작). 이 형태에서는 파일, 네트워크, 외부 프로세스 같은 OS 자원을 모두 쓸 수 있습니다.

커널은 터미널 REPL과 **같은 셀 세션** 위에서 동작하며 [9.7.0](907-jit-cells.md#970-셀-세션의-규칙)의 규칙을 따릅니다. 각 셀은 한 번만 실행되고, 바인딩과 항목은 이후 셀로 이어지며, 실패한 셀은 바인딩에 흔적을 남기지 않습니다. 커널의 엔진은 REPL과 같은 [JIT 세션](906-jit-engine.md)이므로, top-level `await`와 `it`도 REPL에서처럼 동작합니다 ([9.7.1](907-jit-cells.md#971-cell-의-종류), [9.7.7](907-jit-cells.md#977-시각화--출력)).

### 입력

셀은 클라이언트를 거쳐 stdin을 읽습니다. `execute_request`에 `allow_stdin: true`가 있으면, 런타임이 stdin을 읽을 때마다 stdin 채널로 그 요청의 클라이언트에 `input_request` (`prompt`는 빈 문자열, `password`는 false)를 보냅니다. 셀이 읽기 전에 쓴 것(예: 출력한 프롬프트)은 먼저 `stream`으로 보냅니다. `input_reply`의 `value`에 줄바꿈을 붙인 것이 그 읽기의 결과입니다. `allow_stdin`이 없거나 `silent` 요청이면 읽기는 입력의 끝입니다. 셀이 응답을 기다리는 동안 인터럽트하면 셀이 끝납니다 (9.8.4).

### 노트북 셀과 REPL 셀

노트북 셀 하나는 터미널 REPL이 입력을 나누는 것과 같은 규칙으로 REPL 셀 여러 개로 나뉘어 차례로 실행됩니다. 결과로는 IPython처럼 **마지막 값만** 표시합니다. 노트북 셀의 뒷부분이 실패하면, 앞부분 (앞선 REPL 셀)의 효과는 그대로 남습니다.

REPL 셀이 실행 위치 (`<repl cell N>`, [9.7.5](907-jit-cells.md#위치))에서 쓰는 번호는 실행 번호를 따릅니다: REPL 셀 하나인 노트북 셀은 `N`, 여러 개로 나뉘는 노트북 셀은 `N.1`, `N.2`와 같이 번호를 붙입니다. 실행 번호를 늘리지 않는 요청 (`silent`, 또는 `store_history: false`)은 현재 실행 번호 아래 마지막 번호에 이어 REPL 셀 번호를 붙입니다.

## 9.8.2. 로컬 네이티브 커널: 동작

사용자가 노트북의 kernel로 "Rok"를 고르면 Jupyter가 kernelspec (9.8.3)의 명령을 별도 프로세스로 실행합니다.

```
rokc jupyter --connection-file <path>       # --extern 등 rokc 옵션도 받음
rok jupyter --connection-file <path>        # 의존성을 해소한 뒤 rokc jupyter 를 실행
```

커널은 connection file의 ZMQ port와 key를 읽고 다섯 소켓에 bind해 listen합니다. 다섯 소켓이 모두 연결을 받을 수 있게 되면, 출력 캡처를 시작하기 전에 stderr에 `rokc jupyter: ready` 한 줄을 씁니다. 전송은 `tcp`만 지원하며 (`ipc` 없음), CURVE 암호화는 쓰지 않습니다. 메시지는 connection file의 key로 HMAC 서명합니다.

커널은 시작한 곳의 패키지 (현재 디렉토리, 또는 `src/main.rok`이나 `src/lib.rok`이 있는 가장 가까운 상위 디렉토리)를 `rokc repl .`처럼 로드합니다 ([9.7.10](907-jit-cells.md#9710-프로그램-로드-repl-path)): 라이브러리 패키지는 의존성으로 붙이고 `use <name>::*;`를 첫 셀로 넣으며, 프로그램 패키지는 로드해 실행합니다. 로드는 출력 캡처를 시작한 뒤에 일어나며, 그동안 출력한 것은 처음 실행한 셀의 출력과 함께 보냅니다. 패키지 밖에서는 빈 세션으로 시작합니다.

### 소켓 채널

| 채널        | 용도                                                       |
|-------------|------------------------------------------------------------|
| `shell`     | 요청 (execute, kernel_info, complete, is_complete 등)      |
| `iopub`     | 출력과 상태 broadcast (status, stream, execute_result 등) |
| `stdin`     | stdin을 읽는 셀의 `input_request` / `input_reply` (9.8.1) |
| `control`   | interrupt, shutdown                                        |
| `heartbeat` | 받은 것을 그대로 돌려보냄 (echo)                           |

### 메시지 형식

각 메시지는 6개 이상의 ZMQ frame으로 직렬화됩니다.

```
[b"u-u-i-d",         # zmq identity(ies)
 b"<IDS|MSG>",       # delimiter
 b"baddad42",         # HMAC 서명 (key 로)
 b"{header}",          # 헤더 JSON
 b"{parent_header}",   # 부모 헤더 JSON
 b"{metadata}",        # 메타데이터 JSON
 b"{content}",         # 콘텐츠 JSON
]
```

자세한 형식은 Jupyter wire protocol 표준 문서를 따릅니다. 모든 요청은 iopub의 `status: busy`와 `status: idle`로 감쌉니다.

### 지원하는 메시지

| 요청                   | 응답                                                                                   |
|------------------------|----------------------------------------------------------------------------------------|
| `kernel_info_request`  | `language_info`의 name `rok`, mimetype `text/x-rok`, file_extension `.rok`, codemirror_mode와 pygments_lexer `rust` (프런트엔드가 Rok을 알기 전까지의 하이라이팅), 문서 (`https://docs.rok-lang.org/latest/`)와 이 REPL 장으로 가는 `help_links`, REPL의 toolchain 줄인 `banner` ([9.7.8](907-jit-cells.md#줄-편집)), `protocol_version` `"5.3"` |
| `execute_request`      | 아래 "실행" 참고                                                                       |
| `is_complete_request`  | REPL의 이어 쓰기 규칙으로 `complete` / `incomplete` 판정, `indent`는 아직 열린 괄호마다 공백 네 칸 |
| `complete_request`     | REPL의 자동 완성 후보 (9.8.5)                                                         |
| `inspect_request`      | 커서 위 이름의 `text/plain` 도움말과 `found: true` (9.8.6), 아니면 `false`         |
| `history_request`      | 이 커널이 실행한 셀들                                                                  |
| `comm_info_request`    | 빈 목록                                                                                |
| `shutdown_request`     | shell이나 control에서. 응답 후 종료                                                  |
| `interrupt_request`    | control에서 (9.8.4)                                                                   |

### 실행

`execute_request`를 받으면 다음을 보냅니다.

1. iopub `execute_input` (코드와 실행 번호).
2. 셀의 출력을 **쓰는 대로**: 실행 중인 셀이 stdout / stderr에 쓴 것은 커널이 읽는 대로 iopub `stream` (`stdout` / `stderr`)으로 보냅니다. 둘은 서로 다른 파이프이므로, 한쪽의 조각은 순서를 지키고 양쪽의 조각은 읽은 순서로 갑니다. 실행 밖에서 쓴 출력 (진단, `:` 명령)은 다음 실행 전이나 셀이 끝날 때 보냅니다.
3. 값이 있으면 iopub `execute_result`: `text/plain`에 REPL과 같은 출력 (9.7.7).
4. 실패하면 iopub `error`: `evalue`는 아직 보내지 않은 stderr (컴파일되지 않는 셀의 진단, 또는 실행이 어떻게 끝났는지의 보고)의 첫 줄, `traceback`은 그 줄들. panic 메시지처럼 셀 자신이 stderr에 쓴 것은 그 전에 스트림으로 보냈습니다.
5. shell `execute_reply` (`execution_count` 포함, 성공 / 실패 status).

출력 MIME은 `text/plain` 하나입니다.

## 9.8.3. kernelspec 설치

Jupyter는 kernelspec 디렉토리에 설치된 kernel만 인식합니다.

```
rok jupyter install [--env <name>]
```

인자 없는 `rok jupyter`는 대신 무엇을 실행할지 알려 주는 사용법 에러 (종료 코드 `2`)입니다: `` Jupyter starts the kernel itself; to add it, run `rok jupyter install` ``.

kernelspec은 다음 디렉토리 아래의 `kernels/rok/kernel.json`입니다. `--env <name>`으로 설치하면 `kernels/rok-<name>/`이며, 표시 이름은 `rok (<name>)`, 실행 명령은 `rok --env <name> jupyter ...`입니다.

1. `$JUPYTER_DATA_DIR`이 있으면 그 디렉토리
2. macOS이면 `~/Library/Jupyter`
3. Windows이면 `%APPDATA%\jupyter`
4. 그 밖에는 `$XDG_DATA_HOME/jupyter`, 없으면 `~/.local/share/jupyter`

그 옆에는 프런트엔드가 커널에 보여 줄 Rok 로고 `logo-32x32.png`와 `logo-64x64.png`가 있습니다. `kernel.json`은 `rok` proxy ([9.10.6](910-rok-frontend.md#9106-toolchain))를 실행하며, 설치 때의 `ROK_HOME`과 `PATH`를 `env`에 기록합니다.

```json
{
  "argv": ["/home/u/.rok/bin/rok", "jupyter", "--connection-file", "{connection_file}"],
  "display_name": "rok",
  "language": "rok",
  "interrupt_mode": "message",
  "env": { "ROK_HOME": "/home/u/.rok", "PATH": "<설치 때의 PATH>" }
}
```

설치하면 Jupyter, VS Code, nteract 등 어디서든 Rok 커널을 쓸 수 있습니다.

## 9.8.4. 인터럽트 / 종료

### 인터럽트

`interrupt_mode`가 `"message"`이므로 인터럽트는 control 소켓의 `interrupt_request`로 옵니다. 실행 중인 셀을 끝내고, 세션은 그 셀 이전의 상태로 계속됩니다 (실패한 셀과 같음, [9.7.0](907-jit-cells.md#970-셀-세션의-규칙)). `input_reply`를 기다리는 셀도 끝납니다. 아직 컴파일 중인 셀은 인터럽트할 수 없습니다.

### 종료 (kernel restart)

shell이나 control 소켓의 `shutdown_request`에 응답한 뒤 세션을 정리하고 프로세스를 종료합니다. restart이면 Jupyter가 커널을 다시 띄웁니다.

## 9.8.5. 코드 자동 완성

`complete_request`는 커서 앞의 조각을 터미널 REPL의 Tab 자동 완성과 같은 후보 ([9.7.8](907-jit-cells.md#줄-편집))에 맞춥니다: 세션의 바인딩과 항목, core 라이브러리의 이름, 세션의 `use` 항목이 가져온 이름, 키워드이며, `.` 뒤에서는 멤버, `::` 뒤에서는 항목입니다. 응답은 `{ matches, cursor_start, cursor_end }`입니다.

## 9.8.6. inspect_request: 도움말

`inspect_request` (Shift+Tab)에는 `found: true`와 함께 커서의 이름을 `name: type` 형태로, 그 시그니처와 문서 주석 ([어휘 구조](../001-lexical/102-lexical-structure.md)의 `///` / `//!`)을 `text/plain`으로 담아 응답합니다. 모르는 이름이면 `found: false`입니다.

## 9.8.7. REPL 과 Jupyter 의 코드 공유

REPL에서 작성한 코드는 그대로 Jupyter 셀로 옮길 수 있고, 반대도 됩니다. 둘 다 같은 셀 세션과 같은 의미론([9.7.0](907-jit-cells.md#970-셀-세션의-규칙))을 따르기 때문입니다.

### 차이

다만 두 표면에는 다음과 같은 차이가 있습니다.

| 측면        | REPL          | Jupyter               |
|-------------|---------------|-----------------------|
| 입력        | 한 줄 / 한 식 | 한 셀 (여러 줄)       |
| 출력        | text only     | text (`text/plain`)   |
| stdin       | 터미널, 파이프면 빈 입력 | `allow_stdin`이면 `input_request` |
| 상태 보존   | 종료 시 폐기  | 노트북 파일에 셀 보존 |

## 9.8.8. 보안 / 격리

커널은 사용자 권한으로 임의 코드를 실행합니다. 격리는 하지 않습니다.

## 9.8.9. 다음

이어지는 [디버그 / 진단 도구](909-debug-tools.md)에서는 gdb 친화 기능, profile, `--emit` 등을 정의합니다.
