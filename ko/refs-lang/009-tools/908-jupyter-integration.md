# 9.8. Jupyter 커널 통합

이 장에서는 [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 에서 다룬 Jupyter 표면의 두 배포 형태, 즉 로컬 네이티브 커널과 브라우저 WASM 커널에 대한 구체적인 통합 방식을 정의합니다.

## 9.8.1. 두 배포 형태

먼저 첫 번째 형태인 로컬 네이티브 커널은 기본 형태에 해당합니다. 사용자 PC 의 Jupyter 가 별도 프로세스로 rokc 커널 바이너리를 실행하며, ZeroMQ 기반 Jupyter 5.5 wire protocol 로 통신합니다. 이 형태에서는 OS 자원, 즉 파일과 네트워크와 외부 프로세스를 모두 사용할 수 있습니다.

두 번째 형태인 브라우저 WASM 커널은 부가 형태에 해당합니다. JupyterLite 구도를 따르며 모든 것이 브라우저 안에서 동작합니다. rokc 커널을 wasm32 으로 빌드해서 페이지에 로드하며, 통신은 브라우저 메시지 채널인 Web Worker 와 SharedArrayBuffer 를 사용합니다. 다만 OS 자원이 없기 때문에 file 과 net 사용에 제한이 따릅니다.

두 형태는 같은 RokJitSession 엔진 ([JIT 엔진 (RokJitSession)](906-jit-engine.md)) 위에 서로 다른 통신 계층을 추가하는 구조입니다.

## 9.8.2. 로컬 네이티브 커널: 동작

로컬 네이티브 커널의 동작 흐름을 살펴보겠습니다. 먼저 사용자 PC 에서 jupyter notebook 또는 jupyter lab 을 실행합니다. 이어서 사용자가 새 노트북을 생성하면서 kernel 로 "rok" 를 선택하면, Jupyter 가 `rokc jupyter --connection-file <path>` 를 별도 프로세스로 실행합니다. 그러면 rokc 가 connection file 의 ZMQ port 정보를 읽고 5 개 소켓에 연결합니다. 이때부터 Jupyter 와 rokc 사이에서 Jupyter wire protocol 5.5 메시지가 교환됩니다. 사용자가 셀을 실행하면 execute_request 메시지가 전달되고, rokc 가 RokJitSession 으로 처리한 뒤 결과 메시지를 돌려보내게 됩니다.

### 5 개 ZMQ 소켓 채널

다음 표는 Jupyter wire protocol 의 표준 5 채널을 정리한 것입니다.

| 채널        | 용도                                             |
|-------------|--------------------------------------------------|
| `shell`     | 실행 요청 (execute_request, complete_request 등) |
| `iopub`     | 출력 broadcast (display_data, stream, error 등)  |
| `stdin`     | 사용자 입력 요청 (input_request)                 |
| `control`   | 제어 (interrupt, shutdown)                       |
| `heartbeat` | 연결 유지                                        |

각 메시지는 connection file 의 key 를 사용해 HMAC 서명을 통과하게 됩니다.

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
 b"\xf0\x9f\x90\xb1"   # 추가 raw 데이터 buffer (선택)
]
```

자세한 형식은 Jupyter wire protocol 5.5 표준 문서를 따르고 있습니다.

### `execute_request` 처리

사용자가 셀을 실행할 때의 처리 흐름은 다음과 같습니다. 먼저 shell 소켓에 execute_request 가 도착하는데, 그 content 에는 `{ code: "...", silent: false, store_history: true, ... }` 형태의 정보가 담겨 있습니다. 이를 받은 rokc 는 RokJitSession 의 cell 처리 흐름 ([JIT 셀 실행 흐름과 상태 관리](907-jit-cells.md)) 을 시작합니다. cell 시작 시점에 iopub 으로 status: busy 메시지가 전달되고, 진행 중에 발생하는 print() 등의 출력은 iopub stream 메시지로 전송됩니다. 결과 식의 값은 iopub 의 execute_result 메시지로 전달되며, 셀이 완료되면 shell 에 execute_reply 응답이 돌아갑니다. 마지막으로 iopub 에 status: idle 메시지가 발신되어 처리가 마무리됩니다.

### `display_data`: 풍부한 출력

표준 라이브러리의 시각화나 표 출력은 `display_data` 메시지로 MIME 형식을 지정합니다.

```
display_data:
  content:
    data:
      "text/plain": "..."
      "image/png": "..." (base64)
      "application/vnd.rok.dataframe+json": "..."
    metadata: { ... }
