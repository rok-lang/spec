# 3.7. `process::`: 자식 프로세스

`rok::std::process` 모듈은 자식 프로세스를 띄우고 신호를 다룹니다. 기본 골격은 Rust의 `Command` builder 패턴을 따릅니다. 그 위에 현대적인 안전장치인 `pidfd`, `kill_on_drop`, `posix_spawn` 빠른 경로를 더해 더 견고하게 만들었습니다.

## 3.7.1. `Command` 빌더

먼저 자식 프로세스의 청사진인 `Command` struct입니다. 실행할 프로그램, 인자, 환경 변수, 표준 입출력 연결 방식 같은 설정을 차례로 쌓은 뒤 프로세스를 띄웁니다.

경로 매개변수는 tier 공통 [인자 규약](README.md#인자-규약)을 따릅니다(`Path`는 `P: AsPath`).

```rok
#[must_use]
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

    fn kill_on_drop(self, on: bool) -> Command;         // 띄운 Child 가 해제될 때 kill (Drop)
    // unix: process_group, CommandExt (3.11.5)

    // 실행
    fn spawn(self) throws (IoError) -> Child;
    fn output(self) throws (IoError) -> Output;          // 대기 + 수집: Inherit 인 stdout / stderr 는 pipe, Inherit 인 stdin 은 null
    fn status(self) throws (IoError) -> ExitStatus;      // wait, no output
    async fn output_async(self) throws (IoError) -> Output;
    async fn status_async(self) throws (IoError) -> ExitStatus;
}

#[non_exhaustive]
enum Stdio {
    Inherit,                   // 부모 fd 그대로 (default)
    Null,                      // /dev/null
    Piped,                     // pipe, Child 에서 읽기/쓰기 가능
    File(File),                // 이 파일, 이제부터 명령이 소유
}

struct Output {
    status: ExitStatus;
    stdout: List<u8>;
    stderr: List<u8>;
}                                        // + Clone, PartialEq, Eq, Debug
```

빌더 메서드는 바뀐 명령을 돌려주고, 호출한 쪽 명령은 그대로 둡니다. 그래서 `Command`의 복사본들은 서로 독립입니다. `let a = base.arg("x")` 뒤에도 `base`에는 `x`가 없습니다. 결과를 버린 빌더 호출(`base.arg("x");`)은 아무것도 바꾸지 않고 경고를 냅니다(`#[must_use]`).

`Stdio`는 네 가지입니다. 부모의 fd를 그대로 물려주는 `Inherit`, 출력을 모두 버리는 `Null`, 양방향 통신용 `Piped`, 열린 파일을 연결하는 `File`입니다.

`Stdio::File`로 옮긴 `File`은 프로그램이 아니라 명령의 것입니다. 다른 `File` 이동과 같이 이 이동은 값을 소비하고([2.13.4](../200-core/213-marker.md)), 프로그램은 그 파일을 닫지 않습니다. 명령을 실행하면(`spawn`, `output`, `status`와 그 `_async` 쌍둥이) 자식을 띄운 직후 부모 쪽 사본을 닫고, 띄우기에 실패해도 닫습니다. 자식은 자기 사본을 따로 가집니다. 이때 닫기가 실패하면 그 실행의 `IoError`가 됩니다. 이미 띄운 자식은 먼저 죽이고 회수하므로 잃어버리는 `Child`는 없습니다. 앞선 실행이 파일을 닫은 명령의 복사본은 실행하지 않고 `InvalidInput`으로 실패합니다. 실행마다 따로 파일을 주세요(`File::try_clone`). 한 번도 실행하지 않은 명령의 파일은 수집기가 해제할 때까지 열려 있습니다.

## 3.7.2. `Child`

`spawn`이 성공하면 실행 중인 프로세스의 핸들인 `Child`를 돌려줍니다. 이 핸들로 표준 입출력 스트림에 접근하거나, 종료를 기다리거나, 신호를 보낼 수 있습니다.

```rok
struct Child { /* opaque: pid, 아직 꺼내지 않은 pipe 스트림 */ }

impl Child {
    fn id(self) -> u32;                                  // pid
    fn take_stdin(ref self) -> ChildStdin | none;        // none: Piped 가 아니거나 이미 꺼냄
    fn take_stdout(ref self) -> ChildStdout | none;
    fn take_stderr(ref self) -> ChildStderr | none;
    fn kill(self) throws (IoError);
    fn wait(ref self) throws (IoError) -> ExitStatus;
    async fn wait_async(ref self) throws (IoError) -> ExitStatus;
    fn try_wait(self) throws (IoError) -> ExitStatus | none;

    // Linux: pidfd, ChildExt (3.11.5)

    fn send_signal(self, sig: Signal) throws (IoError);  // Windows: Kill 만
}

impl ChildStdin {                        // 자식 stdin pipe 의 쓰기 끝, impl io::Write + io::Close, io::AsyncWrite + io::AsyncClose
    fn close(self) throws (IoError);     // 자식은 입력 끝을 읽음
}
impl ChildStdout {                       // 자식 stdout pipe 의 읽기 끝, impl io::Read + io::Close, io::AsyncRead + io::AsyncClose
    fn close(self) throws (IoError);
}
impl ChildStderr {                       // 자식 stderr pipe 의 읽기 끝, impl io::Read + io::Close, io::AsyncRead + io::AsyncClose
    fn close(self) throws (IoError);
}
// + 세 pipe 모두 AsRawFd (unix), AsRawHandle (Windows): 3.11

struct ExitStatus { /* opaque */ }

impl ExitStatus {
    fn success(self) -> bool;
    fn code(self) -> i32 | none;        // none = signal terminated (unix)
    // unix: signal, ExitStatusExt (3.11.5)
}
impl Display for ExitStatus { ... }      // "exit status: 1" / "signal: 9"
impl Debug for ExitStatus { ... }        // "ExitStatus { code: 1 }" / "ExitStatus { signal: 9 }"
// ExitStatus: + Clone, Copy, PartialEq, Eq, Hash
```

종료 결과는 불투명 타입 `ExitStatus`에 담깁니다. `code()`가 `none`이면 Unix에서 신호로 종료되었다는 뜻입니다. 그 신호는 `signal()`(`ExitStatusExt`, Unix에서 prelude에 있음)이 `Signal`로 알려 줍니다.

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

let stdin = child.take_stdin() ?? panic("stdin not piped");
try stdin.write_all(input_bytes);
try stdin.close();                                       // 입력 끝: grep 이 끝날 수 있음
let stdout = child.take_stdout() ?? panic("stdout not piped");
let result = try stdout.read_to_end();
try stdout.close();
try child.wait();

// 파일로 출력: 명령이 파일을 가져가고 우리 쪽 사본을 닫음
let log = try File::create("ls.txt");
try Command::new("ls").stdout(Stdio::File(log)).status();

// kill_on_drop 으로 process leak 방지
using child = try Command::new("long_running")
        .kill_on_drop(true)
        .spawn() {
    // 에러 전파 / early return 시에도 child kill 보장
}    // ← using 해제 시 Drop → kill
```

마지막 예처럼 `kill_on_drop`을 켠 `Child`를 `using`으로 묶으면 에러 전파나 조기 반환 같은 흐름에서도 자식이 함께 정리됩니다. 소유자가 `Child`를 해제하는 곳, 곧 `using` 블록의 끝이나 마지막 소유자가 스코프를 떠나는 곳에서 `Drop`이 자식을 죽입니다([2.13.4](../200-core/213-marker.md)). `ChildStdin` / `ChildStdout` / `ChildStderr`는 파이프의 파일 핸들입니다. `File`과 같은 방식으로 읽고 씁니다. `Child`는 `Piped` 스트림을 들고 있다가 `take_stdin` / `take_stdout` / `take_stderr`가 하나를 넘겨주면 그 자리에 `none`을 남깁니다. 그때부터 파이프는 꺼낸 쪽의 것이고, `File`처럼 직접 닫아야 합니다. 꺼내지 않은 파이프는 `Child`가 해제되는 곳에서 닫힙니다. `wait`는 `Child`가 아직 들고 있는 stdin을 먼저 닫으므로 입력 끝까지 읽는 자식도 끝날 수 있습니다. 꺼낸 stdin은 꺼낸 쪽이 `wait` 전에 닫습니다.

`Child`의 복사본들은 같은 프로세스를 공유합니다. `wait` / `try_wait`는 다른 복사본으로 진행 중인 `send_signal` / `kill` 호출이 돌아온 뒤에야 자식을 회수합니다. 그래서 시그널이 재사용된 pid나 pidfd에 닿는 일은 없습니다. 자식이 끝나 회수되는 중에 보낸 `send_signal`은 아무 일도 하지 않습니다. 동시에 기다리는 복사본들은 모두 같은 `ExitStatus`를 받습니다.

## 3.7.4. 최신 추가 기능: pidfd / posix_spawn / CLOEXEC

표준 라이브러리는 builder 패턴 위에 몇 가지 현대적인 보강을 더합니다. 각 기능의 의미와 효과는 다음과 같습니다.

| 기능                    | 의미                          | 효과                    |
|-------------------------|-------------------------------|-------------------------|
| `posix_spawn` 빠른 경로 | fork+exec 대신 syscall        | 큰 프로세스에서도 빨리 띄움 |
| `pidfd` (Linux 5.3+)    | pid 대신 fd로 추적           | 경쟁 없는 신호          |
| `CLOEXEC` 기본값        | 모든 fd가 자동 close-on-exec | fd 누수 방지            |
| `kill_on_drop`          | 소유자가 해제할 때 kill       | 프로세스 누수 방지      |

이 결정은 모두 fd 누수, 프로세스 누수, 신호 경쟁을 줄이는 쪽을 향합니다.

`Command::env`를 써도 프로그램은 자식이 아니라 부모의 `PATH`에서 찾습니다.

## 3.7.5. 신호 처리(고급)

한 단계 더 내려가면 신호 처리 함수가 있습니다.

```rok
#[non_exhaustive]
enum Signal {
    Hangup, Interrupt, Quit, IllegalInstruction, Trap, Abort, Bus, FloatingPoint, Kill,
    Segmentation, Pipe, Alarm, Terminate, User1, User2, Child, Continue, Stop,
    TerminalStop, TerminalInput, TerminalOutput, Urgent, CpuLimit, FileSizeLimit,
    VirtualAlarm, Profile, WindowChange, Io, BadSyscall,
}                                        // SIGHUP .. SIGSYS

impl Signal {
    fn number(self) -> i32;              // OS 의 번호 (SIGHUP = 1, ..; Linux 의 것, 아니면 BSD 의 것)
}                                        // + Clone, Copy, PartialEq, Eq, Hash, Debug

fn on_signal(sig: Signal, handler: fn() + Send) throws (SignalError);
fn send_signal(pid: u32, sig: Signal) throws (IoError);
fn raise(sig: Signal) throws (IoError);  // Windows: Interrupt, Hangup 만
fn id() -> u32;                          // 이 프로세스의 id. wasm32 에는 없으므로 panic
fn exit(code: i32) -> !;                 // 런타임 rok_exit 로 즉시 종료; finally 와 Drop 은 돌지 않음 (refs-lang 4.16)
fn abort() -> !;                         // 비정상 종료 (SIGABRT); 아무것도 정리하지 않음

struct SignalError { .. }                // 비공개 필드 (0.5.10)
impl SignalError {
    fn signal(self) -> Signal;           // 설치하려던 신호
    fn os_code(self) -> i32 | none;      // errno (Windows: 에러 코드); 디스패처 스레드를 띄우지 못했으면 none
    fn message(self) -> str;
}
// + Display, Debug, Error, PartialEq, Eq, Clone
```

`on_signal`의 핸들러는 신호 문맥(signal handler) 안에서 돌지 않습니다. 런타임이 신호를 받아 전용 디스패처 스레드 `rok-signal`로 넘깁니다. 핸들러는 그 스레드에서 일반 함수처럼 실행됩니다. 그래서 핸들러는 `Send`여야 하고, 할당하거나 락을 잡아도 됩니다. 같은 신호에 `on_signal`을 다시 호출하면 핸들러가 바뀝니다. `Signal`은 어느 OS에서나 신호를 이름으로 가리킵니다. `process_group(pgid)`(`CommandExt`, unix)는 자식을 그 프로세스 그룹에 넣습니다(`0`이면 자식의 pid로 새 그룹을 만듭니다).

Windows에서는 콘솔 이벤트를 신호에 대응합니다. Ctrl-C → `Interrupt`(2), 콘솔 닫기 / 로그오프 / 종료 → `Hangup`(1)입니다. Ctrl-Break에는 `Signal`이 없고, `os::windows::process::on_ctrl_break`가 받습니다([3.11.7](311-os.md#3117-oswindowsprocess-ctrl-break)). `send_signal`은 `Signal::Kill`만 `TerminateProcess`로 보내고, `raise`는 `Interrupt`와 `Hangup`만 올립니다. 다른 `Signal`은 `IoError`(`Unsupported`)이고, `Interrupt`나 `Hangup`이 아닌 신호에 대한 `on_signal`은 `SignalError`입니다.

대부분의 사용자는 이 부분을 직접 다룰 일이 없습니다. 데몬, 서버, 대화형 CLI처럼 신호를 직접 처리해야 하는 곳에서만 필요합니다.

## 3.7.6. 플랫폼 가용성

플랫폼별 지원 범위는 다음과 같습니다.

| 항목             | linux      | macos | windows         | wasi |
|------------------|------------|-------|-----------------|------|
| `Command::spawn` | ✓          | ✓     | ✓               | ✗    |
| `posix_spawn`    | ✓          | ✓     | (CreateProcess) | ✗    |
| `pidfd`          | linux 5.3+ (`os::linux`) | ✗ | ✗         | ✗    |
| `kill_on_drop`   | ✓          | ✓     | ✓               | ✗    |
| 신호 핸들러      | ✓          | ✓     | (제한적)        | ✗    |
| `process::id()`  | ✓          | ✓     | ✓               | panic (wasm32) |

wasm32 타깃에서 ✗로 표시한 항목은 실행 중에 `IoErrorKind::Unsupported`로 실패합니다.
