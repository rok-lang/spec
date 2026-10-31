# 3.7. `process::`: 자식 프로세스

`rok::std::process` 모듈은 자식 프로세스를 띄우고 신호를 다룹니다. 기본 골격은 Rust의 `Command` builder 패턴을 따릅니다. 그 위에 현대적인 안전장치인 `pidfd`, `kill_on_drop`, `posix_spawn` 빠른 경로를 더해 더 견고하게 만들었습니다.

## 3.7.1. `Command` 빌더

먼저 자식 프로세스의 청사진인 `Command` struct입니다. 실행할 프로그램, 인자, 환경 변수, 표준 입출력 연결 방식 같은 설정을 차례로 쌓은 뒤 프로세스를 띄웁니다.

경로 매개변수는 tier 공통 [인자 규약](README.md#인자-규약)을 따릅니다(`Path`는 `P: AsPath`).

```rok
struct Command { /* opaque: 프로그램, 인자, 환경, stream, cwd, 옵션 */ }

impl Command {
    fn new(program: Path) -> Command;

    fn arg(self, arg: str) -> Command;
    fn args<I: IntoIterator<Item = str>>(self, args: I) -> Command;
    fn env(self, key: str, value: str) -> Command;
    fn envs<I: IntoIterator<Item = (str, str)>>(self, envs: I) -> Command;
    fn env_clear(self) -> Command;
    fn current_dir(self, dir: Path) -> Command;

    fn stdin(self, cfg: Stdio) -> Command;
    fn stdout(self, cfg: Stdio) -> Command;
    fn stderr(self, cfg: Stdio) -> Command;

    fn kill_on_drop(self, on: bool) -> Command;         // using 으로 묶은 Child 가 해제될 때 kill
    fn process_group(self, pgid: i32) -> Command;       // unix

    // 실행
    fn spawn(self) throws (IoError) -> Child;
    fn output(self) throws (IoError) -> Output;          // 대기 + 수집: Inherit 인 stdout / stderr 는 pipe, Inherit 인 stdin 은 null
    fn status(self) throws (IoError) -> ExitStatus;      // wait, no output
}

enum Stdio {
    Inherit,                   // 부모 fd 그대로 (default)
    Null,                      // /dev/null
    Piped,                     // pipe, Child 에서 읽기/쓰기 가능
    File(File),                // 명시 fd
}

struct Output {
    status: ExitStatus;
    stdout: List<u8>;
    stderr: List<u8>;
}
```

`Stdio`는 네 가지입니다. 부모의 fd를 그대로 물려주는 `Inherit`, 출력을 모두 버리는 `Null`, 양방향 통신용 `Piped`, 임의의 fd를 명시적으로 연결하는 `File`입니다.

## 3.7.2. `Child`

`spawn`이 성공하면 실행 중인 프로세스의 핸들인 `Child`를 돌려줍니다. 이 핸들로 표준 입출력 스트림에 접근하거나, 종료를 기다리거나, 신호를 보낼 수 있습니다.

```rok
struct Child {
    stdin: ChildStdin | none;
    stdout: ChildStdout | none;
    stderr: ChildStderr | none;
}

impl Child {
    fn id(self) -> u32;                                  // pid
    fn kill(self) throws (IoError);
    fn wait(self) throws (IoError) -> ExitStatus;
    fn try_wait(self) throws (IoError) -> ExitStatus | none;

    // pidfd (Linux only): race-free signal
    #[cfg(target_os = "linux")]
    fn pidfd(self) throws (IoError) -> RawFd;

    fn send_signal(self, sig: Signal) throws (IoError);  // Windows: Kill 만
    #[deprecated(since = "0.1.6", note = "use `Child::send_signal(Signal::..)`")]
    fn signal(self, sig: i32) throws (IoError);          // unix
}

impl ChildStdin {                        // 자식 stdin pipe 의 쓰기 끝, impl Writer
    fn fd(self) -> RawFd;
    fn close(self) throws (IoError);     // 자식은 입력 끝을 읽음
}
impl ChildStdout {                       // 자식 stdout pipe 의 읽기 끝, impl Reader
    fn fd(self) -> RawFd;
    fn close(self) throws (IoError);
}
impl ChildStderr {                       // 자식 stderr pipe 의 읽기 끝, impl Reader
    fn fd(self) -> RawFd;
    fn close(self) throws (IoError);
}

struct ExitStatus { /* opaque */ }

impl ExitStatus {
    fn success(self) -> bool;
    fn code(self) -> i32 | none;        // none = signal terminated (unix)
    fn signal(self) -> i32 | none;       // unix
}
impl Display for ExitStatus { ... }      // "exit status: 1" / "signal: 9"
impl Debug for ExitStatus { ... }        // "ExitStatus { code: 1 }" / "ExitStatus { signal: 9 }"
```

종료 결과는 불투명 타입 `ExitStatus`에 담깁니다. `code()`가 `none`이면 Unix에서 신호로 종료되었다는 뜻입니다. 신호 번호는 `signal()`로 따로 꺼낼 수 있습니다.

## 3.7.3. 사용 예

```rok
use rok::std::process::{Command, Stdio};

// 단순 실행
let status = try Command::new("ls").arg("-la").status();
if !status.success() { panic("ls failed") };

// 출력 수집
let output = try Command::new("git")
    .args(["log", "--oneline", "-10"])
    .output();
let log = try str::from_utf8(output.stdout);

// pipe 통한 입출력
let child = try Command::new("grep")
    .arg("error")
    .stdin(Stdio::Piped)
    .stdout(Stdio::Piped)
    .spawn();

let stdin = child.stdin ?? panic("stdin not piped");     // 필드는 ChildStdin | none
let stdout = child.stdout ?? panic("stdout not piped");
try stdin.write_all(input_bytes);
let result = try stdout.read_to_end();
try child.wait();

// kill_on_drop 으로 process leak 방지
using child = try Command::new("long_running")
        .kill_on_drop(true)
        .spawn() {
    // 에러 전파 / early return 시에도 child kill 보장
}    // ← using 해제 시 Drop → kill
```

마지막 예처럼 `kill_on_drop`을 켠 `Child`를 `using`으로 묶으면 에러 전파나 조기 반환 같은 흐름에서도 자식이 함께 정리됩니다. `using` 블록을 빠져나갈 때 `Drop`이 자식을 죽입니다. `using` 밖의 `Child`는 자동으로 죽지 않습니다([2.13.4](../200-core/213-marker.md#2134-drop-using-해제-시-정리)). `ChildStdin` / `ChildStdout` / `ChildStderr`는 파이프의 파일 핸들입니다. `File`과 같은 방식으로 읽고 씁니다.

`Child`의 복사본들은 같은 프로세스를 공유합니다. `wait` / `try_wait`는 다른 복사본으로 진행 중인 `signal` / `kill` 호출이 돌아온 뒤에야 자식을 회수합니다. 그래서 시그널이 재사용된 pid나 pidfd에 닿는 일은 없습니다. 자식이 끝나 회수되는 중에 보낸 `signal`은 아무 일도 하지 않습니다. 동시에 기다리는 복사본들은 모두 같은 `ExitStatus`를 받습니다.

## 3.7.4. 최신 추가 기능: pidfd / posix_spawn / CLOEXEC

표준 라이브러리는 builder 패턴 위에 몇 가지 현대적인 보강을 더합니다. 각 기능의 의미와 효과는 다음과 같습니다.

| 기능                    | 의미                          | 효과                    |
|-------------------------|-------------------------------|-------------------------|
| `posix_spawn` 빠른 경로 | fork+exec 대신 syscall        | 큰 프로세스에서도 빨리 띄움 |
| `pidfd` (Linux 5.3+)    | pid 대신 fd로 추적           | 경쟁 없는 신호          |
| `CLOEXEC` 기본값        | 모든 fd가 자동 close-on-exec | fd 누수 방지            |
| `kill_on_drop`          | `using` 해제 시 kill          | 프로세스 누수 방지      |

이 결정은 모두 fd 누수, 프로세스 누수, 신호 경쟁을 줄이는 쪽을 향합니다.

`Command::env`를 써도 프로그램은 자식이 아니라 부모의 `PATH`에서 찾습니다.

## 3.7.5. 신호 처리(고급)

한 단계 더 내려가면 신호 처리 함수가 있습니다.

```rok
#[non_exhaustive]
enum Signal { Hangup, Interrupt, Quit, Kill, Pipe, Alarm, Terminate, User1, User2, Child, Continue, Stop }

impl Signal {
    fn number(self) -> i32;              // OS 의 번호 (SIGHUP = 1, ..)
}                                        // + PartialEq, Debug

fn on_signal(sig: Signal, handler: fn() + Send) throws (SignalError);
fn send_signal(pid: u32, sig: Signal) throws (IoError);
#[deprecated(since = "0.1.6", note = "use `process::on_signal(Signal::.., handler)`")]
fn signal(sig: i32, handler: fn() + Send) throws (SignalError);
#[deprecated(since = "0.1.6", note = "use `process::send_signal(pid, Signal::..)`")]
fn kill(pid: u32, sig: i32) throws (IoError);
fn raise(sig: i32) throws (IoError);
fn id() -> u32;                          // 이 프로세스의 id
fn exit(code: i32) -> !;                 // 런타임 rok_exit 로 즉시 종료; finally 와 Drop 은 돌지 않음 (refs-lang 4.16)
fn abort() -> !;                         // 비정상 종료 (SIGABRT); 아무것도 정리하지 않음

struct SignalError {
    sig: i32;          // 설치하려던 신호
    code: i32;         // errno
}
```

`on_signal`의 핸들러는 신호 문맥(signal handler) 안에서 돌지 않습니다. 런타임이 신호를 받아 전용 디스패처 스레드 `rok-signal`로 넘깁니다. 핸들러는 그 스레드에서 일반 함수처럼 실행됩니다. 그래서 핸들러는 `Send`여야 하고, 할당하거나 락을 잡아도 됩니다. 같은 신호에 `on_signal`을 다시 호출하면 핸들러가 바뀝니다. `Signal`은 어느 OS에서나 신호를 이름으로 가리킵니다. `i32`를 받는 `signal`, `kill`, `Child::signal`은 지금도 동작하지만 경고(`deprecated`)를 내고, 0.2.0에서 사라집니다. `Command::process_group(pgid)`는 자식을 그 프로세스 그룹에 넣습니다(`0`이면 자식의 pid로 새 그룹을 만듭니다).

Windows에서는 콘솔 이벤트를 신호에 대응합니다. Ctrl-C → `Interrupt`(2), 콘솔 닫기 / 로그오프 / 종료 → `Hangup`(1)입니다. 폐기 예정인 `signal`은 Ctrl-Break(21)도 받습니다. `send_signal`은 `Signal::Kill`만 `TerminateProcess`로 보냅니다. 다른 `Signal`은 `IoError`(`Unsupported`)이고, `Interrupt`나 `Hangup`이 아닌 신호에 대한 `on_signal`은 `SignalError`입니다.

대부분의 사용자는 이 부분을 직접 다룰 일이 없습니다. 데몬, 서버, 대화형 CLI처럼 신호를 직접 처리해야 하는 곳에서만 필요합니다.

## 3.7.6. 플랫폼 가용성

플랫폼별 지원 범위는 다음과 같습니다.

| 항목             | linux      | macos | windows         | wasi |
|------------------|------------|-------|-----------------|------|
| `Command::spawn` | ✓          | ✓     | ✓               | ✗    |
| `posix_spawn`    | ✓          | ✓     | (CreateProcess) | ✗    |
| `pidfd`          | linux 5.3+ | ✗     | ✗               | ✗    |
| `kill_on_drop`   | ✓          | ✓     | ✓               | ✗    |
| 신호 핸들러      | ✓          | ✓     | (제한적)        | ✗    |
| `process::id()`  | ✓          | ✓     | ✓               | `0` (wasm32) |

wasm32 타깃에서 ✗로 표시한 항목은 실행 중에 `IoErrorKind::Unsupported`로 실패합니다.