```

Jupyter 가 각 MIME 타입을 적절한 renderer 로 표시해 줍니다.

### `kernel_info`

Jupyter 가 시작될 때 kernel 정보를 묻는 `kernel_info_request` 가 들어오게 되는데, 이에 대해 rokc 는 다음과 같이 응답합니다.

```
{
  "language": "rok",
  "language_version": "1.0",
  "implementation": "rokc",
  "implementation_version": "1.0.0",
  "protocol_version": "5.5",
  "banner": "rok REPL ...",
  "language_info": {
    "name": "rok",
    "file_extension": ".rok",
    "mimetype": "text/x-rok",
    ...
  }
}
```

## 9.8.3. kernelspec 설치

Jupyter 는 kernelspec 디렉토리에 설치된 kernel 만 인식합니다. 따라서 `rokc jupyter --install-kernel` 을 실행하면 다음과 같은 구조가 만들어집니다.

```
~/.local/share/jupyter/kernels/rok/
├── kernel.json
└── (optional) logo-32x32.png, logo-64x64.png
```

> optional: 선택.

이때 `kernel.json` 의 내용은 다음과 같이 구성됩니다.

```json
{
  "argv": [
    "/path/to/rokc",
    "jupyter",
    "--connection-file",
    "{connection_file}"
  ],
  "display_name": "rok",
  "language": "rok"
}
```

이렇게 설치하고 나면 Jupyter, VS Code, nteract 등 어디서든 rok 커널을 사용할 수 있습니다.

## 9.8.4. 인터럽트 / 종료

### 인터럽트 (Ctrl+C)

`control` 소켓에 `interrupt_request` 가 도착하면, 먼저 진행 중인 cell 의 cooperative cancellation 을 시도합니다. 이어서 rok 의 task 들이 cancellation token 을 검사하게 됩니다. 협력하지 않는 task 는 강제 종료되며, 이 부분은 구현 결정 사항으로 남겨 두었습니다. 마지막으로 control 소켓에 interrupt_reply 응답을 돌려보냅니다.

### 종료 (kernel restart)

`control` 소켓에 `shutdown_request` 가 도착할 경우의 처리 순서는 다음과 같습니다. 먼저 모든 진행 중 task 를 cancel 하고, 이어서 RokJitSession 을 정리합니다. 그다음 ORCv2 dylib 를 해제하고 마지막으로 프로세스를 종료하게 됩니다.

restart 의 경우에는 Jupyter 가 자동으로 다시 띄워 줍니다.

## 9.8.5. 코드 자동 완성

`shell` 의 `complete_request` 처리 흐름을 살펴보면 다음과 같습니다. 먼저 입력과 커서 위치를 받은 뒤, rokc 가 그 위치의 유효한 식별자 후보를 추론합니다. 그러고 나서 `{ matches: [...], cursor_start, cursor_end }` 형태의 complete_reply 를 응답으로 돌려보냅니다.

자동 완성은 RokJitSession 의 hir_env 를 활용하고 있어서, 이미 정의된 변수와 가시 모듈의 항목, 키워드 등이 후보에 포함됩니다.

## 9.8.6. inspect_request: 도움말

`shell` 의 `inspect_request` 는 Jupyter 에서 Shift+Tab 으로 호출됩니다. 처리 순서는 우선 커서 위치의 식별자 또는 식을 식별하고, 그 항목의 시그니처와 문서 주석, 예시를 추출한 뒤 inspect_reply 로 응답하는 흐름입니다.

문서 주석은 [어휘 구조](../001-lexical/102-lexical-structure.md) 에서 정의한 `///` 와 `//!` 가 항목에 부착된 결과를 활용합니다.

