# Rok 표준 라이브러리 참조

이 문서는 Rok 표준 라이브러리의 형식 사양(formal specification)입니다. 언어 사양은 문법과 의미 메커니즘을 정의합니다. 표준 라이브러리 사양은 그 메커니즘이 실제로 동작하는 데 필요한 타입, 트레잇, 함수의 의미를 채웁니다. 연산자가 어떤 트레잇으로 풀리는지, 컬렉션에 어떤 인터페이스가 있는지, 비동기와 I/O가 어떤 보장 위에서 동작하는지를 한곳에서 정의합니다.

언어 사양은 [`refs-lang/`](../refs-lang/README.md)에 따로 있습니다. 이 문서는 사양만 다룹니다. 라이브러리의 실제 구현은 사양의 범위 밖입니다.

> [!IMPORTANT]
> **언어 사양 ↔ 표준 라이브러리 사양의 경계**
>
> [refs-lang](../refs-lang/README.md)은 **문법과 의미 메커니즘**을 정의합니다(예: `+`
> 토큰이 어떤 트레잇 메서드에 매핑되는지). [refs-standard-libraries](README.md)(이 문서)는
> **그 트레잇 자체의 시그니처와 의미**를 정의합니다(예: `Add::add`가 어떤 숫자 변환을
> 적용하는지).
>
> 두 사양 모두 같은 [표기법 규약](../refs-lang/000-meta/008-notation.md)을 따릅니다.

---

## 3계층 모델

표준 라이브러리는 하나의 평면이 아니라 세 층으로 나뉩니다. 시스템 코드는 거의 모두 안전한 추상 위에서 작성합니다. 하지만 그 추상을 구현하려면 어딘가에서는 원시 메모리와 FFI를 다뤄야 합니다. 또 파일이나 네트워크처럼 호스트 환경이 있어야 의미가 있는 기능은, 임베디드처럼 그런 환경이 없는 타깃에는 아예 있으면 안 됩니다. 이 차이를 의존성 그래프에 그대로 드러내려고 가상 네임스페이스 `rok::` 아래에 3계층 구조를 둡니다. `rok::bare` / `rok::core` / `rok::std`입니다.

```
rok::bare   (base, runtime extern "C" 만 의존)
  ↓
rok::core   (foundation, 안전, 모든 target; uses rok::bare internally)
  ↓
rok::std   (target-dep; uses rok::core + rok::bare)
```

`rok::bare`는 맨 아래에서 원시 기능만 노출합니다. `rok::core`는 그 위에 안전한 기반을 놓고, 모든 타깃에서 똑같이 동작합니다. `rok::std`는 `rok::core` 위에 호스트 의존 기능을 얹어, 보통의 응용 프로그램이 기대하는 표면을 완성합니다. 일상적인 코드는 `rok::core`만으로 충분합니다. OS 자원이 필요해지면 `rok::std`를 씁니다. 추상 경계를 직접 넘어야 할 때만 `rok::bare`를 명시적으로 가져옵니다. 자세한 의존성 규칙은 [0.1 tier 모델](000-meta/001-tier-model.md)에 있습니다.

| Tier       | 경로          | 디렉터리                        | 영역                                           |
|------------|---------------|---------------------------------|------------------------------------------------|
| `rok::bare` | `rok::bare::*` | [`100-bare/`](100-bare/README.md) | `Ptr`, `ConstPtr`, `transmute`, raw alloc, pin |
| `rok::core` | `rok::core::*` | [`200-core/`](200-core/README.md) | 숫자, 연산자, iter, 컬렉션, fmt, sync, task    |
| `rok::std` | `rok::std::*` | [`300-std/`](300-std/README.md) | fs, net, time, thread, io, env, process, path, crypto (random) |

---

## 장 목차

### 0. 메타: [000-meta/](000-meta/README.md)

라이브러리 전체에 걸친 정책과 표입니다.

- [0.1. 3계층 모델](000-meta/001-tier-model.md)
- [0.2. import 정책 (`use rok::core::**;` 묵시 적용)](000-meta/002-import.md)
- [0.3. intrinsic 메커니즘: 컴파일러 ↔ 표준 라이브러리 경계](000-meta/003-intrinsics.md)
- [0.4. `#[lang]` 슬롯 매핑 표](000-meta/004-lang-slot-table.md)
- [0.5. 표기법](000-meta/005-notation.md)

### 1. Unsafe: [100-bare/](100-bare/README.md)

