# 1.10. `rok::bare::mlir::plugin`: 사용자 정의 다이얼렉트 등록

`rok::bare::mlir::plugin`은 Rok 컴파일러가 외부에서 빌드한 MLIR 다이얼렉트를 동적으로 로드해 [`__mlir_op`](109-mlir.md)의 호출 대상으로 추가할 수 있게 하는 인터페이스를 정의합니다. 이 장은 도메인 특화 컴파일러(암호화, 신호 처리, 형식 검증, 새 가속기 ISA 등)를 Rok 위에 얹으려는 라이브러리 작성자를 위한 것입니다.

이 절은 플러그인의 **등록 메커니즘과 안전 책임**을 정의합니다. 다이얼렉트 자체의 정의(op / type / attribute의 의미와 스키마)는 MLIR의 영역(TableGen / IRDL)입니다. 이 사양의 범위 밖입니다.

## 1.10.1. 영역 분리: 사양 안과 밖

| 책임                                                     | 위치                                       |
|----------------------------------------------------------|--------------------------------------------|
| 다이얼렉트 정의 (op 스키마, type 시스템)                 | **사양 밖**: MLIR (TableGen `.td` / IRDL) |
| 다이얼렉트 빌드 (`.so` / `.dylib` / `.dll`)              | **사양 밖**: 사용자 빌드 도구             |
| Rok 컴파일러가 플러그인을 로드 / 등록하는 인터페이스   | **사양 안** (이 절)                        |
| 로드된 플러그인 op을 `__mlir_op`으로 호출              | **사양 안** ([1.9](109-mlir.md))           |
| 플러그인 op의 lowering / 패스                               | **사양 밖**: 플러그인 자체가 책임         |

이렇게 나누면 MLIR의 다이얼렉트 정의 DSL이 바뀌어도(TableGen → IRDL → 차세대 형식) Rok은 영향을 받지 않습니다.

## 1.10.2. plugin 빌드 형태

