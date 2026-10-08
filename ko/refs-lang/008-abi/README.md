# 8. 런타임 ABI / MLIR lowering 개요

본 블록은 컴파일러 (`nokc`) 와 런타임 (`libnokr.a`) 사이의 공유 contract 를 정의한다.
컴파일러와 런타임은 본 사양 텍스트만으로 연결된다. 공유 코드나 헤더 파일은 없다.

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

컴파일러가 생성한 코드와 `libnokr.a` 를 링크해 함께 실행할 때의 인터페이스.

- 객체의 메모리 레이아웃: 컴파일러가 만든 객체를 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 가 정확히 해석한다.
- [extern "C" 심볼](../000-meta/007-glossary.md#extern-"c"-심볼): 컴파일러가 호출할 수 있는 런타임 함수.
- TLS 슬롯: 컴파일러가 emit 한 인라인 코드가 읽고 쓰는 thread-local 자리.
- Calling convention: 함수 호출 시의 레지스터·스택 사용.

### 2. MLIR lowering

[컴파일 파이프라인](../000-meta/002-pipeline.md) 의 nok-mir → 표준 MLIR 단계에서, nok 의 고차원 의미를 표준
MLIR 다이얼렉트로 변환하는 규칙.

- `tagged union`, `any` 슬롯, 패턴 매칭, try/catch, generics. 모두 표준 MLIR 다이얼렉트로
  lower.
- 커스텀 nok 다이얼렉트는 없다.

## 왜 본 사양 텍스트가 권위인가

컴파일러와 런타임은 공유 코드나 헤더 파일을 갖지 않는다. 두 쪽이 같은 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 를 따른다는
보장은 본 사양 텍스트와 통합 테스트의 end-to-end 검증에서 온다.

이 모델은 Swift ABI Stability 의 접근과 같다. 사양이 권위이고, 구현은 따른다.

## 안정성

본 블록의 contract 는 언어 버전 (SemVer) 의 일부다. ABI 변경은 명시적 버전 업을 동반한다.

- minor / patch: 추가만 가능 (호환을 깨면 안 된다).
- major: 호환을 깨는 변경 가능, 단 패키지 매니저가 알 수 있게 한다.

[컴파일 파이프라인](../000-meta/002-pipeline.md) 의 `.nok.meta` 에 ABI 버전 해시가 박힌다. 다른 버전의 .a 들을
링크하면 명확한 에러가 난다.
