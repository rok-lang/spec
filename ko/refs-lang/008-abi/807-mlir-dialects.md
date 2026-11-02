# 8.7. MLIR 다이얼렉트 사용

[컴파일 파이프라인](../000-meta/002-pipeline.md)에서 정한 표준 MLIR 다이얼렉트를 이 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 블록의 lowering 사양에 적용합니다. 일반 코드는 upstream 다이얼렉트 5개로 낮춥니다. 가속 영역의 라이브러리는 8.7.5에서 다룹니다. 이 문서는 MLIR → LLVM 백엔드(`-O1` 이상)의 lowering을 다룹니다. 네이티브 백엔드(`-O0`, REPL)는 rok-mir에서 바로 기계어를 만들고 같은 ABI를 따릅니다([0.2.4](../000-meta/002-pipeline.md#024-백엔드)).

## 8.7.1. 일반 코드 사용 다이얼렉트(재확인)

[컴파일 파이프라인](../000-meta/002-pipeline.md)의 표를 다시 옮기면 다음과 같습니다.

| 다이얼렉트 | 용도                                     | 이 블록의 사용                                                          |
|------------|------------------------------------------|-------------------------------------------------------------------------|
| `func`     | 함수 정의 / 호출                         | 함수 lowering, Rok 함수와 대부분의 런타임 함수 호출                     |
| `arith`    | 정수 / 부동소수 산술 / 비교              | 연산자 lowering, 부동소수 `min` / `max`                                 |
| `math`     | 정확한 부동소수 함수                     | `sqrt`, 반올림 계열, `abs`, `copysign`, `mul_add`                       |
| `cf`       | 분기 (`cf.br`, `cf.cond_br`)             | 모든 제어 흐름: `if`, 루프, `match`, `try`                              |
| `llvm`     | 저수준 op (intrinsic, `struct`, `alloca`, load / store) | 스택 슬롯, 객체 / 배열, [GC](../000-meta/007-glossary.md#gc-garbage-collector), C 호출 |

커스텀 Rok 다이얼렉트는 없습니다. 일반 Rok 코드의 의미는 위의 표준 다이얼렉트 다섯 개로 표현합니다. 가속 라이브러리는 8.7.5에서 다룹니다. `scf`와 `index`는 내보내지 않습니다. rok-mir는 이미 기본 블록으로 된 제어 흐름 그래프이고, `scf`를 `cf`로 바꾸는 변환은 같은 분기를 만들기 때문입니다(8.7.4).

## 8.7.2. 변환 흐름

전체 변환 흐름은 다음과 같습니다.

```
rok 소스
   ↓ (parse, name resolution, type check)
rok-hir
   ↓ (specialize / monomorphize, desugar)
rok-mir
   ↓ (lowering: 본 블록의 808 ~ 810)
표준 MLIR  (일반 코드는 5 개 다이얼렉트, 라이브러리는 추가 가능)
   ↓ (표준 패스, LLVM 다이얼렉트로 변환)
LLVM IR
   ↓ (LLVM 최적화와 코드 생성)
object
   ↓ (lld)
네이티브 바이너리
```

모든 단계는 `rokc` 안에서 실행됩니다. 이 ABI 블록은 그중 rok-mir에서 표준 MLIR로 가는 단계의 lowering 규칙을 정의합니다.

## 8.7.3. 다이얼렉트별 핵심 op

`rokc`는 `func`, `arith`, `math`, `cf`, `llvm`을 내보냅니다. 아래는 각 다이얼렉트의 핵심 op입니다.

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

### `cf`

```
cf.br ^next(args)
cf.cond_br %cond, ^true_block(args), ^false_block(args)
```

rok-mir의 분기는 모두 `cf.br`(goto) 또는 `cf.cond_br`(두 갈래 `if`)입니다. [`if` 표현식](../003-expressions/310-if-expression.md), [루프](../003-expressions/313-loop-expressions.md), [`match`](../003-expressions/312-match-expression.md)의 decision tree, 에러 전달이 모두 그렇습니다. 태그나 정수에 대한 `match`는 같은 값을 각 상수와 비교하는 `cf.cond_br` 사슬입니다(8.9.3). `-O1` 이상에서 LLVM 최적화기가 이 사슬을 하나의 `switch`(점프 테이블이나 이진 탐색)로 바꿉니다.

### `math`

```
math.sqrt %x : f64
math.floor %x : f64
math.fma %a, %b, %c : f64
```

결과가 정확하거나 올바르게 반올림되는 부동소수 메서드는 인라인으로 낮춥니다. 그래서 최적화기는 이들을 LLVM intrinsic으로 봅니다(대부분의 타깃에서 명령어 하나).

| 메서드                                  | op                                         |
|-----------------------------------------|--------------------------------------------|
| `sqrt`                                  | `math.sqrt`                                |
| `floor` / `ceil` / `trunc`              | `math.floor` / `math.ceil` / `math.trunc`  |
| `round` (0에서 먼 쪽으로)               | `math.round`                               |
| `abs` / `copysign`                      | `math.absf` / `math.copysign`              |
| `mul_add` (반올림 한 번)                | `math.fma`                                 |
| `min` / `max` (NaN 피연산자는 무시)     | `arith.minnumf` / `arith.maxnumf`          |

`f64`와 `f32` 모두 같습니다. 나머지 수학 함수(`sin`, `exp`, `ln`, `pow`, `powi`, `cbrt`, `hypot`, ..)는 libm을 부르는 런타임의 `rok_f64_*` / `rok_f32_*` 함수 호출입니다([8.5](805-extern-symbols.md)). 그래서 결과는 항상 타깃의 libm이 내고, 호스트에서 컴파일 시간에 계산하지 않습니다.

### `llvm`

LLVM 다이얼렉트는 마지막 수단입니다. 다른 다이얼렉트로 표현할 수 없는 의미에만 씁니다.

```
llvm.alloca / llvm.load / llvm.store
llvm.struct<(...)>
llvm.intr.expect / llvm.intr.assume
```

스택 슬롯, 객체 접근, C 호출(`byval` / `sret` 속성이 붙은 `llvm.call`, 8.6), 저수준 intrinsic(`llvm.intr.ctpop`, `llvm.intr.ctlz`, `llvm.call_intrinsic`을 거치는 포화 변환)은 `llvm` 다이얼렉트의 영역입니다. 코루틴은 컴파일러가 만든 상태 기계이므로 `llvm.coro.*`를 쓰지 않습니다([7.8](../007-runtime/708-coroutine.md)). GC 루트는 intrinsic 없이 일반 load / store로 shadow stack 프레임을 다룹니다([8.9.12](809-lowering-control.md#8912-shadow-stack-프레임)). 에러 전달은 반환값과 일반 분기로 표현하므로([8.9.6](809-lowering-control.md#896-try--catch--finally-값-기반-분기)) `llvm` 다이얼렉트의 특수 기능을 쓰지 않습니다.

## 8.7.4. 다이얼렉트 사용 우선순위

`arith`, `math`, `func`의 op로 의미를 표현할 수 있으면 그 op를 쓰고, 그럴 수 없는 곳에서만 `llvm`을 씁니다. 제어 흐름은 `cf`에 둡니다. rok-mir에는 되살릴 구조적 영역이 없고, `scf`는 LLVM IR로 가기 전에 같은 `cf` 분기로 되돌아가므로 얻는 것이 없습니다. MLIR 파이프라인은 (`-O1` 이상에서) `canonicalize`와 `cse`를 돌린 뒤 `math`, `func`, `arith`, `cf`를 `llvm` 다이얼렉트로 변환합니다.

## 8.7.5. 가속 영역 라이브러리

Rok 소스는 MLIR에 직접 접근하지 않습니다. MLIR은 컴파일러만 만들고, 8.7.1의 다이얼렉트만 씁니다. 텐서 / 자동 미분 / SIMD / GPU 같은 가속 영역의 라이브러리(`affine`, `linalg`, `tensor`, `vector`, `gpu`, `async`)는 Rok 0.2.2에 도입할 예정인 백엔드 인터페이스로 컴파일러 백엔드에 닿습니다.

## 8.7.6. `extern` 함수(런타임 함수)

[Extern "C" 심볼](805-extern-symbols.md)에 나오는 `rok_*` 함수는 `func.func private`로 선언하고 `func.call`로 부릅니다. 참조와 헤더만 쓰는 세 함수(`rok_runtime_yield`, `rok_gc_alloc_slow`, `rok_gc_wb_slow`)는 `llvm.func`로 선언하고 `llvm.call`로 부릅니다. `llvm.call`은 호출 너머에서도 다른 힙 load가 유효하도록 지키는 TBAA 태그를 실을 수 있기 때문입니다.

```
func.func private @rok_str_cmp(!llvm.struct<(ptr, i64)>, !llvm.struct<(ptr, i64)>) -> i32
llvm.func @rok_gc_alloc_slow(i64, i64, i32) -> !llvm.ptr
llvm.func @rok_runtime_yield()
```

아예 부르지 않는 런타임 함수도 있습니다. `rok_str_len`과 `rok_str_byte`는 `str` 쌍을 바로 읽고, `rok_str_eq`는 16바이트까지의 문자열을 인라인으로 비교하며(그보다 길면 호출), 정확한 부동소수 함수는 `math` op입니다(8.7.3).

## 8.7.7. TLS 접근

[Mutator TLS 슬롯](803-tls-slots.md)의 TLS 슬롯에 접근할 때는 `llvm` 다이얼렉트의 thread_local 전역 `rok_tls`에 각 슬롯의 오프셋을 더합니다.

```
llvm.mlir.global external thread_local @rok_tls() : !llvm.array<N x i8>

%g = llvm.mlir.addressof @rok_tls : !llvm.ptr
%t = "llvm.intr.threadlocal.address"(%g) : (!llvm.ptr) -> !llvm.ptr
%flag_ptr = llvm.getelementptr %t[88] : (!llvm.ptr) -> !llvm.ptr, i8   ; yield_flag at 10p+8, p = 8
%flag = llvm.load %flag_ptr : !llvm.ptr -> i8
```

스레드의 기준 주소는 entry 블록에서 한 번 계산해 모든 슬롯에 다시 씁니다. LLVM 백엔드가 이를 타깃별 TLS 명령어로 낮춥니다.

## 8.7.8. 각 단계의 책임

```
rok-mir → 표준 MLIR         rokc lowering (본 블록)
MLIR → LLVM IR              MLIR 패스와 변환, 프로세스 안
LLVM IR → object            LLVM, 프로세스 안, 요청한 -O 수준
object → 바이너리           lld, 프로세스 안 (ROK_CC 로 대체, 9.2.14)
```

## 8.7.9. 변환 검증

종단 간(end-to-end) 테스트(`compiler/rokc/tests/integration.rs`, `compiler/rokc/tests/cases/`의 프로그램 대상)는 .rok 파일을 빌드하고 실행합니다. `-O0`에서 돌리고, `ROK_TEST_OPT=-O2`이면 이 lowering을 거칩니다.

이 검증이 이 블록에서 정의한 lowering 규칙이 정확함을 보장합니다.