`rokc`는 MLIR을 내장하고([0.2.4](../../refs-lang/000-meta/002-pipeline.md#024-백엔드)) 플러그인을 자신의 MLIR context에 로드합니다. 플러그인은 MLIR의 표준 플러그인 형태를 따르는 동적 라이브러리입니다.

| 종류            | 진입점 (C ABI)                    |
|-----------------|-----------------------------------|
| 다이얼렉트      | `mlirGetDialectPluginInfo()`      |
| 패스 (선택)     | `mlirGetPassPluginInfo()`         |

다이얼렉트 플러그인의 namespace가 `__mlir_op::<"NS.op_name", ...>`의 `NS` 자리에 들어갑니다. 플러그인은 `rokc`에 내장된 LLVM 버전으로 빌드해야 합니다. 플러그인은 자기 MLIR을 따로 두지 않고 `rokc`의 MLIR을 씁니다. 플러그인의 MLIR / LLVM 심볼은 해소하지 않은 채 둡니다. 로드할 때 이 심볼은 `rokc`가 내보내는 심볼에 묶입니다. `rokc`는 자기가 링크하는 MLIR([refs-lang 8.7.1](../../refs-lang/008-abi/807-mlir-dialects.md#871-일반-코드-사용-다이얼렉트재확인)과 [8.7.5](../../refs-lang/008-abi/807-mlir-dialects.md#875-라이브러리--가속-영역-다이얼렉트)의 다이얼렉트와 그 다이얼렉트가 쓰는 기반)와 LLVM의 support 라이브러리를 내보냅니다. 플러그인은 이것을 기반으로 만들 수 있습니다. 그 밖의 것(예: 다른 upstream 다이얼렉트)은 플러그인이 직접 갖춰야 합니다. 자기 MLIR 사본을 링크한 플러그인은 `rokc`가 보지 못하는 registry에 등록됩니다. Windows에서 `rokc.exe`는 import library로 이 심볼을 내보냅니다. 이 import library는 toolchain의 MLIR 플러그인 개발 파일과 함께 배포됩니다. 플러그인 DLL은 이 import library에 링크하고, `rokc`와 같은 C 런타임(동적 release CRT)을 쓰고, 자기 진입점을 내보냅니다. DLL은 `rokc.exe`가 내보내는 데이터에 `__declspec(dllimport)`로만 닿을 수 있습니다. MLIR의 explicit TypeID는 플러그인 개발 헤더가 이 처리를 해 둡니다. 그 밖의 그런 데이터는 플러그인이 쓸 수 없습니다. 정확한 빌드 플래그는 구현이 정합니다.

## 1.10.3. plugin 로드

플러그인을 지정하는 방식은 두 가지입니다. 어느 쪽이든 `rokc`가 lowering 전에 로드합니다.

### CLI 옵션

```bash
rokc build --load-mlir-plugin /path/to/libmydialect.so main.rok
rokc build --load-mlir-plugin mydialect main.rok         # 이름으로 탐색
rokc build --load-mlir-pass-plugin mypasses --mlir-pass my-lower main.rok
```

| 옵션                                   | 효과                                            |
|----------------------------------------|-------------------------------------------------|
| `--load-mlir-plugin <path\|name>`      | 다이얼렉트 플러그인 로드                        |
| `--load-mlir-pass-plugin <path\|name>` | 패스 플러그인 로드                              |
| `--mlir-pass <pass>`                   | LLVM 다이얼렉트로 변환하기 전에 `<pass>` 실행   |

모두 여러 번 쓸 수 있습니다. 경로가 아닌 이름 `name`은 `ROK_MLIR_PLUGIN_PATH`의 디렉터리에서 찾습니다. 찾는 파일 이름은 호스트의 라이브러리 파일 이름입니다. macOS는 `lib<name>.dylib` 또는 `lib<name>.so`, Linux는 `lib<name>.so`, Windows는 `<name>.dll`입니다. 탐색 세부는 구현이 정합니다. 플러그인을 로드하지 못하면 빌드 에러입니다. 플러그인 파일은 증분 빌드의 fingerprint에 들어갑니다([refs-lang 9.2.16](../../refs-lang/009-tools/902-aot-pipeline.md#9216-증분-빌드)). `rok`는 이 옵션을 `rokc`에 그대로 넘깁니다.

### 패키지 매니페스트 선언

패키지 매니페스트(`rok.rokon`, [refs-lang 9.10](../../refs-lang/009-tools/910-rok-frontend.md))에 플러그인을 선언하면 빌드할 때 자동으로 로드됩니다. 매니페스트 형식은 [refs-lang 9.10.3](../../refs-lang/009-tools/910-rok-frontend.md#9103-rokrokon--roklock)이 정의합니다. 플러그인을 선언하는 필드는 아직 그 형식에 없습니다. 정해지면 9.10.3에 추가합니다.

## 1.10.4. plugin op 호출

로드된 플러그인의 op은 upstream MLIR op과 같은 문법으로 호출합니다.

```rok
use rok::bare::mlir::__mlir_op;

unsafe {
    // plugin namespace 가 "crypto" 라고 가정
    let cipher: Vector<u8, 16> = __mlir_op::<"crypto.aes_round", _>(
        plaintext, key,
        attributes = { "round_index": __mlir_attr::<"3 : i32"> },
    );
}
```

[1.9 rok::bare::mlir](109-mlir.md)의 모든 호출 문법(속성 / region / variadic)이 그대로 적용됩니다. op 이름과 시그니처는 플러그인을 로드한 MLIR context가 검증합니다.

## 1.10.5. 안전 책임

플러그인 사용자(라이브러리 작성자 + 그 라이브러리를 쓰는 사용자)의 책임은 다음과 같습니다.

- **LLVM 버전 일치**: 플러그인은 `rokc`에 내장된 LLVM 버전으로 빌드해야 하고, `rokc`가 내보내는 MLIR / LLVM 심볼에 묶여야 합니다(1.10.2). 그렇지 않으면 로드가 실패합니다.
- **op 의미 검증**: op 이름과 시그니처는 MLIR이 확인합니다. op의 의미 / 부수 효과 / GC 안전성은 플러그인 작성자 책임입니다.
- **GC 루트 추적**: 플러그인 op이 GC 힙 객체를 다룰 때는 루트 추적을 지켜야 합니다. 그 포인터를 shadow stack 프레임 슬롯에 두는 방식입니다([refs-lang 7.4.1](../../refs-lang/007-runtime/704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-내보내는-shadow-stack)). 이 규칙은 플러그인의 lowering 단계가 지킵니다. 그렇지 않으면 GC 주기에서 객체가 회수될 수 있습니다.
- **타깃 가용성**: 플러그인이 특정 타깃(GPU / FPGA / 양자 하드웨어)만 지원할 수 있습니다. 그 타깃을 켜지 않은 빌드에서 호출하면 lowering 단계에서 에러가 납니다.

## 1.10.6. ABI 버전 정책

플러그인 ABI는 Rok이 정하지 않고 MLIR의 플러그인 API(`MLIR_PLUGIN_API_VERSION`)를 따릅니다. 호환성은 MLIR이 판정합니다. 내장 LLVM 버전이 바뀌면 플러그인을 다시 빌드해야 합니다. 이 정책은 `rok::bare` tier가 안정적이지 않다는 일반 원칙([0.5 표준 라이브러리 계층](../../refs-lang/000-meta/005-standard-libraries-tiers.md))과 맞습니다.

## 1.10.7. 빌드 격리

플러그인으로 등록된 다이얼렉트는 **그 빌드에서만 활성화**됩니다. 일반 사용자가 작성하는 Rok 코드의 표면 의미는 플러그인 유무와 무관합니다.

```rok
// crypto plugin 없이 빌드되면 빌드 에러 (MLIR 이 op 을 모름)
unsafe {
    __mlir_op::<"crypto.aes_round", _>(...)
}
```

`rokc`는 플러그인의 namespace를 알 수 없으므로 op 이름 검사는 MLIR이 합니다. 그 진단은 빌드 에러로 보고됩니다.

이 격리 덕분에 플러그인 다이얼렉트는 Rok 언어 자체의 의미에 끼어들 수 없습니다. 일반 사용자 코드의 의미는 항상 이 사양만으로 결정됩니다.

## 1.10.8. 다른 언어와 비교

| 언어       | 플러그인 다이얼렉트 메커니즘                        | 위치       |
|------------|-----------------------------------------------------|------------|
| Mojo       | 컴파일러 빌드 시점 통합 (외부 플러그인 로드는 비공식) | 빌드       |
| MLIR C API | `mlirContextLoadAllAvailableDialects` 등            | 라이브러리 |
| LLVM       | `-load=plugin.so` (legacy pass manager)             | CLI        |
| **Rok**    | **`--load-mlir-plugin ...` → `rokc`의 MLIR에 로드** | CLI      |

Rok은 자체 플러그인 ABI를 두지 않고 MLIR의 플러그인 형태를 그대로 씁니다.

## 1.10.9. 범위 밖

다음은 이 사양의 범위 밖입니다.

- Rok 소스 코드 안에서 다이얼렉트 / op 정의 (Mojo도 하지 않음)
- 같은 namespace의 플러그인 두 개를 함께 로드 (충돌, MLIR이 거부)
- 런타임(실행 시점)에 다이얼렉트 추가
- 플러그인의 LLVM 버전 간 호환 (MLIR 플러그인 API를 따름)
