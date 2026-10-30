# 3. Std 계층

타깃에 의존하는 영역의 표준 라이브러리입니다. 파일 시스템, 네트워크, 시간, OS 스레드, I/O, 환경 변수, 프로세스, 경로를 다룹니다. 사용자 환경(운영체제 / 플랫폼)에 따라 쓸 수 있는 기능이 다릅니다.

## 위치

사양 본체는 `spec/ko/refs-standard-libraries/300-std/` 아래에 있습니다.

## 의존성

위로는 `rok`와 `unsafe`에 의존합니다. 아래로는 사용자 코드가 std에 의존합니다. 단, std는 명시적으로 가져올 때만 끌려 들어옵니다.

## 사용자 import 정책

묵시 `use rok::core::**;`에는 std 항목이 들어 있지 않습니다. `rok::std::*`는 따로 명시적으로 가져와야 합니다. 타깃마다 가용성이 다르므로 사용자가 플랫폼 의존성을 **명시적으로** 드러내도록 설계했습니다.

```rok
use rok::std::fs;
use rok::std::time::Instant;
use rok::std::thread::Builder;
```

## 모듈 목차

이 tier는 다음 열 모듈로 이루어집니다.

- [3.1. `fs::` - 파일 시스템 (atomic ops, Dir capability, reflink copy)](301-fs.md)
- [3.2. `net::` - TCP / UDP / Unix + TLS 1.3](302-net.md)
- [3.3. `time::` - `Instant` / `SystemTime` / `Duration` / `BootTime`](303-time.md)
- [3.4. `thread::` - OS 스레드 (1:1, `task::`와 별개)](304-thread.md)
- [3.5. `io::` - `Reader` / `Writer` / stdin/out/err / `BufReader`](305-io.md)
- [3.6. `env::` - 환경 변수 / args / current_dir / home_dir](306-env.md)
- [3.7. `process::` - `Command` / `Child` / pidfd / kill_on_drop](307-process.md)
- [3.8. `path::` - UTF-8 `Path` (Python 식) + `RawPath` 이스케이프](308-path.md)
- [3.9. `rokon::` - ROKON 데이터 형식 (`.rokon`)](309-rokon.md)
- [3.10. `crypto::`: OS 난수 원천](310-crypto.md)

## 핵심 설계 원칙

### 1. 블로킹 API와 `*_async`: 비동기 트레잇 없음

