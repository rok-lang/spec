# 8.7. MLIR 다이얼렉트 사용

[컴파일 파이프라인](../000-meta/002-pipeline.md)에서 정한 표준 MLIR 다이얼렉트를 이 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 블록의 lowering 사양에 적용합니다. 일반 코드는 핵심 다이얼렉트 7개로 낮춥니다. 가속 영역의 라이브러리는 `linalg` / `tensor` / `vector` / `gpu` / `affine` / `async` 같은 추가 다이얼렉트를 [`rok::bare::mlir`](../../refs-standard-libraries/100-bare/109-mlir.md)로 직접 쓸 수 있습니다. 이 문서는 MLIR → LLVM 백엔드(`-O1` 이상)의 lowering을 다룹니다. 네이티브 백엔드(`-O0`, REPL)는 rok-mir에서 바로 기계어를 만들고 같은 ABI를 따릅니다([0.2.4](../000-meta/002-pipeline.md#024-백엔드)).

## 8.7.1. 일반 코드 사용 다이얼렉트(재확인)

[컴파일 파이프라인](../000-meta/002-pipeline.md)의 표를 다시 옮기면 다음과 같습니다.

| 다이얼렉트 | 용도                                     | 이 블록의 사용                                                          |
|------------|------------------------------------------|-------------------------------------------------------------------------|
| `func`     | 함수 정의 / 호출                         | 함수 lowering                                                           |
| `arith`    | 정수 / 부동소수 산술 / 비교              | 연산자 lowering                                                         |
| `scf`      | 구조적 제어흐름 (`if` / `for` / `while`) | `if` / 루프 lowering                                                    |
| `cf`       | 비구조적 분기 (switch, cond_br)          | `match` decision tree                                                   |
| `index`    | 인덱스 타입 연산                         | 인덱싱 lowering                                                         |
| `math`     | 수학 함수                                | sqrt / sin / cos 등                                                     |
| `llvm`     | 저수준 op (intrinsic, `struct`, `alloca`, load / store) | 스택 슬롯, 객체 / 배열, [GC](../000-meta/007-glossary.md#gc-garbage-collector) |

커스텀 Rok 다이얼렉트는 없습니다. 일반 Rok 코드의 의미는 위의 표준 다이얼렉트 일곱 개로 표현합니다. 가속 라이브러리가 추가로 쓰는 다이얼렉트는 8.7.5에서 다룹니다.

## 8.7.2. 변환 흐름

전체 변환 흐름은 다음과 같습니다.

```
rok 소스
   ↓ (parse, name resolution, type check)
rok-hir
   ↓ (specialize / monomorphize, desugar)
rok-mir
   ↓ (lowering: 본 블록의 808 ~ 810)
표준 MLIR  (일반 코드는 7 개 다이얼렉트, 라이브러리는 추가 가능)
   ↓ (표준 패스, LLVM 다이얼렉트로 변환)
LLVM IR
   ↓ (LLVM 최적화와 코드 생성)
object
   ↓ (lld)
네이티브 바이너리
```

모든 단계는 `rokc` 안에서 실행됩니다. 이 ABI 블록은 그중 rok-mir에서 표준 MLIR로 가는 단계의 lowering 규칙을 정의합니다.

## 8.7.3. 다이얼렉트별 핵심 op

`rokc`가 어느 다이얼렉트를 내보내는지는 구현이 정합니다. `rokc`는 `func`, `arith`, `cf`, `llvm`을 내보냅니다. 수학 intrinsic을 낮추는 곳에서는 `math`도 내보냅니다. `scf`와 `index`는 써도 됩니다. 아래는 각 다이얼렉트의 핵심 op입니다.

### `func`

```
func.func @name(%arg: i32) -> (i32) { ... }
func.call @name(%arg) : (i32) -> (i32)
func.return %val : i32
```

함수 정의, 호출, 반환을 다루는 표준 op입니다.

### `arith`

```
arith.addi %a, %b : i32
arith.subi %a, %b : i32
arith.muli %a, %b : i32
arith.cmpi "eq", %a, %b : i32
arith.addf %a, %b : f64
arith.cmpf "olt", %a, %b : f64
```

[연산자 디스패치](../003-expressions/301-operator-dispatch.md)의 산술 연산과 비교 연산은 `arith` 다이얼렉트로 낮춥니다.

다만 사용자 정의 트레잇 구현은 제네릭 단형화를 거친 뒤 함수 호출로 낮춥니다.

### `scf`

```
scf.if %cond -> (i32) {
  scf.yield %a : i32
} else {
  scf.yield %b : i32
}

scf.while ... do { ... }
scf.for %i = %lb to %ub step %step { ... }
```

[`if` 표현식](../003-expressions/310-if-expression.md)과 [루프](../003-expressions/313-loop-expressions.md)의 정상 제어 흐름은 `scf`로 낮춥니다.

### `cf`

```
cf.cond_br %cond, ^true_block(args), ^false_block(args)
cf.switch %v : i32, [
  default: ^default,
  0: ^case0,
  1: ^case1
]
```

복잡한 [`match`](../003-expressions/312-match-expression.md)의 decision tree는 `cf` 다이얼렉트의 cond_br와 switch로 낮춥니다.

### `index`

```
%idx = index.constant 5
%new_idx = index.add %idx, %step
```

`index` 타입이 필요한 자리(`scf.for`의 경계 등)는 `index` 다이얼렉트로 낮춥니다.

### `math`

```
math.sqrt %x : f64
math.sin %x : f64
math.cos %x : f64
```

표준 라이브러리의 수학 함수는 가능하면 `math` 다이얼렉트로 표현합니다. `math`에 직접 대응하는 op가 없는 함수는 일반 함수 호출로 처리합니다.

### `llvm`

LLVM 다이얼렉트는 마지막 수단입니다. 다른 다이얼렉트로 표현할 수 없는 의미에만 씁니다.

```
llvm.alloca / llvm.load / llvm.store
llvm.struct<(...)>
llvm.intr.expect / llvm.intr.assume
```

스택 슬롯, 객체 접근, 저수준 intrinsic은 `llvm` 다이얼렉트의 영역입니다. 코루틴은 컴파일러가 만든 상태 기계이므로 `llvm.coro.*`를 쓰지 않습니다([7.8](../007-runtime/708-coroutine.md)). GC 루트는 intrinsic 없이 일반 load / store로 shadow stack 프레임을 다룹니다([8.9.12](809-lowering-control.md#8912-shadow-stack-프레임)). 에러 전달은 반환값과 일반 분기로 표현하므로([8.9.6](809-lowering-control.md#896-try--catch--finally-값-기반-분기)) `llvm` 다이얼렉트의 특수 기능을 쓰지 않습니다.

## 8.7.4. 다이얼렉트 사용 우선순위

lowering 단계는 `scf`, `index`, `math`로 의미를 표현할 수 있으면 이들을 써도 됩니다. 다른 것으로 표현할 수 없는 곳에서는 `llvm`을 써도 됩니다. 어느 쪽을 고를지는 구현이 정합니다(8.7.3).

## 8.7.5. 라이브러리 / 가속 영역 다이얼렉트

다음 다이얼렉트는 Rok의 '일반 코드'를 낮출 때는 쓰지 않습니다. 하지만 라이브러리 작성자가 [`rok::bare::mlir`](../../refs-standard-libraries/100-bare/109-mlir.md)로 직접 쓸 수 있습니다.

| 다이얼렉트 | 활용 영역                                     |
|------------|-----------------------------------------------|
| `affine`   | affine 분석 가능한 루프 (수치 라이브러리)     |
| `linalg`   | element-wise / matmul / reduction (텐서 lib)  |
| `tensor`   | shape / extract / insert / reshape (텐서 lib) |
| `vector`   | SIMD 폭 / 정렬 / shuffle (SIMD lib)           |
| `gpu`      | GPU launch / kernel / shared memory (GPU lib) |
| `async`    | async fork / await (전용 런타임 라이브러리)   |

일반 Rok 코드를 낮출 때는 8.7.1의 다이얼렉트 일곱 개면 충분합니다. 텐서 / 자동 미분 / SIMD / GPU 같은 가속 영역의 라이브러리는 위 다이얼렉트를 직접 호출합니다. 이렇게 PyTorch / JAX / Mojo에 견줄 만한 성능 표면을 만듭니다.

라이브러리 작성자가 다이얼렉트를 호출하는 메커니즘과 작성자가 지는 안전 책임은 [`rok::bare::mlir`](../../refs-standard-libraries/100-bare/109-mlir.md)에서 정의합니다.

## 8.7.6. `extern` 함수(런타임 함수)

[Extern "C" 심볼](805-extern-symbols.md)에 나오는 `rok_*` 함수는 외부 `llvm.func` 선언으로 선언합니다(선언 형태는 구현이 정함).

```
llvm.func @rok_gc_alloc_slow(i64, i64, i32) -> !llvm.ptr
llvm.func @rok_runtime_yield()
```

호출에는 `llvm.call`을 씁니다.

## 8.7.7. TLS 접근

[Mutator TLS 슬롯](803-tls-slots.md)의 TLS 슬롯에 접근할 때는 `llvm` 다이얼렉트의 thread_local 전역 `rok_tls`에 각 슬롯의 오프셋을 더합니다.

```
llvm.mlir.global external thread_local @rok_tls() : !llvm.array<N x i8>

%t = llvm.mlir.addressof @rok_tls : !llvm.ptr
%flag_ptr = llvm.getelementptr %t[88] : (!llvm.ptr) -> !llvm.ptr, i8   ; yield_flag at 10p+8, p = 8
%flag = llvm.load %flag_ptr : !llvm.ptr -> i8
```

LLVM 백엔드가 이 코드를 타깃별 TLS 명령어로 자동으로 낮춥니다.

## 8.7.8. 각 단계의 책임

```
rok-mir → 표준 MLIR         rokc lowering (본 블록)
MLIR → LLVM IR              MLIR 패스와 변환, 프로세스 안
LLVM IR → object            LLVM, 프로세스 안, 요청한 -O 수준
object → 바이너리           lld, 프로세스 안 (ROK_CC 로 대체, 9.2.14)
```

## 8.7.9. 변환 검증

`tests/integration/` 아래의 종단 간(end-to-end) 테스트는 사용자가 작성한 .rok 파일을 빌드하고 실행해서 검증합니다.

이 검증이 이 블록에서 정의한 lowering 규칙이 정확함을 보장합니다.
