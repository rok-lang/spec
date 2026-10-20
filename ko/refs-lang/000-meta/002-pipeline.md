# 0.2. 컴파일 파이프라인

rok 컴파일러를 본 사양에서는 `rokc` 라고 부릅니다. `rokc` 는 LLVM, MLIR, lld 를 내장하며, 소스에서 링크된 바이너리까지 파이프라인 전체를 프로세스 안에서 실행합니다.

## 0.2.1. 단계

전체 흐름은 다음과 같습니다.

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
   standard MLIR   (upstream MLIR dialects — `func`, `arith`, `scf`, ...)    -O1 이상
        │
        ▼
   LLVM IR → native object → lld
```

`-O0` (빌드와 REPL) 에서는 rok-mir 가 rok 의 native backend 를 거쳐 곧바로 기계어가 됩니다 (0.2.4).

> rok-hir 는 고수준 IR 이며, 타입 추론 이전 단계로서 사용자 코드 구조를 그대로 유지합니다. rok-mir 는 중간 IR 로서, 단형화와 연산자 desugar 가 완료되고 모든 타입이 확정된 형태입니다.
> standard MLIR 은 upstream MLIR 다이얼렉트를 뜻합니다. 마지막으로 LLVM IR 과 네이티브 코드가 산출됩니다.

## 0.2.2. 단방향 가시성

각 단계는 직전 단계만을 바라봅니다. `rok-hir` 생성기는 오직 소스만을 참조합니다. `rok-mir` 생성기는 `rok-hir` 만을 참조합니다. 그리고 MLIR 방출기는 `rok-mir` 만을 참조합니다.

따라서 상위 단계의 결과를 아래 단계에서 다시 참고하지 않습니다. 사양 문서 또한 같은 규칙을 따르며, 번호가 작은 파일은 번호가 큰 파일을 참조하지 않습니다.

## 0.2.3. 표준 MLIR 다이얼렉트만 사용

rok 전용 MLIR 다이얼렉트는 별도로 정의하지 않습니다. 사용하는 다이얼렉트는 모두 upstream MLIR 의 공식 다이얼렉트입니다.

| 다이얼렉트 | 용도                                     |
|------------|------------------------------------------|
| `func`     | 함수 정의 / 호출                         |
| `arith`    | 정수 / 부동소수 산술 / 비교              |
| `scf`      | 구조적 제어흐름 (`if` / `for` / `while`) |
| `cf`       | 비구조적 분기 (switch, cond_br)          |
| `index`    | 인덱스 타입 연산                         |
| `math`     | 수학 함수                                |
| `llvm`     | 저수준 op (intrinsic, `struct`)          |

이러한 선택은 다음과 같은 사항을 보장합니다. 먼저, upstream MLIR 의 개선이 자동으로 rok 에 흘러들어옵니다. 또한 IREE, Enzyme, Triton 같은 외부 MLIR 기반 도구와 자연스럽게 상호운용합니다. 마지막으로, 자체 다이얼렉트의 등록과 유지보수에 드는 비용이 발생하지 않습니다.

그 대신 rok 의 고차원 의미를 표준 다이얼렉트로 떨어뜨리는 책임이 컴파일러, 특히 `rok-mir → MLIR` 단계로 모이게 됩니다. 이는 의도된 부담입니다. 파이프라인 끝단의 선택지를 닫지 않기 위해 부담을 안쪽으로 옮긴 결과입니다.

## 0.2.4. Backend

`rokc` 에는 코드 생성기가 둘 있습니다. 둘 다 rok-mir 를 입력으로 받고 같은 런타임 ABI ([8](../008-abi/README.md)) 를 따르므로, 그 object 들은 함께 링크됩니다.

| Backend           | 쓰이는 곳                    | 경로                                         |
|-------------------|------------------------------|----------------------------------------------|
| native            | `-O0` 빌드와 REPL 셀         | rok-mir → 기계어                             |
| MLIR → LLVM       | `-O1` 이상                   | rok-mir → MLIR → LLVM IR → 최적화된 object   |

LLVM, MLIR, lld 는 `rokc` 의 일부입니다. 외부 LLVM 도구를 실행하지 않으며 사용자는 LLVM 을 설치하지 않습니다. macOS 대상은 SDK 를 위해 Xcode Command Line Tools 가 필요합니다. 크로스 컴파일은 내장되어 있습니다: `--target <triple>` 은 어느 host 에서나 동작하고, toolchain 은 지원하는 Linux 대상의 libc 헤더와 stub 라이브러리 (musl 과 glibc 심볼 stub) 를 함께 제공하므로 Linux 대상에는 sysroot 가 필요 없습니다. Apple 대상은 재배포할 수 없는 macOS SDK (Xcode Command Line Tools) 가 필요합니다 ([9.2.14](../009-tools/902-aot-pipeline.md#9214-링킹-실행-파일의-경우), [0.6](006-platform-targets.md)). `--emit=mlir` / `--emit=llvmir` 는 중간 형태를 내보냅니다 ([9.9.1](../009-tools/909-debug-tools.md#991-중간-산출물-출력---emit)).
