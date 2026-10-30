# 8.9. Lowering: 제어 흐름과 GC 통합

이 절은 [`if` 표현식](../003-expressions/310-if-expression.md), [`match`](../003-expressions/312-match-expression.md), [루프](../003-expressions/313-loop-expressions.md), [에러 제어 흐름](../003-expressions/314-error-control-flow.md)을 낮추는 규칙을 다룹니다. [GC](../000-meta/007-glossary.md#gc-garbage-collector)의 alloc / barrier / safepoint 코드를 내보내는 규칙도 정의합니다.

컴파일러가 생성하는 MLIR / LLVM 모듈에는 빌드 타깃(`--target`)의 타깃 트리플과 data layout이 들어갑니다.

## 8.9.1. `if` 표현식

```rok
if cond { x } else { y }
```

이 표현식은 `scf.if`를 쓰는 다음 MLIR로 낮춥니다.

```
%result = scf.if %cond -> (i32) {
  scf.yield %x : i32
} else {
  scf.yield %y : i32
}
```

`else` 분기가 없고 값이 unit인 `if`의 예입니다.

```rok
if cond { do_thing(); }
```

이 경우는 더 단순한 형태로 낮춥니다.

```
scf.if %cond {
  func.call @do_thing()
}
```

## 8.9.2. `if let`

[`if` 표현식](../003-expressions/310-if-expression.md)의 `if let pattern = expr` 형태의 예입니다.

```rok
if let v: T = compute() { use(v) } else { fallback() }
```

이 표현식은 decision tree 형태의 MLIR로 낮춥니다.

```
%opt = func.call @compute() : () -> !sum_T_none
%tag = llvm.extractvalue %opt[0] : i32
%is_some = arith.cmpi "ne", %tag, %TAG_NONE : i32

scf.if %is_some {
  %payload = llvm.extractvalue %opt[1] : !T
  func.call @use(%payload)
} else {
  func.call @fallback()
}
```

## 8.9.3. `match`의 결정 트리

[`match`](../003-expressions/312-match-expression.md)의 가지는 decision tree로 낮춥니다.

```rok
match v {
    1 => a,
    2 => b,
    n if n > 100 => c,
    _ => d,
}
```

이 `match` 식은 다음 MLIR이 됩니다.

```
cf.switch %v : i32, [
  default: ^check_guard,
  1: ^arm_a,
  2: ^arm_b
]

^check_guard:
  %is_big = arith.cmpi "sgt", %v, %100 : i32
  cf.cond_br %is_big, ^arm_c, ^arm_d

^arm_a: ; ... = a
  cf.br ^end(%a)

^arm_b: ; ... = b
  cf.br ^end(%b)

^arm_c: ; ... = c
  cf.br ^end(%c)

^arm_d: ; ... = d
  cf.br ^end(%d)

^end(%result: i32):
  ; ...
```

튜플, `struct`, `enum` 변형, 범위 같은 복잡한 패턴은 분해 단계마다 내부 switch / cmp로 낮춥니다.

## 8.9.4. `loop` / `while` / `for`

### `loop`

`loop`는 다음과 같이 씁니다.

```rok
let v = loop { ...; if cond { break x; } }
```

이 식은 `scf.while`을 쓰거나 `cf.br`로 직접 표현해서 MLIR로 낮춥니다.

```
%v = scf.while () : () -> (i32) {
  ; condition check (loop 은 항상 true)
  scf.condition (%true) ()
} do {
  ; body
  ; if cond { break x; } 는 별도 처리 (아래)
  scf.yield
}
```

`break value`는 `scf.while` 안에서 표현하기 어려우므로 `scf.execute_region`이나 `cf.br`로 풀어 씁니다.

### `while`

`while`은 다음과 같이 씁니다.

```rok
while cond { body }
```

이 식은 `scf.while`로 낮춥니다.

```
scf.while () : () -> () {
  ; condition
  %c = ...
  scf.condition (%c) ()
} do {
  ; body
  scf.yield
}
```

### `for`

`for`는 다음과 같이 씁니다.

```rok
for x in xs { body }
```

[루프 표현식](../003-expressions/313-loop-expressions.md)의 규칙이 이 식을 `let it = xs.into_iter(); loop { match it.next() { ... } }`로 풉니다. 그 결과는 위에서 설명한 lowering 과정을 똑같이 거칩니다.

## 8.9.5. `break` / `continue`

루프 안의 `break`와 `continue`는 `cf.br`로 루프의 시작이나 끝으로 점프합니다.

`scf.while`만으로는 표현하기 불편하므로 `cf` 다이얼렉트의 직접 분기로 낮춥니다.

라벨(`'outer:`)도 알맞은 [블록](../000-meta/007-glossary.md#block-블록)으로 점프합니다.

## 8.9.6. `try` / `catch` / `finally`: 값 기반 분기

[에러 제어 흐름](../003-expressions/314-error-control-flow.md)의 에러는 [Calling convention](806-calling-convention.md#865-던지는-함수의-반환-값-기반-에러-전달)에서 정의한 `{ is_err, payload }` 반환값으로 전달됩니다. 따라서 `try`와 `catch`는 둘 다 일반 분기로 낮춥니다.

```rok
let v = try f(x);              // f: fn(X) throws (E) -> T
```

`try`로 표시한 호출은 반환값의 `is_err`를 검사합니다. 성공이면 페이로드를 `T`로 꺼내고, 에러면 에러 경로로 분기합니다.

```
%r = func.call @f(%x) : (!X) -> !Throws_T_E
%is_err = llvm.extractvalue %r[0] : !Throws_T_E
cf.cond_br %is_err, ^err, ^ok

^ok:
  %v = ; payload 를 T 로 해석
  ...

^err:
  %e = ; payload 를 E 로 해석
  ; 가장 가까운 catch 가 있으면 그 dispatch 블록으로,
  ; 없으면 둘러싼 함수의 반환값 { 1, e } 를 만들어 반환
  cf.br ^catch_dispatch(%e)
```

`catch` 절이 붙은 식에는 왼쪽 식의 모든 에러 경로가 모이는 dispatch 블록이 하나 있습니다.

```rok
try body catch e: A { handler_a } catch _ { handler_any }
```

```
^catch_dispatch(%e: !E):
  ; 절의 순서대로 패턴 검사 (8.9.3 의 decision tree 와 같은 방식)
  cf.cond_br %is_a, ^handler_a(%e), ^handler_any(%e)

^handler_a(%ea: !A):
  ; handler_a
  cf.br ^end(%result_a)

^handler_any(%ex: !E):
  ; handler_any
  cf.br ^end(%result_any)

^end(%result: ...):
  ...
```

매칭되는 절이 없으면 dispatch 블록은 에러를 바깥 `catch`의 dispatch 블록으로 넘깁니다. 넘길 곳이 없으면 둘러싼 함수의 에러 반환값을 만들어 반환합니다. 에러 타입을 적지 않은 `throws`의 에러(`any Error`)를 타입 패턴으로 검사하는 코드는 [`any`](808-lowering-types.md)의 tag 비교로 낮춥니다.

`try?`는 에러 경로에서 `none`을 만들어 `^end`로 가고, `try!`는 에러 경로에서 `panic_fn`을 호출합니다.

## 8.9.7. `throw`

`throw`는 다음과 같이 씁니다.

```rok
throw err;
```

`throw`는 에러 경로로 가는 무조건 분기로 낮춥니다. 가장 가까운 `catch`가 같은 함수 안에 있으면 그 dispatch 블록으로 가고, 없으면 둘러싼 함수의 에러 반환값을 만들어 반환합니다.

```
; 같은 함수 안에 catch 가 있는 경우
cf.br ^catch_dispatch(%err)

; 없는 경우: 함수의 에러 반환
%r0 = llvm.mlir.undef : !Throws_T_E
%r1 = llvm.insertvalue %true, %r0[0] : !Throws_T_E
%r2 = ; payload 에 %err 저장
func.return %r2 : !Throws_T_E
```

함수 밖으로 에러를 돌려주는 경로도 일반 `return`과 같습니다. 그래서 shadow stack pop 같은 epilogue가 그대로 실행됩니다. 런타임 호출은 없습니다.

## 8.9.8. `finally`

`finally`는 다음과 같이 씁니다.

```rok
try { body } finally { cleanup }
```

왼쪽 식을 벗어나는 모든 경로가 cleanup 블록을 거치도록 낮춥니다. 정상 종료, `catch`로 처리한 뒤의 종료, 처리하지 않은 에러의 전파, `return` / `break` / `continue`의 각 경로가 여기에 해당합니다. 각 경로는 cleanup 블록으로 분기한 뒤 원래 목적지로 이어집니다. 에러 전파도 일반 분기이므로 별도의 unwind 메커니즘이 필요 없습니다.

```
^body_ok(%v):        cf.br ^cleanup(%k_ok, %v)
^body_err(%e):       cf.br ^cleanup(%k_err, %e)
^body_return(%rv):   cf.br ^cleanup(%k_ret, %rv)

^cleanup(%kind, %val):
  ; cleanup
  ; %kind 에 따라 ^end / 에러 전파 / 함수 return / 루프 목적지로 분기
```

cleanup 블록 안에서 새 에러를 던지면 그 에러가 원래 경로를 대신합니다. panic은 프로세스를 끝내므로 cleanup 블록을 거치지 않습니다.

## 8.9.9. GC 할당: fast / slow path

[할당](../007-runtime/702-allocation.md)도 함께 참고하세요.

```rok
let p = Point { x: 1.0, y: 2.0 };
```

이 식은 fast path를 인라인한 다음 MLIR로 낮춥니다.

```
; fast path
%t = llvm.mlir.addressof @rok_tls : !llvm.ptr
%cursor_ptr = llvm.getelementptr %t[8] : (!llvm.ptr) -> !llvm.ptr, i8    ; tlab_cursor at p
%limit_ptr = llvm.getelementptr %t[16] : (!llvm.ptr) -> !llvm.ptr, i8    ; tlab_limit at 2p

%cursor = llvm.load %cursor_ptr : !llvm.ptr -> !llvm.ptr
%new_cursor = llvm.getelementptr %cursor[24] : !llvm.ptr
%limit = llvm.load %limit_ptr : !llvm.ptr -> !llvm.ptr

%overflow = llvm.icmp "ugt" %new_cursor, %limit : !llvm.ptr
cf.cond_br %overflow, ^slow, ^fast

^fast:
  llvm.store %new_cursor, %cursor_ptr : !llvm.ptr, !llvm.ptr
  cf.br ^cont(%cursor)

^slow:
  %p_slow = func.call @rok_gc_alloc_slow(%24, %8, %TYPE_ID_POINT) : (i64, i64, i32) -> !llvm.ptr
  cf.br ^cont(%p_slow)

^cont(%p: !llvm.ptr):
  ; %p 가 객체 헤더의 시작. 헤더 채우고 필드 채우기
  ; ...
```

위 fast path에서 `%24`는 객체 총 크기, `%8`은 정렬, `%TYPE_ID_POINT`는 type_id 상수입니다. fast path는 TLAB의 bump 포인터를 올리고 헤더의 `type_id`와 `gc_word`(= 크기, [8.1.4](801-object-header-abi.md#814-gc_word오프셋-8-u64-또는-ptr))를 인라인으로 씁니다.

slow path는 `rok_gc_alloc_slow`를 호출합니다.

### 헤더 채우기

객체 헤더와 필드는 다음과 같이 채웁니다.

```
; type_id 채우기
%hdr_type_addr = llvm.getelementptr %p[0, 0] : ...
llvm.store %TYPE_ID_POINT, %hdr_type_addr : i32

; gc_flags 초기값 0
; ... (slow path 가 채울 수도)

; gc_word = 객체 크기 (8.1.4)
%hdr_word_addr = llvm.getelementptr %p[0, 2] : ...
llvm.store %24, %hdr_word_addr : i64

; 필드 채우기
%x_addr = llvm.getelementptr %p[0, 3] : ...
llvm.store %1.0, %x_addr : f64
%y_addr = llvm.getelementptr %p[0, 4] : ...
llvm.store %2.0, %y_addr : f64
```

## 8.9.10. write barrier

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md#742-write-barrier-incremental-update)에서 정한 객체 기록(object-remembering) barrier입니다.

```rok
*p.field = q;       // GC 객체에 GC 포인터 쓰기
```

이 식은 다음 MLIR로 낮춥니다.

```
; 1. write barrier: 목적지 p 의 헤더만 검사 (store 앞이어도 됨)
%hdr_flags = llvm.load %p[1] : i32
%unlogged = llvm.and %hdr_flags, %UNLOGGED_BIT : i32
%need_log = arith.cmpi "ne", %unlogged, %0 : i32
cf.cond_br %need_log, ^log, ^store

^log:
  func.call @rok_gc_wb_slow(%p) : (!llvm.ptr) -> ()   ; UNLOGGED 를 atomic 으로 지우고 remembered set 에
  cf.br ^store

^store:
  ; 2. 일반 store
  %addr = llvm.getelementptr %p[0, 4] : ...
  llvm.store %q, %addr : !llvm.ptr
```

- 저장하는 값 `q`의 헤더는 읽지 않습니다. `q`는 `any` 슬롯의 인라인 정수나 `str`의 내부 포인터일 수 있으므로 헤더로 읽으면 안전하지 않습니다.
- fast path는 load 하나와 분기 하나입니다. 실제로는 `UNLOGGED | CARDED`를 함께 검사합니다. `CARDED`(큰 old 객체)이면 호출 없이 card table에 dirty를 표시합니다. 표시하는 card는 쓴 슬롯이 걸치는 512 B card(`(slot - p) >> 9` ..= `(slot + len - 1 - p) >> 9`)입니다. card table 포인터는 헤더 바로 앞 word입니다. `UNLOGGED`이면 위처럼 `rok_gc_wb_slow`를 호출합니다. 그래서 barrier는 슬롯 주소와 저장하는 값의 크기를 씁니다([7.4.2](../007-runtime/704-barriers-and-roots.md#742-write-barrier-incremental-update)).
- `rok_gc_wb_slow`는 safepoint가 아닙니다. 그래서 barrier와 store 사이에 수집이 끼어들지 않습니다.
- 방금 할당한 객체를 초기화할 때, `static`에 store할 때, 원시 `Ptr<T>` 메모리, 참조가 아닌 atomic에는 barrier를 넣지 않습니다.

## 8.9.11. safepoint poll

[safepoint 프로토콜](804-safepoint.md)도 참고하세요.

```
; safepoint poll
%t = llvm.mlir.addressof @rok_tls : !llvm.ptr
%flag_ptr = llvm.getelementptr %t[88] : (!llvm.ptr) -> !llvm.ptr, i8    ; yield_flag at 10p+8
%flag = llvm.load %flag_ptr : !llvm.ptr -> i8
%need_yield = arith.cmpi "ne", %flag, %0 : i8

cf.cond_br %need_yield, ^slow, ^cont

^slow:
  func.call @rok_runtime_yield() : () -> ()
  cf.br ^cont

^cont:
  ; ... 계속 ...
```

컴파일러는 함수 진입 지점(또는 각 경로의 첫 safepoint 앞, [8.4.7](804-safepoint.md#847-safepoint-불변식))과 루프의 back-edge에 위 코드를 내보냅니다.

## 8.9.12. shadow stack 프레임

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-내보내는-shadow-stack)도 함께 참고하세요. 컴파일러는 LLVM의 `gc "shadow-stack"` 전략이나 `llvm.gcroot`를 쓰지 않습니다. 프레임을 체인에 잇고 떼는 코드(link / unlink)를 직접 내보냅니다. LLVM의 체인 머리 `llvm_gc_root_chain`은 프로세스 전역 하나뿐이라 스레드별 체인을 만들 수 없기 때문입니다.

GC 참조 word가 2개(`%x`, `%y`)인 함수의 예입니다. 슬롯에는 word의 주소를 넣습니다. 그래서 변수에 대입할 때는 평소처럼 `%x`에 store하기만 하면 됩니다.

```
; 진입: word 0 초기화, frame 할당과 link
%x = llvm.alloca %c1 x !llvm.ptr : (i64) -> !llvm.ptr
%y = llvm.alloca %c1 x !llvm.ptr : (i64) -> !llvm.ptr
llvm.store %null, %x : !llvm.ptr, !llvm.ptr
llvm.store %null, %y : !llvm.ptr, !llvm.ptr
%frame = llvm.alloca %c1 x !llvm.struct<(ptr, i64, array<2 x ptr>)> : (i64) -> !llvm.ptr
%head_ptr = llvm.mlir.addressof @rok_tls : !llvm.ptr             ; shadow stack head at offset 0
%head = llvm.load %head_ptr : !llvm.ptr -> !llvm.ptr
%prev_p = llvm.getelementptr %frame[0, 0] : (!llvm.ptr) -> !llvm.ptr
llvm.store %head, %prev_p : !llvm.ptr, !llvm.ptr
%count_p = llvm.getelementptr %frame[0, 1] : (!llvm.ptr) -> !llvm.ptr
llvm.store %c2_i64, %count_p : i64, !llvm.ptr          ; count: usize
%s0 = llvm.getelementptr %frame[0, 2, 0] : (!llvm.ptr) -> !llvm.ptr
llvm.store %x, %s0 : !llvm.ptr, !llvm.ptr          ; 값이 아니라 주소
%s1 = llvm.getelementptr %frame[0, 2, 1] : (!llvm.ptr) -> !llvm.ptr
llvm.store %y, %s1 : !llvm.ptr, !llvm.ptr
llvm.store %frame, %head_ptr : !llvm.ptr, !llvm.ptr

; 변수 대입: frame 갱신 없음
llvm.store %obj, %x : !llvm.ptr, !llvm.ptr

; 모든 return 직전 (에러 반환 포함): unlink
%saved = llvm.load %prev_p : !llvm.ptr -> !llvm.ptr
llvm.store %saved, %head_ptr : !llvm.ptr, !llvm.ptr
```

`@rok_tls`는 [8.3.8](803-tls-slots.md#838-컴파일러-출력-형식)의 스레드 로컬 블록입니다. shadow stack 머리는 이 블록의 오프셋 0 슬롯입니다. 에러 반환도 일반 `return`이므로([8.9.6](#896-try--catch--finally-값-기반-분기)) 프레임을 떼는 같은 코드를 거칩니다. 프레임은 위 예보다 늦게, 각 경로의 첫 safepoint 앞에서 이어도 됩니다([8.4.7](804-safepoint.md#847-safepoint-불변식)). 그때는 프레임을 이은 경로의 return에서만 프레임을 뗍니다. 프레임은 GC 참조를 쥐고 있고 safepoint에 닿을 수 있는 함수에만 필요합니다([8.4.2](804-safepoint.md#842-삽입-위치)). 이때 반환하지 않는 함수(panic)의 호출과, 할당도 block도 하지 않는 런타임 함수([8.5.2](805-extern-symbols.md#safepoint가-아닌-함수))의 호출은 세지 않습니다.

### 모듈의 데이터 레이아웃

컴파일러가 내보내는 모듈은 `llvm.data_layout` 속성을 타깃에 맞게 설정해야 합니다. 그렇지 않으면 `gep null[1]`을 접어 얻는 크기(객체 크기, 프레임 크기)가 타깃과 어긋날 수 있습니다. data layout을 설정하지 않는 경로에서는 접은 `gep`으로 크기를 구하지 않고 상수로 계산합니다.

## 8.9.13. 인덱싱

[인덱싱과 슬라이싱](../003-expressions/306-indexing-and-slicing.md)의 `arr[i]` 예입니다.

```rok
arr[5]
```

이 표현식은 표준 라이브러리 함수 `Index::index(arr, 5)` 호출이 되고, 일반 함수 호출과 똑같이 낮춥니다.

`Index` 트레잇이 인라인되면 결과는 다음과 같습니다.

```
%ptr = llvm.getelementptr %arr_data, %5 : ...
%val = llvm.load %ptr : ...
```

이 결과는 제네릭 단형화와 인라인만으로 저절로 나옵니다.

## 8.9.14. 종합: hot path의 모습

대표적인 hot path인 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) alloc과 필드 쓰기의 LLVM IR 비용은 다음과 같습니다.

- TLAB 커서 검사: load / add / icmp / cond_br (명령어 4개)
- 커서 갱신: store (명령어 1개)
- 헤더 채우기: store \* 3 (type_id, gc_flags, gc_word)
- 필드 채우기: store \* N (필드 수)
- write barrier (필요 시): load / and / icmp / cond_br + (slow) call (명령어 4~6개)
- safepoint 폴링 (필요 시): load / icmp / cond_br (명령어 3개)

작은 객체를 할당하고 채우는 데는 명령어가 대략 10~20개 듭니다. LLVM 최적화(인라인, mem2reg, simplification)가 그중 일부를 더 없앱니다.