저수준 / 위험 영역입니다. 명시적으로 가져와야 합니다.

- 1.1. [Pointer 계열 (`Ptr`/`ConstPtr`/`ManyPtr`/`CPtr` - Zig arity-encoded + Rust strict provenance)](100-bare/101-ptr.md)
- 1.2. [cast 연산 (`ptr_cast`/`align_cast`/`const_cast`/`reinterpret`)](100-bare/102-cast.md)
- 1.3. [`MaybeUninit<T>` + `Pod` + raw mem ops](100-bare/103-mem.md)
- 1.4. [raw allocation](100-bare/104-alloc.md)
- 1.5. [volatile 메모리 접근](100-bare/105-volatile.md)
- 1.6. [`Pinned<T>` - GC pin (Go 1.21 Pinner 모델)](100-bare/106-gc-pin.md)
- 1.7. [FFI - `extern "C"` / `CStr` / `CString`](100-bare/107-ffi.md)
- 1.8. [WASM Component Model (`ffi.wit`)](100-bare/108-wit.md)
- 1.9. [`rok::bare::mlir`: MLIR 다이얼렉트 직접 접근](100-bare/109-mlir.md)
- 1.10. [MLIR 다이얼렉트 plugin - 사용자 정의 다이얼렉트를 `rokc`에 로드](100-bare/110-mlir-plugin.md)

### 2. Rok: [200-core/](200-core/README.md)

안전 영역입니다. 모든 타깃에서 쓸 수 있습니다. `use rok::core::**;`가 묵시적으로 적용됩니다. 별도 prelude 모듈은 없습니다.

- 2.1. [숫자 타입 (`int`, `float`, auto-promote)](200-core/201-numbers.md)
- 2.2. [niche wrapper (`NonZero`, `NonNan`, `Finite`, `Positive`)](200-core/202-non-zero.md)
- 2.3. [산술 / 비트 트레잇 (`Add`, `Sub`, ..., `Shl`, `Shr`)](200-core/203-ops.md)
- 2.4. [복합 대입 트레잇 (`AddAssign`, ...)](200-core/204-ops-assign.md)
- 2.5. [비교 트레잇 (`PartialEq`, `Eq`, `Ord`, `PartialOrd`, `Hash`)](200-core/205-cmp.md)
- 2.6. [broadcast 트레잇 (`DotAdd`, `DotMul`, ...)](200-core/206-broadcast.md)
- 2.7. [iter 트레잇 (`Iterator`, `IntoIterator`)](200-core/207-iter.md)
- 2.8. [`Stream` (async iterator)](200-core/208-stream.md)
- 2.9. [컬렉션 (`List`, `Dict`, `Set`, `Tuple`, `Slice`)](200-core/209-collections.md)
- 2.10. [fmt (`Display`, `Debug`, `Formatter`)](200-core/210-fmt.md)
- 2.11. [변환 (`From`, `Into`, `TryFrom`, `saturating_into`, ...)](200-core/211-conversion.md)
- 2.12. [리플렉션 (`Reflect`, `Inspect`)](200-core/212-reflect.md)
- 2.13. [marker 트레잇 (`Clone`, `Copy`)](200-core/213-marker.md)
- 2.14. [에러 타입 (`TypeError`, `ValueError`, ...)](200-core/214-error.md)
- 2.15. [문자열 (`str` 가변 단일, `char`, `c_char`)](200-core/215-string.md)
- 2.16. [sync primitive (`Atomic`, `Mutex`, `RwLock`, `Cell`)](200-core/216-sync.md)
- 2.17. [`Channel`](200-core/217-channel.md)
- 2.18. [`task::` API](200-core/218-task.md)
- 2.19. [`par::` API](200-core/219-par.md)
- 2.20. [`Future` / `await` 통합](200-core/220-future.md)
- 2.21. [`crypto::` - 해시, MAC, AEAD, 키 교환, 서명](200-core/221-crypto.md)

> [!NOTE]
> 텐서 / 자동 미분 / SIMD / GPU 같은 과학 컴퓨팅 / AI 영역은 표준 라이브러리에 넣지 않고 외부 라이브러리로 나눕니다. 외부 라이브러리는 [`rok::bare::mlir`](100-bare/109-mlir.md) 모듈로 MLIR 다이얼렉트에 직접 접근할 수 있습니다.

### 3. Std: [300-std/](300-std/README.md)

타깃 의존 영역입니다. 명시적으로 가져옵니다.

