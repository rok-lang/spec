# 3.7. `process::` - child processes

The `rok::std::process` module spawns child processes and handles signals. Its basic skeleton follows Rust's `Command` builder pattern, hardened with modern safeguards on top: `pidfd`, `kill_on_drop` and a `posix_spawn` fast path.

## 3.7.1. `Command` builder

The first thing you meet is the `Command` struct, the blueprint of a child process. All settings (the program to run, arguments, environment variables, how standard I/O is connected) are built up step by step before spawning.

Path parameters follow the tier-wide [argument conventions](README.md#argument-conventions) (`Path` is `P: AsPath`).

```rok
#[must_use]
struct Command { /* opaque: program, args, environment, streams, cwd, options */ }

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

    fn kill_on_drop(self, on: bool) -> Command;         // kill when the spawned Child is released (Drop)
    // unix: process_group, CommandExt (3.11.5)

    // execution
    fn spawn(self) throws (IoError) -> Child;
    fn output(self) throws (IoError) -> Output;          // wait + collect: stdout / stderr left at Inherit are piped; an Inherit stdin is null
    fn status(self) throws (IoError) -> ExitStatus;      // wait, no output
    async fn output_async(self) throws (IoError) -> Output;
    async fn status_async(self) throws (IoError) -> ExitStatus;
}

#[non_exhaustive]
enum Stdio {
    Inherit,                   // parent's fd as is (default)
    Null,                      // /dev/null
    Piped,                     // pipe, readable/writable from Child
    File(File),                // this file; the command owns it from here
}

struct Output {
    status: ExitStatus;
    stdout: List<u8>;
    stderr: List<u8>;
}                                        // + Clone, PartialEq, Eq, Debug
```

A builder method returns the changed command and leaves the one it was called on as it was, so copies of a `Command` are independent: after `let a = base.arg("x")`, `base` has no `x`. A `Command` is not affine although a `Stdio::File` in it is ([2.13.4](../200-core/213-marker.md)): that one file is the exception, shared by every copy (below). A builder call whose result is dropped (`base.arg("x");`) changes nothing and warns (`#[must_use]`).

Note that `Stdio` has four branches: `Inherit`, which passes on the parent's fd; `Null`, which discards output entirely; `Piped`, for two-way communication; and `File`, which plugs in an open file.

A `File` moved into `Stdio::File` belongs to the command, not to the program: the move consumes it, as any move of a `File` does ([2.13.4](../200-core/213-marker.md)), and the program does not close it. Running the command (`spawn`, `output`, `status` and their `_async` twins) closes the parent's copy right after the child is started, and also when starting it fails; the child keeps its own. A failed close there is the run's `IoError`: the child, already started, is killed and reaped first, so no `Child` is lost. A copy of a command whose file an earlier run closed fails with `InvalidInput` instead of running; give each run its own file (`File::try_clone`). A command that never runs leaves its file open until the collector frees it.

## 3.7.2. `Child`

When `spawn` succeeds, it returns a `Child`, the handle to the running process. Through it you can access the standard I/O streams, wait for exit, or send signals.

```rok
struct Child { /* opaque: pid, the piped streams not yet taken */ }

impl Child {
    fn id(self) -> u32;                                  // pid
    fn take_stdin(ref self) -> ChildStdin | none;        // none: not Piped, or taken already
    fn take_stdout(ref self) -> ChildStdout | none;
    fn take_stderr(ref self) -> ChildStderr | none;
    fn kill(self) throws (IoError);
    fn wait(ref self) throws (IoError) -> ExitStatus;
    async fn wait_async(ref self) throws (IoError) -> ExitStatus;
    fn try_wait(self) throws (IoError) -> ExitStatus | none;

    // Linux: pidfd, ChildExt (3.11.5)

    fn send_signal(self, sig: Signal) throws (IoError);  // Windows: Kill only
}

impl ChildStdin {                        // write end of the child's stdin pipe; impl io::Write + io::Close, io::AsyncWrite + io::AsyncClose
    fn close(self) throws (IoError);     // the child reads end of input
}
impl ChildStdout {                       // read end of the child's stdout pipe; impl io::Read + io::Close, io::AsyncRead + io::AsyncClose
    fn close(self) throws (IoError);
}
impl ChildStderr {                       // read end of the child's stderr pipe; impl io::Read + io::Close, io::AsyncRead + io::AsyncClose
    fn close(self) throws (IoError);
}
// + AsRawFd (unix), AsRawHandle (Windows) for the three pipes: 3.11

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

The exit result is held in the opaque type `ExitStatus`. If `code()` is `none`, the process was terminated by a signal on unix, and `signal()` (`ExitStatusExt`, in the prelude on unix) names it as a `Signal`.

## 3.7.3. Examples

```rok
use rok::std::process::{Command, Stdio};

// simple execution
let status = try Command::new("ls").arg("-la").status();
if !status.success() { panic("ls failed") };

// collecting output
let output = try Command::new("git")
    .args(["log", "--oneline", "-10"])
    .output();
let log = try str::from_utf8(output.stdout);

// I/O through pipes
let child = try Command::new("grep")
    .arg("error")
    .stdin(Stdio::Piped)
    .stdout(Stdio::Piped)
    .spawn();

let stdin = child.take_stdin() ?? panic("stdin not piped");
try stdin.write_all(input_bytes);
try stdin.close();                                       // end of input: grep can finish
let stdout = child.take_stdout() ?? panic("stdout not piped");
let result = try stdout.read_to_end();
try stdout.close();
try child.wait();

// output into a file: the command takes the file and closes our copy
let log = try File::create("ls.txt");
try Command::new("ls").stdout(Stdio::File(log)).status();

// kill_on_drop prevents process leaks
using child = try Command::new("long_running")
        .kill_on_drop(true)
        .spawn() {
    // child is guaranteed to be killed even on error propagation / early return
}    // ← on using release, Drop → kill
```

As the last example shows, binding a `Child` with `kill_on_drop` enabled in `using` cleans up the child even on flows such as error propagation or early return. `Drop` issues the kill where the owner releases the `Child`: at the end of the `using` block, or where its last owner leaves scope ([2.13.4](../200-core/213-marker.md)). `ChildStdin` / `ChildStdout` / `ChildStderr` are file handles of the pipes and are read and written the same way as `File`. The `Child` holds its `Piped` streams until `take_stdin` / `take_stdout` / `take_stderr` hand one over, leaving `none`; from then on the taker owns the pipe and must close it, like a `File`. The pipes never taken close where the `Child` is released. `wait` closes a stdin still held by the `Child` first, so a child reading to end of input can finish; a taken stdin is the taker's to close before `wait`.

Copies of a `Child` share the process. `wait` / `try_wait` reap the child only after the `send_signal` / `kill` calls in flight through other copies return, so a signal never reaches a pid or pidfd that was reused. A `send_signal` once the child has ended and is being reaped is a no-op. Copies that wait at the same time all get the same `ExitStatus`.

## 3.7.4. Modern additions - pidfd / posix_spawn / CLOEXEC

On top of the builder pattern, the standard library adds several modern reinforcements. What each means and what it achieves:

| Feature                 | Meaning                       | Effect                  |
|-------------------------|-------------------------------|-------------------------|
| `posix_spawn` fast path | A syscall instead of fork+exec | Faster spawn from huge processes |
| `pidfd` (Linux 5.3+)    | Track by fd instead of pid    | race-free signal        |
| `CLOEXEC` default       | Every fd is close-on-exec automatically | Prevents fd leaks |
| `kill_on_drop`          | Kill where the owner releases it | Prevents process leaks  |

These decisions consistently aim at reducing fd leaks, process leaks and signal races.

With `Command::env`, the program is still looked up on the parent's `PATH`, not the child's.

## 3.7.5. Signal handling (advanced)

Going a level deeper, there are the signal handling functions.

```rok
#[non_exhaustive]
enum Signal {
    Hangup, Interrupt, Quit, IllegalInstruction, Trap, Abort, Bus, FloatingPoint, Kill,
    Segmentation, Pipe, Alarm, Terminate, User1, User2, Child, Continue, Stop,
    TerminalStop, TerminalInput, TerminalOutput, Urgent, CpuLimit, FileSizeLimit,
    VirtualAlarm, Profile, WindowChange, Io, BadSyscall,
}                                        // SIGHUP .. SIGSYS

impl Signal {
    fn number(self) -> i32;              // the OS's number (SIGHUP = 1, ..; Linux's, else the BSD one)
}                                        // + Clone, Copy, PartialEq, Eq, Hash, Debug

fn on_signal(sig: Signal, handler: fn() + Send) throws (SignalError);
fn send_signal(pid: u32, sig: Signal) throws (IoError);
fn raise(sig: Signal) throws (IoError);  // Windows: Interrupt and Hangup only
fn id() -> u32;                          // this process's id; panics on wasm32, which has none
fn exit(code: i32) -> !;                 // exit now via the runtime's rok_exit; finally and Drop do not run (refs-lang 4.16)
fn abort() -> !;                         // abnormal termination (SIGABRT); nothing is cleaned up

struct SignalError { .. }                // private fields (0.5.10)
impl SignalError {
    fn signal(self) -> Signal;           // the signal being installed
    fn os_code(self) -> i32 | none;      // errno (Windows: the error code); none if the dispatcher thread could not start
    fn message(self) -> str;
}
// + Display, Debug, Error, PartialEq, Eq, Clone
```

An `on_signal` handler does not run in signal context (inside a signal handler). The runtime receives the signal and hands it to a dedicated dispatcher thread `rok-signal`, where the handler runs like an ordinary function. The handler must therefore be `Send`, and it may allocate or take locks. Calling `on_signal` again for the same signal replaces the handler. `Signal` names the signal on every OS. `process_group(pgid)` (`CommandExt`, unix) puts the child in that process group (`0` means a new group with the child's pid).

On Windows, console events map to signals: Ctrl-C → `Interrupt` (2), console close / logoff / shutdown → `Hangup` (1). Ctrl-Break has no `Signal`; `os::windows::process::on_ctrl_break` routes it ([3.11.7](311-os.md#3117-oswindowsprocess---ctrl-break)). `send_signal` sends only `Signal::Kill`, as `TerminateProcess`, and `raise` raises only `Interrupt` and `Hangup`; any other `Signal` is an `IoError` (`Unsupported`), and `on_signal` of a signal other than `Interrupt` or `Hangup` is a `SignalError`.

Most users never need to touch this. These tools become necessary only where signals must be faced head-on, such as daemons, servers and interactive CLIs.

## 3.7.6. Platform availability

Per-platform support:

| Item             | linux      | macos | windows         | wasi |
|------------------|------------|-------|-----------------|------|
| `Command::spawn` | ✓          | ✓     | ✓               | ✗    |
| `posix_spawn`    | ✓          | ✓     | (CreateProcess) | ✗    |
| `pidfd`          | linux 5.3+ (`os::linux`) | ✗ | ✗         | ✗    |
| `kill_on_drop`   | ✓          | ✓     | ✓               | ✗    |
| signal handler   | ✓          | ✓     | (limited)       | ✗    |
| `process::id()`  | ✓          | ✓     | ✓               | panics (wasm32) |

On wasm32 targets the items marked ✗ fail at run time with `IoErrorKind::Unsupported`.
