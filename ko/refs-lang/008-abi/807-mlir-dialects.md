# 8.7. MLIR 다이얼렉트 사용

[컴파일 파이프라인](../000-meta/002-pipeline.md) 에서 정한 표준 MLIR 다이얼렉트만 사용 원칙을 본 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 블록의
lowering 사양에 적용한다.

## 8.7.1. 사용 다이얼렉트 (재확인)

[컴파일 파이프라인](../000-meta/002-pipeline.md) 의 표 그대로:

| 다이얼렉트 | 용도                                     | 본 블록의 사용                                                          |
| ---------- | ---------------------------------------- | ----------------------------------------------------------------------- |
| `func`     | 함수 정의 / 호출                         | 함수 lowering                                                           |
| `arith`    | 정수 / 부동소수 산술 / 비교              | 연산자 lowering                                                         |
| `scf`      | 구조적 제어흐름 (`if` / `for` / `while`) | `if` / 루프 lowering                                                    |
| `cf`       | 비구조적 분기 (switch, cond_br)          | `match` decision tree                                                   |
| `memref`   | 스택 / 힙 메모리 참조                    | 객체 / 배열 lowering                                                    |
| `index`    | memref 인덱싱                            | 인덱싱 lowering                                                         |
| `math`     | 수학 함수                                | sqrt / sin / cos 등                                                     |
| `llvm`     | 저수준 op (intrinsic, `struct`, setjmp)  | [GC](../000-meta/007-glossary.md#gc-garbage-collector) / coroutine / EH |

커스텀 nok 다이얼렉트는 없다. 모든 nok 의 의미를 위 8 개 다이얼렉트로 표현한다.

## 8.7.2. 변환 흐름 정리

```
nok 소스
   ↓ (parse, name resolution, type check)
nok-hir
   ↓ (specialize / monomorphize, desugar)
nok-mir
   ↓ (lowering — 본 블록의 808 ~ 810)
표준 MLIR 텍스트  (위 8 개 다이얼렉트만)
   ↓ (mlir-opt — 표준 패스)
   ↓ (mlir-translate)
LLVM IR
   ↓ (clang / lld)
네이티브 바이너리
```

본 ABI 블록은 nok-mir → 표준 MLIR 텍스트 단계의 lowering 규칙을 정의한다.

## 8.7.3. 다이얼렉트 별 핵심 op

각 다이얼렉트의 nok 가 사용하는 핵심 op 들.

### `func`

```
func.func @name(%arg: i32) -> (i32) { ... }
func.call @name(%arg) : (i32) -> (i32)
func.return %val : i32
```

함수 정의 / 호출 / 반환의 표준.

### `arith`

```
arith.addi %a, %b : i32
arith.subi %a, %b : i32
arith.muli %a, %b : i32
arith.cmpi "eq", %a, %b : i32
arith.addf %a, %b : f64
arith.cmpf "olt", %a, %b : f64
```

[연산자 디스패치](../003-expressions/301-operator-dispatch.md) 의 산술 / 비교는 `arith` 다이얼렉트로 lower.

사용자 정의 트레잇 구현은 함수 호출로 lower (제네릭 단형화 후).

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

[`if` 표현식](../003-expressions/310-if-expression.md) / [루프](../003-expressions/313-loop-expressions.md) 의 정상 제어 흐름은 `scf` 로 lower.

### `cf`

```
cf.cond_br %cond, ^true_block(args), ^false_block(args)
cf.switch %v : i32, [
  default: ^default,
  0: ^case0,
  1: ^case1
]
```

복잡한 [`match`](../003-expressions/312-match-expression.md) 의 decision tree 는 `cf` 의 cond_br / switch 로 lower.

### `memref`

```
%ref = memref.alloca() : memref<i32>
memref.store %v, %ref[] : memref<i32>
%loaded = memref.load %ref[] : memref<i32>
```

객체의 필드 슬롯을 표현한다. GC 객체는 heap memref 로 lower (자세한 내용은 [lowering
타입](808-lowering-types.md) 에서 정의).

### `index`

```
%idx = index.constant 5
%new_idx = index.add %idx, %step
```

memref 인덱싱의 정수 자리는 `index` 다이얼렉트로 lower.

### `math`

```
math.sqrt %x : f64
math.sin %x : f64
math.cos %x : f64
```

표준 라이브러리의 수학 함수는 가능하면 `math` 다이얼렉트로. `math` 가 직접 지원 안 하는 함수
는 일반 함수 호출.

### `llvm`

LLVM 다이얼렉트는 마지막 수단. 다른 다이얼렉트가 표현 못 하는 의미만 사용한다.

```
llvm.gcroot
llvm.coro.id / llvm.coro.alloc / llvm.coro.begin / ...
llvm.call @setjmp / @longjmp
llvm.struct<(...)>
llvm.intr.expect / llvm.intr.assume
```

GC / coroutine / 예외 처리 / 저수준 intrinsic 은 `llvm` 다이얼렉트.

## 8.7.4. 다이얼렉트 사용 우선순위

```
가능하면 → arith / scf / cf / memref / index / math / func
필요시 → llvm
```

한 의미를 여러 다이얼렉트로 표현 가능하면 고차원 다이얼렉트 (scf, arith) 우선. `llvm` 은
그것 외에 표현 못 할 때만.

이 우선순위로 MLIR 의 표준 패스가 더 효과적으로 최적화한다.

## 8.7.5. 사용 안 하는 다이얼렉트

다음 표준 다이얼렉트는 본 사양 범위에서 사용하지 않는다.

| 다이얼렉트 | 이유                                         |
| ---------- | -------------------------------------------- |
| `affine`   | nok 의 일반 코드는 affine 분석 대상 아님     |
| `linalg`   | 추후 SIMD / GPU 통합 시 검토                 |
| `tensor`   | 추후 텐서 라이브러리 통합 시 검토            |
| `vector`   | 추후 SIMD 통합 시 검토                       |
| `gpu`      | GPU 코드 생성 시 (추후)                      |
| `async`    | nok 자체 스케줄러 + LLVM coroutine 으로 대체 |

이들은 현재 사양에서 사용하지 않을 뿐, 영원히 배제하는 것은 아니다. 추후 라이브러리나 도구가
활용 가능하다.

## 8.7.6. `extern` function (런타임 함수)

[Extern "C" 심볼](805-extern-symbols.md) 의 `nok_*` 함수들은 `func.func` 의 external declaration 으로
선언한다.

```
func.func private @nok_gc_alloc_slow(i64, i64, i32) -> !llvm.ptr
func.func private @nok_runtime_yield()
```

호출은 `func.call` 또는 `llvm.call`.

## 8.7.7. TLS 접근

[Mutator TLS 슬롯](803-tls-slots.md) 의 TLS 슬롯 접근은 `llvm` 다이얼렉트의 thread_local global.

```
llvm.mlir.global thread_local @nok_tls_yield_flag(0 : i8) : i8
llvm.mlir.global thread_local @nok_tls_tlab_cursor(0 : i64) : i64
...

%flag_ptr = llvm.mlir.addressof @nok_tls_yield_flag
%flag = llvm.load %flag_ptr : !llvm.ptr -> i8
```

타깃 별 TLS 명령어로 LLVM 백엔드가 자동 lower.

## 8.7.8. 각 단계의 책임

```
nokc:           nok-mir → 표준 MLIR 텍스트  (본 블록)
mlir-opt:       표준 MLIR → 최적화된 MLIR  (외부 도구)
mlir-translate: MLIR → LLVM IR  (외부 도구)
clang/lld:      LLVM IR → 네이티브  (외부 도구)
```

`nokc` 의 책임은 표준 MLIR 텍스트 생성까지. 그 뒤 단계는 외부 도구. `nokc` 가 그
명령어들을 호출하지만, 도구 자체는 외부다.

## 8.7.9. 변환 검증

`tests/integration/` 의 end-to-end 테스트는 사용자 .nok → 빌드 → 실행 결과로 검증한다.

이 검증이 본 블록의 lowering 규칙이 정확함을 보증한다.
