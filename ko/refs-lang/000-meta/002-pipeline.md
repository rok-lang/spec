# 0.2. 컴파일 파이프라인

이 사양은 Rok 컴파일러를 `rokc`라고 부릅니다. `rokc`는 LLVM, MLIR, lld를 내장합니다. 소스에서 링크된 바이너리까지 파이프라인 전체를 한 프로세스 안에서 실행합니다.

## 0.2.1. 단계

전체 흐름은 다음과 같습니다.

```
  source code
        │
        ▼
   rok-hir         (High-level IR - pre type inference, user code structure preserved)
        │
        ▼
   rok-mir         (Mid-level IR - monomorphized / operators desugared, all types fixed)
        │
        ▼
   standard MLIR   (upstream MLIR dialects - `func`, `arith`, `scf`, ...)    -O1 이상
        │
        ▼
   LLVM IR → native object → lld
```

`-O0`(빌드와 REPL)에서는 rok-mir가 Rok의 네이티브 백엔드를 거쳐 곧바로 기계어가 됩니다(0.2.4).

> rok-hir는 고수준 IR입니다. 타입 추론 이전 단계이며 사용자 코드 구조를 그대로 유지합니다. rok-mir는 중간 IR입니다. 단형화와 연산자 풀어 쓰기(desugaring)가 끝나고 모든 타입이 확정된 형태입니다.
> standard MLIR은 업스트림(upstream) MLIR 다이얼렉트를 뜻합니다. 마지막 단계에서 LLVM IR과 네이티브 코드가 나옵니다.

## 0.2.2. 단방향 가시성

각 단계는 바로 앞 단계만 봅니다. `rok-hir` 생성기는 소스만 참조합니다. `rok-mir` 생성기는 `rok-hir`만 참조합니다. MLIR 방출기는 `rok-mir`만 참조합니다.

그러므로 뒤 단계는 앞 단계의 결과를 다시 참고하지 않습니다. 사양 문서도 같은 규칙을 따릅니다. 번호가 작은 파일은 번호가 큰 파일을 참조하지 않습니다.

## 0.2.3. 표준 MLIR 다이얼렉트만 사용

Rok 전용 MLIR 다이얼렉트는 따로 정의하지 않습니다. 사용하는 다이얼렉트는 모두 업스트림 MLIR의 공식 다이얼렉트입니다.

| 다이얼렉트 | 용도                                     |
|------------|------------------------------------------|
| `func`     | 함수 정의 / 호출                         |
| `arith`    | 정수 / 부동소수 산술 / 비교              |
| `scf`      | 구조적 제어 흐름(`if` / `for` / `while`) |
| `cf`       | 비구조적 분기(switch, cond_br)           |
| `index`    | 인덱스 타입 연산                         |
| `math`     | 수학 함수                                |
| `llvm`     | 저수준 op(intrinsic, `struct`)           |

이렇게 하면 세 가지를 보장할 수 있습니다. 업스트림 MLIR이 개선되면 Rok에 자동으로 들어옵니다. IREE, Enzyme, Triton 같은 외부 MLIR 기반 도구와 그대로 상호운용합니다. 자체 다이얼렉트를 등록하고 유지보수하는 비용이 들지 않습니다.

대신 Rok의 고수준 의미를 표준 다이얼렉트로 낮추는 책임이 컴파일러, 특히 `rok-mir → MLIR` 단계에 모입니다. 의도한 부담입니다. 파이프라인 끝단의 선택지를 닫지 않으려고 부담을 안쪽으로 옮겼습니다.

## 0.2.4. 백엔드

`rokc`에는 코드 생성기가 둘 있습니다. 둘 다 rok-mir를 입력으로 받고 같은 런타임 ABI([8](../008-abi/README.md))를 따릅니다. 그래서 두 생성기가 만든 오브젝트를 함께 링크할 수 있습니다.

| Backend           | 쓰이는 곳                    | 경로                                         |
|-------------------|------------------------------|----------------------------------------------|
| native            | `-O0` 빌드와 REPL 셀         | rok-mir → 기계어                             |
| MLIR → LLVM       | `-O1` 이상                   | rok-mir → MLIR → LLVM IR → 최적화된 object   |

LLVM, MLIR, lld는 `rokc`의 일부입니다. 외부 LLVM 도구는 실행하지 않고, 사용자가 LLVM을 설치하지 않아도 됩니다. macOS 대상은 SDK를 위해 Xcode Command Line Tools가 필요합니다. 크로스 컴파일도 내장되어 있습니다. `--target <triple>`는 어느 호스트에서나 동작합니다. toolchain이 지원 대상 Linux용 C 라이브러리(musl, glibc 링크 스텁)를 함께 담고 있으므로 Linux 대상에는 sysroot가 필요 없습니다. C 헤더는 없습니다. Rok은 C를 컴파일하지 않습니다. Windows 대상은 toolchain에 든 module definition으로 만든 import library를 UCRT 위에서 링크합니다. MSVC 도구나 Windows SDK는 필요 없습니다. Apple 대상에는 macOS SDK(Xcode Command Line Tools)가 필요하고, 이 SDK는 재배포할 수 없습니다([9.2.14](../009-tools/902-aot-pipeline.md#9214-링킹실행-파일의-경우), [0.6](006-platform-targets.md)). `--emit=mlir` / `--emit=llvmir`는 중간 형태를 내보냅니다([9.9.1](../009-tools/909-debug-tools.md#991-중간-산출물-출력---emit)).
