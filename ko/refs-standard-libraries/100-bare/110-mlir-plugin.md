# 1.10. `rok::bare::mlir::plugin` — 사용자 정의 다이얼렉트 등록

`rok::bare::mlir::plugin` 은 rok 컴파일러가 외부에서 빌드된 MLIR 다이얼렉트를 동적으로 로드하여 [`__mlir_op`](109-mlir.md) 의 호출 대상으로 추가할 수 있도록 하는 인터페이스를 정의합니다. 도메인 특화 컴파일러 (암호화, 신호 처리, 형식 검증, 새 가속기 ISA 등) 를 rok 위에 얹고 싶은 라이브러리 작성자를 위한 자리입니다.

본 절은 plugin 의 **등록 메커니즘과 안전 책임** 을 정의합니다. 다이얼렉트 자체의 정의 (op / type / attribute 의 의미와 스키마) 는 MLIR 의 영역 (TableGen / IRDL) 이며 본 사양 범위 밖입니다.

## 1.10.1. 영역 분리 — 무엇이 spec 안 / 밖인가

| 책임                                                     | 위치                                       |
|----------------------------------------------------------|--------------------------------------------|
| 다이얼렉트 정의 (op 스키마, type 시스템)                 | **spec 밖** — MLIR (TableGen `.td` / IRDL) |
| 다이얼렉트 빌드 (`.so` / `.dylib` / `.dll`)              | **spec 밖** — 사용자 빌드 도구             |
| rok 컴파일러가 plugin 을 로드 / register 하는 인터페이스 | **spec 안** (본 절)                        |
| 로드된 plugin op 을 `__mlir_op` 으로 호출                | **spec 안** ([1.9](109-mlir.md))           |
| plugin op 의 lowering / 패스                             | **spec 밖** — plugin 자체가 책임           |

이 분리는 rok 이 MLIR 자체의 dialect-definition DSL 진화 (TableGen → IRDL → 차세대 형식) 에 종속되지 않도록 합니다.

## 1.10.2. plugin 빌드 형태

