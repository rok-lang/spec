# 0.6. 플랫폼 타깃

rok 은 LLVM 과 clang 이 코드 생성을 지원하는 플랫폼이라면 어디에서나 빌드된다는 원칙을 따릅니다. 다만 실제로 동작이 보증되는 플랫폼은 한정된 목록(Tier 1)으로 정해 두며, 그 외의 플랫폼은 필요에 따라 사안별로 검토합니다.

## 0.6.1. Tier 1: 정식 지원

다음 13 개의 타깃은 모든 PR 에 대해 CI 가 실행되며, 빌드와 테스트의 통과가 의무로 요구됩니다.

### 64-bit 데스크탑 / 서버

이 영역은 Linux 와 macOS, Windows 의 주요 아키텍처를 모두 포괄하며, 정적 바이너리를 위한 musl 변형까지 포함합니다.

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

### WebAssembly

다음으로 WebAssembly 계열을 봅니다. 브라우저 기반 실행과 서버사이드 실행을 모두 염두에 두고 두 가지 타깃을 함께 지원합니다.

| 타깃                     | 비고                             |
|--------------------------|----------------------------------|
| `wasm32-unknown-unknown` | 브라우저 환경                    |
| `wasm32-wasi`            | 서버사이드 WASM                  |

### 32-bit

마지막으로 32-bit 영역인데, 여전히 활발하게 사용되는 임베디드 및 레거시 환경을 위해 다음 세 타깃을 정식으로 지원합니다.

| 타깃                            | 비고                                 |
|---------------------------------|--------------------------------------|
| `i686-unknown-linux-gnu`        | 32-bit x86 Linux                     |
| `i686-pc-windows-msvc`          | 32-bit Windows                       |
| `armv7-unknown-linux-gnueabihf` | 32-bit ARM Linux (Raspberry Pi 4 등) |

이렇게 총 13 개의 타깃을 통해 과학·데이터·HPC·Jupyter·임베디드 분야의 사용자 대부분을 직접 커버합니다.

## 0.6.2. Tier 1 외 타깃: 보증 없음

다음 타깃들은 Tier 1 에 속하지 않으며, 본 사양은 이들에 대한 동작을 보증하지 않습니다. 구체적으로는 RISC-V 서버용 `riscv64gc-unknown-linux-gnu`, IBM Power HPC 환경의 `powerpc64le-unknown-linux-gnu`, FreeBSD 용 `x86_64-unknown-freebsd`, RISC-V 임베디드용 `riscv32imac-unknown-none`, 그리고 IBM Z 메인프레임을 위한 `s390x-unknown-linux-gnu` 가 여기에 해당합니다.

## 0.6.3. 32-bit 지원의 함의

32-bit 타깃이 Tier 1 에 포함되어 있는 만큼, 사양 전반은 32-bit 와 64-bit 양쪽 환경에서 모두 동작하도록 규정됩니다. 우선 객체 헤더 레이아웃은 두 비트 폭에서 동일하게 유지되며, 64-bit 정수에 대한 atomic 연산은 32-bit 환경에서 시스템의 atomic 라이브러리에 의존하여 에뮬레이트됩니다. 또한 `usize` 의 비트 폭은 타깃의 기본 포인터 폭에 자동으로 종속되며, 가비지 컬렉터가 활용할 수 있는 힙 상한은 32-bit 에서 약 3.5 GB, 64-bit 에서는 사실상 무제한으로 사용할 수 있습니다.

## 0.6.4. SIMD

각 Tier 1 타깃에서는 다음과 같은 SIMD 명령 세트가 자동으로 감지되며 기본적으로 활성화됩니다.

| 타깃      | SIMD                |
|-----------|---------------------|
| `x86_64`  | SSE / AVX / AVX-512 |
| `aarch64` | NEON / SVE          |
| `i686`    | SSE2 (baseline)     |
| `armv7`   | NEON                |
| `wasm32`  | WASM SIMD128        |

다만 SIMD 의 라이브러리 표면 자체는 본 사양의 다른 문서에서 별도로 정의하고 있으니, 그쪽을 함께 참고합니다.
