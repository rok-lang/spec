# 0.6. Platform targets

rok builds for any platform for which LLVM supports code generation, from any host (`--target`, [0.2.4](002-pipeline.md#024-backends)). The platforms on which behavior is actually guaranteed, however, are fixed in a limited list (Tier 1); other platforms are considered case by case as needed.

## 0.6.1. Tier 1: official support

For the following 14 targets, CI runs on every PR, and builds and tests are required to pass.

### 64-bit desktop / server

This group covers the major architectures of Linux, macOS and Windows, including musl variants for static binaries.

| Target                       | Notes                                      |
|------------------------------|--------------------------------------------|
| `x86_64-unknown-linux-gnu`   | Primary Linux server / desktop             |
| `x86_64-apple-darwin`        | macOS Intel                                |
| `aarch64-apple-darwin`       | macOS Apple Silicon                        |
| `aarch64-unknown-linux-gnu`  | ARM servers (AWS Graviton, Raspberry Pi 5, etc.) |
| `x86_64-pc-windows-msvc`     | Windows desktop                            |
| `aarch64-pc-windows-msvc`    | Windows ARM                                |
| `x86_64-unknown-linux-musl`  | Alpine / `static` binary                   |
| `aarch64-unknown-linux-musl` | Same, ARM                                  |

### WebAssembly

Three WebAssembly targets are supported, covering browser-based and server-side execution. WASI sockets (preview 2 components only) and WASI threads (a preview 1 core-module proposal) cannot be combined in one module today, so server-side WASM has two targets.

| Target                   | Notes                            |
|--------------------------|----------------------------------|
| `wasm32-unknown-unknown` | Browser environment; `target_os = "none"` ([6.3.2](../006-meta-system/603-cfg.md)) |
| `wasm32-wasi`            | Server-side WASM: a WASI preview 2 component. `wasi:sockets` networking; no threads (`thread::spawn` throws `Unsupported`) |
| `wasm32-wasi-threads`    | Server-side WASM with threads: a WASI preview 1 core module using wasi-threads (shared memory). Networking only on listeners the host passes in; connect / bind / DNS throw `Unsupported` |

> [!NOTE]
> wasmtime removed wasi-threads after v46, so `wasm32-wasi-threads` will depend on WASI shared-everything-threads in the future.

### 32-bit

For embedded and legacy environments still in active use, the following three 32-bit targets are officially supported.

| Target                          | Notes                                |
|---------------------------------|--------------------------------------|
| `i686-unknown-linux-gnu`        | 32-bit x86 Linux                     |
| `i686-pc-windows-msvc`          | 32-bit Windows                       |
| `armv7-unknown-linux-gnueabihf` | 32-bit ARM Linux (Raspberry Pi 4, etc.) |

These 14 targets directly cover most users in science, data, HPC, Jupyter and embedded work.

## 0.6.2. Targets outside Tier 1: no guarantee

The following targets are not in Tier 1, and this specification does not guarantee behavior on them: `riscv64gc-unknown-linux-gnu` for RISC-V servers, `powerpc64le-unknown-linux-gnu` for IBM Power HPC environments, `x86_64-unknown-freebsd` for FreeBSD, `riscv32imac-unknown-none` for RISC-V embedded, and `s390x-unknown-linux-gnu` for IBM Z mainframes.

## 0.6.3. Implications of 32-bit support

Because 32-bit targets are in Tier 1, the specification as a whole is defined to work in both 32-bit and 64-bit environments. The object header layout is the same at both bit widths. Atomic operations on 64-bit integers use the target's native 64-bit atomic instructions on 32-bit environments (`cmpxchg8b` on i686, `ldrexd` / `strexd` on ARMv7) or, where none exist, the system atomic library. The bit width of `usize` follows the target's native pointer width automatically. On wasm32 the heap is linear memory, grown on demand up to the 4 GiB wasm32 limit. On other 32-bit targets the heap available to the garbage collector is about 3.5 GB: the block heap reserves at most 2 GiB contiguous (less if the OS refuses), the rest of the address space holds large objects, and Windows x86 images are large-address-aware. On 64-bit it is effectively unlimited. Time uses 64-bit interfaces (`time64`) on 32-bit targets too, so `SystemTime` has the same range everywhere.

## 0.6.4. SIMD

Builds target the triple's baseline CPU by default, so binaries are portable: x86-64 baseline (SSE2) for `x86_64`, `apple-m1` for `aarch64-apple-darwin`, armv8-a (NEON) for other `aarch64` targets, and SIMD128 for `wasm32`. Wider instruction sets (AVX, AVX-512, SVE, ...) are opt-in with `-C target-cpu=<cpu>|native` and `-C target-feature=+avx2,...` ([9.1](../009-tools/901-rokc-cli.md)). The compiler does no runtime feature detection; a program that wants it does it itself.

The library surface for SIMD itself is defined separately in other documents of this specification.

## 0.6.5. Target specifications

Every target is described by one target specification; the compiler, the linker and the runtime build read nothing else about a target. A target is named by `--target`: a built-in name (`rokc --print target-list`: the Tier 1 targets of 0.6.1 and the targets of 0.6.2), or the path of a file `<name>.rokon` holding a ROKON value `TargetSpec { .. }` (a custom target, named by the file's stem).

`rokc --print target-spec [--target=<t>]` prints a target's specification in that form, so a custom target usually starts as a copy of a built-in one. A custom file must give every field; an unknown field is an error. A custom target's runtime and, where the target ships one, its C library come from `lib/rok/targets/<name>/` of the toolchain; without that directory, a build that links fails naming it.

| Field | Meaning |
|-------|---------|
| `llvm_target`, `deployment_target` | LLVM's triple; the minimum OS version appended to it (Mach-O; `MACOSX_DEPLOYMENT_TARGET` overrides) |
| `arch`, `os`, `env`, `family`, `endian`, `pointer_width` | The `target_*` cfg values ([6.3.2](../006-meta-system/603-cfg.md)); an empty `env` / `family` sets no key |
| `vendor` | Informational |
| `wide_align`, `max_atomic_width`, `c_abi`, `data_layout` | `i64` / `u64` / `f64` alignment, the widest atomic, the C calling convention ([8.6.3](../008-abi/806-calling-convention.md)), LLVM's data layout |
| `cpu`, `features` | The default CPU and features (the SIMD baseline of 0.6.4) |
| `object_format`, `linker`, `pre_link_args`, `crt_objects`, `late_link_args` | Object format and lld flavor; the target's own linker arguments; C runtime objects; arguments or files after the inputs |
| `dynamic_linking`, `dynamic_loader`, `libc` | Shared-library support, the ELF interpreter, the C library and whether it ships with the toolchain |
| `rust_target` | The Rust target the runtime is built for |
| `exe_suffix`, `staticlib_prefix` / `staticlib_suffix`, `dylib_prefix` / `dylib_suffix`, `c_symbol_prefix` | File and symbol naming |
| `stack_size`, `stack_probes`, `threads` | The main thread's stack, stack probing, thread support |
| `native_backend`, `repl`, `profile_runtime` | rok's own `-O0` backend, the REPL host, the runtime of `-C profile-generate` |

A specification has a stable hash; every build cache keys on it.
