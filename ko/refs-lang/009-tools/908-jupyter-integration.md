# 9.8. Jupyter 커널 통합

이 절은 [Jupyter 표면](../000-meta/001-execution-modes.md)의 Rok 커널을 정의합니다.

## 9.8.1. 커널

Jupyter는 Rok 커널을 별도 프로세스로 실행하고, ZeroMQ 기반 Jupyter wire protocol로 통신합니다. 커널은 `protocol_version`을 `"5.3"`으로 알립니다. `jupyter_client`가 기대하는 값이고, 5.3 이상의 클라이언트와 동작합니다. 이 형태에서는 파일, 네트워크, 외부 프로세스 같은 OS 자원을 모두 쓸 수 있습니다.

커널은 터미널 REPL과 **같은 셀 세션** 위에서 동작하고 [9.7.0](907-jit-cells.md#970-셀-세션의-규칙)의 규칙을 따릅니다. 각 셀은 한 번만 실행되고 바인딩과 항목은 이후 셀로 이어집니다. 실패한 셀은 바인딩에 흔적을 남기지 않습니다. 커널의 엔진은 REPL과 같은 [JIT 세션](906-jit-engine.md)입니다.

### 노트북 셀과 REPL 셀

노트북 셀 하나는 REPL 셀 여러 개로 나뉘어 차례로 실행됩니다. 나누는 규칙은 터미널 REPL이 입력을 나누는 규칙과 같습니다. 결과로는 IPython처럼 **마지막 값만** 표시합니다. 노트북 셀의 뒷부분이 실패해도 앞부분(앞선 REPL 셀)의 효과는 그대로 남습니다.

## 9.8.2. 로컬 네이티브 커널: 동작

사용자가 노트북의 커널로 "Rok"를 고르면 Jupyter가 kernelspec(9.8.3)의 명령을 별도 프로세스로 실행합니다.

```
rokc jupyter --connection-file <path>       # --extern 등 rokc 옵션도 받음
rok jupyter --connection-file <path>        # 의존성을 해소한 뒤 rokc jupyter 를 실행
```

커널은 connection file에서 ZMQ 포트와 키를 읽고 다섯 소켓에 바인드해 연결을 기다립니다. 다섯 소켓이 모두 연결을 받을 수 있게 되면, 출력 캡처를 시작하기 전에 stderr에 `rokc jupyter: ready` 한 줄을 씁니다. 전송은 `tcp`만 지원하고(`ipc` 없음), CURVE 암호화는 쓰지 않습니다. 메시지는 connection file의 키로 HMAC 서명합니다.

### 소켓 채널

| 채널        | 용도                                                       |
|-------------|------------------------------------------------------------|
| `shell`     | 요청(execute, kernel_info, complete, is_complete 등)       |
| `iopub`     | 출력과 상태 브로드캐스트(status, stream, execute_result 등) |
| `stdin`     | 쓰지 않음. 입력 요청을 보내지 않음(stdin은 `/dev/null`) |
| `control`   | interrupt, shutdown                                        |
| `heartbeat` | 받은 것을 그대로 돌려보냄(echo)                            |

### 메시지 형식

각 메시지는 ZMQ 프레임 6개 이상으로 직렬화됩니다.

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
| `kernel_info_request`  | `language_info`의 name `rok`, mimetype `text/x-rok`, file_extension `.rok`, `protocol_version` `"5.3"` |
| `execute_request`      | 아래 "실행" 참고                                                                       |
| `is_complete_request`  | REPL의 이어 쓰기 규칙으로 `complete` / `incomplete` 판정                              |
| `complete_request`     | 세션의 이름, 멤버, 키워드를 후보로 함(9.8.5)                                          |
| `inspect_request`      | 커서 위 이름의 `text/plain` 도움말과 `found: true`(9.8.6), 없으면 `false`          |
| `history_request`      | 이 커널이 실행한 셀                                                                    |
| `comm_info_request`    | 빈 목록                                                                                |
| `shutdown_request`     | shell이나 control에서. 응답 후 종료                                                  |
| `interrupt_request`    | control에서(9.8.4)                                                                    |

### 실행

`execute_request`를 받으면 다음을 보냅니다.

1. iopub `execute_input`(코드와 실행 번호).
2. 셀이 **끝난 뒤** 그 출력을 보냅니다. stdout / stderr는 iopub `stream`으로 보냅니다. 실행 중에 조금씩 보내지 않습니다.
3. 값이 있으면 iopub `execute_result`. `text/plain`에 REPL과 같은 `타입: 값`을 담습니다(9.7.7).
4. 실패하면 iopub `error`. `evalue`는 stderr의 첫 줄이고, `traceback`은 stderr의 줄 전체입니다.
5. shell `execute_reply`(`execution_count` 포함, 성공 / 실패 status).

출력 MIME은 `text/plain` 하나입니다.

## 9.8.3. kernelspec 설치

Jupyter는 kernelspec 디렉터리에 설치된 커널만 인식합니다.

```
rok jupyter install [--env <name>]
```

kernelspec은 다음 디렉터리 아래의 `kernels/rok/kernel.json`입니다. `--env <name>`으로 설치하면 `kernels/rok-<name>/`입니다. 이때 표시 이름은 `rok (<name>)`, 실행 명령은 `rok --env <name> jupyter ...`입니다.

1. `$JUPYTER_DATA_DIR`이 있으면 그 디렉터리
2. macOS이면 `~/Library/Jupyter`
3. 그 밖에는 `$XDG_DATA_HOME/jupyter`, 없으면 `~/.local/share/jupyter`

`kernel.json`은 `rok` 프록시([9.10.6](910-rok-frontend.md#9106-toolchain))를 실행하고, 설치할 때의 `ROK_HOME`과 `PATH`를 `env`에 기록합니다.

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

`interrupt_mode`가 `"message"`이므로 인터럽트는 control 소켓의 `interrupt_request`로 옵니다. 커널은 실행 중인 셀을 끝내고, 세션은 그 셀 이전의 상태로 계속됩니다(실패한 셀과 같음, [9.7.0](907-jit-cells.md#970-셀-세션의-규칙)). 아직 컴파일 중인 셀은 인터럽트할 수 없습니다.

### 종료(kernel restart)

커널은 shell이나 control 소켓의 `shutdown_request`에 응답한 뒤 세션을 정리하고 프로세스를 종료합니다. 재시작이면 Jupyter가 커널을 다시 띄웁니다.

## 9.8.5. 코드 자동 완성

`complete_request`는 커서 앞의 조각을 세션의 바인딩, 항목, 모듈 경로, 멤버, 키워드에 맞춰 보고, `{ matches, cursor_start, cursor_end }`로 응답합니다.

## 9.8.6. inspect_request: 도움말

`inspect_request`(Shift+Tab)에는 `found: true`와 함께 `text/plain` 도움말로 응답합니다. 도움말에는 커서의 이름을 `name: type` 형태로 적고, 그 시그니처와 문서 주석([어휘 구조](../001-lexical/102-lexical-structure.md)의 `///` / `//!`)을 담습니다. 모르는 이름이면 `found: false`입니다.

## 9.8.7. REPL과 Jupyter의 코드 공유

REPL에서 작성한 코드는 그대로 Jupyter 셀로 옮길 수 있고, 반대도 됩니다. 둘 다 같은 셀 세션과 의미론([9.7.0](907-jit-cells.md#970-셀-세션의-규칙))을 따르기 때문입니다.

### 차이

다만 두 표면에는 다음과 같은 차이가 있습니다.

| 측면        | REPL          | Jupyter               |
|-------------|---------------|-----------------------|
| 입력        | 한 줄 / 한 식 | 한 셀(여러 줄)        |
| 출력        | 텍스트만      | 텍스트(`text/plain`)  |
| 상태 보존   | 종료 시 폐기  | 노트북 파일에 셀 보존 |

## 9.8.8. 보안 / 격리

커널은 사용자 권한으로 임의 코드를 실행합니다. 격리는 하지 않습니다.

## 9.8.9. 다음

이어지는 [디버그 / 진단 도구](909-debug-tools.md)는 gdb 친화 기능, 프로파일, `--emit` 등을 정의합니다.
