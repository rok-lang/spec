# 3. Std tier

target-dep 영역의 표준 라이브러리입니다. 파일시스템, 네트워크, 시간, OS 스레드, I/O, 환경 변수, 프로세스, 경로를 포괄하며, 사용자 환경 (운영체제 / 플랫폼) 에 따라 가용성에 차이가 납니다.

## 위치

spec 본체는 `spec/ko/refs-standard-libraries/300-std/` 아래 둡니다.

## 의존성

위로는 `rok` 와 `unsafe` 에 의존합니다. 아래로는 사용자 코드가 의존자 위치에 서지만, 어디까지나 명시 import 가 있어야 끌려 들어옵니다.

## 사용자 import 정책

묵시 `use rok::;` 는 std 항목을 포함하지 않습니다 — `rok::std::*` 는 별도 명시 import 를 강제합니다. target 별 가용성 차이가 있기 때문에, 사용자가 **명시적으로** 플랫폼 의존성을 표현하도록 설계했습니다.

```rok
use rok::std::fs;
use rok::std::time::Instant;
use rok::std::thread::Builder;
```

## 모듈 목차

본 tier 는 다음 여덟 모듈로 구성합니다.

- [3.1. `fs::` — 파일 시스템 (atomic ops, Dir 캡슐, reflink copy)](301-fs.md)
- [3.2. `net::` — TCP / UDP / Unix + TLS 1.3](302-net.md)
- [3.3. `time::` — `Instant` / `SystemTime` / `Duration` / `BootTime`](303-time.md)
- [3.4. `thread::` — OS 스레드 (1:1, `task::` 와 별개)](304-thread.md)
- [3.5. `io::` — `Reader` / `Writer` / stdin/out/err / `BufReader`](305-io.md)
- [3.6. `env::` — 환경 변수 / args / current_dir / home_dir](306-env.md)
- [3.7. `process::` — `Command` / `Child` / pidfd / kill_on_drop](307-process.md)
- [3.8. `path::` — UTF-8 `Path` (Python 식) + `RawPath` 이스케이프](308-path.md)

## 핵심 디자인 원칙

### 1. blocking API + `*_async` — async 트레잇 없음

