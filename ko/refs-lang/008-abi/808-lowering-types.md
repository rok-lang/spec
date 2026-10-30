# 8.8. Lowering: 데이터 타입과 표현식

이 절은 Rok의 값, 타입, 식이 표준 MLIR 다이얼렉트로 어떻게 lower되는지 정의합니다.

## 8.8.1. 원시 정수 / 부동소수

원시 수치 타입은 다음과 같이 MLIR 타입에 매핑됩니다.

| Rok 타입          | MLIR 타입                                                                                   |
|-------------------|---------------------------------------------------------------------------------------------|
| `i8` ~ `i64`      | `i8` ~ `i64`                                                                                |
| `u8` ~ `u64`      | `i8` ~ `i64` (sign은 op가 결정)                                                             |
| `usize` / `isize` | `index` 또는 타깃별 `iN`                                                                    |
| `f32` / `f64`     | `f32` / `f64`                                                                               |
| `bool`            | `i1` (또는 i8, [ABI](../000-meta/007-glossary.md#abi-application-binary-interface)에 따라)  |
| `char`            | `i32` (Unicode 코드포인트)                                                                  |
| `c_char`          | `i8`                                                                                        |

`str`은 `!llvm.struct<(ptr, iN)>`(포인터와 `usize` 바이트 길이)로 값 전달되는 불변 쌍입니다. 리터럴은 immortal 상수를 가리키고, 런타임에 만든 문자열은 GC 객체(type id `1`, [8.2.12](802-scan-table.md#8212-런타임-예약-type-id))의 payload를 가리킵니다.

- `s[a:b]`는 `(ptr + a, b - a)`이며 원래 바이트를 공유합니다(O(1)).
- `ptr`은 GC 객체의 어느 바이트든(끝의 NUL 포함) 가리킬 수 있고, 객체 전체를 살려 둡니다. GC는 이를 내부 포인터로 다루며, evacuation은 offset을 유지한 채 고쳐 씁니다.
- 리터럴과 런타임이 새로 만든 문자열은 바이트 뒤에 NUL이 있습니다. slice에는 없을 수 있습니다.

MLIR의 정수 타입 자체에는 부호 정보가 없으며, 부호 해석은 연산자(`addi`, `subi` 등)가 정합니다.

## 8.8.2. 산술 연산

산술 식은 다음과 같이 `arith` 다이얼렉트의 연산으로 lower됩니다.

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

[연산자 디스패치](../003-expressions/301-operator-dispatch.md)의 트레잇 매핑은 원시 타입에서 위와 같이 lower됩니다. 사용자 정의 트레잇 구현은 제네릭 단형화를 거친 뒤 함수 호출로 lower됩니다.

## 8.8.3. `struct` 의 표현

`struct`는 값 타입이며([2.3.8](../002-types/203-struct.md#238-값-의미)) 객체 헤더가 없습니다.

```rok
struct Point { x: f64; y: f64; }
```

필드로 이루어진 LLVM struct로 lower되며, 배치는 [2.3.6](../002-types/203-struct.md#236-메모리-레이아웃)을 따릅니다.

```
!Point = !llvm.struct<"Point", (f64, f64)>
```

구조체 값은 SSA 레지스터나 스택 슬롯에 있거나, 담는 쪽 안에 inline으로 있습니다. 담는 쪽은 다른 구조체의 필드, `List` 버퍼의 원소, 클로저 환경이나 coroutine frame의 슬롯입니다. 대입과 인자 전달은 필드를 복사합니다. 컴파일러는 box를 관찰할 수 없는 곳에서만 값을 내부적으로 box할 수 있습니다.

### 필드 접근

SSA 값에서 `p.x`는 `extractvalue`이고, 스택 슬롯이나 inline 위치에서는 필드 주소를 거친 load입니다.

```
%val  = llvm.extractvalue %p[0] : !Point
%addr = llvm.getelementptr %slot[0, 0] : (!llvm.ptr) -> !llvm.ptr, !Point
%val2 = llvm.load %addr : !llvm.ptr -> f64
```

### 필드 쓰기

`p.x = 1.0f64`는 변수 자신의 사본을 바꿉니다. SSA 값이면 `insertvalue`이고, 슬롯이면 필드 주소로의 store입니다.

```
%p2 = llvm.insertvalue %one, %p[0] : !Point
```

구조체가 GC heap 객체 안에 inline으로 있고 필드가 GC 참조를 담으면, 그 store는 write barrier를 거칩니다([배리어와 루트](../007-runtime/704-barriers-and-roots.md)). 스택 슬롯에 있는 구조체의 포인터 필드는 GC 루트입니다([7.4](../007-runtime/704-barriers-and-roots.md)).

## 8.8.4. tuple `struct` 의 표현

다음은 tuple `struct`의 예입니다.

```rok
struct Pair(i32, i32);
```

일반 `struct`와 똑같이 표현하며, 필드 이름 대신 위치 인덱스로 접근한다는 점만 다릅니다.

## 8.8.5. tuple 타입

다음 tuple 타입은

```rok
(i32, str, bool)
```

`!llvm.struct<(i32, !llvm.ptr, i1)>`로 lower됩니다. `struct`처럼 헤더 없는 값입니다.

## 8.8.6. `enum` 의 표현

[`enum`](../002-types/204-enum.md)의 `enum`은 tagged `union`으로 lower됩니다.

```rok
enum Result {
    Ok(i32),
    Err(str),
    Pending,
}
```

다음과 같이 표현합니다.

```
!Result = !llvm.struct<"Result", (
    i32,                                // type_id
    i32,                                // gc_flags
    i64,                                // gc_word
    i32,                                // discriminant (tag)
    !llvm.struct<(i64)>                 // 가장 큰 변형의 payload (8 바이트)
)>
```

discriminant는 변형 식별자(`Ok`=0, `Err`=1, `Pending`=2)를 담고, payload는 모든 변형의 `union`이며 크기는 가장 큰 변형의 크기입니다.

### 변형 생성

변형을 만드는 식은 다음과 같습니다.

```rok
Result::Ok(42)
```

이 식은 다음 MLIR 시퀀스로 lower됩니다.

```
; struct 할당 + 헤더 채우기 + tag 채우기 + payload 채우기
%obj = call @rok_gc_alloc_slow(...)
; ... 헤더 ...
%tag_addr = llvm.getelementptr %obj [0, 3] : ...
llvm.store i32 0, %tag_addr  ; Ok = 0
%payload_addr = llvm.getelementptr %obj [0, 4] : ...
llvm.store %42_i32, %payload_addr
```

더 짧은 변형의 payload로 만든 union 값은 그 payload 뒤의 바이트를 0으로 채웁니다. 그래서 거기서 복사한 root word는 null이나 참조이며, 오래된 스택 값일 수 없습니다([8.9.12](809-lowering-control.md#8912-shadow-stack-frame)).

### 변형 매칭

변형을 매칭하는 식은 다음과 같습니다.

```rok
match v {
    Ok(n) => ...,
    Err(s) => ...,
    Pending => ...,
}
```

이 식은 decision tree 형태의 MLIR로 lower됩니다.

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

[`union`](../002-types/205-union.md)의 `union`의 예입니다.

```rok
union EpollData {
    ptr: usize;
    fd: i32;
    val: u64;
}
```

가장 큰 필드의 크기를 갖는 `!llvm.struct<(i64)>`로 lower됩니다. 각 필드 접근은 bitcast나 getelementptr로 같은 메모리를 다른 타입으로 해석합니다. 필드 읽기는 소스에서 `unsafe` 안에 있어야 합니다([2.5](../002-types/205-union.md)).

## 8.8.8. 합 타입 (`A | B`) 의 표현

[합 타입](../002-types/206-sum-types.md)의 합 타입은 다음과 같이 다룹니다.

```rok
type X = i32 | f64
```

이 타입은 `!llvm.struct<(i32, i64)>`로 표현하며, tag(i32)와 payload(i64, 가장 큰 변형 크기)의 조합입니다.

niche 최적화([niche 최적화](../002-types/207-niche-optimization.md))가 적용되면 tag 자리가 사라지고 합집합 비트 패턴만 남습니다.

```rok
Ptr<u8> | none
```

위 타입은 `!llvm.ptr`(단일 포인터, null이 `none`을 뜻하는 자동 niche)로 lower됩니다.

## 8.8.9. `any` 슬롯의 표현

[`any` 슬롯](../002-types/202-any-slot.md)의 동적 슬롯은 다음과 같이 씁니다.

```rok
let x: any = ...
```

`(i32 tag, usize payload)` 쌍으로 lower되며, tag는 type_id입니다. `any Tr`과 맨 `any`의 표현은 같습니다.

```
!Any = !llvm.struct<(i32, iN)>      ; iN = 타깃의 포인터 폭 (usize)
```

포인터 폭 이하의 숫자·`bool`·`char` 값은 payload에 직접 담습니다. 그 밖의 값은 GC heap의 box이고, payload는 그 box의 포인터입니다. 따라서 payload가 포인터인지는 tag로만 알 수 있으며, GC는 `any` 슬롯을 scan할 때 tag를 보고 판단해야 합니다.

### `any` 의 동적 디스패치

다음 식을 예로 듭니다.

```rok
v + 1i32     // v: any
```

이 식은 런타임 검사가 들어간 다음 MLIR로 lower됩니다.

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

런타임에 가능한 모든 타입을 검사해야 하는 이 분기 구조가 `any` 슬롯의 비용입니다.

## 8.8.10. 배열 타입: `[T; N]`

[타입 표현식](../002-types/201-type-expressions.md)의 고정 크기 배열의 예입니다.

```rok
[i32; 8]
```

`!llvm.array<8 x i32>`로 lower됩니다. GC 객체 안에서는 배열 앞에 16바이트 헤더가 붙습니다.

## 8.8.11. 함수 타입: `fn(...) -> ...`

[함수 타입](../004-functions/404-function-type.md)의 예입니다.

```rok
fn(i32, i32) -> i32
```

`(i32, i32) -> i32` 타입의 함수 포인터, 곧 `!llvm.ptr`이나 `!llvm.func<...>`로 lower됩니다. 함수 포인터 값은 값으로서 변수에 저장할 수 있습니다.

## 8.8.12. 클로저의 표현

캡처가 있는 [클로저](../004-functions/405-closure.md)의 예입니다.

```rok
let n = 10;
let f = |x| x + n;       // n 캡처
```

다음 MLIR로 lower됩니다.

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

호출할 때 환경은 함수에 추가된 첫 인자로 전달됩니다.

## 8.8.13. 식별자 / 변수 접근

[변수 선언](../005-modules/501-variable-declaration.md)의 지역 변수의 예입니다.

```rok
let x = 42i32;
let y = x + 1i32;
```

LLVM의 SSA 형태로 lower됩니다.

```
%x = arith.constant 42 : i32
%y = arith.addi %x, %1 : i32
```

지역 변수는 SSA로 lower하며, 스택 슬롯이 필요한 지역 변수는 `llvm.alloca`와 load / store로 다룹니다(debug와 release 모두 같음).

```rok
let counter = 0i32;
counter = counter + 1i32;
```

다음과 같이 lower됩니다.

```
%counter_ref = llvm.alloca %c1 x i32 : (i64) -> !llvm.ptr
llvm.store %0, %counter_ref : i32, !llvm.ptr

%loaded = llvm.load %counter_ref : !llvm.ptr -> i32
%new = arith.addi %loaded, %1 : i32
llvm.store %new, %counter_ref : i32, !llvm.ptr
```

최적화 빌드에서는 LLVM의 mem2reg가 이를 다시 SSA로 올립니다.

## 8.8.14. 리터럴 desugar

[`#[literal]` 매직](../006-meta-system/605-literal-magic.md)의 컬렉션 리터럴의 예입니다.

```rok
[1, 2, 3]
```

표준 라이브러리 함수 `list_literal(1, 2, 3)` 호출로 desugar되며, 일반 함수 호출 lowering을 따릅니다.

문자열 리터럴과 숫자 리터럴은 .rodata에 저장되는 컴파일 타임 `const`로 처리하며, 그 뒤에는 포인터나 값으로 씁니다.