- 3.1. [`fs::` - 파일 시스템 (atomic ops, Dir capability, reflink)](300-std/301-fs.md)
- 3.2. [`net::` - TCP/UDP/Unix + TLS 1.3 (HTTP 외부)](300-std/302-net.md)
- 3.3. [`time::` - `Instant` / `SystemTime` / `Duration` / `BootTime`](300-std/303-time.md)
- 3.4. [`thread::` - OS 스레드 (1:1, task와 별개)](300-std/304-thread.md)
- 3.5. [`io::` - `Reader` / `Writer` / stdin/out/err (동기만, async 트레잇 없음)](300-std/305-io.md)
- 3.6. [`env::` - 환경 변수 / args / 디렉터리](300-std/306-env.md)
- 3.7. [`process::` - `Command` / pidfd / kill_on_drop](300-std/307-process.md)
- 3.8. [`path::` - UTF-8 `Path` + `/` 연산자 (Python pathlib 식)](300-std/308-path.md)
- 3.9. [`rokon::` - ROKON 데이터 형식 (`.rokon`)](300-std/309-rokon.md)
- 3.10. [`crypto::`: OS 난수 원천](300-std/310-crypto.md)

---

## 읽는 방법

### 처음 읽을 때

처음이라면 [`refs-lang`](../refs-lang/README.md)의 0~6장으로 언어의 메커니즘을 먼저 이해한 뒤 이 문서를 읽기를 권장합니다. 이 문서는 refs-lang이 정의한 메커니즘에 의미를 부여합니다. 메커니즘을 모르고 읽으면 트레잇과 시그니처가 허공에 뜬 것처럼 느껴지기 쉽습니다. 두 문서를 나란히 놓고 오가며 읽는 편이 가장 좋습니다.

### 참조용

각 항목은 안정 절 번호(`<chapter>.<section>`)로 식별합니다. 이 번호는 내부 링크와 외부 참조에서 똑같이 씁니다. 그래서 파일을 옮겨도 같은 항목을 가리킵니다.

```
refs-standard-libraries 2.3        →  rok ops (Add/Sub/...)
refs-standard-libraries 1.1        →  unsafe Ptr<T>
refs-standard-libraries 3.5.2      →  std io (특정 하위 섹션)
```

### 검색

특정 트레잇이나 함수를 빨리 찾으려면 `ripgrep`으로 디렉터리를 좁혀 검색하는 방법이 가장 가볍습니다.

```bash
rg "trait Add" spec/ko/refs-standard-libraries/200-core/
rg "fn into" spec/ko/refs-standard-libraries/
```

---

## 디렉터리와 파일 이름

이 문서는 `refs-lang`과 다른 디렉터리 규칙을 따릅니다. 두 사양의 모양이 다르기 때문입니다. refs-lang은 0장부터 9장까지 모든 장의 권위가 같은 평면 구조입니다. refs-standard-libraries는 세 tier가 상호 참조의 핵심 정보 역할을 하는 계층 구조입니다. 이 구조 차이를 디렉터리 이름의 자릿수로 그대로 드러냅니다.

| 사양                    | prefix           | 이유                                                   |
|-------------------------|------------------|--------------------------------------------------------|
| refs-lang               | 한 자리 (`002-`) | 10개 장(0~9)이 모두 같은 권위, 평면 구조               |
| refs-standard-libraries | 세 자리 (`200-`) | 3 tier × N 모듈, tier가 상호 참조의 핵심 정보          |

구체적인 파일 이름 규칙은 다음과 같습니다. tier 디렉터리에는 100 단위 접두어가 붙습니다. 그 안의 모듈 파일은 tier 한 자리와 모듈 두 자리를 합친 세 자리 접두어로 시작합니다.

| 위치   | 형태                            | 예                                |
|--------|---------------------------------|-----------------------------------|
| Tier   | `<TNN>-<descriptive-name>/`     | `100-bare/`, `200-core/`            |
| 모듈   | `<tier><module>-<name>.md`      | `203-ops.md`, `211-conversion.md` |
|        | (tier 한 자리 + module 두 자리) |                                   |
| README | 각 tier / 장에 `README.md`      | TOC + 개요                        |

파일 경로에도 tier 정보가 들어 있습니다. 하지만 외부에서 이 문서를 인용할 때 정식 표기는 안정 절 번호(`2.3.1` 등)입니다. 파일 경로는 IDE와 GitHub에서 탐색하기 편하도록 쓰는 보조 표기일 뿐입니다.