## 9.8.7. 브라우저 WASM 커널

JupyterLite 구도를 따르며 모든 것이 브라우저에서 동작합니다.

### wasm32 빌드

다음 명령으로 wasm32 빌드를 수행합니다.

```
rokc build --target=wasm32-unknown-unknown --out=rokc-kernel.wasm
```

빌드한 wasm 모듈은 Jupyter 의 web kernel 로 로드해서 사용하면 됩니다.

### 브라우저 메시지 채널

Web Worker 안에서 `postMessage` 로 메시지를 주고받습니다. ZMQ 가 아니라 브라우저 네이티브 채널을 사용하는 점이 특징입니다.

```
Jupyter Frontend (UI)
    ↕  postMessage
Web Worker (rokc as wasm)
```

> rokc as wasm: rokc 의 wasm.

메시지 내용은 Jupyter wire protocol 과 동일한 JSON 형식이며, 다만 전송 매체만 다릅니다.

### SharedArrayBuffer

이미지나 dataframe 같은 대량 데이터는 `SharedArrayBuffer` 로 zero-copy 전달합니다.

이때 CORS 와 Cross-Origin-Embedder-Policy 등의 브라우저 보안 정책을 만족해야 하며, 자세한 사양은 JupyterLite 문서를 참조하면 좋습니다.

### OS 자원 제한

브라우저 안의 rok 에는 다음과 같은 라이브러리 영역이 존재하지 않습니다 ([표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md)).

- `rok::std::fs`: 파일 시스템.
- `rok::std::net`: 네트워크 (TCP / UDP).
- `rok::std::process`: 프로세스 생성.

다만 `rok` 영역의 모든 것, 즉 일반 rok 코드와 컬렉션, 동시성 primitive 는 정상적으로 동작합니다.

브라우저의 fetch 나 IndexedDB 등은 별도 라이브러리로 추가할 수 있는데, 이 부분은 사양 외의 영역에 해당합니다.

## 9.8.8. REPL 과 Jupyter 의 코드 공유

REPL 에서 작성한 코드는 그대로 Jupyter 셀로 옮길 수 있고, 반대 방향도 마찬가지로 가능합니다. 같은 RokJitSession 엔진과 같은 의미론을 따르고 있기 때문입니다.

### 차이

다만 양쪽 표면 사이에는 몇 가지 차이가 존재하며, 다음 표에 정리합니다.

| 측면        | REPL          | Jupyter               |
|-------------|---------------|-----------------------|
| 입력        | 한 줄 / 한 식 | 한 셀 (여러 줄)       |
| 출력        | text only     | text + 시각화         |
| undo        | 없음          | 셀 삭제               |
| 상태 보존   | 종료 시 폐기  | 노트북 파일에 셀 보존 |
| 디스크 캐시 | 옵션          | 옵션                  |

## 9.8.9. 보안 / 격리

로컬 커널은 사용자 권한으로 실행되므로 임의 코드를 실행할 수 있으며, 이는 사용자의 책임 영역으로 남게 됩니다.

반면 브라우저 커널은 브라우저 샌드박스 안에서 동작하기 때문에 OS 자원에 대한 접근이 제한됩니다.

자세한 보안 모델은 Jupyter 표준 문서를 참조합니다.

## 9.8.10. 다음

이어지는 [디버그 / 진단 도구](909-debug-tools.md) 에서는 gdb 친화 기능과 profile, `--emit` 등을 정의합니다.
