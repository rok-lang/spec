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

이 분리는 rok 이 MLIR 자체의 dialect-definition DSL 진화 (TableGen → IRDL → 차세대 형식) 에 종속되지 않도록 해 줍니다.

## 1.10.2. plugin 빌드 형태

plugin 은 다음 심볼을 노출하는 동적 라이브러리입니다.

```c
// C ABI — plugin 의 진입점
extern "C" const RokMlirPluginV1* rok_mlir_plugin_v1(void);
```

반환되는 구조체 형식은 다음과 같이 정의됩니다.

```c
struct RokMlirPluginV1 {
    uint32_t  abi_version;        // 본 사양의 plugin ABI 버전 — 현재 1
    const char* dialect_namespace; // 예: "mydialect"
    const char* compatible_llvm;   // 예: "20"
    void (*register_dialect)(MlirContext ctx);
    void (*register_passes)(MlirPassManager pm);  // 선택, NULL 가능
};
```

각 필드의 책임:
- **`abi_version`**: rok plugin ABI 버전. 본 사양은 v1 만 정의합니다.
- **`dialect_namespace`**: `__mlir_op::<"NS.op_name", ...>` 의 `NS` 자리에 들어갈 namespace.
- **`compatible_llvm`**: 이 plugin 이 빌드된 LLVM major 버전. `rokc` 가 자기 LLVM 버전과 비교하여 mismatch 면 거부.
- **`register_dialect`**: `MlirContext` 에 dialect 를 등록하는 callback. plugin 작성자가 MLIR C API 를 통해 작성.
- **`register_passes`**: 다이얼렉트 lowering / 변환 패스를 PassManager 에 등록하는 callback. NULL 이면 패스 등록 단계가 건너뛰어집니다.

## 1.10.3. plugin 로드

`rokc` 가 plugin 을 로드하는 방식은 두 가지입니다.

### CLI 옵션

```bash
rokc build --load-mlir-plugin=/path/to/libmydialect.so main.rok
rokc build --load-mlir-plugin=mydialect main.rok    # 검색 경로에서 자동 탐색
```

여러 번 사용 가능합니다.

```bash
rokc build \
    --load-mlir-plugin=/path/to/libcrypto_dialect.so \
    --load-mlir-plugin=/path/to/libsignal_dialect.so \
    main.rok
```

### 패키지 매니페스트 선언

패키지의 매니페스트 (e.g., `rok.toml`) 에 `mlir_plugin` 의존성을 선언하면 빌드 시 자동 로드됩니다. 매니페스트 형식 자체는 패키지 매니저 사양의 영역이며 본 사양 외부입니다.

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

[1.9 rok::bare::mlir](109-mlir.md) 의 모든 호출 syntax (속성 / region / variadic) 가 그대로 적용됩니다. `rokc` 는 `crypto` namespace 가 등록되어 있는지 확인한 뒤 op 이름과 시그니처를 plugin 에 위임하여 검증합니다.

## 1.10.5. 안전 책임

plugin 사용자 (라이브러리 작성자 + 그 라이브러리를 쓰는 사용자) 가 부담하는 책임은 다음과 같습니다.

- **LLVM 버전 일치**: plugin 이 빌드된 LLVM major 버전이 `rokc` 와 다르면 로드 거부됩니다. patch 버전 차이는 허용.
- **ABI 안정성**: `RokMlirPluginV1` 의 필드 추가 / 변경은 본 사양의 SemVer 정책을 따릅니다. 다음 ABI 버전이 나오면 v1 / v2 두 entry point 가 공존할 수 있습니다.
- **op 의미 검증**: rok 컴파일러는 op 이름이 plugin 에 등록되어 있는지만 확인합니다. op 의 의미 / 부수효과 / GC 안전성은 plugin 작성자 책임입니다.
- **GC root 추적**: plugin op 이 GC heap 객체를 다룰 때는 `llvm.gcroot` 를 통한 root 추적을 plugin lowering 에서 명시해야 하며, 그렇지 않으면 GC cycle 에서 객체가 회수될 수 있습니다.
- **target 가용성**: plugin 이 특정 target (GPU / FPGA / 양자 hw) 만 지원하는 경우, 해당 target 이 활성화되지 않은 빌드에서 호출하면 lowering 단계에서 에러가 납니다.

