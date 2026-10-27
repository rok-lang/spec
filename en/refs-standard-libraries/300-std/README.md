# 3. Std tier

The standard library of the target-dependent area. It covers the file system, networking, time, OS threads, I/O, environment variables, processes and paths, and availability varies with the user's environment (operating system / platform).

## Location

The spec body lives under `spec/en/refs-standard-libraries/300-std/`.

## Dependencies

Upward, it depends on `rok` and `unsafe`. Downward, user code stands in the dependent position, but std is pulled in only by an explicit import.

## User import policy

The implicit `use rok::core::**;` does not include std items - `rok::std::*` requires a separate explicit import. Because availability differs by target, the design makes users express platform dependencies **explicitly**.

```rok
use rok::std::fs;
use rok::std::time::Instant;
use rok::std::thread::Builder;
```

## Module index

This tier consists of the following eleven modules.

- [3.1. `fs::` - file system (atomic ops, Dir capability, reflink copy)](301-fs.md)
- [3.2. `net::` - TCP / UDP / Unix + TLS 1.3](302-net.md)
- [3.3. `time::` - `Instant` / `SystemTime` / `Duration` / `BootTime`](303-time.md)
- [3.4. `thread::` - OS threads (1:1, separate from `task::`)](304-thread.md)
- [3.5. `io::` - `Reader` / `Writer` / stdin/out/err / `BufReader`](305-io.md)
- [3.6. `env::` - environment variables / args / current_dir / home_dir](306-env.md)
- [3.7. `process::` - `Command` / `Child` / pidfd / kill_on_drop](307-process.md)
- [3.8. `path::` - UTF-8 `Path` (Python-style) + `RawPath` escape hatch](308-path.md)
- [3.9. `rokon::` - the ROKON data format (`.rokon`)](309-rokon.md)
- [3.10. `crypto::` - the OS random source](310-crypto.md)
- `wit::` - the component model's async ABI that generated WIT bindings call ([1.8](../100-bare/108-wit.md)). Bindings-internal: public so the generated code can reach it, not meant to be called directly, and not specified here.

## Argument conventions

Two shorthands hold in every signature of this tier.

