# 8.8. Lowering: 데이터 타입과 표현식

이 절에서는 rok 의 값과 타입, 그리고 식이 MLIR 표준 다이얼렉트로 어떻게 lower 되는지를 정의합니다.

## 8.8.1. 원시 정수 / 부동소수

원시 수치 타입은 다음과 같이 MLIR 타입으로 매핑됩니다.

| rok 타입          | MLIR 타입                                                                                   |
|-------------------|---------------------------------------------------------------------------------------------|
| `i8` ~ `i64`      | `i8` ~ `i64`                                                                                |
| `u8` ~ `u64`      | `i8` ~ `i64` (sign 은 op 가 결정)                                                           |
| `usize` / `isize` | `index` 또는 타깃별 `iN`                                                                    |
| `f32` / `f64`     | `f32` / `f64`                                                                               |
| `bool`            | `i1` (또는 i8, [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 에 따라) |
| `char`            | `i32` (Unicode 코드포인트)                                                                  |
| `c_char`          | `i8`                                                                                        |

`str` 은 `!llvm.struct<(ptr, iN)>` (포인터와 `usize` 바이트 길이) 로 값 전달되는 불변 쌍입니다. 리터럴은 immortal 상수를, 런타임에 만든 문자열은 GC 객체 (type id `1`, [8.2.12](802-scan-table.md#8212-런타임-예약-type-id)) 의 payload 를 가리킵니다. `str` 슬롯은 GC 가 scan 할 때 `ptr` 을 객체 내부 포인터로 다룹니다.

여기서 한 가지 유의할 점이 있습니다. MLIR 의 정수 타입 자체에는 부호 정보가 없으며, 부호 해석은 연산자 (`addi` 나 `subi` 등) 가 결정합니다.

## 8.8.2. 산술 연산

산술 식은 아래와 같이 `arith` 다이얼렉트의 연산으로 lower 됩니다.

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

[연산자 디스패치](../003-expressions/301-operator-dispatch.md) 의 트레잇 매핑은 원시 타입에 대해 위와 같이 lower 됩니다. 한편 사용자 정의 트레잇 구현은 제네릭 단형화를 거친 뒤 함수 호출로 lower 됩니다.

## 8.8.3. `struct` 의 표현

[`struct`](../002-types/203-struct.md) 의 named `struct` 를 살펴보겠습니다.

```rok
struct Point { x: f64; y: f64; }
```

이는 다음과 같은 MLIR 로 lower 됩니다.

```
!Point = !llvm.struct<"Point", (
    i32,        // type_id
    i32,        // gc_flags
    i64,        // gc_word
    f64,        // x
    f64         // y
)>
```

또한 `memref<!Point>` 나 `!llvm.ptr` 와 같은 포인터 형태로도 표현될 수 있습니다.

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 객체는 항상 heap 에 자리하며 ([TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 할당), 스택 할당은 escape 분석을 거친 뒤에 가능합니다 (CoroElide 와 비슷한 방식입니다).

### 필드 접근

다음과 같이 필드에 접근하는 식을 봅니다.

```rok
p.x
```

이는 포인터를 통한 접근으로 lower 됩니다.

```
%addr = llvm.getelementptr %p [0, 3] : (!llvm.ptr) -> !llvm.ptr
%val = llvm.load %addr : !llvm.ptr -> f64
```

여기서 [0, 3] 은 `struct` 의 4 번째 멤버 (즉 x, 0-base 로 인덱스 3, 헤더 3 슬롯 다음) 를 가리킵니다.

### 필드 쓰기

필드에 값을 쓰는 식은 다음과 같이 처리됩니다.

```rok
p.x = 1.0f64
```

해당 식은 다음 MLIR 로 lower 됩니다.

```
%addr = llvm.getelementptr %p [0, 3] : (!llvm.ptr) -> !llvm.ptr
llvm.store %val, %addr : f64, !llvm.ptr
```

만약 write barrier 가 필요한 경우라면 (sticky 비트 검사와 remembered set 등), [배리어와 루트](../007-runtime/704-barriers-and-roots.md) 의 규칙에 따라 추가 코드를 emit 합니다.

## 8.8.4. tuple `struct` 의 표현

다음과 같은 tuple `struct` 를 살펴보겠습니다.

```rok
struct Pair(i32, i32);
```

이는 일반 `struct` 와 동일하게 표현되며, 다만 필드 이름 대신 위치 인덱스로 접근하게 됩니다.

## 8.8.5. tuple 타입

다음과 같은 tuple 타입은

```rok
(i32, str, bool)
```

`!llvm.struct<(i32, !llvm.ptr, i1)>` 로 lower 됩니다. 만약 GC 객체로 처리되는 경우라면 헤더가 추가됩니다.

## 8.8.6. `enum` 의 표현

[`enum`](../002-types/204-enum.md) 의 `enum` 은 tagged `union` 형태로 lower 됩니다.

```rok
enum Result {
    Ok(i32),
    Err(str),
    Pending,
}
```

이는 다음과 같이 표현됩니다.

```
!Result = !llvm.struct<"Result", (
    i32,                                // type_id
    i32,                                // gc_flags
    i64,                                // gc_word
    i32,                                // discriminant (tag)
    !llvm.struct<(i64)>                 // 가장 큰 변형의 payload (8 바이트)
)>
```

여기서 discriminant 는 변형 식별자 (`Ok`=0, `Err`=1, `Pending`=2) 를 담고, payload 는 모든 변형의 `union` 으로서 가장 큰 변형의 크기를 갖습니다.

### 변형 생성

변형을 생성하는 식은 다음과 같습니다.

```rok
Result::Ok(42)
```

이는 다음 MLIR 시퀀스로 lower 됩니다.

```
; struct 할당 + 헤더 채우기 + tag 채우기 + payload 채우기
%obj = call @rok_gc_alloc_slow(...)
; ... 헤더 ...
%tag_addr = llvm.getelementptr %obj [0, 3] : ...
llvm.store i32 0, %tag_addr  ; Ok = 0
%payload_addr = llvm.getelementptr %obj [0, 4] : ...
llvm.store %42_i32, %payload_addr
```

### 변형 매칭

변형을 매칭하는 식을 살펴봅니다.

```rok
match v {
    Ok(n) => ...,
    Err(s) => ...,
    Pending => ...,
}
```

이는 decision tree 형태의 MLIR 로 lower 됩니다.

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

[`union`](../002-types/205-union.md) 의 `union` 을 살펴보겠습니다.

```rok
union EpollData {
    ptr: usize;
    fd: i32;
    val: u64;
}
```

이는 가장 큰 필드 크기를 갖는 `!llvm.struct<(i64)>` 로 lower 됩니다. 각 필드 접근은 bitcast 나 getelementptr 를 통해 같은 메모리를 다른 형으로 해석하는 방식으로 처리됩니다.

## 8.8.8. 합 타입 (`A | B`) 의 표현

[합 타입](../002-types/206-sum-types.md) 의 합 타입은 다음과 같이 다룹니다.

```rok
type X = i32 | f64
```

이는 `!llvm.struct<(i32, i64)>` 로 표현되며, tag (i32) 와 payload (i64, 가장 큰 변형 크기) 의 조합입니다.

만약 niche 최적화 ([niche 최적화](../002-types/207-niche-optimization.md)) 가 적용된다면 tag 자리가 사라지고 합집합 비트 패턴만 남게 됩니다.

```rok
Ptr<u8> | none
```

위 타입은 `!llvm.ptr` (단일 포인터, null 이 곧 `none` 을 의미하는 자동 niche) 로 lower 됩니다.

## 8.8.9. `any` 슬롯의 표현

[`any` 슬롯](../002-types/202-any-slot.md) 의 동적 슬롯을 살펴봅니다.

```rok
let x: any = ...
```

이는 `(i32 tag, usize payload)` 쌍으로 lower 되며, tag 는 type_id 입니다. `any Tr` 과 맨 `any` 의 표현은 같습니다.

```
!Any = !llvm.struct<(i32, iN)>      ; iN = 타깃의 포인터 폭 (usize)
```

포인터 폭 이하의 숫자·`bool`·`char` 값은 payload 에 직접 담깁니다. 그 밖의 값은 GC heap 의 box 이고 payload 는 그 포인터입니다. 따라서 payload 가 포인터인지는 tag 로만 알 수 있으며, GC 는 `any` 슬롯을 scan 할 때 tag 를 보고 판단해야 합니다.

### `any` 의 동적 디스패치

다음 식을 봅니다.

```rok
v + 1i32     // v: any
```

이는 런타임 검사를 동반하는 다음 MLIR 로 lower 됩니다.

```
%tag = llvm.extractvalue %v[0] : i32

cf.switch %tag, [
  default: ^trap,
  TAG_I32: ^add_i32,
  TAG_F64: ^add_f64,       ; 다른 피연산자 타입 불일치는 ^trap (panic, exit 101)
  ...
]

^add_i32:
  %payload = llvm.extractvalue %v[1] : i64     ; 64-bit 타깃
  %i = llvm.trunc %payload : i64 to i32
  %sum = arith.addi %i, %1_const : i32
  ; 결과를 any 로 다시 wrap
  ...
```

이러한 분기 구조는 런타임에 모든 가능한 타입을 검사해야 한다는 점에서, `any` 슬롯이 가지는 비용이라 할 수 있습니다.

## 8.8.10. 배열 타입: `[T; N]`

[타입 표현식](../002-types/201-type-expressions.md) 의 고정 크기 배열을 살펴보겠습니다.

```rok
[i32; 8]
```

이는 `!llvm.array<8 x i32>` 또는 `memref<8xi32>` 로 lower 됩니다. GC 객체 안에 자리하는 경우라면 16 바이트 헤더가 배열 앞에 붙습니다.

## 8.8.11. 함수 타입: `fn(...) -> ...`

[함수 타입](../004-functions/404-function-type.md) 을 살펴봅니다.

```rok
fn(i32, i32) -> i32
```

이는 `(i32, i32) -> i32` 타입의 함수 포인터, 즉 `!llvm.ptr` 또는 `!llvm.func<...>` 로 lower 됩니다. 함수 포인터 값은 값으로서 변수에 저장될 수 있습니다.

## 8.8.12. 클로저의 표현

[클로저](../004-functions/405-closure.md) 중 캡처가 있는 클로저를 살펴봅니다.

```rok
let n = 10;
let f = |x| x + n;       // n 캡처
```

이는 다음과 같은 MLIR 로 lower 됩니다.

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

호출 시 환경은 함수의 추가 첫 인자로 전달됩니다.

## 8.8.13. 식별자 / 변수 접근

[변수 선언](../005-modules/501-variable-declaration.md) 의 지역 변수를 살펴보겠습니다.

```rok
let x = 42i32;
let y = x + 1i32;
```

이는 LLVM 의 SSA 형태로 lower 됩니다.

```
%x = arith.constant 42 : i32
%y = arith.addi %x, %1 : i32
```

지역 변수는 SSA 로 lower 되며, 변경 가능한 변수의 경우에는 `memref.alloca` 와 load/store 의 조합으로 처리됩니다.

```rok
let counter = 0i32;
counter = counter + 1i32;
```

이는 다음과 같이 lower 됩니다.

```
%counter_ref = memref.alloca() : memref<i32>
memref.store %0, %counter_ref[] : memref<i32>

%loaded = memref.load %counter_ref[] : memref<i32>
%new = arith.addi %loaded, %1 : i32
memref.store %new, %counter_ref[] : memref<i32>
```

이렇게 생성된 코드는 LLVM 의 mem2reg 패스에 의해 다시 SSA 로 promote 됩니다.

## 8.8.14. 리터럴 desugar

[`#[literal]` 매직](../006-meta-system/605-literal-magic.md) 의 컬렉션 리터럴을 살펴봅니다.

```rok
[1, 2, 3]
```

이는 표준 라이브러리의 `list_literal(1, 2, 3)` 함수 호출로 desugar 되어, 일반적인 함수 호출 lowering 을 따라갑니다.

한편 문자열과 숫자 리터럴은 컴파일 타임 `const` 로 처리되어 .rodata 에 저장되며, 이후에는 포인터 또는 값으로 사용됩니다.
