# 9.3. 패키지 아카이브 (.a) 포맷

[AOT 빌드 파이프라인](902-aot-pipeline.md) 산출물인 `libfoo.a` 의 내부 구조를 정의한다.

## 9.3.1. 표준 ar archive

`libfoo.a` 는 표준 Unix `ar` 아카이브 형식이다. `ar`, `ranlib`, `lld` 등 표준 도구로 그대로 처리할 수 있다.

```
ar t libfoo.a
foo.concrete.o
foo.hir.o
foo.concrete.o.1     (큰 패키지의 경우 추가 TU)
...
```

## 9.3.2. 두 종류의 .o 파일

### `foo.concrete.o`: [concrete fn](../000-meta/007-glossary.md#concrete-fn-/-generic-fn) 의 native code

[AOT 빌드 파이프라인](902-aot-pipeline.md) 단계 12 의 결과로, MLIR 을 거쳐 컴파일된 native 코드이다.

표준 ELF / Mach-O / COFF object 파일이며 다음 섹션을 포함한다.

| 섹션      | 의미                    |
| --------- | ----------------------- |
| `.text`   | concrete `fn` 들의 코드 |
| `.rodata` | `const` / 리터럴        |
| `.data`   | `static` 의 초기값      |
| `.bss`    | 0 초기 `static`         |
| ...       | 표준 섹션들             |

특별 섹션:

| 섹션         | 의미                                                                     |
| ------------ | ------------------------------------------------------------------------ |
| `.nok.scan`  | ScanTable 데이터 ([ScanTable 직렬화 포맷](../008-abi/802-scan-table.md)) |
| `.nok.types` | 타입 등록 정보 (`type_id → 타입 이름` 등)                                |
| `.nok.init`  | 패키지 초기화 함수 포인터                                                |

이 특별 섹션들은 링커가 표준대로 처리하지만 nok 부팅 시점에 사용한다.

### `foo.hir.o`: 제네릭 `fn` 의 [NHIR](../000-meta/007-glossary.md#nhir-nok-hir-bytecode) bytecode

