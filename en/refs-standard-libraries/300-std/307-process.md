# 3.7. `process::` — child processes

The `rok::std::process` module spawns child processes and handles signals. Its basic skeleton follows Rust's `Command` builder pattern, hardened with modern safeguards on top: `pidfd`, `kill_on_drop` and a `posix_spawn` fast path.

## 3.7.1. `Command` builder

The first thing you meet is the `Command` struct, the blueprint of a child process. All settings — the program to run, arguments, environment variables, how standard I/O is connected — are built up step by step before moving on to spawn.

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
    kill_on_drop: bool;       // kill when a Child bound with using is released
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

    // execution
    fn spawn(self) throws (IoError) -> Child;
    fn output(self) throws (IoError) -> Output;          // wait + collect
    fn status(self) throws (IoError) -> ExitStatus;      // wait, no output
}

enum Stdio {
    Inherit,                   // parent's fd as is (default)
    Null,                      // /dev/null
    Piped,                     // pipe — readable/writable from Child
    File(File),                // explicit fd
}

struct Output {
    status: ExitStatus;
    stdout: List<u8>;
    stderr: List<u8>;
}
```

Note that `Stdio` has four branches: `Inherit`, which passes on the parent's fd; `Null`, which discards output entirely; `Piped`, for two-way communication; and `File`, which plugs in an arbitrary fd explicitly.

## 3.7.2. `Child`

When `spawn` succeeds, it returns a `Child`, the handle to the running process. Through it you can access the standard I/O streams, wait for exit, or send signals.

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

impl ChildStdin { ... }                  // file handle — impl Writer
impl ChildStdout { ... }                 // file handle — impl Reader
impl ChildStderr { ... }                 // file handle — impl Reader

struct ExitStatus { /* opaque */ }

impl ExitStatus {
    fn success(self) -> bool;
    fn code(self) -> i32 | none;        // none = signal terminated (unix)
    fn signal(self) -> i32 | none;       // unix
}
```

The exit result is held in the opaque type `ExitStatus`. If `code()` is `none`, the process was terminated by a signal on unix, and the signal number can be retrieved separately with `signal()`.

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

let stdin = child.stdin ?? panic("stdin not piped");     // the field is ChildStdin | none
let stdout = child.stdout ?? panic("stdout not piped");
try stdin.write_all(input_bytes);
let result = try stdout.read_to_end();
try child.wait();

// kill_on_drop — prevents process leaks
using child = try Command::new("long_running")
        .kill_on_drop(true)
        .spawn() {
    // child is guaranteed to be killed even on error propagation / early return
}    // ← on using release, Drop → kill
```

As the last example shows, binding a `Child` with `kill_on_drop` enabled in `using` cleans up the child even on flows such as error propagation or early return. `Drop` issues the kill when leaving the `using` block. A `Child` outside `using` is not killed automatically ([2.13.4](../200-core/213-marker.md#2134-drop--cleanup-on-using-release)). `ChildStdin` / `ChildStdout` / `ChildStderr` are file handles of the pipes and are read and written the same way as `File`.

## 3.7.4. Modern additions — pidfd / posix_spawn / CLOEXEC

On top of the builder pattern, the standard library adds several modern reinforcements. What each means and what it achieves:

| Feature                 | Meaning                       | Effect                  |
|-------------------------|-------------------------------|-------------------------|
| `posix_spawn` fast path | A syscall instead of fork+exec | Faster spawn from huge processes |
| `pidfd` (Linux 5.3+)    | Track by fd instead of pid    | race-free signal        |
| `CLOEXEC` default       | Every fd is close-on-exec automatically | Prevents fd leaks |
| `kill_on_drop`          | Kill on `using` release       | Prevents process leaks  |

These decisions consistently aim at reducing fd leaks, process leaks and signal races.

With `Command::env`, the program is still looked up on the parent's `PATH`, not the child's.

## 3.7.5. Signal handling (advanced)

Going a level deeper, there are the signal handling functions.

```rok
fn signal(sig: i32, handler: fn() + Send) throws (SignalError);
fn kill(pid: u32, sig: i32) throws (IoError);
fn raise(sig: i32) throws (IoError);

struct SignalError {
    sig: i32;          // the signal being installed
    code: i32;         // errno
}
```

A `signal` handler does not run in signal context (inside a signal handler). The runtime receives the signal and hands it to a dedicated dispatcher thread `rok-signal`, where the handler runs like an ordinary function. The handler must therefore be `Send`, and it may allocate or take locks. Calling `signal` again for the same signal replaces the handler. `Command::process_group(pgid)` puts the child in that process group (`0` means a new group with the child's pid).

On Windows, console events map to signal numbers: Ctrl-C → 2, Ctrl-Break → 21, console close / logoff / shutdown → 1. `kill` and `signal` sent to a child call `TerminateProcess`.

Most users never need to touch this. These tools become necessary only where signals must be faced head-on, such as daemons, servers and interactive CLIs.

## 3.7.6. Platform availability

Per-platform support:

| Item             | linux      | macos | windows         | wasi |
|------------------|------------|-------|-----------------|------|
| `Command::spawn` | ✓          | ✓     | ✓               | ✗    |
| `posix_spawn`    | ✓          | ✓     | (CreateProcess) | ✗    |
| `pidfd`          | linux 5.3+ | ✗     | ✗               | ✗    |
| `kill_on_drop`   | ✓          | ✓     | ✓               | ✗    |
| signal handler   | ✓          | ✓     | (limited)       | ✗    |
| `process::id()`  | ✓          | ✓     | ✓               | `0` (wasm32) |

On wasm32 targets the items marked ✗ fail at run time with `IoErrorKind::Unsupported`.
