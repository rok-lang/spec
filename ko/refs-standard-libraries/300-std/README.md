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

### 1. Sync API over M:N scheduler — async 트레잇 없음

rok 의 가장 큰 구조적 우위가 여기 있습니다. [task](../200-core/218-task.md) 의 M:N 스케줄러가 blocking 호출을 자동으로 suspend 하므로, 사용자 코드는 sync 처럼 작성하면 됩니다.

```rok
// rok 식 — sync 처럼 보이지만 task 안에서 blocking 안 함
fn fetch(url: str) -> Result<str, NetError> {
    let conn = net::tcp_connect(url)?;       // suspend (parking)
    let bytes = conn.read_all()?;             // suspend
    Ok(parse(bytes))
}

task::spawn { fetch("...") };
task::spawn { fetch("...") };
// ↑ 동시에 실행 — 한 task suspend 시 worker 가 다른 task 실행
```

그 결과 `AsyncRead` / `AsyncWrite` 트레잇은 **존재하지 않으며**, `Future<T>` 를 직접 다룰 일도 거의 없습니다. 무엇보다 function color 문제가 **완화** 됩니다 — sync API 가 곧 async 이기 때문입니다.

이 정책은 [Tokio io_uring 글](https://tokio.rs/blog/2021-07-tokio-uring) 이 지적한 completion vs readiness model 충돌도 회피합니다. sync API 라 두 모델 모두를 자유롭게 선택할 수 있는 까닭입니다.

### 2. io_uring / IOCP / kqueue / epoll — runtime detail

I/O backend 는 **runtime 의 책임** 입니다. 사용자 코드는 어떤 backend 가 돌아가는지 알 필요가 없습니다.

| 플랫폼          | backend                             |
|-----------------|-------------------------------------|
| Linux 5.15+     | io_uring (default) + epoll fallback |
| Linux 5.10-5.14 | epoll                               |
| macOS / FreeBSD | kqueue                              |
| Windows         | IOCP + wepoll                       |
| WASI            | poll_oneoff                         |

io_uring backend 는 `IORING_SETUP_DEFER_TASKRUN` (Linux 5.19+) 을 활용해 epoll 대비 2-5x throughput 이득을 얻습니다. 보안 CVE 측면에서는 opt-in 환경 변수와 epoll fallback 을 함께 제공해, Google/Android 처럼 io_uring 을 비활성화하는 환경에서도 동작하도록 설계되어 있습니다.

### 3. TLS 1.3 in std — HTTP out

TLS 1.3 는 **보안 critical** 영역입니다. 모든 앱이 TLS 를 새로 구현하면 보안 취약점이 폭발하므로 std 에 통합하는 쪽을 택했습니다.

```rok
use rok::std::net::tls;

let conn = net::tcp_connect("example.com:443")?;
let tls_conn = tls::Client::new(conn, "example.com")?;
let req = b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n";
tls_conn.write_all(req)?;
let resp = tls_conn.read_all()?;
```

backend 로는 rustls 또는 aws-lc-rs 식을 상정합니다 (memory-safe, no OpenSSL FFI).

반면 HTTP/1, HTTP/2, HTTP/3 (QUIC) 은 **std 외부** 에 둡니다 — 별도 `rok/http` 패키지로 분리합니다. Go `net/http` 의 **1.0 frozen** 사례에서 얻은 학습입니다.

### 4. Capability-based fs — Dir 핸들

WASI Preview 2 의 pre-opened directory handle 모델을 따릅니다. 모든 fs op 가 **Dir 을 통해** 일어납니다.

```rok
use rok::std::fs::{Dir, File};

let root: Dir = Dir::root();                  // capability — process 시작 시
let data_dir: Dir = root.open_dir("data")?;   // sub-cap

let f: File = data_dir.open("config.toml")?;
let bytes = f.read_to_end()?;
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

### 6. 모든 I/O 가 `Result<T, IoError>`

panic 은 거의 발생하지 않습니다 (OOM 외). file not found, permission denied, network failure 어느 쪽이든 모두 `Result` 로 반환되며, `?` 연산자로 자연스럽게 체이닝할 수 있습니다.

```rok
fn load_config() -> Result<Config, ConfigError> {
    let bytes = fs::read("config.toml")?;       // IoError → ConfigError (From 변환)
    let cfg = parse(bytes)?;
    Ok(cfg)
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
