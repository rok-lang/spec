# 3.7. `process::` — 자식 프로세스

`rok::std::process` 모듈은 자식 프로세스를 spawn 하고 그 신호를 다루는 일을 담당합니다. 기본 골격은 Rust 의 `Command` builder 패턴을 따르되, 그 위에 모던한 안전 장치—`pidfd`, `kill_on_drop`, 그리고 `posix_spawn` fast path—를 얹어 한층 견고하게 만들었다.

## 3.7.1. `Command` builder

가장 먼저 만나게 되는 것은 자식 프로세스의 청사진을 그리는 `Command` 구조체입니다. 실행할 프로그램, 인자, 환경 변수, 표준 입출력의 연결 방식 같은 모든 설정을 단계적으로 쌓아 올린 뒤에야 비로소 spawn 으로 옮겨 가게 됩니다.

```rok
struct Command {
    program: Path;
    args: List<str>;
    envs: Dict<str, str>;
    env_clear: bool;
    stdin: Stdio;
    stdout: Stdio;
    stderr: Stdio;
    cwd: Path | none;
    kill_on_drop: bool;       // using 으로 묶은 Child 가 해제될 때 kill
    process_group: i32 | none;
    // ...
}

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

    fn kill_on_drop(self, on: bool) -> Command;
    fn process_group(self, pgid: i32) -> Command;       // unix

    // 실행
    fn spawn(self) throws (IoError) -> Child;
    fn output(self) throws (IoError) -> Output;          // wait + collect
    fn status(self) throws (IoError) -> ExitStatus;      // wait, no output
}

enum Stdio {
    Inherit,                   // 부모 fd 그대로 (default)
    Null,                      // /dev/null
    Piped,                     // pipe — Child 에서 읽기/쓰기 가능
    File(File),                // 명시 fd
}

struct Output {
    status: ExitStatus;
    stdout: List<u8>;
    stderr: List<u8>;
}
```

여기서 눈여겨볼 점은 `Stdio` 가 네 가지 갈래로 나뉜다는 것입니다. 부모의 fd 를 그대로 물려주는 `Inherit`, 출력을 통째로 버리는 `Null`, 양방향 통신을 위한 `Piped`, 그리고 임의의 fd 를 명시적으로 꽂아 넣는 `File` 이 그것입니다.

## 3.7.2. `Child`

`spawn` 이 성공하면 이제 살아 움직이는 프로세스의 손잡이로서 `Child` 가 돌아옵니다. 이 핸들을 통해 표준 입출력 스트림에 접근하거나, 종료를 기다리거나, 신호를 보낼 수 있습니다.

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

    // pidfd (Linux only) — race-free signal
    #[cfg(target_os = "linux")]
    fn pidfd(self) throws (IoError) -> RawFd;

    fn signal(self, sig: i32) throws (IoError);          // unix
}

impl ChildStdin { ... }                  // 파일 핸들 — impl Writer
impl ChildStdout { ... }                 // 파일 핸들 — impl Reader
impl ChildStderr { ... }                 // 파일 핸들 — impl Reader

struct ExitStatus { /* opaque */ }

impl ExitStatus {
    fn success(self) -> bool;
    fn code(self) -> i32 | none;        // none = signal terminated (unix)
    fn signal(self) -> i32 | none;       // unix
}
```

종료 결과는 `ExitStatus` 라는 불투명 타입에 담긴다. `code()` 가 `none` 이라면 unix 환경에서 신호에 의해 종료되었다는 뜻이고, 그 신호 번호는 `signal()` 로 따로 꺼내 볼 수 있습니다.

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

// kill_on_drop — process leak 방지
using child = try Command::new("long_running")
        .kill_on_drop(true)
        .spawn() {
    // 에러 전파 / early return 시에도 child kill 보장
}    // ← using 해제 시 Drop → kill
```

마지막 예시가 보여주듯, `kill_on_drop` 을 켠 `Child` 를 `using` 으로 묶으면 에러 전파나 조기 return 같은 흐름에서도 자식이 함께 정리됩니다. `using` 블록을 빠져나갈 때 `Drop` 이 kill 을 겁니다. `using` 밖의 `Child` 는 자동으로 kill 되지 않습니다 ([2.13.4](../200-core/213-marker.md#2134-drop--using-해제-시-정리)). `ChildStdin` / `ChildStdout` / `ChildStderr` 는 파이프의 파일 핸들이며 `File` 과 같은 방식으로 읽고 씁니다.

## 3.7.4. 모던 추가 — pidfd / posix_spawn / CLOEXEC

이러한 builder 패턴 위에 표준 라이브러리는 몇 가지 현대적 보강을 더합니다. 각각이 어떤 의미를 갖고 어떤 효과로 이어지는지 다음 표에 정리해 두었다.

| 기능                    | 의미                          | 효과                    |
|-------------------------|-------------------------------|-------------------------|
| `posix_spawn` fast path | fork+exec 대신 syscall        | huge process spawn 빠름 |
| `pidfd` (Linux 5.3+)    | pid 대신 fd 로 추적           | race-free signal        |
| `CLOEXEC` default       | 모든 fd 가 자동 close-on-exec | fd leak 방지            |
| `kill_on_drop`          | `using` 해제 시 kill          | process leak 방지       |

이러한 결정은 fd 누수와 process 누수, 그리고 신호 race 를 줄이는 방향으로 일관되게 정렬됩니다.

## 3.7.5. 신호 처리 (advanced)

조금 더 깊은 영역으로 내려가면 신호 처리 함수들이 자리 잡고 있습니다.

```rok
fn signal(sig: i32, handler: fn() + Send) throws (SignalError);
fn kill(pid: u32, sig: i32) throws (IoError);
fn raise(sig: i32) throws (IoError);

struct SignalError {
    sig: i32;          // 설치하려던 신호
    code: i32;         // errno
}
```

`signal` 의 handler 는 신호 문맥 (signal handler) 안에서 돌지 않습니다. 런타임이 신호를 받아 전용 dispatcher 스레드 `rok-signal` 로 넘기고, handler 는 그 스레드에서 보통 함수처럼 실행됩니다. 그래서 handler 는 `Send` 여야 하며 할당이나 lock 을 써도 됩니다. 같은 신호에 다시 `signal` 을 부르면 handler 가 바뀝니다. `Command::process_group(pgid)` 는 자식을 그 프로세스 그룹에 넣습니다 (`0` 이면 자식의 pid 로 새 그룹).

다만 대부분의 사용자는 이 부분을 직접 만질 일이 없습니다. daemon 이나 서버, interactive CLI 처럼 신호를 정면으로 마주해야 하는 자리에서나 비로소 필요해지는 도구들이라 보면 됩니다.

## 3.7.6. 플랫폼 가용성

플랫폼별 지원 범위는 다음과 같이 갈립니다.

| 항목             | linux      | macos | windows         | wasi |
|------------------|------------|-------|-----------------|------|
| `Command::spawn` | ✓          | ✓     | ✓               | ✗    |
| `posix_spawn`    | ✓          | ✓     | (CreateProcess) | ✗    |
| `pidfd`          | linux 5.3+ | ✗     | ✗               | ✗    |
| `kill_on_drop`   | ✓          | ✓     | ✓               | ✗    |
| signal handler   | ✓          | ✓     | (limited)       | ✗    |
