# 9.3. 패키지 아카이브 (.a) 포맷

이 장에서는 [AOT 빌드 파이프라인](902-aot-pipeline.md) 의 산출물인 `libfoo.a` 의 내부 구조를 정의합니다.

> [!NOTE]
> 현재 구현의 `libfoo.a` 는 concrete 코드의 `foo.o` 만 담습니다. RHIR 멤버 (`foo.hir.o`, [9.4](904-rhir-bytecode.md)), `rokc inspect`, 그리고 미리 빌드한 `.a` 를 의존 패키지로 쓰는 것 ([9.2.3](902-aot-pipeline.md#923-의존성-해소)) 은 아직 구현되지 않았습니다. 의존 패키지는 소스로 함께 컴파일됩니다.

## 9.3.1. 표준 ar archive

`libfoo.a` 는 표준 Unix `ar` 아카이브 형식을 그대로 따르고 있습니다. 따라서 `ar`, `ranlib`, `lld` 같은 표준 도구로 별도의 변환 없이 처리할 수 있습니다.

```
ar t libfoo.a
foo.concrete.o
foo.hir.o
foo.concrete.o.1     (큰 패키지의 경우 추가 TU)
...
```

## 9.3.2. 두 종류의 .o 파일

### `foo.concrete.o`: [concrete fn](../000-meta/007-glossary.md#concrete-fn--generic-fn) 의 native code

이 파일은 [AOT 빌드 파이프라인](902-aot-pipeline.md) 의 단계 12 결과로, MLIR 을 거쳐 컴파일된 native 코드를 담고 있습니다.

표준 ELF / Mach-O / COFF object 파일이며 다음과 같은 섹션들을 포함합니다.

| 섹션      | 의미                    |
|-----------|-------------------------|
| `.text`   | concrete `fn` 들의 코드 |
| `.rodata` | `const` / 리터럴        |
| `.data`   | `static` 의 초기값      |
| `.bss`    | 0 초기 `static`         |
| ...       | 표준 섹션들             |

여기에 더해 다음과 같은 특별 섹션들이 있습니다.

| 섹션         | 의미                                                                     |
|--------------|--------------------------------------------------------------------------|
| `.rok.scan`  | ScanTable 데이터 ([ScanTable 직렬화 포맷](../008-abi/802-scan-table.md)) |
| `.rok.types` | 타입 등록 정보 (`type_id → 타입 이름` 등)                                |
| `.rok.init`  | 패키지 초기화 함수 포인터                                                |

이 특별 섹션들은 링커가 표준 절차대로 처리하지만, 실제로는 rok 부팅 시점에 사용됩니다.

### `foo.hir.o`: 제네릭 `fn` 의 [RHIR](../000-meta/007-glossary.md#rhir-rok-hir-bytecode) bytecode

이 파일은 [AOT 빌드 파이프라인](902-aot-pipeline.md) 의 단계 6 ~ 7 결과 중 [generic](../000-meta/007-glossary.md#generic--제네릭) 함수의 [HIR](../000-meta/007-glossary.md#hir-high-level-ir--mir-mid-level-ir) 을 직렬화한 결과입니다.

표준 object 파일의 형태를 갖추고 있지만, 다음과 같은 커스텀 섹션만 포함합니다.

| 섹션        | 의미                                  |
|-------------|---------------------------------------|
| `.rok.hir`  | RHIR bytecode (제네릭 `fn` body)      |
| `.rok.meta` | 패키지 메타 (export 목록, 버전, 해시) |

`.text` 같은 코드 섹션은 비어 있습니다. 링커는 이 obj 를 심볼 충돌 없이 [archive](../000-meta/007-glossary.md#archive) 안에 그대로 두게 됩니다.

## 9.3.3. archive 구성 이유

### 표준 도구 호환

앞서 언급했듯 `libfoo.a` 는 표준 ar 아카이브이기 때문에 `ar`, `ranlib`, `nm`, `lld`, `gcc`/`clang` 이 모두 그대로 처리할 수 있습니다. 별도의 도구 없이 시스템에 설치된 도구만으로 디버깅과 검사, 링크를 모두 수행할 수 있습니다.

### Rust rlib 와 동일

Rust 의 `rlib` 역시 표준 ar archive 에 커스텀 섹션(LLVM bitcode) 을 더한 형식을 채택하고 있습니다. rok 도 같은 모델을 따르고 있습니다.

### LLVM SHT_LLVM_BITCODE 선례

LLVM 의 thinLTO 는 `.llvmbc` 커스텀 섹션을 archive 안에 두는데, 링커는 이 섹션을 무시하고 LLVM 만 꺼내 쓰는 구조입니다.

rok 의 `.rok.hir` 도 같은 패턴을 따르고 있습니다. 링커는 이 섹션을 무시하고 `rokc` 만 꺼내 [단형화](../000-meta/007-glossary.md#monomorphization-단형화) 에 활용합니다.

### 정적 .a 원칙

모든 산출물은 정적 .a 로 만들어집니다. dylib 나 so 같은 동적 라이브러리는 rok 의 표준 산출물이 아닙니다(다만 [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 호출은 가능합니다).

## 9.3.4. 단일 archive 의 내용

하나의 archive 안에는 다음과 같은 파일들이 들어 있습니다.

```
libfoo.a
├── foo.concrete.o          (native code)
├── foo.hir.o               (RHIR bytecode + meta)
├── foo.concrete.o.1        (when TU is split)
├── foo.concrete.o.2        (when TU is split)
└── ...
```

> when TU is split: TU 분할 시.

각 `.o` 는 표준 ELF / Mach-O / COFF 형식을 따르고 있고, archive 는 이들을 묶어주는 컨테이너 역할을 합니다.

## 9.3.5. .rok.meta 섹션의 내용

`foo.hir.o` 의 `.rok.meta` 섹션은 패키지의 메타 정보를 담고 있습니다.

```
ROK_META magic       (4 bytes)
version              (u16)
rokc_version_hash    (16 bytes)
package_hash         (16 bytes — package hash of [source unit](../000-meta/003-source-units.md))
abi_version_hash     (16 bytes — ABI hash of chapter 8)

export_list_offset    (u32)
export_list_count     (u16)
... (continued)

serialized info: name / item / visibility / generic flag / signature, etc.
```

> package hash of source unit: 소스 단위의 패키지 해시. ABI hash of chapter 8: 8 장의 ABI 해시. continued: 계속. serialized info: 직렬화된 정보. name / item / visibility / generic flag / signature: 이름 / 항목 / 가시성 / generic 여부 / 시그니처.

자세한 형식은 `rokc` 가 결정하도록 되어 있습니다. 본 사양에서는 어떤 정보가 들어가는지만 정의해 두었습니다.

## 9.3.6. 버전 호환 검사

서로 다른 rokc 버전이나 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 버전, 또는 다른 패키지 버전의 archive 를 함께 링크하려고 하면 다음과 같은 절차가 진행됩니다.

```
rokc build → read .rok.meta of each dep .a
            → compare rokc_version_hash / abi_version_hash
            → mismatch → error (rebuild required)
```

> read .rok.meta of each dep .a: 각 의존 .a 의 .rok.meta 읽음. compare: 비교. mismatch → error: 불일치면 에러.

이러한 검사 덕분에 Swift binary swiftmodule 이 버전 간 불안정했던 전철을 피할 수 있습니다. 명확한 에러 메시지를 통해 사용자에게 무엇을 다시 빌드해야 하는지 안내합니다.

## 9.3.7. 심볼 노출

archive 가 외부로 노출하는 심볼은 `foo.concrete.o` 의 전역 심볼입니다.

[심볼 맹글링 스킴](905-symbol-mangling.md) 에 따라 명명된 심볼은 `nm libfoo.a` 명령으로 확인할 수 있습니다.

```
nm libfoo.a
0000000000000000 T _Y3f8a1c2b_foo_geom_distance_a7c91d8e44b2f190
0000000000000040 T _Y3f8a1c2b_foo_geom_Point_origin_b1234567...
...
```

## 9.3.8. 표준 라이브러리 archive

`librok_bare.a`, `librok_core.a`, `librok_std.a` 도 모두 같은 형식을 따르고 있습니다. 표준 라이브러리 또한 일반 패키지와 다를 바 없는 archive 로 제공됩니다.

## 9.3.9. archive 의 압축 / 최소화

본 사양에서는 archive 의 압축이나 최소화를 강제하지 않습니다. `rokc build --release` 는 작은 산출물을 만들기 위한 추가 옵션을 가질 수 있는데, 이는 구현 결정사항으로 남겨두었습니다.

## 9.3.10. 외부 도구를 통한 검사

다음과 같이 표준 도구만으로도 archive 를 자유롭게 검사할 수 있습니다.

```
ar t libfoo.a            # 내용 목록
ar x libfoo.a foo.hir.o   # 추출
nm libfoo.a              # 심볼 목록
objdump -h libfoo.a       # 섹션 정보
```

이렇게 사용자와 개발자가 표준 도구로 archive 를 직접 들여다볼 수 있습니다.

## 9.3.11. 링크 명령 예

링크는 다음과 같은 형태로 수행됩니다.

```
clang main.o \
  libfoo.a \
  librokr.a \
  librok_bare.a librok_core.a librok_std.a \
  -o my_binary
```

링커는 archive 에 들어 있는 모든 .o 중 필요한 심볼만 골라서 링크하고, `.rok.hir` 처럼 rokc 전용 섹션은 무시합니다.

이 모델 덕분에 rok 코드를 C / C++ 프로젝트에서 직접 링크해 사용할 수 있습니다.

## 9.3.12. FFI 친화성

C 프로그램에서 다음과 같이 rok 함수를 호출할 수 있습니다.

```c
extern uint32_t rok_compute(int32_t a, int32_t b);  // 사용자가 export_name 으로 노출

int main() {
    uint32_t r = rok_compute(1, 2);
    ...
}
```

```
clang main.c libfoo.a librokr.a librok_*.a -o my_binary
```

표준 archive 형식을 채택한 덕분에 C 와의 직접 통합이 자연스럽게 이루어집니다.

## 9.3.13. 산출물 안정성

archive 의 binary 형식은 본 사양의 일부이며, 변경 시에는 [개요](../008-abi/README.md) 의 ABI 안정성 정책을 따르게 됩니다.

`.rok.hir` 의 RHIR bytecode 형식은 [RHIR bytecode 포맷](904-rhir-bytecode.md) 에서 정의하고 있고, 동일한 안정성 정책이 적용됩니다.
