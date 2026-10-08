# 9.8. Jupyter 커널 통합

[실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 의 Jupyter 표면의 두 배포 형태 (로컬 네이티브 커널과 브라우저 WASM 커널) 에 대한 구체 통합을 정의한다.

## 9.8.1. 두 배포 형태

```
1. 로컬 네이티브 커널 (기본 형태)
   - 사용자 PC 의 Jupyter 가 별도 프로세스로 nokc 커널 바이너리를 실행
   - ZeroMQ 기반 Jupyter 5.5 wire protocol 로 통신
   - OS 자원 (파일 / 네트워크 / 외부 프로세스) 모두 사용 가능

2. 브라우저 WASM 커널 (부가 형태)
   - JupyterLite 구도. 모든 것이 브라우저 안에서 동작
   - nokc 커널을 wasm32 으로 빌드해 페이지에 로드
   - 통신은 브라우저 메시지 채널 (Web Worker / SharedArrayBuffer)
   - OS 자원 없음 (file / net 제한)
```

두 형태는 같은 NokJitSession 엔진 ([JIT 엔진 (NokJitSession)](906-jit-engine.md)) 위에 서로 다른 통신 계층을 얹는다.

## 9.8.2. 로컬 네이티브 커널: 동작

```
1. 사용자 PC 에서 jupyter notebook 또는 jupyter lab 실행.
2. 사용자가 새 노트북 생성 — kernel = "nok" 선택.
3. Jupyter 가 nokc jupyter --connection-file <path> 를 별도 프로세스로 실행.
4. nokc 가 connection file 의 ZMQ port 정보를 읽고 5 개 소켓에 연결.
5. Jupyter 와 nokc 사이 — Jupyter wire protocol 5.5 메시지 교환.
6. 사용자 셀 실행 → execute_request 메시지 → nokc 가 NokJitSession 으로 처리 → 결과 메시지.
```

### 5 개 ZMQ 소켓 채널

Jupyter wire protocol 의 표준 5 채널이다.

| 채널        | 용도                                             |
| ----------- | ------------------------------------------------ |
| `shell`     | 실행 요청 (execute_request, complete_request 등) |
| `iopub`     | 출력 broadcast (display_data, stream, error 등)  |
| `stdin`     | 사용자 입력 요청 (input_request)                 |
| `control`   | 제어 (interrupt, shutdown)                       |
| `heartbeat` | 연결 유지                                        |

각 메시지는 HMAC 서명을 거친다 (connection file 의 key 사용).

### 메시지 형식

각 메시지는 6+ 개 ZMQ frame 으로 직렬화된다.

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

자세한 형식은 Jupyter wire protocol 5.5 표준 문서에 따른다.

### `execute_request` 처리

사용자가 셀 실행:

```
1. shell 소켓에 execute_request 도착.
   content: { code: "...", silent: false, store_history: true, ... }
2. nokc 가 NokJitSession 의 cell 처리 흐름 ([JIT 셀 실행 흐름과 상태 관리](907-jit-cells.md)) 을 시작.
3. cell 시작 시 iopub 에 status: busy 메시지.
4. 진행 중 print() 등의 출력은 iopub stream 메시지로 전달.
5. 결과 식의 값은 iopub 의 execute_result 메시지로 전달.
6. 셀 완료 시 shell 의 execute_reply 응답.
7. 마지막에 iopub 의 status: idle.
```

### `display_data`: 풍부한 출력

표준 라이브러리의 시각화 / 표 출력은 `display_data` 메시지로 MIME 형식을 지정한다.

```
display_data:
  content:
    data:
      "text/plain": "..."
      "image/png": "..." (base64)
      "application/vnd.nok.dataframe+json": "..."
    metadata: { ... }
```

Jupyter 가 각 MIME 타입을 적절한 renderer 로 표시한다.

### `kernel_info`

Jupyter 시작 시 kernel 정보를 묻는 `kernel_info_request` 가 온다. 이에 nokc 가 응답한다.

```
{
  "language": "nok",
  "language_version": "1.0",
  "implementation": "nokc",
  "implementation_version": "1.0.0",
  "protocol_version": "5.5",
  "banner": "nok REPL ...",
  "language_info": {
    "name": "nok",
    "file_extension": ".nok",
    "mimetype": "text/x-nok",
    ...
  }
}
```

## 9.8.3. kernelspec 설치

Jupyter 는 kernelspec 디렉토리에 설치된 kernel 만 인식한다.

`nokc jupyter --install-kernel` 실행 시:

```
~/.local/share/jupyter/kernels/nok/
├── kernel.json
└── (선택) logo-32x32.png, logo-64x64.png
```

`kernel.json` 내용:

```json
{
  "argv": [
    "/path/to/nokc",
    "jupyter",
    "--connection-file",
    "{connection_file}"
  ],
  "display_name": "nok",
  "language": "nok"
}
```

설치하면 Jupyter, VS Code, nteract 등 어디서든 nok 커널을 쓸 수 있다.

## 9.8.4. 인터럽트 / 종료

### 인터럽트 (Ctrl+C)

`control` 소켓에 `interrupt_request` 도착.

```
1. 진행 중인 cell 의 cooperative cancellation 을 시도.
2. nok 의 task 들이 cancellation token 을 검사.
3. 협력하지 않는 task 는 강제 종료 (구현 결정).
4. control 소켓에 interrupt_reply 응답.
```

### 종료 (kernel restart)

`control` 소켓에 `shutdown_request`.

```
1. 모든 진행 중 task cancel.
2. NokJitSession 정리.
3. ORCv2 dylib 해제.
4. 프로세스 종료.
```

restart 인 경우 Jupyter 가 자동으로 다시 띄운다.

## 9.8.5. 코드 자동 완성

`shell` 의 `complete_request`.

```
1. 입력 + 커서 위치.
2. nokc 가 그 위치의 유효한 식별자 후보를 추론.
3. complete_reply 응답: { matches: [...], cursor_start, cursor_end }
```

자동 완성은 NokJitSession 의 hir_env 를 활용한다. 이미 정의된 변수, 가시 모듈의 항목, 키워드 등이 후보가 된다.

## 9.8.6. inspect_request: 도움말

`shell` 의 `inspect_request` (Shift+Tab in Jupyter).

```
1. 커서 위치의 식별자 또는 식 식별.
2. 그 항목의 시그니처, 문서 주석, 예시를 추출.
3. inspect_reply 응답.
```

문서 주석은 [어휘 구조](../001-lexical/102-lexical-structure.md) 의 `///` / `//!` 가 항목에 부착된 결과를 활용한다.

## 9.8.7. 브라우저 WASM 커널

JupyterLite 구도. 모든 것이 브라우저에서 동작한다.

### wasm32 빌드

```
nokc build --target=wasm32-unknown-unknown --out=nokc-kernel.wasm
```

빌드한 wasm 모듈을 Jupyter 의 web kernel 로 로드한다.

### 브라우저 메시지 채널

Web Worker 안에서 `postMessage` 로 메시지를 주고받는다. ZMQ 가 아니라 브라우저 네이티브 채널이다.

```
Jupyter Frontend (UI)
    ↕  postMessage
Web Worker (nokc 의 wasm)
```

메시지 내용은 Jupyter wire protocol 과 같은 형식 (JSON) 이다. 전송 매체만 다르다.

### SharedArrayBuffer

대량 데이터 (이미지, dataframe 등) 는 `SharedArrayBuffer` 로 zero-copy 전달한다.

CORS, Cross-Origin-Embedder-Policy 등 브라우저 보안 정책을 만족해야 한다. 자세한 사양은 JupyterLite 문서를 참조한다.

### OS 자원 제한

브라우저 안의 nok 에는 다음 라이브러리 영역이 없다 ([표준 라이브러리 계층](../000-meta/005-stdlib-tiers.md)).

- `std::fs`: 파일 시스템.
- `std::net`: 네트워크 (TCP / UDP).
- `std::process`: 프로세스 생성.

`core` 영역의 모든 것 (일반 nok 코드, 컬렉션, 동시성 primitive) 은 동작한다.

브라우저의 fetch, IndexedDB 등은 별도 라이브러리로 추가할 수 있다 (사양 외).

## 9.8.8. REPL 과 Jupyter 의 코드 공유

REPL 에서 작성한 코드는 그대로 Jupyter 셀로 옮길 수 있다 (반대도 마찬가지). 같은 NokJitSession 엔진과 같은 의미론을 따른다.

### 차이

| 측면        | REPL          | Jupyter               |
| ----------- | ------------- | --------------------- |
| 입력        | 한 줄 / 한 식 | 한 셀 (여러 줄)       |
| 출력        | text only     | text + 시각화         |
| undo        | 없음          | 셀 삭제               |
| 상태 보존   | 종료 시 폐기  | 노트북 파일에 셀 보존 |
| 디스크 캐시 | 옵션          | 옵션                  |

## 9.8.9. 보안 / 격리

로컬 커널은 사용자 권한으로 실행된다. 임의 코드를 실행할 수 있다 (사용자 책임).

브라우저 커널은 브라우저 샌드박스 안에서 동작한다. OS 자원 접근이 제한된다.

자세한 보안 모델은 Jupyter 표준 문서를 참조한다.

## 9.8.10. 다음

[디버그 / 진단 도구](909-debug-tools.md) 에서 gdb 친화, profile, `--emit` 등을 정의한다.