[AOT 빌드 파이프라인](902-aot-pipeline.md) 단계 6 ~ 7 의 결과 중 [generic](../000-meta/007-glossary.md#generic-/-제네릭) 함수의 [HIR](../000-meta/007-glossary.md#hir-high-level-ir-/-mir-mid-level-ir) 을 직렬화한 결과이다.

표준 object 파일이지만 다음 커스텀 섹션만 담는다.

| 섹션        | 의미                                  |
| ----------- | ------------------------------------- |
| `.nok.hir`  | NHIR bytecode (제네릭 `fn` body)      |
| `.nok.meta` | 패키지 메타 (export 목록, 버전, 해시) |

`.text` 등 코드 섹션은 비어 있다. 링커는 이 obj 를 심볼 충돌 없이 [archive](../000-meta/007-glossary.md#archive) 안에 그대로 둔다.

## 9.3.3. archive 구성 이유

### 표준 도구 호환

`libfoo.a` 는 표준 ar 아카이브이다. `ar`, `ranlib`, `nm`, `lld`, `gcc`/`clang` 이 모두 그대로 처리한다. 별도 도구 없이 시스템에 설치된 도구만으로 디버그 / 검사 / 링크할 수 있다.

### Rust rlib 와 동일

Rust 의 `rlib` 도 표준 ar archive + 커스텀 섹션 (LLVM bitcode) 형식이다. nok 도 같은 모델을 따른다.

### LLVM SHT_LLVM_BITCODE 선례

LLVM 의 thinLTO 가 `.llvmbc` 커스텀 섹션을 archive 안에 둔다. 링커는 무시하고 LLVM 만 꺼내 쓴다.

nok 의 `.nok.hir` 도 같은 패턴이다. 링커는 무시하고 `nokc` 만 꺼내 [specialize](../000-meta/007-glossary.md#specialize) 한다.

### 정적 .a 원칙

모든 산출물은 정적 .a 이다. dylib, so 등은 nok 의 표준 산출물이 아니다 ([FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 호출은 가능).

## 9.3.4. 단일 archive 의 내용

```
libfoo.a
├── foo.concrete.o          (native code)
├── foo.hir.o                (NHIR bytecode + meta)
├── foo.concrete.o.1         (TU 분할 시)
├── foo.concrete.o.2         (TU 분할 시)
└── ...
```

각 `.o` 는 표준 ELF / Mach-O / COFF 형식이다. archive 는 이들을 묶는 컨테이너이다.

## 9.3.5. .nok.meta 섹션의 내용

`foo.hir.o` 의 `.nok.meta` 섹션은 패키지 메타 정보를 담는다.

```
NOK_META magic       (4 bytes)
version              (u16)
nokc_version_hash    (16 bytes)
package_hash         (16 bytes — [소스 단위](../000-meta/003-source-units.md) 의 패키지 해시)
abi_version_hash     (16 bytes — 8 장의 ABI 해시)

export_list_offset    (u32)
export_list_count     (u16)
... (계속)

이름 / 항목 / 가시성 / generic 여부 / 시그니처 등의 직렬화된 정보.
```

자세한 형식은 `nokc` 가 결정한다. 본 사양은 어떤 정보가 들어가는지만 정의한다.

## 9.3.6. 버전 호환 검사

다른 nokc 버전, [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 버전, 패키지 버전의 archive 를 함께 링크하면:

```
nokc build → 각 의존 .a 의 .nok.meta 읽음
            → nokc_version_hash / abi_version_hash 비교
            → 불일치면 에러 (rebuild required)
```

이 검사로 Swift binary swiftmodule 이 버전 간 불안정했던 전철을 피한다. 명확한 에러 메시지로 사용자가 무엇을 다시 빌드해야 하는지 안내한다.

## 9.3.7. 심볼 노출

archive 가 외부로 노출하는 심볼은 `foo.concrete.o` 의 전역 심볼이다.

[심볼 맹글링 스킴](905-symbol-mangling.md) 으로 명명된 심볼이 `nm libfoo.a` 로 보인다.

```
nm libfoo.a
0000000000000000 T _N3f8a1c2b_foo_geom_distance_a7c91d8e44b2f190
0000000000000040 T _N3f8a1c2b_foo_geom_Point_origin_b1234567...
...
```

## 9.3.8. 표준 라이브러리 archive

`libnok_unsafe.a`, `libnok_core.a`, `libnok_std.a` 는 같은 형식이다. 표준 라이브러리도 일반 패키지처럼 archive 이다.

## 9.3.9. archive 의 압축 / 최소화

본 사양은 archive 의 압축이나 최소화를 강제하지 않는다. `nokc build --release` 는 작은 산출물을 위한 추가 옵션을 가질 수 있다 (구현 결정).

## 9.3.10. 외부 도구를 통한 검사

```
ar t libfoo.a            # 내용 목록
ar x libfoo.a foo.hir.o   # 추출
nm libfoo.a              # 심볼 목록
objdump -h libfoo.a       # 섹션 정보
```

사용자와 개발자가 표준 도구로 archive 를 직접 검사할 수 있다.

## 9.3.11. 링크 명령 예

```
clang main.o \
  libfoo.a \
  libnokr.a \
  libnok_unsafe.a libnok_core.a libnok_std.a \
  -o my_binary
```

링커는 archive 의 모든 .o 중 필요한 심볼만 골라 링크한다. `.nok.hir` 같은 nokc 전용 섹션은 무시한다.

이 모델 덕분에 nok 코드를 C / C++ 프로젝트에서 직접 링크할 수 있다.

## 9.3.12. FFI 친화성

C 프로그램에서:

```c
extern uint32_t nok_compute(int32_t a, int32_t b);  // 사용자가 export_name 으로 노출

int main() {
    uint32_t r = nok_compute(1, 2);
    ...
}
```

```
clang main.c libfoo.a libnokr.a libnok_*.a -o my_binary
```

표준 archive 형식 덕분에 C 와의 직접 통합이 자연스럽다.

## 9.3.13. 산출물 안정성

archive 의 binary 형식은 본 사양의 일부이다. 변경 시 [개요](../008-abi/README.md) 의 ABI 안정성 정책을 따른다.

`.nok.hir` 의 NHIR bytecode 형식은 [NHIR bytecode 포맷](904-nhir-bytecode.md) 에서 정의하며 같은 안정성 정책을 적용한다.
