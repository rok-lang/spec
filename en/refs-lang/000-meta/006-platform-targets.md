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

On each Tier 1 target, the following SIMD instruction sets are detected automatically and enabled by default.

| Target    | SIMD                |
|-----------|---------------------|
| `x86_64`  | SSE / AVX / AVX-512 |
| `aarch64` | NEON / SVE          |
| `i686`    | SSE2 (baseline)     |
| `armv7`   | NEON                |
| `wasm32`  | WASM SIMD128        |

The library surface for SIMD itself is defined separately in other documents of this specification.
