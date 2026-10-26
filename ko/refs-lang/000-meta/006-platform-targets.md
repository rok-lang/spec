# 0.6. 플랫폼 타깃

Rok 은 LLVM 이 코드 생성을 지원하는 플랫폼이라면 어느 host 에서든 그 플랫폼용으로 빌드합니다 (`--target`, [0.2.4](002-pipeline.md#024-backends)). 다만 실제로 동작이 보증되는 플랫폼은 한정된 목록(Tier 1)으로 정해 두며, 그 외의 플랫폼은 필요에 따라 사안별로 검토합니다.

## 0.6.1. Tier 1: 정식 지원

다음 15 개의 타깃은 모든 PR 에 대해 CI 가 실행되며, 빌드와 테스트의 통과가 의무로 요구됩니다.

### 64-bit 데스크탑 / 서버

이 영역은 Linux 와 macOS, Windows 의 주요 아키텍처를 모두 포괄하며, 정적 바이너리를 위한 musl 변형과 64-bit RISC-V Linux 까지 포함합니다.

| 타깃                         | 비고                                       |
|------------------------------|--------------------------------------------|
| `x86_64-unknown-linux-gnu`   | Linux 서버 / 데스크탑 주력                 |
| `x86_64-apple-darwin`        | macOS Intel                                |
| `aarch64-apple-darwin`       | macOS Apple Silicon                        |
| `aarch64-unknown-linux-gnu`  | ARM 서버 (AWS Graviton, Raspberry Pi 5 등) |
| `x86_64-pc-windows-msvc`     | Windows 데스크탑                           |
| `aarch64-pc-windows-msvc`    | Windows ARM                                |
| `x86_64-unknown-linux-musl`  | Alpine / `static` binary                   |
| `aarch64-unknown-linux-musl` | 동일, ARM                                  |
| `riscv64gc-unknown-linux-gnu`| RISC-V 64-bit Linux 서버                   |

### WebAssembly

다음으로 WebAssembly 계열을 봅니다. 브라우저 기반 실행과 서버사이드 실행을 모두 염두에 두고 세 가지 타깃을 지원합니다. 지금은 WASI 소켓 (preview 2 component 전용) 과 WASI threads (preview 1 core module 제안) 를 한 모듈에서 함께 쓸 수 없으므로 서버사이드 WASM 은 타깃이 둘입니다.

| 타깃                     | 비고                             |
|--------------------------|----------------------------------|
| `wasm32-unknown-unknown` | 브라우저 환경. `target_os = "none"` ([6.3.2](../006-meta-system/603-cfg.md)) |
| `wasm32-wasi`            | 서버사이드 WASM: WASI preview 2 component. `wasi:sockets` 네트워크, 스레드 없음 (`thread::spawn` 은 `Unsupported` 를 throw) |
| `wasm32-wasi-threads`    | 스레드가 있는 서버사이드 WASM: wasi-threads 를 쓰는 WASI preview 1 core module (공유 메모리). 네트워크는 host 가 넘긴 listener 에서만, connect / bind / DNS 는 `Unsupported` 를 throw |

> [!NOTE]
> wasmtime 은 v46 뒤에 wasi-threads 를 제거했으므로, `wasm32-wasi-threads` 는 앞으로 WASI shared-everything-threads 에 기대게 됩니다.

### 32-bit

마지막으로 32-bit 영역인데, 여전히 활발하게 사용되는 임베디드 및 레거시 환경을 위해 다음 세 타깃을 정식으로 지원합니다.

| 타깃                            | 비고                                 |
|---------------------------------|--------------------------------------|
| `i686-unknown-linux-gnu`        | 32-bit x86 Linux                     |
| `i686-pc-windows-msvc`          | 32-bit Windows                       |
| `armv7-unknown-linux-gnueabihf` | 32-bit ARM Linux (Raspberry Pi 4 등) |

이렇게 총 15 개의 타깃을 통해 과학·데이터·HPC·Jupyter·임베디드 분야의 사용자 대부분을 직접 커버합니다.

## 0.6.2. Tier 1 외 타깃: 보증 없음

다음 타깃들은 Tier 1 에 속하지 않으며, 본 사양은 이들에 대한 동작을 보증하지 않습니다. 구체적으로는 IBM Power HPC 환경의 `powerpc64le-unknown-linux-gnu`, FreeBSD 용 `x86_64-unknown-freebsd`, RISC-V 임베디드용 `riscv32imac-unknown-none`, 그리고 IBM Z 메인프레임을 위한 `s390x-unknown-linux-gnu` 가 여기에 해당합니다.

## 0.6.3. 32-bit 지원의 함의

32-bit 타깃이 Tier 1 에 포함되어 있는 만큼, 사양 전반은 32-bit 와 64-bit 양쪽 환경에서 모두 동작하도록 규정됩니다. 객체 헤더 레이아웃은 두 비트 폭에서 동일합니다. 32-bit 환경에서 64-bit 정수의 atomic 연산은 타깃의 native 64-bit atomic 명령 (i686 의 `cmpxchg8b`, ARMv7 의 `ldrexd` / `strexd`) 을 쓰며, 그런 명령이 없으면 시스템 atomic 라이브러리를 씁니다. `usize` 의 비트 폭은 타깃의 기본 포인터 폭을 따릅니다. wasm32 의 힙은 linear memory 이며, 필요할 때 wasm32 한도인 4 GiB 까지 늘어납니다. 그 밖의 32-bit 타깃에서 가비지 컬렉터가 쓸 수 있는 힙은 약 3.5 GB 입니다: block heap 은 연속 공간을 최대 2 GiB 예약하고 (OS 가 거절하면 그보다 적게), 주소 공간의 나머지는 큰 객체가 차지하며, Windows x86 image 는 large-address-aware 입니다. 64-bit 에서는 사실상 무제한입니다. 시간은 32-bit 타깃에서도 64-bit 인터페이스 (`time64`) 를 쓰므로 `SystemTime` 의 범위는 어디서나 같습니다.

## 0.6.4. SIMD

빌드는 기본으로 triple 의 baseline CPU 를 대상으로 하므로 바이너리는 이식 가능합니다: `x86_64` 는 x86-64 baseline (SSE2), `aarch64-apple-darwin` 은 `apple-m1`, 그 밖의 `aarch64` 는 armv8-a (NEON), `wasm32` 는 SIMD128. 더 넓은 명령 세트 (AVX, AVX-512, SVE 등) 는 `-C target-cpu=<cpu>|native` 와 `-C target-feature=+avx2,...` 로 켭니다 ([9.1](../009-tools/901-rokc-cli.md)). 컴파일러는 런타임 기능 감지를 하지 않으며, 필요한 프로그램은 스스로 합니다.

다만 SIMD 의 라이브러리 표면 자체는 본 사양의 다른 문서에서 별도로 정의하고 있으니, 그쪽을 함께 참고합니다.

## 0.6.5. 타깃 명세

모든 타깃은 하나의 타깃 명세로 기술됩니다. 컴파일러, 링커, 런타임 빌드는 타깃에 대해 그 밖의 것을 읽지 않습니다. 타깃은 `--target` 으로 지정합니다: 내장 이름 (`rokc --print target-list`: 0.6.1 의 Tier 1 타깃과 0.6.2 의 타깃), 또는 ROKON 값 `TargetSpec { .. }` 을 담은 파일 `<name>.rokon` 의 경로 (custom 타깃, 이름은 파일의 stem).

`rokc --print target-spec [--target=<t>]` 은 타깃의 명세를 그 형태로 출력하므로, custom 타깃은 보통 내장 타깃의 사본에서 시작합니다. custom 파일은 모든 필드를 주어야 하며, 모르는 필드는 에러입니다. custom 타깃의 런타임과 (그 타깃이 제공한다면) C 라이브러리는 toolchain 의 `lib/rok/targets/<name>/` 에서 오며, 그 디렉토리가 없으면 링크하는 빌드는 그것을 밝히며 실패합니다.

| 필드 | 의미 |
|------|------|
| `llvm_target`, `deployment_target` | LLVM triple. 그 뒤에 붙는 최소 OS 버전 (Mach-O, `MACOSX_DEPLOYMENT_TARGET` 이 덮음) |
| `arch`, `os`, `env`, `family`, `endian`, `pointer_width` | `target_*` cfg 값 ([6.3.2](../006-meta-system/603-cfg.md)). 비어 있는 `env` / `family` 는 키를 두지 않음 |
| `vendor` | 정보용 |
| `wide_align`, `max_atomic_width`, `c_abi`, `data_layout` | `i64` / `u64` / `f64` 정렬, 가장 넓은 atomic, C 호출 규약 ([8.6.3](../008-abi/806-calling-convention.md)), LLVM data layout |
| `cpu`, `features` | 기본 CPU 와 feature (0.6.4 의 SIMD 기준선) |
| `object_format`, `linker`, `pre_link_args`, `crt_objects`, `late_link_args` | object 형식과 lld flavor, 타깃 자체의 링커 인자, C 런타임 object, 입력 뒤의 인자나 파일 |
| `dynamic_linking`, `dynamic_loader`, `libc` | 공유 라이브러리 지원, ELF interpreter, C 라이브러리와 그것이 toolchain 과 함께 오는지 |
| `rust_target` | 런타임을 빌드하는 Rust 타깃 |
| `exe_suffix`, `staticlib_prefix` / `staticlib_suffix`, `dylib_prefix` / `dylib_suffix`, `c_symbol_prefix` | 파일과 심볼 이름 |
| `stack_size`, `stack_probes`, `threads` | main 스레드의 스택, stack probing, 스레드 지원 |
| `native_backend`, `repl`, `profile_runtime` | Rok 자신의 `-O0` backend, REPL host, `-C profile-generate` 의 런타임 |

명세에는 안정된 해시가 있으며, 모든 빌드 cache 는 그것을 키로 씁니다.
