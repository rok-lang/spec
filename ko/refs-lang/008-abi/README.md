# 8. 런타임 ABI / MLIR lowering 개요

이 블록은 컴파일러(`rokc`)와 런타임(`librokr.a`)이 함께 따르는 계약(contract)을 정의합니다. 두 쪽을 잇는 것은 이 사양 텍스트입니다. 둘이 공유하는 코드는 GC ABI 상수의 정의 하나(`gc_abi`, [8.1](801-object-header-abi.md))뿐입니다.

## 목차

- [8.1. 객체 헤더 ABI](801-object-header-abi.md)
- [8.2. ScanTable 직렬화 포맷](802-scan-table.md)
- [8.3. mutator TLS 슬롯 레이아웃](803-tls-slots.md)
- [8.4. safepoint 프로토콜](804-safepoint.md)
- [8.5. Extern "C" 심볼](805-extern-symbols.md)
- [8.6. 호출 규약](806-calling-convention.md)
- [8.7. MLIR 다이얼렉트 사용](807-mlir-dialects.md)
- [8.8. Lowering: 데이터 타입과 표현식](808-lowering-types.md)
- [8.9. Lowering: 제어 흐름과 GC 통합](809-lowering-control.md)
- [8.10. Lowering: 코루틴과 제네릭](810-lowering-coroutine-generic.md)

## 두 종류의 계약

### 1. 런타임 ABI

컴파일러가 생성한 코드를 `librokr.a`와 링크해 함께 실행할 때 쓰는 인터페이스입니다. 객체의 메모리 레이아웃은 컴파일러가 만든 객체를 [GC](../000-meta/007-glossary.md#gc-garbage-collector)가 정확히 해석할 수 있게 정합니다. [extern "C" 심볼](../000-meta/007-glossary.md#extern-c-심볼)은 컴파일러가 호출할 수 있는 런타임 함수입니다. 각 심볼의 스레드 안전성 분류(`thread-safe` / `caller-thread-only` / `process-init` 등)는 [8.5 Extern "C" 심볼](805-extern-symbols.md#850-스레드-안전성-분류)에서 정의합니다. TLS 슬롯은 컴파일러가 내보낸 인라인 코드가 읽고 쓰는 스레드 로컬 자리입니다. 호출 규약(calling convention)은 함수를 호출할 때 레지스터와 스택을 어떻게 쓰는지 정합니다.

### 2. MLIR lowering

[컴파일 파이프라인](../000-meta/002-pipeline.md)의 rok-mir → 표준 MLIR 단계에서 Rok의 고수준 의미를 표준 MLIR 다이얼렉트로 바꾸는 규칙입니다. 컴파일러는 `tagged union`, `any` 슬롯, 패턴 매칭, try/catch, 제네릭을 모두 표준 MLIR 다이얼렉트로 낮춥니다. 커스텀 Rok 다이얼렉트는 없습니다.

## 이 사양 텍스트가 권위를 갖는 이유

컴파일러와 런타임은 `gc_abi` 말고는 코드나 헤더 파일을 공유하지 않습니다. 두 쪽이 같은 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface)를 따른다는 보장은 이 사양 텍스트와 통합 테스트의 종단 간(end-to-end) 검증에서 나옵니다.

이 모델은 Swift ABI Stability와 같은 접근입니다. 사양이 권위이고, 구현은 사양을 따릅니다.

## 안정성

이 블록의 계약은 언어 버전(SemVer)의 일부입니다. ABI를 바꾸면 버전을 명시적으로 올립니다. minor / patch 릴리스에서는 추가만 할 수 있고 호환성을 깨면 안 됩니다. major 릴리스에서는 호환성을 깰 수 있습니다. 이때는 패키지 매니저가 알아챌 수 있게 깹니다.

미리 빌드한 패키지의 ROKHIR 헤더에는 그 패키지를 만든 `rokc`가 기록됩니다([9.4](../009-tools/904-rokhir-bytecode.md)). 다른 `rokc`가 만든 패키지를 쓰면 에러입니다. 에러는 다시 빌드하라고 안내합니다.
