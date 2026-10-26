# 8.7. MLIR 다이얼렉트 사용

[컴파일 파이프라인](../000-meta/002-pipeline.md) 에서 정한 표준 MLIR 다이얼렉트를 본 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 블록의 lowering 사양에 적용합니다. 일반 코드는 7 개 핵심 다이얼렉트로 lower 되며, 가속 영역의 라이브러리는 `linalg` / `tensor` / `vector` / `gpu` / `affine` / `async` 같은 추가 다이얼렉트를 [`rok::bare::mlir`](../../refs-standard-libraries/100-bare/109-mlir.md) 로 직접 활용할 수 있습니다. 이것은 MLIR → LLVM backend (`-O1` 이상) 의 lowering 입니다. native backend (`-O0`, REPL) 는 rok-mir 에서 바로 기계어를 만들며 같은 ABI 를 따릅니다 ([0.2.4](../000-meta/002-pipeline.md#024-backends)).

## 8.7.1. 일반 코드 사용 다이얼렉트 (재확인)

[컴파일 파이프라인](../000-meta/002-pipeline.md) 에 정리된 표를 다시 옮겨 보면 다음과 같습니다.

| 다이얼렉트 | 용도                                     | 본 블록의 사용                                                          |
|------------|------------------------------------------|-------------------------------------------------------------------------|
| `func`     | 함수 정의 / 호출                         | 함수 lowering                                                           |
| `arith`    | 정수 / 부동소수 산술 / 비교              | 연산자 lowering                                                         |
| `scf`      | 구조적 제어흐름 (`if` / `for` / `while`) | `if` / 루프 lowering                                                    |
| `cf`       | 비구조적 분기 (switch, cond_br)          | `match` decision tree                                                   |
| `index`    | 인덱스 타입 연산                         | 인덱싱 lowering                                                         |
| `math`     | 수학 함수                                | sqrt / sin / cos 등                                                     |
| `llvm`     | 저수준 op (intrinsic, `struct`, `alloca`, load / store) | 스택 슬롯, 객체 / 배열, [GC](../000-meta/007-glossary.md#gc-garbage-collector) |

별도의 커스텀 Rok 다이얼렉트는 두지 않습니다. 일반 Rok 코드의 의미는 위에 나열한 일곱 개의 표준 다이얼렉트로 표현되며, 가속 라이브러리가 추가로 활용하는 다이얼렉트는 8.7.5 에서 다룹니다.

## 8.7.2. 변환 흐름 정리

전체 변환 흐름은 다음과 같이 이어집니다.

```
rok 소스
   ↓ (parse, name resolution, type check)
rok-hir
   ↓ (specialize / monomorphize, desugar)
rok-mir
   ↓ (lowering - 본 블록의 808 ~ 810)
표준 MLIR  (일반 코드는 7 개 다이얼렉트, 라이브러리는 추가 가능)
   ↓ (표준 패스, LLVM 다이얼렉트로 변환)
LLVM IR
   ↓ (LLVM 최적화와 코드 생성)
object
   ↓ (lld)
네이티브 바이너리
```

모든 단계는 `rokc` 안에서 실행됩니다. 본 ABI 블록에서는 이 가운데 rok-mir 에서 표준 MLIR 로 가는 단계의 lowering 규칙을 정의합니다.

## 8.7.3. 다이얼렉트 별 핵심 op

`rokc` 가 어느 다이얼렉트를 emit 하는지는 구현이 정합니다. `rokc` 는 `func`, `arith`, `cf`, `llvm` 을 (그리고 수학 intrinsic 을 낮추는 곳에서는 `math` 를) emit 하며, `scf` 와 `index` 는 쓸 수 있습니다. 아래는 각 다이얼렉트의 핵심 op 입니다.

### `func`

```
func.func @name(%arg: i32) -> (i32) { ... }
func.call @name(%arg) : (i32) -> (i32)
func.return %val : i32
```

함수의 정의와 호출, 그리고 반환을 다루는 표준 op 들입니다.

### `arith`

```
arith.addi %a, %b : i32
arith.subi %a, %b : i32
arith.muli %a, %b : i32
arith.cmpi "eq", %a, %b : i32
arith.addf %a, %b : f64
arith.cmpf "olt", %a, %b : f64
```

[연산자 디스패치](../003-expressions/301-operator-dispatch.md) 의 산술과 비교 연산은 `arith` 다이얼렉트로 lower 됩니다.

다만 사용자 정의 트레잇 구현은 제네릭 단형화를 거친 뒤 함수 호출의 형태로 lower 된다는 점에 유의합니다.

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

[`if` 표현식](../003-expressions/310-if-expression.md) 이나 [루프](../003-expressions/313-loop-expressions.md) 의 정상적인 제어 흐름은 `scf` 로 lower 됩니다.

### `cf`

```
cf.cond_br %cond, ^true_block(args), ^false_block(args)
cf.switch %v : i32, [
  default: ^default,
  0: ^case0,
  1: ^case1
]
```

복잡한 [`match`](../003-expressions/312-match-expression.md) 의 decision tree 는 `cf` 다이얼렉트의 cond_br 와 switch 를 사용해 lower 됩니다.

### `index`

```
%idx = index.constant 5
%new_idx = index.add %idx, %step
```

`index` 타입이 필요한 자리 (`scf.for` 의 경계 등) 는 `index` 다이얼렉트로 lower 됩니다.

### `math`

```
math.sqrt %x : f64
math.sin %x : f64
math.cos %x : f64
```

표준 라이브러리의 수학 함수는 가능한 한 `math` 다이얼렉트로 표현합니다. `math` 가 직접 지원하지 않는 함수의 경우에는 일반적인 함수 호출로 처리합니다.

### `llvm`

LLVM 다이얼렉트는 어디까지나 마지막 수단으로 두고, 다른 다이얼렉트로는 표현할 수 없는 의미가 있을 때에만 사용합니다.

```
llvm.alloca / llvm.load / llvm.store
llvm.struct<(...)>
llvm.intr.expect / llvm.intr.assume
```

스택 슬롯, 객체 접근, 저수준 intrinsic 은 `llvm` 다이얼렉트의 영역입니다. 코루틴은 컴파일러가 만든 상태 기계이므로 `llvm.coro.*` 를 쓰지 않습니다 ([7.8](../007-runtime/708-coroutine.md)). GC 루트는 intrinsic 없이 일반 load / store 로 shadow stack frame 을 다룹니다 ([8.9.12](809-lowering-control.md#8912-shadow-stack-frame)). 에러 전달은 반환값과 일반 분기로 표현되므로 ([8.9.6](809-lowering-control.md#896-try--catch--finally-값-기반-분기)) `llvm` 다이얼렉트의 특수 기능을 쓰지 않습니다.

## 8.7.4. 다이얼렉트 사용 우선순위

lowering 은 의미를 표현할 수 있는 곳에서 `scf`, `index`, `math` 를, 그 밖의 것으로는 표현할 수 없는 곳에서 `llvm` 을 쓸 수 있으며, 그 선택은 구현이 정합니다 (8.7.3).

## 8.7.5. 라이브러리 / 가속 영역 다이얼렉트

다음 다이얼렉트들은 Rok 의 *일반 코드* lowering 에서는 사용하지 않지만, 라이브러리 작성자가 [`rok::bare::mlir`](../../refs-standard-libraries/100-bare/109-mlir.md) 를 통해 직접 활용할 수 있습니다.

| 다이얼렉트 | 활용 영역                                     |
|------------|-----------------------------------------------|
| `affine`   | affine 분석 가능한 루프 (수치 라이브러리)     |
| `linalg`   | element-wise / matmul / reduction (텐서 lib)  |
| `tensor`   | shape / extract / insert / reshape (텐서 lib) |
| `vector`   | SIMD 폭 / 정렬 / shuffle (SIMD lib)           |
| `gpu`      | GPU launch / kernel / shared memory (GPU lib) |
| `async`    | async fork / await (전용 런타임 라이브러리)   |

일반 Rok 코드의 lowering 은 8.7.1 의 일곱 다이얼렉트만으로 충분하지만, 텐서 / 자동 미분 / SIMD / GPU 같은 가속 영역의 라이브러리는 위 다이얼렉트를 직접 호출하여 PyTorch / JAX / Mojo 와 같은 성능 표면을 구성합니다.

라이브러리 작성자가 다이얼렉트를 호출하는 메커니즘과 안전 책임은 [`rok::bare::mlir`](../../refs-standard-libraries/100-bare/109-mlir.md) 에서 정의합니다.

## 8.7.6. `extern` function (런타임 함수)

[Extern "C" 심볼](805-extern-symbols.md) 에 등장하는 `rok_*` 함수들은 외부 `llvm.func` 선언으로 선언합니다 (선언 형태는 구현이 정함).

```
llvm.func @rok_gc_alloc_slow(i64, i64, i32) -> !llvm.ptr
llvm.func @rok_runtime_yield()
```

호출은 `llvm.call` 로 이루어집니다.

## 8.7.7. TLS 접근

[Mutator TLS 슬롯](803-tls-slots.md) 에서 다루는 TLS 슬롯 접근에는 `llvm` 다이얼렉트의 thread_local global `rok_tls` 를 각 슬롯의 offset 으로 사용합니다.

```
llvm.mlir.global external thread_local @rok_tls() : !llvm.array<N x i8>

%t = llvm.mlir.addressof @rok_tls : !llvm.ptr
%flag_ptr = llvm.getelementptr %t[88] : (!llvm.ptr) -> !llvm.ptr, i8   ; yield_flag at 10p+8, p = 8
%flag = llvm.load %flag_ptr : !llvm.ptr -> i8
```

타깃별 TLS 명령어로의 변환은 LLVM 백엔드가 자동으로 lower 합니다.

## 8.7.8. 각 단계의 책임

```
rok-mir → 표준 MLIR         rokc lowering (본 블록)
MLIR → LLVM IR              MLIR 패스와 변환, 프로세스 안
LLVM IR → object            LLVM, 프로세스 안, 요청한 -O 수준
object → 바이너리           lld, 프로세스 안 (ROK_CC 로 대체, 9.2.14)
```

## 8.7.9. 변환 검증

`tests/integration/` 아래의 end-to-end 테스트는 사용자가 작성한 .rok 파일을 빌드해서 실행한 결과를 가지고 검증을 수행합니다.

이 검증을 통해 본 블록에서 정의한 lowering 규칙이 정확하다는 점을 보증합니다.
