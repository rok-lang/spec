# 9.8. Jupyter kernel integration

This section defines the Rok kernel of the [Jupyter surface](../000-meta/001-execution-modes.md).

## 9.8.1. Kernel

Jupyter runs the Rok kernel as a separate process and communicates over the ZeroMQ-based Jupyter wire protocol. The kernel announces `protocol_version` as `"5.3"` (the value `jupyter_client` expects; works with clients 5.3 and later). In this form all OS resources, i.e. files, network and external processes, are available.

The kernel runs on **the same cell session** as the terminal REPL and follows the rules of [9.7.0](907-jit-cells.md#970-rules-of-the-cell-session): each cell runs exactly once, bindings and items carry over to later cells, and a failed cell leaves no trace in the bindings. The kernel's engine is the same [JIT session](906-jit-engine.md) as the REPL: a top-level `await` and `it` work as there ([9.7.1](907-jit-cells.md#971-kinds-of-cell), [9.7.7](907-jit-cells.md#977-visualization--output)).

### Input

A cell reads stdin through the client. When the `execute_request` has `allow_stdin: true`, each read the runtime makes of stdin sends an `input_request` (`prompt` empty, `password` false) on the stdin channel, to the client of that request; what the cell wrote before the read, such as a prompt it printed, has been sent as `stream` first. The `value` of the `input_reply`, with a newline added, is what the read returns. Without `allow_stdin`, or for a `silent` request, a read is end of input. An interrupt while the cell waits for the reply ends the cell (9.8.4).

### Notebook cells and REPL cells

A notebook cell is split into several REPL cells by the same rules the terminal REPL uses to split input, and they run in order. As in IPython, **only the last value** is displayed as the result. If a later part of a notebook cell fails, the effects of the earlier parts (the preceding REPL cells) remain.

## 9.8.2. Local native kernel: operation

When the user selects "Rok" as the notebook's kernel, Jupyter runs the command of the kernelspec (9.8.3) as a separate process.

```
rokc jupyter --connection-file <path>       # also accepts rokc options such as --extern
rok jupyter --connection-file <path>        # resolves dependencies, then runs rokc jupyter
```

The kernel reads the ZMQ ports and key from the connection file, then binds and listens on its five sockets. Once all five accept connections, it writes the line `rokc jupyter: ready` to its stderr, before output capture starts. Only the `tcp` transport is supported (no `ipc`), and CURVE encryption is not used. Messages are HMAC-signed with the connection file's key.

The kernel loads the package it starts in (the current directory or the nearest one above it with `src/main.rok` or `src/lib.rok`) as `rokc repl .` does ([9.7.10](907-jit-cells.md#9710-loading-a-program-repl-path)): a library package is a dependency with `use <name>::*;` as the first cell; a program package is loaded and run. This happens after output capture starts; what it prints is sent with the output of the first executed cell. Outside a package the session starts empty.

### Socket channels

| Channel     | Purpose                                                    |
|-------------|------------------------------------------------------------|
| `shell`     | Requests (execute, kernel_info, complete, is_complete, etc.) |
| `iopub`     | Output and status broadcast (status, stream, execute_result, etc.) |
| `stdin`     | `input_request` / `input_reply` for a cell that reads stdin (9.8.1) |
| `control`   | interrupt, shutdown                                        |
| `heartbeat` | Sends back whatever it receives (echo)                     |

### Message format

Each message is serialized as six or more ZMQ frames.

```
[b"u-u-i-d",         # zmq identity(ies)
 b"<IDS|MSG>",       # delimiter
 b"baddad42",         # HMAC signature (with key)
 b"{header}",          # header JSON
 b"{parent_header}",   # parent header JSON
 b"{metadata}",        # metadata JSON
 b"{content}",         # content JSON
]
```

The detailed format follows the Jupyter wire protocol standard. Every request is wrapped in iopub `status: busy` and `status: idle`.

### Supported messages

| Request                | Reply                                                                                  |
|------------------------|----------------------------------------------------------------------------------------|
| `kernel_info_request`  | `language_info` with name `rok`, mimetype `text/x-rok`, file_extension `.rok`, codemirror_mode and pygments_lexer `rust` (highlighting until front ends know Rok); `help_links` to the documentation (`https://docs.rok-lang.org/latest/`) and this REPL chapter; `protocol_version` `"5.3"` |
| `execute_request`      | See "Execution" below                                                                  |
| `is_complete_request`  | `complete` / `incomplete` decided by the REPL continuation rules; `indent` is four spaces per bracket still open |
| `complete_request`     | Session names, members and keywords as candidates (9.8.5)                              |
| `inspect_request`      | `found: true` with `text/plain` help for the name at the cursor (9.8.6), else `false`  |
| `history_request`      | The cells executed by this kernel                                                      |
| `comm_info_request`    | Empty list                                                                             |
| `shutdown_request`     | On shell or control. Exits after replying                                              |
| `interrupt_request`    | On control (9.8.4)                                                                     |

### Execution

On receiving an `execute_request`, the kernel sends:

1. iopub `execute_input` (code and execution count).
2. The cell's output **as it is written**: what a running cell writes to stdout / stderr is sent as iopub `stream` (`stdout` / `stderr`) as the kernel reads it. The two are separate pipes, so chunks of one keep their order and chunks of both come in the order they are read. Output written outside a run (diagnostics, `:` commands) is sent before the next run or when the cell finishes.
3. If there is a value, iopub `execute_result`: `text/plain` holding the same output as the REPL (9.7.7).
4. On failure, iopub `error`: `evalue` is the first line of the stderr not yet sent (the diagnostics of a cell that does not compile, or the report of how a run ended), `traceback` its lines. What the cell itself wrote to stderr, such as a panic message, has been streamed before.
5. shell `execute_reply` (including `execution_count`, with success / failure status).

The only output MIME type is `text/plain`.

## 9.8.3. kernelspec installation

Jupyter only recognizes kernels installed in a kernelspec directory.

```
rok jupyter install [--env <name>]
```

The kernelspec is `kernels/rok/kernel.json` under the directory below. Installed with `--env <name>`, it is `kernels/rok-<name>/`, with display name `rok (<name>)` and command `rok --env <name> jupyter ...`.

1. `$JUPYTER_DATA_DIR` if set
2. `~/Library/Jupyter` on macOS
3. `%APPDATA%\jupyter` on Windows
4. Otherwise `$XDG_DATA_HOME/jupyter`, or `~/.local/share/jupyter` if unset

Next to it are `logo-32x32.png` and `logo-64x64.png`, the Rok logo front ends show for the kernel. `kernel.json` runs the `rok` proxy ([9.10.6](910-rok-frontend.md#9106-toolchain)) and records the `ROK_HOME` and `PATH` at installation time in `env`.

```json
{
  "argv": ["/home/u/.rok/bin/rok", "jupyter", "--connection-file", "{connection_file}"],
  "display_name": "rok",
  "language": "rok",
  "interrupt_mode": "message",
  "env": { "ROK_HOME": "/home/u/.rok", "PATH": "<PATH at install time>" }
}
```

Once installed, the Rok kernel is usable from Jupyter, VS Code, nteract and others.

## 9.8.4. Interrupt / shutdown

### Interrupt

Since `interrupt_mode` is `"message"`, an interrupt arrives as an `interrupt_request` on the control socket. The running cell is terminated and the session continues from the state before that cell (as for a failed cell, [9.7.0](907-jit-cells.md#970-rules-of-the-cell-session)). A cell waiting for an `input_reply` ends as well. A cell still being compiled cannot be interrupted.

### Shutdown (kernel restart)

After replying to a `shutdown_request` on the shell or control socket, the kernel cleans up the session and exits the process. On restart, Jupyter launches it again automatically.

## 9.8.5. Code completion

`complete_request` matches the fragment before the cursor against the session's bindings, items, module paths, members and keywords, and replies with `{ matches, cursor_start, cursor_end }`.

## 9.8.6. inspect_request: help

`inspect_request` (Shift+Tab) replies with `found: true` and, as `text/plain`, the name at the cursor as `name: type`, its signature and doc comments (`///` / `//!` of [Lexical structure](../001-lexical/102-lexical-structure.md)). For an unknown name it is `found: false`.

## 9.8.7. Code sharing between REPL and Jupyter

Code written in the REPL can be moved to Jupyter cells unchanged, and vice versa, because both follow the same cell session and the same semantics ([9.7.0](907-jit-cells.md#970-rules-of-the-cell-session)).

### Differences

The two surfaces differ in a few respects, summarized below.

| Aspect      | REPL              | Jupyter                      |
|-------------|-------------------|------------------------------|
| Input       | One line / one expression | One cell (multiple lines) |
| Output      | text only         | text (`text/plain`)          |
| stdin       | The terminal; empty when piped | `input_request` with `allow_stdin` |
| State       | Discarded on exit | Cells kept in the notebook file |

## 9.8.8. Security / isolation

The kernel runs arbitrary code with the user's privileges. No isolation is performed.

## 9.8.9. Next

The following [Debug / diagnostic tools](909-debug-tools.md) defines gdb-friendly features, profiling, `--emit` and more.