std 의 I/O 함수는 sync 이며 호출한 OS 스레드를 막습니다. task 안에서 기다리는 동안 worker 를 내주려면 같은 연산의 `*_async` 메서드를 `await` 합니다 ([3.2.0](302-net.md#320-공통-규칙)). stackless 코루틴에서는 보통 함수가 task 를 중단시킬 수 없으므로, sync 호출이 저절로 async 가 되지는 않습니다.

```rok
// blocking — OS 스레드 (thread::spawn) 나 task 밖에서
fn fetch(addr: str) throws (IoError) -> List<u8> {
    using conn = try net::tcp_connect(addr);
    return try conn.read_all();
}

// async — task 만 중단
async fn fetch_async(addr: str) throws (IoError) -> List<u8> {
    using conn = try await net::tcp_connect_async(addr);
    return try await conn.read_all_async();
}
```

`AsyncRead` / `AsyncWrite` 트레잇은 **두지 않습니다**. async 판은 타입의 고유 메서드이며, 아직 소켓에만 있습니다. 일반 파일의 async I/O 는 제공하지 않습니다.

### 2. I/O backend — runtime detail

준비 대기 (readiness) 는 **runtime 의 책임** 입니다 ([refs-lang 7.7](../../refs-lang/007-runtime/707-io-and-timer.md)). 사용자 코드는 어떤 backend 가 돌아가는지 알 필요가 없습니다.

| 플랫폼          | backend                     |
|-----------------|-----------------------------|
| Linux           | epoll (edge-triggered)      |
| macOS / FreeBSD | kqueue (`EV_CLEAR`)         |
| Windows         | IOCP (향후)                 |
| WASI            | poll_oneoff (향후)          |

io_uring 과 IOCP 같은 completion 모델 backend 는 향후 과제입니다.

### 3. TLS 1.3 in std — HTTP out

TLS 1.3 는 **보안 critical** 영역입니다. 모든 앱이 TLS 를 새로 구현하면 보안 취약점이 폭발하므로 std 에 통합하는 쪽을 택했습니다.

```rok
use rok::std::net::tls;

let conn = try net::tcp_connect("example.com:443");
let tls_conn = try tls::Client::new(conn, "example.com");
let req = b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n";
try tls_conn.write_all(req);
let resp = try tls_conn.read_all();
```

backend 로는 rustls 또는 aws-lc-rs 식을 상정합니다 (memory-safe, no OpenSSL FFI).

반면 HTTP/1, HTTP/2, HTTP/3 (QUIC) 은 **std 외부** 에 둡니다 — 별도 `rok/http` 패키지로 분리합니다. Go `net/http` 의 **1.0 frozen** 사례에서 얻은 학습입니다.

### 4. Capability-based fs — Dir 핸들

WASI Preview 2 의 pre-opened directory handle 모델을 따릅니다. 모든 fs op 가 **Dir 을 통해** 일어납니다.

```rok
use rok::std::fs::{Dir, File};

let root: Dir = Dir::root();                  // capability — process 시작 시
let data_dir: Dir = try root.open_dir("data"); // sub-cap

let f: File = try data_dir.open("config.toml");
let bytes = try f.read_to_end();
```

이 모델의 장점은 세 가집니다. 첫째, TOCTOU 를 회피합니다 — `open_dir` 후 `open` 이 race-free 다. 둘째, 샌드박스가 자연스럽습니다 — 자식 task 에 **좁은 Dir** 만 전달하면 됩니다. 셋째, WASI 와 호환됩니다 — 같은 코드가 native 와 WASI 양쪽에서 동작합니다.

편의 차원에서 `fs::open(absolute_path)` 도 가능하게 합니다. 내부적으로는 `Dir::root().open(...)` 으로 풀린다.

### 5. UTF-8 Path (Python 식)

Rust `Path` / `PathBuf` + `OsStr` 가 안기는 **ergonomics 부담** 을 회피하려는 결정입니다.

```rok
use rok::std::path::Path;

let p: Path = Path::from("data/config.toml");
let parent: Path = p.parent()?;
let with_ext: Path = p.with_extension("bak");

// `/` 연산자 (Python pathlib 식)
let full = Path::from("/etc") / "nginx" / "nginx.conf";
```

`Path` 는 **UTF-8 newtype** 입니다. non-UTF-8 파일명 (드물게 Unix 에 존재) 은 `RawPath` (bytes) 로 별도 처리합니다. 코드의 99% 는 `Path` 만으로 충분할 것입니다.

자세한 비교와 권위 근거는 [3.8 path](308-path.md) 에서 다룹니다.

### 6. 모든 I/O 가 `throws (IoError)`

panic 은 거의 발생하지 않습니다 (OOM 외). file not found, permission denied, network failure 어느 쪽이든 모두 `throws (IoError)` 시그니처로 던져지며, 호출을 `try` 로 표시해 자연스럽게 전파할 수 있습니다. 에러 타입 사이의 자동 변환은 없으므로, 다른 에러 타입으로 바꾸려면 `catch` 에서 감싸 다시 던집니다.

```rok
fn load_config() throws (ConfigError) -> Config {
    let bytes = try fs::read("config.toml") catch e: IoError {
        throw ConfigError::Io(e);              // IoError → ConfigError (명시 변환)
    };
    try parse(bytes)
}
```

## target 가용성

플랫폼별 모듈 가용성은 다음과 같습니다.

| 모듈                | linux/macos/windows | wasi          | wasm32 (browser) |
|---------------------|---------------------|---------------|------------------|
| `rok::std::fs`      | ✓                   | ✓ (cap-based) | ✗ (no FS API)    |
| `rok::std::net`     | ✓ (+ TLS 1.3)       | ✓             | ✗ (use `fetch`)  |
| `rok::std::time`    | ✓                   | ✓             | ✓ (Date.now)     |
| `rok::std::thread`  | ✓                   | partial       | ✗ (no threads)   |
| `rok::std::io`      | ✓                   | ✓             | ✗ (no stdin/out) |
| `rok::std::env`     | ✓                   | ✓             | ✗                |
| `rok::std::process` | ✓                   | ✗             | ✗                |
| `rok::std::path`    | ✓                   | ✓             | ✓                |

각 모듈 spec 의 가용성 절에서 플랫폼별 동작을 더 자세히 명시합니다.