## 1.10.6. ABI 버전 정책

`RokMlirPluginV1` 의 `abi_version` 필드는 plugin 이 따르는 ABI 의 메이저 버전을 명시합니다. 정책은 다음과 같이 정리됩니다.

- **거부 조건**: 컴파일러는 로드 시점에 `abi_version` 을 확인하여 자신이 알고 있는 버전 집합과 매칭하지 않으면 거부합니다. 메시지는 "plugin ABI version N is not supported by this rokc (supports: ..)" 형태로 표준화됩니다.
- **새 버전 신설 기준**: 다음 중 하나라도 발생하면 메이저 버전이 올라갑니다.
  - 기존 함수 포인터의 시그니처 변경
  - 기존 필드 의미 변경 또는 제거
  - GC root 추적 메커니즘 등 안전 책임 모델 변경
- **하위 호환 추가**: 새 버전이 등장하더라도 한 메이저 사이클 동안은 직전 버전 (v1) 의 entry point 를 함께 노출하여 plugin 이 점진적으로 전환할 시간을 확보합니다. 두 entry point 가 공존할 때는 plugin 이 둘 중 하나만 채워 두면 충분합니다.
- **deprecation 경로**: 한 메이저 사이클 (보통 한 LLVM 메이저 cycle 정도) 이 지나면 직전 버전 entry point 는 제거됩니다. 컴파일러 release note 에 명시되며, 사용자는 plugin 을 재빌드해야 합니다.
- **patch / minor 변경**: 기존 함수 포인터 추가 (필드 끝에 append) 는 minor 변경으로 처리되며 `abi_version` 은 그대로 둡니다. plugin 이 모르는 새 필드는 컴파일러가 옵션으로 취급합니다.

본 정책은 `rok::bare` tier 의 일반 안정성 정책 ([0.5 표준 라이브러리 계층](../../refs-lang/000-meta/005-standard-libraries-tiers.md)) 의 특수 형태이며, bare tier 가 stable 하지 않다는 일반 원칙은 그대로 유지됩니다.

## 1.10.7. 빌드 격리

plugin 으로 등록된 다이얼렉트는 **그 빌드에서만 활성화** 됩니다. 일반 사용자가 작성하는 rok 코드의 표면 의미는 plugin 유무와 무관합니다.

```rok
// crypto plugin 없이 빌드되면 컴파일 에러
unsafe {
    __mlir_op::<"crypto.aes_round", _>(...)
}
// 에러: "unknown MLIR op: crypto.aes_round
//        (plugin 'crypto' not loaded)"
```

이 격리는 plugin 다이얼렉트가 rok 언어 자체의 의미에 침투하지 못하도록 하는 장치입니다. 일반 사용자 코드의 의미는 항상 본 사양만으로 결정됩니다.

## 1.10.8. 다른 언어와의 비교

| 언어       | plugin dialect 메커니즘                             | 위치       |
|------------|-----------------------------------------------------|------------|
| Mojo       | 컴파일러 빌드 시점 통합 (외부 plugin 로드 미공식화) | 빌드       |
| MLIR C API | `mlirContextLoadAllAvailableDialects` 등            | 라이브러리 |
| LLVM       | `-load=plugin.so` (legacy pass manager)             | CLI        |
| **rok**    | **`--load-mlir-plugin=...` + entry point ABI**      | spec 정의  |

rok 의 모델은 LLVM 의 pass plugin 과 가장 닮았습니다. 동적 로드 + 명시 entry point + ABI 버전 일치 검사로 안전하게 확장합니다.

## 1.10.9. 미지원 / 향후 영역

다음은 본 사양 범위 밖입니다.

- rok 소스 코드 안에서 다이얼렉트 / op 정의 (Mojo 도 안 함)
- 동일 namespace 의 plugin 두 개를 같이 로드 (충돌 — `rokc` 가 거부)
- 런타임 (실행 시점) 에 다이얼렉트 추가
- plugin 의 cross-version 호환 (v1 → v2 자동 마이그레이션)

이 영역들이 필요해지면 본 사양을 v1 → v2 로 bump 하는 시점에 검토합니다.
