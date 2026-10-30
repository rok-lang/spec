# 0.6. 플랫폼 타깃

Rok은 LLVM이 코드 생성을 지원하는 플랫폼이면 어느 host에서든 그 플랫폼용으로 빌드합니다(`--target`, [0.2.4](002-pipeline.md#024-backends)). 다만 동작을 보증하는 플랫폼은 한정된 목록(Tier 1)으로 정해 두고, 그 밖의 플랫폼은 필요할 때 사안별로 검토합니다.

## 0.6.1. Tier 1: 정식 지원

다음 15개 타깃은 모든 PR마다 CI를 실행하며, 빌드와 테스트를 반드시 통과해야 합니다.

### 64-bit 데스크탑 / 서버

Linux, macOS, Windows의 주요 아키텍처를 모두 다루며, 정적 바이너리용 musl 변형과 64-bit RISC-V Linux까지 포함합니다.

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

브라우저 실행과 서버사이드 실행을 모두 고려해 WebAssembly 타깃 세 가지를 지원합니다. 지금은 WASI 소켓 (preview 2 component 전용)과 WASI threads (preview 1 core module 제안)를 한 모듈에서 함께 쓸 수 없어서 서버사이드 WASM 타깃이 둘입니다.

| 타깃                     | 비고                             |
|--------------------------|----------------------------------|
| `wasm32-unknown-unknown` | 브라우저 환경. `target_os = "none"` ([6.3.2](../006-meta-system/603-cfg.md)) |
| `wasm32-wasi`            | 서버사이드 WASM: WASI preview 2 component. `wasi:sockets` 네트워크, 스레드 없음 (`thread::spawn`은 panic, `thread::Builder::spawn`은 `Unsupported`를 throw) |
| `wasm32-wasi-threads`    | 스레드가 있는 서버사이드 WASM: wasi-threads를 쓰는 WASI preview 1 core module (공유 메모리). 네트워크는 host가 넘긴 listener 에서만, connect / bind / DNS는 `Unsupported`를 throw |

> [!NOTE]
> wasmtime이 v46 이후 wasi-threads를 제거했으므로, `wasm32-wasi-threads`는 앞으로 WASI shared-everything-threads에 기댑니다.

### 32-bit

아직 활발히 쓰이는 임베디드와 레거시 환경을 위해 다음 32-bit 타깃 세 가지를 정식으로 지원합니다.

| 타깃                            | 비고                                 |
|---------------------------------|--------------------------------------|
| `i686-unknown-linux-gnu`        | 32-bit x86 Linux                     |
| `i686-pc-windows-msvc`          | 32-bit Windows                       |
| `armv7-unknown-linux-gnueabihf` | 32-bit ARM Linux (Raspberry Pi 4 등) |

이 15개 타깃으로 과학·데이터·HPC·Jupyter·임베디드 분야 사용자 대부분을 직접 지원합니다.

## 0.6.2. Tier 1 외 타깃: 보증 없음

다음 타깃은 Tier 1에 속하지 않으며, 본 사양은 이 타깃들의 동작을 보증하지 않습니다. IBM Power HPC 환경의 `powerpc64le-unknown-linux-gnu`, FreeBSD용 `x86_64-unknown-freebsd`, RISC-V 임베디드용 `riscv32imac-unknown-none`, IBM Z 메인프레임용 `s390x-unknown-linux-gnu`입니다.

## 0.6.3. 32-bit 지원의 함의

32-bit 타깃이 Tier 1에 들어 있으므로, 사양 전체는 32-bit와 64-bit 환경 양쪽에서 동작하도록 규정합니다. 객체 헤더 레이아웃은 두 비트 폭에서 동일합니다. 32-bit 환경에서 64-bit 정수의 atomic 연산은 타깃의 native 64-bit atomic 명령 (i686의 `cmpxchg8b`, ARMv7의 `ldrexd` / `strexd`)을 쓰며, 그런 명령이 없으면 시스템 atomic 라이브러리를 씁니다. `usize`의 비트 폭은 타깃의 기본 포인터 폭을 자동으로 따릅니다. wasm32의 힙은 linear memory이며, 필요할 때 wasm32 한도인 4 GiB까지 늘어납니다. 그 밖의 32-bit 타깃에서 가비지 컬렉터가 쓸 수 있는 힙은 약 3.5 GB입니다. block heap은 연속 공간을 최대 2 GiB 예약하고(OS가 거절하면 그보다 적게), 주소 공간의 나머지는 큰 객체가 차지하며, Windows x86 image는 large-address-aware입니다. 64-bit에서는 사실상 무제한입니다. 시간은 32-bit 타깃에서도 64-bit 인터페이스 (`time64`)를 쓰므로 `SystemTime`의 범위는 어디서나 같습니다.

## 0.6.4. SIMD

빌드는 기본적으로 triple의 baseline CPU를 대상으로 하므로 바이너리를 다른 기기로 옮겨도 동작합니다. `x86_64`는 x86-64 baseline(SSE2), `aarch64-apple-darwin`은 `apple-m1`, 그 밖의 `aarch64`는 armv8-a(NEON), `wasm32`는 SIMD128이 기준입니다. 더 넓은 명령 세트(AVX, AVX-512, SVE 등)는 `-C target-cpu=<cpu>|native`와 `-C target-feature=+avx2,...`로 켭니다 ([9.1](../009-tools/901-rokc-cli.md)). 컴파일러는 런타임 기능 감지를 하지 않으며, 필요한 프로그램이 직접 합니다.

SIMD의 라이브러리 표면은 본 사양의 다른 문서에서 따로 정의합니다.

## 0.6.5. 타깃 명세

모든 타깃은 타깃 명세 하나로 기술합니다. 컴파일러, 링커, 런타임 빌드는 타깃에 관해 명세 말고는 아무것도 읽지 않습니다. 타깃은 `--target`으로 지정하며, 값은 내장 이름 (`rokc --print target-list`: 0.6.1의 Tier 1 타깃과 0.6.2의 타깃), 또는 ROKON 값 `TargetSpec { .. }`을 담은 파일 `<name>.rokon`의 경로(custom 타깃, 이름은 파일의 stem)입니다.

`rokc --print target-spec [--target=<t>]`은 타깃 명세를 이 형태로 출력하므로, custom 타깃은 보통 내장 타깃을 복사해서 시작합니다. custom 파일은 모든 필드를 적어야 하며, 모르는 필드는 에러입니다. custom 타깃의 런타임과 C 라이브러리(타깃이 함께 제공하는 경우)는 toolchain의 `lib/rok/targets/<name>/`에서 가져오며, 이 디렉토리가 없으면 링크하는 빌드는 그 디렉토리 이름을 밝히며 실패합니다.

| 필드 | 의미 |
|------|------|
| `llvm_target`, `deployment_target` | LLVM triple. 그 뒤에 붙는 최소 OS 버전 (Mach-O, `MACOSX_DEPLOYMENT_TARGET`이 덮음) |
| `arch`, `os`, `env`, `family`, `endian`, `pointer_width` | `target_*` cfg 값 ([6.3.2](../006-meta-system/603-cfg.md)). 비어 있는 `env` / `family`는 키를 두지 않음 |
| `vendor` | 정보용 |
| `wide_align`, `max_atomic_width`, `c_abi`, `data_layout` | `i64` / `u64` / `f64` 정렬, 가장 넓은 atomic, C 호출 규약 ([8.6.3](../008-abi/806-calling-convention.md)), LLVM data layout |
| `cpu`, `features` | 기본 CPU와 feature (0.6.4의 SIMD 기준선) |
| `object_format`, `linker`, `pre_link_args`, `crt_objects`, `late_link_args` | object 형식과 lld flavor, 타깃 자체의 링커 인자, C 런타임 object, 입력 뒤의 인자나 파일 |
| `dynamic_linking`, `dynamic_loader`, `libc` | 공유 라이브러리 지원, ELF interpreter, C 라이브러리와 그것이 toolchain과 함께 오는지 |
| `rust_target` | 런타임을 빌드하는 Rust 타깃 |
| `exe_suffix`, `staticlib_prefix` / `staticlib_suffix`, `dylib_prefix` / `dylib_suffix`, `c_symbol_prefix` | 파일과 심볼 이름 |
| `stack_size`, `stack_probes`, `threads` | main 스레드의 스택, stack probing, 스레드 지원 |
| `native_backend`, `repl`, `profile_runtime` | Rok 자신의 `-O0` backend, REPL host, `-C profile-generate`의 런타임 |

명세에는 안정된 해시가 있으며, 모든 빌드 cache는 이 해시를 키로 씁니다.
