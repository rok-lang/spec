# 8.8. Lowering: 데이터 타입과 표현식

nok 의 값, 타입, 식이 MLIR 표준 다이얼렉트로 어떻게 lower 되는지 정의한다.

## 8.8.1. 원시 정수 / 부동소수

| nok 타입          | MLIR 타입                                                                                   |
| ----------------- | ------------------------------------------------------------------------------------------- |
| `i8` ~ `i64`      | `i8` ~ `i64`                                                                                |
| `u8` ~ `u64`      | `i8` ~ `i64` (sign 은 op 가 결정)                                                           |
| `usize` / `isize` | `index` 또는 타깃별 `iN`                                                                    |
| `f32` / `f64`     | `f32` / `f64`                                                                               |
| `bool`            | `i1` (또는 i8, [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 에 따라) |
| `char`            | `i32` (Unicode 코드포인트)                                                                  |
| `c_char`          | `i8`                                                                                        |

MLIR 의 정수는 부호 없음. 부호는 op (`addi` / `subi` 등) 가 결정한다.

## 8.8.2. 산술 연산

```
a + b            →   arith.addi %a, %b : i32
a - b            →   arith.subi %a, %b : i32
a * b            →   arith.muli %a, %b : i32
a / b            →   arith.divsi %a, %b : i32  (signed) / divui (unsigned)
a % b            →   arith.remsi / remui

a + b (f64)      →   arith.addf %a, %b : f64
a < b            →   arith.cmpi "slt", %a, %b : i32  (signed less than)
                       또는 arith.cmpf "olt", %a, %b : f64
```

[연산자 디스패치](../003-expressions/301-operator-dispatch.md) 의 트레잇 매핑이 원시 타입에 대해 위와 같이 lower 된다.

사용자 정의 트레잇 구현은 함수 호출로 lower (제네릭 단형화 후).

## 8.8.3. `struct` 의 표현

[`struct`](../002-types/203-struct.md) 의 named `struct`.

```nok
struct Point { x: f64; y: f64; }
```

→ MLIR:

```
!Point = !llvm.struct<"Point", (
    i32,        // type_id
    i32,        // gc_flags
    i64,        // gc_word
    f64,        // x
    f64         // y
)>
```

또는 `memref<!Point>` / `!llvm.ptr` 로 포인터 형태.

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 객체는 항상 heap 에 있다 ([TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 할당). 스택 할당은 escape 분석 후 가능 (CoroElide 와 비슷).

### 필드 접근

```nok
p.x
```

→ MLIR (포인터를 통한 접근):

```
%addr = llvm.getelementptr %p [0, 3] : (!llvm.ptr) -> !llvm.ptr
%val = llvm.load %addr : !llvm.ptr -> f64
```

[0, 3] 은 `struct` 의 4 번째 멤버 (x, 0-base index 3, 헤더 3 슬롯 후).

### 필드 쓰기

```nok
p.x = 1.0f64
```

→ MLIR:

```
%addr = llvm.getelementptr %p [0, 3] : (!llvm.ptr) -> !llvm.ptr
llvm.store %val, %addr : f64, !llvm.ptr
```

write barrier 가 필요하면 (sticky 비트 검사 + remembered set) [배리어와 루트](../007-runtime/704-barriers-and-roots.md) 의
규칙에 따라 추가 코드를 emit 한다.

## 8.8.4. tuple `struct` 의 표현

```nok
struct Pair(i32, i32);
```

→ `struct` 와 동일. 필드 이름 대신 위치 인덱스.

## 8.8.5. tuple 타입

```nok
(i32, str, bool)
```

→ `!llvm.struct<(i32, !llvm.ptr, i1)>`.

GC 객체로 처리되는 경우 헤더 추가.

## 8.8.6. `enum` 의 표현

[`enum`](../002-types/204-enum.md) 의 `enum` 은 tagged `union` 형태로 lower.

```nok
enum Result {
    Ok(i32),
    Err(str),
    Pending,
}
```

→ MLIR:

```
!Result = !llvm.struct<"Result", (
    i32,                                // type_id
    i32,                                // gc_flags
    i64,                                // gc_word
    i32,                                // discriminant (tag)
    !llvm.struct<(i64)>                 // 가장 큰 변형의 payload (8 바이트)
)>
```

- discriminant 는 변형 식별자 (`Ok`=0, `Err`=1, `Pending`=2).
- payload 는 모든 변형의 `union`. 가장 큰 변형의 크기.

### 변형 생성

```nok
Result::Ok(42)
```

→ MLIR:

```
; struct 할당 + 헤더 채우기 + tag 채우기 + payload 채우기
%obj = call @nok_gc_alloc_slow(...)
; ... 헤더 ...
%tag_addr = llvm.getelementptr %obj [0, 3] : ...
llvm.store i32 0, %tag_addr  ; Ok = 0
%payload_addr = llvm.getelementptr %obj [0, 4] : ...
llvm.store %42_i32, %payload_addr
```

### 변형 매칭

```nok
match v {
    Ok(n) => ...,
    Err(s) => ...,
    Pending => ...,
}
```

→ MLIR (decision tree):

```
%tag = llvm.load %v[3] : i32

cf.switch %tag, [
  default: ^unreachable,    ; 모든 변형이 case 에 있어 default 도달 안 함
  0: ^ok,
  1: ^err,
  2: ^pending
]

^ok:
  %payload = llvm.load %v[4] : i32
  ; ... n 으로 사용 ...
  br ^end

^err:
  %payload_str = llvm.load %v[4] : !llvm.ptr
  ; ... s 로 사용 ...
  br ^end

^pending:
  ; ...
  br ^end
```

## 8.8.7. `union` (비태그) 의 표현

[`union`](../002-types/205-union.md) 의 `union`.

```nok
union EpollData {
    ptr: usize;
    fd: i32;
    val: u64;
}
```

→ `!llvm.struct<(i64)>` (가장 큰 필드 크기). 각 필드 접근은 bitcast / getelementptr 로
같은 메모리를 다른 형으로 해석한다.

## 8.8.8. 합 타입 (`A | B`) 의 표현

[합 타입](../002-types/206-sum-types.md) 의 합 타입.

```nok
type X = i32 | f64
```

→ `!llvm.struct<(i32, i64)>`. tag (i32) + payload (i64, 가장 큰 변형 크기).

niche 최적화 ([niche 최적화](../002-types/207-niche-optimization.md)) 가 적용되면 tag 자리가 사라지고 합집합 비트 패턴만 남는다.

```nok
Ptr<u8> | none
```

→ `!llvm.ptr` (단일 포인터, null = `none`. niche 자동).

## 8.8.9. `any` 슬롯의 표현

[`any` 슬롯](../002-types/202-any-slot.md) 의 동적 슬롯.

```nok
let x: any = ...
```

→ `!llvm.struct<(i32, i64)>`. tag (type_id 또는 종류 식별자) + payload.

```
!Any = !llvm.struct<(i32, i64)>
```

작은 원시 값은 payload 직접. 큰 값은 payload 가 포인터.

### `any` 의 동적 디스패치

```nok
v + 1i32     // v: any
```

→ MLIR (런타임 검사):

```
%tag = llvm.extractvalue %v[0] : i32

cf.switch %tag, [
  default: ^trap,
  TAG_I32: ^add_i32,
  TAG_F64: ^add_f64,
  ...
]

^add_i32:
  %payload = llvm.extractvalue %v[1] : i64
  %i = llvm.trunc %payload : i64 to i32
  %sum = arith.addi %i, %1_const : i32
  ; 결과를 any 로 다시 wrap
  ...
```

이 분기 구조는 런타임에 모든 가능 타입을 검사해야 한다. `any` 슬롯의 비용.

## 8.8.10. 배열 타입: `[T; N]`

[타입 표현식](../002-types/201-type-expressions.md) 의 고정 크기 배열.

```nok
[i32; 8]
```

→ `!llvm.array<8 x i32>` 또는 `memref<8xi32>`.

GC 객체 안에 있으면 16 바이트 헤더 + 배열.

## 8.8.11. 함수 타입: `fn(...) -> ...`

[함수 타입](../004-functions/404-function-type.md).

```nok
fn(i32, i32) -> i32
```

→ `(i32, i32) -> i32` 의 함수 포인터. `!llvm.ptr` 또는 `!llvm.func<...>`.

값으로서 함수 포인터 값이 변수에 저장 가능하다.

## 8.8.12. 클로저의 표현

[클로저](../004-functions/405-closure.md) 의 캡처가 있는 클로저.

```nok
let n = 10;
let f = |x| x + n;       // n 캡처
```

→ MLIR:

```
; 클로저 객체 = 환경 + 코드 포인터
!Closure = !llvm.struct<(
    i32, i32, i64,                ; GC header
    !llvm.ptr,                    ; 함수 포인터 (코드)
    i32                            ; 환경: n 의 복사본
)>

; 호출 시
%env = llvm.getelementptr %clo[0, 4] : ...
%fn_ptr = llvm.extractvalue %clo[0, 3] : !llvm.ptr
%result = llvm.call %fn_ptr(%env, %x) : ...
```

환경이 함수의 추가 첫 인자로 전달된다.

## 8.8.13. 식별자 / 변수 접근

[변수 선언](../005-modules/501-variable-declaration.md) 의 지역 변수.

```nok
let x = 42i32;
let y = x + 1i32;
```

→ MLIR (LLVM 의 SSA 형태):

```
%x = arith.constant 42 : i32
%y = arith.addi %x, %1 : i32
```

지역 변수는 SSA 로 lower. 변경 가능한 변수는 `memref.alloca` + load/store.

```nok
let counter = 0i32;
counter = counter + 1i32;
```

→

```
%counter_ref = memref.alloca() : memref<i32>
memref.store %0, %counter_ref[] : memref<i32>

%loaded = memref.load %counter_ref[] : memref<i32>
%new = arith.addi %loaded, %1 : i32
memref.store %new, %counter_ref[] : memref<i32>
```

LLVM 의 mem2reg 패스가 이를 SSA 로 다시 promote 한다.

## 8.8.14. 리터럴 desugar

[`#[literal]` 매직](../006-meta-system/605-literal-magic.md) 의 컬렉션 리터럴.

```nok
[1, 2, 3]
```

→ 표준 라이브러리의 `list_literal(1, 2, 3)` 함수 호출. 일반 함수 호출 lowering.

문자열 / 숫자 리터럴은 컴파일 타임 `const` 로 처리한다. .rodata 에 저장 + 포인터 / 값 사용.