`rokc` 는 MLIR 라이브러리를 링크하지 않고 MLIR 을 텍스트로 냅니다 ([0.2.4](../../refs-lang/000-meta/002-pipeline.md#024-mlir-텍스트-직접-생성)). 그래서 plugin 은 `rokc` 가 아니라 `mlir-opt` 가 로드합니다. plugin 은 MLIR 의 표준 plugin 형태를 따르는 동적 라이브러리입니다.

| 종류            | 진입점 (C ABI)                    | `mlir-opt` 에 넘기는 옵션         |
|-----------------|-----------------------------------|-----------------------------------|
| 다이얼렉트      | `mlirGetDialectPluginInfo()`      | `--load-dialect-plugin=<path>`    |
| 패스 (선택)     | `mlirGetPassPluginInfo()`         | `--load-pass-plugin=<path>`       |

다이얼렉트 plugin 의 namespace 가 `__mlir_op::<"NS.op_name", ...>` 의 `NS` 자리에 들어갑니다. plugin 은 `mlir-opt` 와 같은 LLVM major 버전으로 빌드해야 합니다.

## 1.10.3. plugin 로드

plugin 을 지정하는 방식은 두 가지입니다. 어느 쪽이든 `rokc` 는 경로를 그대로 `mlir-opt` 에 넘깁니다.

### CLI 옵션

```bash
rokc build --load-mlir-plugin /path/to/libmydialect.so main.rok
rokc build --load-mlir-plugin mydialect main.rok         # 이름으로 탐색
rokc build --load-mlir-pass-plugin mypasses --mlir-pass my-lower main.rok
```

| 옵션                                   | `mlir-opt` 에 넘기는 것                         |
|----------------------------------------|-------------------------------------------------|
| `--load-mlir-plugin <path\|name>`      | `--load-dialect-plugin=<path>`                  |
| `--load-mlir-pass-plugin <path\|name>` | `--load-pass-plugin=<path>`                     |
| `--mlir-pass <pass>`                   | `--<pass>` (LLVM 다이얼렉트로 변환하기 전에)     |

모두 여러 번 쓸 수 있습니다. 경로가 아닌 이름 `name` 은 `ROK_MLIR_PLUGIN_PATH` 의 디렉토리에서 `lib<name>.dylib` / `lib<name>.so` / `<name>.dll` 을 찾습니다 (탐색 세부는 구현 정의). `mlir-opt` 가 plugin 을 로드하지 못하면 빌드 에러입니다. plugin 파일은 증분 빌드의 fingerprint 에 들어가며 ([refs-lang 9.2.16](../../refs-lang/009-tools/902-aot-pipeline.md#9216-증분-빌드)), `rok` 는 이 옵션들을 `rokc` 에 그대로 넘깁니다.

### 패키지 매니페스트 선언

패키지의 매니페스트 (`rok.rokon`, [refs-lang 9.10](../../refs-lang/009-tools/910-rok-frontend.md)) 에 plugin 을 선언하면 빌드 시 자동 로드됩니다. 매니페스트 형식은 [refs-lang 9.10.3](../../refs-lang/009-tools/910-rok-frontend.md#9103-rokrokon--roklock) 이 정의합니다. plugin 을 선언하는 필드는 아직 그 형식에 없으며, 정해지면 9.10.3 에 추가됩니다.

## 1.10.4. plugin op 호출

로드된 plugin 의 op 은 upstream MLIR op 과 동일한 syntax 로 호출됩니다.

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

[1.9 rok::bare::mlir](109-mlir.md) 의 모든 호출 syntax (속성 / region / variadic) 가 그대로 적용됩니다. op 이름과 시그니처의 검증은 plugin 을 로드한 `mlir-opt` 가 합니다.

## 1.10.5. 안전 책임

plugin 사용자 (라이브러리 작성자 + 그 라이브러리를 쓰는 사용자) 가 부담하는 책임은 다음과 같습니다.

- **LLVM 버전 일치**: plugin 이 빌드된 LLVM major 버전이 `mlir-opt` 와 다르면 `mlir-opt` 가 로드를 거부합니다.
- **op 의미 검증**: op 이름과 시그니처는 `mlir-opt` 가 확인합니다. op 의 의미 / 부수효과 / GC 안전성은 plugin 작성자 책임입니다.
- **GC root 추적**: plugin op 이 GC heap 객체를 다룰 때는 그 포인터를 shadow stack frame 슬롯에 두는 root 추적 ([refs-lang 7.4.1](../../refs-lang/007-runtime/704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-emit-하는-shadow-stack)) 을 plugin lowering 에서 지켜야 하며, 그렇지 않으면 GC cycle 에서 객체가 회수될 수 있습니다.
- **target 가용성**: plugin 이 특정 target (GPU / FPGA / 양자 hw) 만 지원하는 경우, 해당 target 이 활성화되지 않은 빌드에서 호출하면 lowering 단계에서 에러가 납니다.

## 1.10.6. ABI 버전 정책

plugin ABI 는 rok 가 정하지 않고 MLIR 의 plugin API (`MLIR_PLUGIN_API_VERSION`) 를 따릅니다. 호환성 판정은 `mlir-opt` 가 하며, LLVM 메이저 버전이 바뀌면 plugin 을 다시 빌드해야 합니다. 이는 `rok::bare` tier 가 stable 하지 않다는 일반 원칙 ([0.5 표준 라이브러리 계층](../../refs-lang/000-meta/005-standard-libraries-tiers.md)) 과 맞습니다.

## 1.10.7. 빌드 격리

plugin 으로 등록된 다이얼렉트는 **그 빌드에서만 활성화** 됩니다. 일반 사용자가 작성하는 rok 코드의 표면 의미는 plugin 유무와 무관합니다.

```rok
// crypto plugin 없이 빌드되면 빌드 에러 (mlir-opt 가 op 을 모름)
unsafe {
    __mlir_op::<"crypto.aes_round", _>(...)
}
```

`rokc` 는 plugin 의 namespace 를 알 수 없으므로 op 이름 검사는 `mlir-opt` 가 하며, 그 진단이 빌드 에러로 보고됩니다.

이 격리는 plugin 다이얼렉트가 rok 언어 자체의 의미에 침투하지 못하도록 하는 장치입니다. 일반 사용자 코드의 의미는 항상 본 사양만으로 결정됩니다.

## 1.10.8. 다른 언어와의 비교

| 언어       | plugin dialect 메커니즘                             | 위치       |
|------------|-----------------------------------------------------|------------|
| Mojo       | 컴파일러 빌드 시점 통합 (외부 plugin 로드 미공식화) | 빌드       |
| MLIR C API | `mlirContextLoadAllAvailableDialects` 등            | 라이브러리 |
| LLVM       | `-load=plugin.so` (legacy pass manager)             | CLI        |
| **rok**    | **`--load-mlir-plugin ...` → `mlir-opt` 에 전달**   | CLI        |

rok 은 자체 plugin ABI 를 두지 않고 MLIR 의 plugin 형태를 그대로 씁니다. `rokc` 는 MLIR 을 링크하지 않으므로 plugin 은 텍스트 MLIR 을 받는 `mlir-opt` 단계에서만 로드됩니다.

## 1.10.9. 범위 밖

다음은 본 사양 범위 밖입니다.

- rok 소스 코드 안에서 다이얼렉트 / op 정의 (Mojo 도 안 함)
- 동일 namespace 의 plugin 두 개를 같이 로드 (충돌 — `mlir-opt` 가 거부)
- 런타임 (실행 시점) 에 다이얼렉트 추가
- plugin 의 LLVM 버전 간 호환 (MLIR plugin API 를 따름)
