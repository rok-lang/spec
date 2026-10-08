# 8. 런타임 ABI / MLIR lowering 개요

본 블록은 컴파일러(`rokc`) 와 런타임(`librokr.a`) 사이의 공유 contract 를 정의합니다. 컴파일러와 런타임은 본 사양 텍스트만으로 연결되며, 공유 코드나 헤더 파일은 존재하지 않습니다.

## 목차

- [8.1. 객체 헤더 ABI](801-object-header-abi.md)
- [8.2. ScanTable 직렬화 포맷](802-scan-table.md)
- [8.3. Mutator TLS 슬롯 레이아웃](803-tls-slots.md)
- [8.4. Safepoint 프로토콜](804-safepoint.md)
- [8.5. Extern "C" 심볼](805-extern-symbols.md)
- [8.6. Calling Convention](806-calling-convention.md)
- [8.7. MLIR 다이얼렉트 사용](807-mlir-dialects.md)
- [8.8. Lowering: 데이터 타입과 표현식](808-lowering-types.md)
- [8.9. Lowering: 제어 흐름과 GC 통합](809-lowering-control.md)
- [8.10. Lowering: 코루틴과 제네릭](810-lowering-coroutine-generic.md)

## 두 종류의 contract

### 1. 런타임 ABI

컴파일러가 생성한 코드와 `librokr.a` 를 링크해 함께 실행할 때의 인터페이스에 해당합니다. 객체의 메모리 레이아웃은 컴파일러가 만든 객체를 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 가 정확히 해석할 수 있도록 규정됩니다. [extern "C" 심볼](../000-meta/007-glossary.md#extern-c-심볼) 은 컴파일러가 호출할 수 있는 런타임 함수이며, 각 심볼의 thread-safety 분류 (`thread-safe` / `caller-thread-only` / `process-init` 등) 는 [8.5 Extern "C" 심볼](805-extern-symbols.md#850-thread-safety-분류) 에서 정의합니다. TLS 슬롯은 컴파일러가 emit 한 인라인 코드가 읽고 쓰는 thread-local 자리를 의미합니다. Calling convention 은 함수 호출 시의 레지스터·스택 사용을 규정합니다.

### 2. MLIR lowering

[컴파일 파이프라인](../000-meta/002-pipeline.md) 의 rok-mir → 표준 MLIR 단계에서, rok 의 고차원 의미를 표준 MLIR 다이얼렉트로 변환하는 규칙을 의미합니다. `tagged union`, `any` 슬롯, 패턴 매칭, try/catch, generics 는 모두 표준 MLIR 다이얼렉트로 lower 됩니다. 커스텀 rok 다이얼렉트는 존재하지 않습니다.

## 본 사양 텍스트가 권위인 이유

컴파일러와 런타임은 공유 코드나 헤더 파일을 갖지 않습니다. 두 쪽이 같은 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 를 따른다는 보장은 본 사양 텍스트와 통합 테스트의 end-to-end 검증에서 비롯됩니다.

이 모델은 Swift ABI Stability 의 접근과 같습니다. 즉, 사양이 권위이며 구현은 그것을 따릅니다.

## 안정성

본 블록의 contract 는 언어 버전(SemVer) 의 일부에 해당합니다. ABI 변경은 명시적 버전 업을 동반합니다. minor / patch 의 경우 추가만 가능하며 호환을 깨면 안 됩니다. major 의 경우 호환을 깨는 변경이 가능하지만, 패키지 매니저가 알 수 있도록 합니다.

[컴파일 파이프라인](../000-meta/002-pipeline.md) 의 `.rok.meta` 에 ABI 버전 해시가 박힙니다. 다른 버전의 .a 들을 링크하면 명확한 에러가 발생합니다.
