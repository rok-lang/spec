# 0.6. 플랫폼 타깃

nok 은 LLVM 과 clang 이 코드 생성을 지원하는 플랫폼이면 빌드된다는 원칙을 따른다. 다만
실제로 보증하는 플랫폼은 한정된 목록(Tier 1)이며, 그 외 플랫폼은 필요에 따라 검토한다.

## 0.6.1. Tier 1: 정식 지원

다음 13 개 타깃은 모든 PR 에 CI 가 실행되며, 빌드와 테스트의 통과가 의무다.

### 64-bit 데스크탑 / 서버

| 타깃                         | 비고                                       |
| ---------------------------- | ------------------------------------------ |
| `x86_64-unknown-linux-gnu`   | Linux 서버 / 데스크탑 주력                 |
| `x86_64-apple-darwin`        | macOS Intel                                |
| `aarch64-apple-darwin`       | macOS Apple Silicon                        |
| `aarch64-unknown-linux-gnu`  | ARM 서버 (AWS Graviton, Raspberry Pi 5 등) |
| `x86_64-pc-windows-msvc`     | Windows 데스크탑                           |
| `aarch64-pc-windows-msvc`    | Windows ARM                                |
| `x86_64-unknown-linux-musl`  | Alpine / `static` binary                   |
| `aarch64-unknown-linux-musl` | 동일, ARM                                  |

### WebAssembly

| 타깃                     | 비고                             |
| ------------------------ | -------------------------------- |
| `wasm32-unknown-unknown` | 브라우저 환경 (JupyterLite 커널) |
| `wasm32-wasi`            | 서버사이드 WASM                  |

### 32-bit

| 타깃                            | 비고                                 |
| ------------------------------- | ------------------------------------ |
| `i686-unknown-linux-gnu`        | 32-bit x86 Linux                     |
| `i686-pc-windows-msvc`          | 32-bit Windows                       |
| `armv7-unknown-linux-gnueabihf` | 32-bit ARM Linux (Raspberry Pi 4 등) |

총 13 개. 과학, 데이터, HPC, Jupyter, 임베디드 사용자 대부분을 직접 커버한다.

## 0.6.2. 향후 검토: 보증 없음

다음은 시장 성숙도와 사용자 요청에 따라 Tier 1 합류를 검토한다. 현재는 동작 보증이 없다.

- `riscv64gc-unknown-linux-gnu` (RISC-V 서버)
- `powerpc64le-unknown-linux-gnu` (IBM Power HPC)
- `x86_64-unknown-freebsd`
- `riscv32imac-unknown-none` (RISC-V 임베디드)
- `s390x-unknown-linux-gnu` (IBM Z)

## 0.6.3. 32-bit 지원의 함의

32-bit 타깃이 Tier 1 에 포함되므로, 사양 전반은 32-bit 와 64-bit 양쪽에서 동작하도록 규정된다.

- 객체 헤더 레이아웃이 두 비트 폭에서 동일하다.
- 64-bit 정수의 atomic 연산은 32-bit 환경에서 emulated 된다 (시스템 atomic 라이브러리 의존).
- `usize` 의 비트 폭은 타깃 기본 포인터 폭에 자동 종속된다.
- 가비지 컬렉터의 가용 힙 상한은 32-bit 에서 약 3.5 GB, 64-bit 에서 사실상 무제한이다.

## 0.6.4. SIMD

각 Tier 1 타깃에서 자동 감지·기본 활성화되는 SIMD 명령 세트.

| 타깃      | SIMD                |
| --------- | ------------------- |
| `x86_64`  | SSE / AVX / AVX-512 |
| `aarch64` | NEON / SVE          |
| `i686`    | SSE2 (baseline)     |
| `armv7`   | NEON                |
| `wasm32`  | WASM SIMD128        |

SIMD 의 라이브러리 표면은 본 사양의 다른 문서에서 정의한다.
