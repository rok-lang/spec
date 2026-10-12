# 0.2. 컴파일 파이프라인

rok 컴파일러를 본 사양에서는 `rok` 라고 부릅니다. `rok` 는 소스 코드를 표준 MLIR 텍스트로 방출하는 일까지를 책임집니다. 그 이후의 최적화와 바이너리 생성은 표준 도구체인이 담당합니다.

## 0.2.1. 단계

전체 흐름은 다음과 같이 정리할 수 있습니다.

```
  source code
        │
        ▼
   rok-hir         (High-level IR — pre type inference, user code structure preserved)
        │
        ▼
   rok-mir         (Mid-level IR — monomorphized / operators desugared, all types fixed)
        │
        ▼
   standard MLIR   (upstream MLIR dialect text — `func`, `arith`, `scf`, ...)
        │
        ▼
   LLVM IR / native (mlir-opt → mlir-translate → clang / lld)
```

> rok-hir 는 고수준 IR 이며, 타입 추론 이전 단계로서 사용자 코드 구조를 그대로 유지합니다. rok-mir 는 중간 IR 로서, 단형화와 연산자 desugar 가 완료되고 모든 타입이 확정된 형태입니다.
> standard MLIR 은 표준 MLIR 을 의미하며, upstream MLIR 다이얼렉트 텍스트로 표현됩니다. 마지막으로 LLVM IR 또는 네이티브 코드가 산출됩니다.

## 0.2.2. 단방향 가시성

각 단계는 직전 단계만을 바라봅니다. 구체적으로 살펴보면 다음과 같습니다. `rok-hir` 생성기는 오직 소스만을 참조합니다. `rok-mir` 생성기는 `rok-hir` 만을 참조합니다. 그리고 MLIR 방출기는 `rok-mir` 만을 참조합니다.

따라서 상위 단계의 결과를 아래 단계에서 다시 참고하지 않습니다. 사양 문서 또한 같은 규칙을 따르며, 번호가 작은 파일은 번호가 큰 파일을 참조하지 않습니다.

## 0.2.3. 표준 MLIR 다이얼렉트만 사용

rok 전용 MLIR 다이얼렉트는 별도로 정의하지 않습니다. 사용하는 다이얼렉트는 모두 upstream MLIR 의 공식 다이얼렉트입니다.

| 다이얼렉트 | 용도                                     |
|------------|------------------------------------------|
| `func`     | 함수 정의 / 호출                         |
| `arith`    | 정수 / 부동소수 산술 / 비교              |
| `scf`      | 구조적 제어흐름 (`if` / `for` / `while`) |
| `cf`       | 비구조적 분기 (switch, cond_br)          |
| `memref`   | 스택 / 힙 메모리 참조                    |
| `index`    | memref 인덱싱                            |
| `math`     | 수학 함수                                |
| `llvm`     | 저수준 op (intrinsic, `struct`)          |

이러한 선택은 다음과 같은 사항을 보장합니다. 먼저, upstream MLIR 의 개선이 자동으로 rok 에 흘러들어옵니다. 또한 IREE, Enzyme, Triton 같은 외부 MLIR 기반 도구와 자연스럽게 상호운용합니다. 마지막으로, 자체 다이얼렉트의 등록과 유지보수에 드는 비용이 발생하지 않습니다.

그 대신 rok 의 고차원 의미를 표준 다이얼렉트로 떨어뜨리는 책임이 컴파일러, 특히 `rok-mir → MLIR` 단계로 모이게 됩니다. 이는 의도된 부담입니다. 파이프라인 끝단의 선택지를 닫지 않기 위해 부담을 안쪽으로 옮긴 결과입니다.

## 0.2.4. MLIR 텍스트 직접 생성

컴파일러는 MLIR 을 텍스트 형태로 출력합니다. 즉, MLIR 라이브러리 바인딩(C++ 또는 그 외) 을 호출하지 않습니다.

이러한 방식은 몇 가지 이점을 가져다 줍니다. 우선 외부 라이브러리 의존을 줄여 줍니다. 다음으로 크로스 컴파일 매트릭스가 단순해지며, MLIR 도구가 설치된 환경이라면 어디서든 결과를 받을 수 있습니다. 끝으로 `rok --emit=mlir` 같은 디버그 출력이 그대로 표준 도구로 흘러갑니다.