std의 I/O 함수는 동기 함수이고, 호출한 OS 스레드를 막습니다. task 안에서 기다리는 동안 워커를 내주려면 같은 연산의 `*_async` 메서드를 `await`로 기다립니다([3.2.0](302-net.md#320-공통-규칙)). 스택 없는(stackless) 코루틴에서는 일반 함수가 task를 중단할 수 없습니다. 그래서 동기 호출이 저절로 비동기가 되지는 않습니다.

```rok
// blocking - OS 스레드 (thread::spawn) 나 task 밖에서
fn fetch(addr: str) throws (IoError) -> List<u8> {
    using conn = try net::tcp_connect(addr) {
        try conn.read_to_end()
    }
}

// async - task 만 중단
async fn fetch_async(addr: str) throws (IoError) -> List<u8> {
    using conn = try await net::tcp_connect_async(addr) {
        try await conn.read_to_end_async()
    }
}
```

`AsyncRead` / `AsyncWrite` 트레잇은 **두지 않습니다**. async 판은 타입의 고유 메서드와 함수입니다. 소켓은 I/O 드라이버의 준비 상태를 기다립니다([3.2.0](302-net.md#320-공통-규칙)). 일반 파일은 블로킹 호출을 다른 OS 스레드에서 돌립니다([3.1.4a](301-fs.md#314a-비동기-파일-io_async)).

### 2. I/O 백엔드: 런타임 세부 사항

준비 대기(readiness)는 **런타임의 책임**입니다([refs-lang 7.7](../../refs-lang/007-runtime/707-io-and-timer.md)). 사용자 코드는 어떤 백엔드가 돌아가는지 몰라도 됩니다.

| 플랫폼          | 백엔드                      |
|-----------------|-----------------------------|
| Linux           | epoll (edge-triggered)      |
| macOS / FreeBSD | kqueue (`EV_CLEAR`)         |
| Windows         | IOCP                        |
| WASI            | `wasi:io/poll` (`wasm32-wasi`), `poll_oneoff` (`wasm32-wasi-threads`) |

### 3. std의 TLS 1.3: HTTP는 제외

TLS 1.3은 **보안에 결정적인** 영역입니다. 앱마다 TLS를 직접 구현하면 보안 취약점이 크게 늘어납니다. 그래서 std에 넣었습니다.

```rok
use rok::std::net::tls;

let conn = try net::tcp_connect("example.com:443");
let tls_conn = try tls::Client::new(conn, "example.com");
let req = b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n";
try tls_conn.write_all(req);
let resp = try tls_conn.read_to_end();
```

스택은 `rok::core::crypto`([2.21](../200-core/221-crypto.md)) 위에 Rok으로 작성한 자체 구현입니다. OS나 서드파티 TLS 라이브러리는 쓰지 않습니다([3.2.5](302-net.md#325-tls-13-nettls)).

반면 HTTP/1, HTTP/2, HTTP/3(QUIC)은 **std 밖**의 별도 `rok/http` 패키지로 나눕니다. Go `net/http`가 **1.0에서 동결된** 사례에서 얻은 교훈입니다.

### 4. capability 기반 fs: Dir 핸들

WASI Preview 2의 미리 연 디렉터리 핸들(pre-opened directory handle) 모델을 따릅니다. 모든 fs 연산은 **Dir을 거쳐** 일어납니다.

```rok
use rok::std::fs::{Dir, File};

let root: Dir = Dir::root();                  // capability - process 시작 시
let data_dir: Dir = try root.open_dir("data"); // sub-cap

let f: File = try data_dir.open("config.toml");
let bytes = try f.read_to_end();
```

이 모델의 장점은 세 가지입니다. 첫째, TOCTOU를 피합니다. `open_dir` 뒤의 `open`에는 경쟁 상태가 없습니다. 둘째, 샌드박스가 자연스럽습니다. 자식 task에 **좁은 Dir**만 넘기면 됩니다. 셋째, WASI와 호환됩니다. 같은 코드가 네이티브와 WASI 양쪽에서 동작합니다.

편의를 위해 `fs::open(absolute_path)`도 허용합니다. 이 호출은 내부에서 `Dir::root().open(...)`으로 풀립니다.

### 5. UTF-8 Path(Python 방식)

Rust `Path` / `PathBuf` + `OsStr`의 **사용성 부담**을 피하려는 결정입니다.

```rok
use rok::std::path::Path;

let p: Path = Path::from("data/config.toml");
let parent: Path = p.parent()?;
let with_ext: Path = p.with_extension("bak");

// `/` 연산자 (Python pathlib 식)
let full = Path::from("/etc") / "nginx" / "nginx.conf";
```

`Path`는 **UTF-8 newtype**입니다. UTF-8이 아닌 파일명(Unix에 드물게 있음)은 `RawPath`(바이트)로 따로 처리합니다. 코드의 99%는 `Path`만으로 충분합니다.

자세한 비교와 근거는 [3.8 path](308-path.md)에 있습니다.

### 6. 모든 I/O가 `throws (IoError)`

panic은 거의 일어나지 않습니다(OOM 제외). 파일 없음, 권한 거부, 네트워크 실패는 모두 `throws (IoError)` 시그니처로 던집니다. 호출에 `try`를 붙여 전파합니다. 에러 타입 사이의 자동 변환은 없습니다. 다른 에러 타입으로 바꾸려면 `catch`에서 감싸 다시 던집니다.

```rok
fn load_config() throws (ConfigError) -> Config {
    let bytes = try fs::read("config.toml") catch e: IoError {
        throw ConfigError::Io(e);              // IoError → ConfigError (명시 변환)
    };
    try parse(bytes)
}
```

## 타깃 가용성

플랫폼별 모듈 가용성은 다음과 같습니다.

| 모듈                | linux/macos/windows | wasm32-wasi   | wasm32-wasi-threads | wasm32 (browser) |
|---------------------|---------------------|---------------|---------------------|------------------|
| `rok::std::fs`      | ✓                   | ✓ (cap-based) | ✓ (cap-based)       | ✗ (모든 호출이 `Unsupported`) |
| `rok::std::net`     | ✓ (+ TLS 1.3)       | ✓ (`wasi:sockets`) | 호스트가 넘긴 listener만 | ✗ (`fetch` 사용) |
| `rok::std::time`    | ✓                   | ✓             | ✓                   | ✓ (Date.now)     |
| `rok::std::thread`  | ✓                   | 스레드 하나 (`spawn` `Unsupported`) | ✓ (wasi-threads) | 스레드 하나 (`spawn` `Unsupported`) |
| `rok::std::io`      | ✓                   | ✓             | ✓                   | console / stdin은 EOF |
| `rok::std::env`     | ✓                   | ✓             | ✓                   | ✗                |
| `rok::std::process` | ✓                   | ✗             | ✗                   | ✗                |
| `rok::std::path`    | ✓                   | ✓             | ✓                   | ✓                |

✗는 모듈은 빌드되지만 그 연산이 실행 중에 `IoErrorKind::Unsupported`로 실패한다는 뜻입니다. 컴파일 타임에 모듈이 없다는 뜻이 아닙니다. 브라우저에서 stdout / stderr는 호스트 콘솔로 가고 stdin은 EOF입니다. 플랫폼별 동작은 각 모듈 사양의 가용성 절에 더 자세히 있습니다.
