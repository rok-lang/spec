# 9.8. Jupyter 커널 통합

[Jupyter 표면](../000-meta/001-execution-modes.md) 의 rok 커널을 정의합니다.

## 9.8.1. 커널

Jupyter 가 rok 커널을 별도 프로세스로 실행하며, ZeroMQ 기반 Jupyter wire protocol 로 통신합니다. 커널은 `protocol_version` 을 `"5.3"` 으로 알립니다 (`jupyter_client` 가 기대하는 값, 5.3 이상의 클라이언트와 동작). 이 형태에서는 OS 자원, 즉 파일과 네트워크와 외부 프로세스를 모두 사용할 수 있습니다.

커널은 터미널 REPL 과 **같은 셀 세션** 위에서 동작하며 [9.7.0](907-jit-cells.md#970-셀-세션의-규칙) 의 규칙을 따릅니다. 각 셀은 한 번만 실행되고, 바인딩과 항목은 이후 셀로 이어지며, 실패한 셀은 바인딩에 흔적을 남기지 않습니다. 커널의 엔진은 REPL 과 같은 [JIT 세션](906-jit-engine.md) 이며, libLLVM 이 필요합니다.

### 노트북 셀과 REPL 셀

노트북 셀 하나는 터미널 REPL 이 입력을 나누는 것과 같은 규칙으로 REPL 셀 여러 개로 나뉘어 차례로 실행됩니다. 결과로는 IPython 처럼 **마지막 값만** 표시합니다. 노트북 셀의 뒷부분이 실패하면, 앞부분 (앞선 REPL 셀) 의 효과는 그대로 남습니다.

## 9.8.2. 로컬 네이티브 커널: 동작

사용자가 노트북의 kernel 로 "rok" 를 고르면 Jupyter 가 kernelspec (9.8.3) 의 명령을 별도 프로세스로 실행합니다.

```
rokc jupyter --connection-file <path>       # --extern 등 rokc 옵션도 받음
rok jupyter --connection-file <path>        # 의존성을 해소한 뒤 rokc jupyter 를 실행
```

커널은 connection file 의 ZMQ port 와 key 를 읽고 소켓에 연결합니다. 전송은 `tcp` 만 지원하며 (`ipc` 없음), CURVE 암호화는 쓰지 않습니다. 메시지는 connection file 의 key 로 HMAC 서명합니다.

### 소켓 채널

| 채널        | 용도                                                       |
|-------------|------------------------------------------------------------|
| `shell`     | 요청 (execute, kernel_info, complete, is_complete 등)      |
| `iopub`     | 출력과 상태 broadcast (status, stream, execute_result 등) |
| `stdin`     | 쓰지 않음 — 입력 요청을 보내지 않음 (stdin 은 `/dev/null`) |
| `control`   | interrupt, shutdown                                        |
| `heartbeat` | 받은 것을 그대로 돌려보냄 (echo)                           |

### 메시지 형식

각 메시지는 6 개 이상의 ZMQ frame 으로 직렬화됩니다.

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

자세한 형식은 Jupyter wire protocol 표준 문서를 따릅니다. 모든 요청은 iopub 의 `status: busy` 와 `status: idle` 로 감쌉니다.

### 지원하는 메시지

| 요청                   | 응답                                                                                   |
|------------------------|----------------------------------------------------------------------------------------|
| `kernel_info_request`  | `language_info` 의 name `rok`, mimetype `text/x-rok`, file_extension `.rok`, `protocol_version` `"5.3"` |
| `execute_request`      | 아래 "실행" 참조                                                                       |
| `is_complete_request`  | REPL 의 이어 쓰기 규칙으로 `complete` / `incomplete` 판정                              |
| `complete_request`     | 세션의 바인딩과 항목 이름만 후보로                                                     |
| `inspect_request`      | 늘 `found: false`                                                                      |
| `history_request`      | 빈 목록                                                                                |
| `comm_info_request`    | 빈 목록                                                                                |
| `shutdown_request`     | shell 이나 control 에서. 응답 후 종료                                                  |
| `interrupt_request`    | control 에서 (9.8.4)                                                                   |

### 실행

`execute_request` 를 받으면 다음을 보냅니다.

1. iopub `execute_input` (코드와 실행 번호).
2. 셀이 **끝난 뒤** 그 출력: stdout / stderr 는 iopub `stream` 으로. 실행 중에 조금씩 보내지 않습니다.
3. 값이 있으면 iopub `execute_result`: `text/plain` 에 REPL 과 같은 `타입: 값` (9.7.7).
4. 실패하면 iopub `error`: `evalue` 는 stderr 의 첫 줄, `traceback` 은 stderr 의 줄들.
5. shell `execute_reply` (`execution_count` 포함, 성공 / 실패 status).

출력 MIME 은 `text/plain` 하나입니다.

## 9.8.3. kernelspec 설치

Jupyter 는 kernelspec 디렉토리에 설치된 kernel 만 인식합니다.

```
rok jupyter install [--env <name>]
```

kernelspec 은 다음 디렉토리 아래의 `kernels/rok/kernel.json` 입니다. `--env <name>` 으로 설치하면 `kernels/rok-<name>/` 이며, 표시 이름은 `rok (<name>)`, 실행 명령은 `rok --env <name> jupyter ...` 입니다.

1. `$JUPYTER_DATA_DIR` 이 있으면 그 디렉토리
2. macOS 이면 `~/Library/Jupyter`
3. 그 밖에는 `$XDG_DATA_HOME/jupyter`, 없으면 `~/.local/share/jupyter`

`kernel.json` 의 내용은 다음과 같습니다. 설치할 때 `ROK_HOME` 이 설정되어 있었으면 `env` 에 함께 기록합니다.

```json
{
  "argv": ["/path/to/rok", "jupyter", "--connection-file", "{connection_file}"],
  "display_name": "rok",
  "language": "rok",
  "interrupt_mode": "message",
  "env": { "ROK_HOME": "/path/to/rok-home" }
}
```

이렇게 설치하고 나면 Jupyter, VS Code, nteract 등 어디서든 rok 커널을 사용할 수 있습니다.

## 9.8.4. 인터럽트 / 종료

### 인터럽트

`interrupt_mode` 가 `"message"` 이므로 인터럽트는 control 소켓의 `interrupt_request` 로 옵니다. 실행 중인 셀을 끝내고, 세션은 그 셀 이전의 상태로 계속됩니다 (실패한 셀과 같음, [9.7.0](907-jit-cells.md#970-셀-세션의-규칙)). 아직 컴파일 중인 셀은 인터럽트할 수 없습니다.

### 종료 (kernel restart)

shell 이나 control 소켓의 `shutdown_request` 에 응답한 뒤 세션을 정리하고 프로세스를 종료합니다. restart 의 경우에는 Jupyter 가 자동으로 다시 띄워 줍니다.

## 9.8.5. 코드 자동 완성

`complete_request` 는 커서 앞의 조각을 세션의 바인딩, 항목, 모듈 경로, 멤버, 키워드에 맞춰 `{ matches, cursor_start, cursor_end }` 로 응답합니다.

## 9.8.6. inspect_request: 도움말

`inspect_request` (Shift+Tab) 는 커서의 이름의 시그니처와 문서 주석 ([어휘 구조](../001-lexical/102-lexical-structure.md) 의 `///` / `//!`) 을 `text/plain` 으로 응답합니다. 모르는 이름이면 `found: false` 입니다.

## 9.8.7. REPL 과 Jupyter 의 코드 공유

REPL 에서 작성한 코드는 그대로 Jupyter 셀로 옮길 수 있고, 반대 방향도 마찬가지로 가능합니다. 같은 셀 세션과 같은 의미론 ([9.7.0](907-jit-cells.md#970-셀-세션의-규칙)) 을 따르고 있기 때문입니다.

### 차이

다만 양쪽 표면 사이에는 몇 가지 차이가 존재하며, 다음 표에 정리합니다.

| 측면        | REPL          | Jupyter               |
|-------------|---------------|-----------------------|
| 입력        | 한 줄 / 한 식 | 한 셀 (여러 줄)       |
| 출력        | text only     | text (`text/plain`)   |
| 상태 보존   | 종료 시 폐기  | 노트북 파일에 셀 보존 |

## 9.8.8. 보안 / 격리

커널은 사용자 권한으로 임의 코드를 실행합니다. 격리는 하지 않습니다.

## 9.8.9. 다음

이어지는 [디버그 / 진단 도구](909-debug-tools.md) 에서는 gdb 친화 기능과 profile, `--emit` 등을 정의합니다.
