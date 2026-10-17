# 8.9. Lowering: 제어 흐름과 GC 통합

이 절에서는 [`if` 표현식](../003-expressions/310-if-expression.md), [`match`](../003-expressions/312-match-expression.md), [루프](../003-expressions/313-loop-expressions.md), [에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 lowering 규칙을 다루고, 더불어 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 의 alloc / barrier / safepoint 를 emit 하는 규칙을 함께 정의합니다.

생성하는 MLIR / LLVM 모듈은 빌드 타깃 (`--target`) 의 target triple 과 data layout 을 담습니다.

## 8.9.1. `if` 표현식

```rok
if cond { x } else { y }
```

이 표현식은 `scf.if` 를 사용한 MLIR 로 다음과 같이 lower 됩니다.

```
%result = scf.if %cond -> (i32) {
  scf.yield %x : i32
} else {
  scf.yield %y : i32
}
```

한편 `else` 분기가 없고 값이 unit 인 `if` 의 경우는 다음과 같습니다.

```rok
if cond { do_thing(); }
```

이 경우의 lowering 결과는 다음과 같이 단순한 형태가 됩니다.

```
scf.if %cond {
  func.call @do_thing()
}
```

## 8.9.2. `if let`

[`if` 표현식](../003-expressions/310-if-expression.md) 의 `if let pattern = expr` 형태를 살펴보겠습니다.

```rok
if let v: T = compute() { use(v) } else { fallback() }
```

이 표현식은 decision tree 형태의 MLIR 로 lower 되며, 그 결과는 다음과 같습니다.

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

## 8.9.3. `match` 의 decision tree

[`match`](../003-expressions/312-match-expression.md) 의 가지들은 decision tree 로 lower 됩니다.

```rok
match v {
    1 => a,
    2 => b,
    n if n > 100 => c,
    _ => d,
}
```

이 `match` 식은 다음과 같은 MLIR 로 변환됩니다.

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

튜플, `struct`, `enum` 변형, range 등 복잡한 패턴의 경우에는 각각의 분해 과정이 내부적인 switch / cmp 로 lower 됩니다.

## 8.9.4. `loop` / `while` / `for`

### `loop`

`loop` 는 다음과 같은 형태로 사용됩니다.

```rok
let v = loop { ...; if cond { break x; } }
```

이는 `scf.while` 을 사용하거나 `cf.br` 로 직접 표현하여 MLIR 로 lower 됩니다.

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

다만 `break value` 는 `scf.while` 안에서 표현하기 어려우므로, `scf.execute_region` 또는 `cf.br` 로 풀어쓰게 됩니다.

### `while`

`while` 은 다음과 같이 사용합니다.

```rok
while cond { body }
```

이는 `scf.while` 로 lower 되며, 결과는 다음과 같습니다.

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

`for` 는 다음과 같이 사용합니다.

```rok
for x in xs { body }
```

[루프 표현식](../003-expressions/313-loop-expressions.md) 의 desugar 는 `let it = xs.into_iter(); loop { match it.next() { ... } }` 형태로 풀린 뒤, 위에서 설명한 lowering 과정을 동일하게 통과합니다.

## 8.9.5. `break` / `continue`

루프 안의 `break` 와 `continue` 는 `cf.br` 로 루프의 시작 지점이나 끝 지점으로 점프합니다.

`scf.while` 만으로는 표현이 불편하기 때문에, `cf` 다이얼렉트의 직접 분기로 lower 하게 됩니다.

라벨 (`'outer:`) 또한 적절한 [block](../000-meta/007-glossary.md#block-블록) 으로 점프합니다.

## 8.9.6. `try` / `catch` / `finally`: 값 기반 분기

[에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 에러는 [Calling convention](806-calling-convention.md#865-throwing-함수의-반환-값-기반-에러-전달) 에서 정의한 `{ is_err, payload }` 반환값으로 전달됩니다. 따라서 `try` 와 `catch` 는 모두 일반 분기로 lower 됩니다.

```rok
let v = try f(x);              // f: fn(X) throws (E) -> T
```

`try` 로 표시한 호출은 반환값의 `is_err` 를 검사해, 성공이면 payload 를 `T` 로 꺼내고 에러면 에러 경로로 분기합니다.

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

`catch` 절이 붙은 식은 왼쪽 식의 모든 에러 경로가 모이는 dispatch 블록을 하나 갖습니다.

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

매칭되는 절이 없으면 dispatch 블록은 에러를 바깥 `catch` 의 dispatch 블록으로 넘기거나 둘러싼 함수의 에러 반환값을 만들어 반환합니다. untyped `throws` 의 에러 (`any Error`) 에 대한 타입 패턴 검사는 [`any`](808-lowering-types.md) 의 tag 비교로 lower 됩니다.

`try?` 는 에러 경로에서 `none` 을 만들어 `^end` 로 가고, `try!` 는 에러 경로에서 `panic_fn` 을 호출합니다.

## 8.9.7. `throw`

`throw` 는 다음과 같이 사용합니다.

```rok
throw err;
```

`throw` 는 에러 경로로 가는 무조건 분기로 lower 됩니다. 가장 가까운 `catch` 가 같은 함수 안에 있으면 그 dispatch 블록으로 가고, 없으면 둘러싼 함수의 에러 반환값을 만들어 반환합니다.

```
; 같은 함수 안에 catch 가 있는 경우
cf.br ^catch_dispatch(%err)

; 없는 경우 — 함수의 에러 반환
%r0 = llvm.mlir.undef : !Throws_T_E
%r1 = llvm.insertvalue %true, %r0[0] : !Throws_T_E
%r2 = ; payload 에 %err 저장
func.return %r2 : !Throws_T_E
```

함수 밖으로 에러를 돌려주는 경로도 일반 `return` 과 같으므로, shadow stack pop 같은 epilogue 가 그대로 실행됩니다. 런타임 호출은 없습니다.

## 8.9.8. `finally`

`finally` 는 다음과 같이 사용합니다.

```rok
try { body } finally { cleanup }
```

왼쪽 식을 벗어나는 모든 경로가 cleanup 블록을 거치도록 lower 됩니다. 정상 종료, `catch` 처리 후 종료, 처리되지 않은 에러의 전파, 그리고 `return` / `break` / `continue` 의 각 경로는 cleanup 블록으로 분기한 뒤 원래 목적지로 이어집니다. 에러 전파도 일반 분기이므로 별도의 unwind 메커니즘은 필요하지 않습니다.

```
^body_ok(%v):        cf.br ^cleanup(%k_ok, %v)
^body_err(%e):       cf.br ^cleanup(%k_err, %e)
^body_return(%rv):   cf.br ^cleanup(%k_ret, %rv)

^cleanup(%kind, %val):
  ; cleanup
  ; %kind 에 따라 ^end / 에러 전파 / 함수 return / 루프 목적지로 분기
```

cleanup 블록 안에서 새 에러가 던져지면 그 에러가 원래의 경로를 대체합니다. panic 은 프로세스를 종료하므로 cleanup 블록을 거치지 않습니다.

## 8.9.9. GC alloc: fast / slow path

[할당](../007-runtime/702-allocation.md) 부분을 함께 살펴보겠습니다.

```rok
let p = Point { x: 1.0, y: 2.0 };
```

이는 fast path 가 인라인된 다음과 같은 MLIR 로 lower 됩니다.

```
; fast path
%cursor_ptr = llvm.mlir.addressof @rok_tls_tlab_cursor
%limit_ptr = llvm.mlir.addressof @rok_tls_tlab_limit

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
  ; %p 가 객체 헤더의 시작 — 헤더 채우고 필드 채우기
  ; ...
```

위 fast path 에서 `%24` 는 객체 총 크기, `%8` 은 정렬, `%TYPE_ID_POINT` 는 type_id 상수를 의미합니다. fast path 는 TLAB 의 bump 포인터를 올린 뒤 헤더의 `type_id` 와 `gc_word` (= 크기, [8.1.4](801-object-header-abi.md#814-gc_word-offset-8-u64-또는-ptr)) 를 인라인으로 씁니다.

slow path 의 경우에는 `rok_gc_alloc_slow` 를 호출합니다.

### 헤더 채우기

객체 헤더와 필드는 다음과 같이 채워 줍니다.

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

## 8.9.10. Write Barrier

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md#742-write-barrier-incremental-update) 의 object-remembering barrier 입니다.

```rok
*p.field = q;       // GC 객체에 GC 포인터 쓰기
```

이는 다음과 같은 MLIR 로 lower 됩니다.

```
; 1. write barrier — 목적지 p 의 헤더만 검사 (store 앞이어도 됨)
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

- 저장하는 값 `q` 의 헤더는 읽지 않습니다. `q` 는 `any` 슬롯의 inline 정수나 `str` 의 내부 포인터일 수 있어 헤더로 읽으면 안전하지 않습니다.
- fast path 는 load 하나와 분기 하나입니다. 실제로는 `UNLOGGED | CARDED` 를 함께 검사합니다. `CARDED` (큰 old 객체) 이면 호출 없이 쓴 slot 이 걸치는 512 B card (`(slot - p) >> 9` ..= `(slot + len - 1 - p) >> 9`) 를 card table (포인터는 헤더 바로 앞 word) 에 dirty 로 표시하고, `UNLOGGED` 이면 위처럼 `rok_gc_wb_slow` 를 부릅니다. 그래서 barrier 는 slot 주소와 저장하는 값의 크기를 씁니다 ([7.4.2](../007-runtime/704-barriers-and-roots.md#742-write-barrier-incremental-update)).
- `rok_gc_wb_slow` 는 safepoint 가 아닙니다. barrier 와 store 사이에 수집이 끼어들지 않습니다.
- 방금 할당한 객체의 초기화, `static` 에의 store, raw `Ptr<T>` 메모리, 참조가 아닌 atomic 에는 barrier 를 넣지 않습니다.

## 8.9.11. Safepoint poll

[Safepoint 프로토콜](804-safepoint.md) 도 살펴보겠습니다.

```
; safepoint poll
%flag_ptr = llvm.mlir.addressof @rok_tls_yield_flag
%flag = llvm.load %flag_ptr : !llvm.ptr -> i8
%need_yield = arith.cmpi "ne", %flag, %0 : i8

cf.cond_br %need_yield, ^slow, ^cont

^slow:
  func.call @rok_runtime_yield() : () -> ()
  cf.br ^cont

^cont:
  ; ... 계속 ...
```

컴파일러는 함수의 진입 지점과 루프의 back-edge 에 위 코드를 emit 합니다.

## 8.9.12. Shadow stack frame

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-emit-하는-shadow-stack) 을 함께 참고합니다. 컴파일러는 LLVM 의 `gc "shadow-stack"` 전략이나 `llvm.gcroot` 를 쓰지 않고, frame 의 link / unlink 를 직접 emit 합니다. LLVM 의 체인 머리 `llvm_gc_root_chain` 은 프로세스 전역 하나뿐이라 thread 별 체인을 만들 수 없기 때문입니다.

GC 참조 word 가 2 개 (`%x`, `%y`) 인 함수의 예입니다. 슬롯에는 word 의 주소를 넣으므로, 변수 대입은 평소처럼 `%x` 에 store 하는 것으로 끝납니다.

```
; 진입: word 0 초기화, frame 할당과 link
%x = llvm.alloca %c1 x !llvm.ptr : (i64) -> !llvm.ptr
%y = llvm.alloca %c1 x !llvm.ptr : (i64) -> !llvm.ptr
llvm.store %null, %x : !llvm.ptr, !llvm.ptr
llvm.store %null, %y : !llvm.ptr, !llvm.ptr
%frame = llvm.alloca %c1 x !llvm.struct<(ptr, i64, array<2 x ptr>)> : (i64) -> !llvm.ptr
%head_ptr = llvm.mlir.addressof @rok_tls_shadow_stack_top : !llvm.ptr
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

; 변수 대입 — frame 갱신 없음
llvm.store %obj, %x : !llvm.ptr, !llvm.ptr

; 모든 return 직전 (에러 반환 포함): unlink
%saved = llvm.load %prev_p : !llvm.ptr -> !llvm.ptr
llvm.store %saved, %head_ptr : !llvm.ptr, !llvm.ptr
```

`@rok_tls_shadow_stack_top` 은 [8.3.8](803-tls-slots.md#838-컴파일러-emit-의-형식) 의 thread-local 심볼입니다. 에러 반환도 일반 `return` 이므로 ([8.9.6](#896-try--catch--finally-값-기반-분기)) 같은 unlink 를 지납니다. GC 참조를 들고 있고 safepoint 에 닿을 수 있는 함수만 frame 이 필요합니다 ([8.4.2](804-safepoint.md#842-emit-위치)). 반환하지 않는 함수 (panic) 와 할당도 block 도 하지 않는 런타임 함수 ([8.5.2](805-extern-symbols.md#safepoint-가-아닌-함수)) 의 호출은 세지 않습니다.

### 모듈의 data layout

컴파일러가 emit 하는 모듈은 반드시 `llvm.data_layout` 속성을 타깃에 맞게 설정합니다. 그렇지 않으면 `gep null[1]` 을 접어 얻는 크기 (객체 크기, frame 크기) 가 타깃과 어긋날 수 있습니다. data layout 을 설정하지 않는 경로에서는 그런 접힌 `gep` 으로 크기를 구하지 않고 상수로 계산합니다.

## 8.9.13. Indexing

[인덱싱과 슬라이싱](../003-expressions/306-indexing-and-slicing.md) 의 `arr[i]` 를 살펴보겠습니다.

```rok
arr[5]
```

이 표현식은 표준 라이브러리의 `Index::index(arr, 5)` 함수 호출이 되며, 일반 함수 호출과 동일한 방식으로 lowering 됩니다.

`Index` 트레잇이 인라인되어 결과적으로 다음과 같은 형태가 만들어집니다.

```
%ptr = llvm.getelementptr %arr_data, %5 : ...
%val = llvm.load %ptr : ...
```

이러한 결과는 제네릭 단형화와 인라이닝을 통해 자동으로 도출됩니다.

## 8.9.14. 종합: Hot path 의 모습

대표적인 hot path 인 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) alloc 과 필드 쓰기의 LLVM IR 비용을 정리해 드리면 다음과 같습니다.

- TLAB cursor 검사: load / add / icmp / cond_br (4 명령어)
- cursor 갱신: store (1 명령어)
- 헤더 채우기: store \* 3 (type_id, gc_flags, gc_word)
- 필드 채우기: store \* N (필드 수)
- write barrier (필요 시): load / and / icmp / cond_br + (slow) call (4 ~ 6 명령어)
- safepoint poll (필요 시): load / icmp / cond_br (3 명령어)

작은 객체의 할당과 채우기는 대략 10 ~ 20 명령어입니다. LLVM 최적화 (인라인, mem2reg, simplification) 가 일부를 더 지웁니다.