- A path parameter written `Path` stands for a type parameter `P: AsPath`, so it accepts both `str` and `Path` ([3.8.3a](308-path.md#383a-aspath---path-arguments)). Two path parameters are independent (`P: AsPath, Q: AsPath`).
- The data of an inherent write function (`write`, `write_all`, `send`, `send_to`, `send_with_fd` and their `_async` forms, the `fs` write functions) is `B: AsBytes`, so `str`, `List<u8>`, `Slice<u8>` and `[u8; N]` can be passed ([3.5.1](305-io.md#351-asbytes)). The `Writer` trait methods and `write_zerocopy` take a `List<u8>`.

A read buffer is a `List<u8>` whose length is set up in advance: `read(buf)` fills `buf[0..n]` and returns `n`, `0` at the end of the stream.

## Core design principles

### 1. Blocking API + `*_async` - no async traits

std I/O functions are sync and block the calling OS thread. To give up the worker while waiting inside a task, `await` the `*_async` method of the same operation ([3.2.0](302-net.md#320-common-rules)). With stackless coroutines an ordinary function cannot suspend a task, so a sync call never becomes async on its own.

```rok
// blocking - on an OS thread (thread::spawn) or outside a task
fn fetch(addr: str) throws (IoError) -> List<u8> {
    using conn = try net::tcp_connect(addr) {
        try conn.read_to_end()
    }
}

// async - suspends only the task
async fn fetch_async(addr: str) throws (IoError) -> List<u8> {
    using conn = try await net::tcp_connect_async(addr) {
        try await conn.read_to_end_async()
    }
}
```

There are **no** `AsyncRead` / `AsyncWrite` traits. The async versions are inherent methods and functions of the types. Sockets wait for I/O driver readiness ([3.2.0](302-net.md#320-common-rules)); regular files run the blocking call on another OS thread ([3.1.4a](301-fs.md#314a-asynchronous-file-io-_async)).

### 2. I/O backend - runtime detail

Waiting for readiness is **the runtime's responsibility** ([refs-lang 7.7](../../refs-lang/007-runtime/707-io-and-timer.md)). User code does not need to know which backend is running.

| Platform        | Backend                     |
|-----------------|-----------------------------|
| Linux           | epoll (edge-triggered)      |
| macOS / FreeBSD | kqueue (`EV_CLEAR`)         |
| Windows         | IOCP                        |
| WASI            | `wasi:io/poll` (`wasm32-wasi`), `poll_oneoff` (`wasm32-wasi-threads`) |

### 3. TLS 1.3 in std - HTTP out

TLS 1.3 is **security-critical**. If every application implemented TLS itself, security vulnerabilities would multiply, so it is integrated into std.

```rok
use rok::std::net::tls;

let conn = try net::tcp_connect("example.com:443");
let tls_conn = try tls::Client::new(conn, "example.com");
let req = b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n";
try tls_conn.write_all(req);
let resp = try tls_conn.read_to_end();
```

The stack is Rok's own, written on `rok::core::crypto` ([2.21](../200-core/221-crypto.md)); no OS or third-party TLS library is used ([3.2.5](302-net.md#325-tls-13---nettls)).

HTTP/1, HTTP/2 and HTTP/3 (QUIC), on the other hand, live **outside std**, split into a separate `rok/http` package. This is the lesson from Go's `net/http` being **frozen at 1.0**.

### 4. Capability-based fs - Dir handles

Follows the pre-opened directory handle model of WASI Preview 2. Every fs op goes **through a Dir**.

```rok
use rok::std::fs::{Dir, File};

let root: Dir = Dir::root();                  // capability - at process start
let data_dir: Dir = try root.open_dir("data"); // sub-cap

let f: File = try data_dir.open("config.toml");
let bytes = try f.read_to_end();
```

The model has three advantages. First, it avoids TOCTOU - `open` after `open_dir` is race-free. Second, sandboxing comes naturally - pass only a **narrow Dir** to a child task. Third, it is WASI-compatible - the same code works on native and WASI.

For convenience, `fs::open(absolute_path)` is also allowed. Internally it expands to `Dir::root().open(...)`.

### 5. UTF-8 Path (Python-style)

A decision to avoid the **ergonomic burden** of Rust's `Path` / `PathBuf` + `OsStr`.

```rok
use rok::std::path::Path;

let p: Path = Path::from("data/config.toml");
let parent: Path = p.parent()?;
let with_ext: Path = p.with_extension("bak");

// `/` operator (Python pathlib style)
let full = Path::from("/etc") / "nginx" / "nginx.conf";
```

`Path` is a **UTF-8 newtype**. Non-UTF-8 file names (which occasionally exist on Unix) are handled separately with `RawPath` (bytes). `Path` alone should cover 99% of code.

The detailed comparison and rationale are in [3.8 path](308-path.md).

### 6. All I/O `throws (IoError)`

Panics almost never occur (except OOM). File not found, permission denied and network failure are all thrown with a `throws (IoError)` signature and propagate by marking the call with `try`. There is no automatic conversion between error types; to turn one into another error type, wrap it in `catch` and rethrow.

```rok
fn load_config() throws (ConfigError) -> Config {
    let bytes = try fs::read("config.toml") catch e: IoError {
        throw ConfigError::Io(e);              // IoError → ConfigError (explicit conversion)
    };
    try parse(bytes)
}
```

## Target availability

Module availability per platform:

| Module              | linux/macos/windows | wasm32-wasi   | wasm32-wasi-threads | wasm32 (browser) |
|---------------------|---------------------|---------------|---------------------|------------------|
| `rok::std::fs`      | ✓                   | ✓ (cap-based) | ✓ (cap-based)       | ✗ (every call `Unsupported`) |
| `rok::std::net`     | ✓ (+ TLS 1.3)       | ✓ (`wasi:sockets`) | listeners passed in by the host | ✗ (use `fetch`) |
| `rok::std::time`    | ✓                   | ✓             | ✓                   | ✓ (Date.now)     |
| `rok::std::thread`  | ✓                   | one thread (`Builder::spawn` `Unsupported`) | ✓ (wasi-threads) | one thread (`Builder::spawn` `Unsupported`) |
| `rok::std::io`      | ✓                   | ✓             | ✓                   | console / stdin at EOF |
| `rok::std::env`     | ✓                   | ✓             | ✓                   | ✗                |
| `rok::std::process` | ✓                   | ✗             | ✗                   | ✗                |
| `rok::std::path`    | ✓                   | ✓             | ✓                   | ✓                |

✗ means the module builds but its operations fail at run time with `IoErrorKind::Unsupported`; it is not a compile-time absence. In the browser, stdout / stderr go to the host console and stdin is at EOF. The availability section of each module spec gives per-platform behavior in more detail.
