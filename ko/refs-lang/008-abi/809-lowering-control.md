# 8.9. Lowering: 제어 흐름과 GC 통합

이 절에서는 [`if` 표현식](../003-expressions/310-if-expression.md), [`match`](../003-expressions/312-match-expression.md), [루프](../003-expressions/313-loop-expressions.md), [에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 lowering 규칙을 다루고, 더불어 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 의 alloc / barrier / safepoint 를 emit 하는 규칙을 함께 정의합니다.

## 8.9.1. `if` 표현식

가장 단순한 형태의 `if` 표현식부터 살펴보겠습니다.

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

## 8.9.6. `try` / `catch`: setjmp / longjmp

[에러 제어 흐름](../003-expressions/314-error-control-flow.md) 과 [Calling convention](806-calling-convention.md) 에 정의된 EH 모델을 살펴보겠습니다.

```rok
try { body } catch e: T { handler }
```

이 구문은 다음과 같은 MLIR 로 lower 됩니다.

```
%jbuf = llvm.alloca i32 (sizeof_jmp_buf) : !llvm.ptr
%r = llvm.call @setjmp(%jbuf) : (!llvm.ptr) -> i32
%is_normal = arith.cmpi "eq", %r, %0 : i32

cf.cond_br %is_normal, ^try_body, ^catch

^try_body:
  ; body
  ; ... throw 가 있으면 rok_throw 호출 (longjmp)
  cf.br ^end(%result_normal)

^catch:
  ; e 를 받아 (현재 던져진 예외 가져오기)
  %exc = func.call @rok_exception_current() : () -> !llvm.ptr
  ; 타입 검사 후 handler 실행
  ; ...
  cf.br ^end(%result_catch)

^end(%result: ...):
  ...
```

## 8.9.7. `throw`

`throw` 는 다음과 같이 사용합니다.

```rok
throw err;
```

이는 다음과 같은 MLIR 로 lower 됩니다.

```
func.call @rok_throw(%err) : (!llvm.ptr) -> ()
; longjmp 가 가장 가까운 jmp_buf 로 점프 — return 안 함
llvm.unreachable
```

여기서 `rok_throw` 는 `rokr` 가 제공해 주며, setjmp 가 저장한 jmp_buf 로 longjmp 하는 역할을 합니다.

## 8.9.8. `finally`

`finally` 는 다음과 같이 사용합니다.

```rok
try { body } finally { cleanup }
```

이 경우에는 try_body 와 (있다면) `catch` 양쪽의 마지막에 cleanup 코드를 emit 합니다. 또한 `rok_throw` 가 unwind 할 때에도 cleanup 이 호출됩니다.

자세한 unwind 메커니즘은 `rokr` 의 `rok_throw` 구현을 참고해 주시면 됩니다.

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

위 fast path 에서 `%24` 는 객체 총 크기, `%8` 은 정렬, `%TYPE_ID_POINT` 는 type_id 상수를 의미합니다.

slow path 의 경우에는 `rok_gc_alloc_slow` 를 호출하게 됩니다.

### 헤더 채우기

객체 헤더와 필드는 다음과 같이 채워 줍니다.

```
; type_id 채우기
%hdr_type_addr = llvm.getelementptr %p[0, 0] : ...
llvm.store %TYPE_ID_POINT, %hdr_type_addr : i32

; gc_flags 초기값 0
; ... (slow path 가 채울 수도)

; 필드 채우기
%x_addr = llvm.getelementptr %p[0, 3] : ...
llvm.store %1.0, %x_addr : f64
%y_addr = llvm.getelementptr %p[0, 4] : ...
llvm.store %2.0, %y_addr : f64
```

## 8.9.10. Write Barrier

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md) 의 incremental-update 와 SATB 를 살펴보겠습니다.

```rok
*p.field = q;       // GC 객체에 GC 포인터 쓰기
```

이는 다음과 같은 MLIR 로 lower 됩니다.

```
; 1. 일반 store
%addr = llvm.getelementptr %p[0, 4] : ...
llvm.store %q, %addr : !llvm.ptr

; 2. write barrier 검사
%hdr_flags = llvm.load %p[1] : i32
%sticky = llvm.and %hdr_flags, %STICKY_BIT : i32
%is_sticky = arith.cmpi "ne", %sticky, %0 : i32

cf.cond_br %is_sticky, ^check_q, ^cont

^check_q:
  %q_flags = llvm.load %q[1] : i32
  %q_sticky = llvm.and %q_flags, %STICKY_BIT : i32
  %q_is_young = arith.cmpi "eq", %q_sticky, %0 : i32
  cf.cond_br %q_is_young, ^add_log, ^cont

^add_log:
  func.call @rok_gc_wb_slow(%p) : (!llvm.ptr) -> ()
  cf.br ^cont

^cont:
  ; ...
```

barrier 는 모든 GC 포인터 쓰기에 추가됩니다. fast path 의 비용은 분기 1 ~ 2 개에 load 1 개를 더한 정도이며, 비교적 가벼운 편입니다.

write barrier 의 세부 내용은 GC 구현에 따라 달라지며, Yuasa SATB 또는 Dijkstra-style 등 일부 변형도 가능합니다.

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

컴파일러는 함수의 진입 지점과 루프의 back-edge 에 위 코드를 emit 해 줍니다.

## 8.9.12. llvm.gcroot

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md) 을 함께 참고해 주세요.

```
; 변수 선언 시점
%root = llvm.alloca i32 1 x !llvm.ptr : !llvm.ptr
llvm.intr.gcroot %root, %null

; 변수에 GC 객체 저장 시
llvm.store %obj, %root : !llvm.ptr, !llvm.ptr
```

여기서 `llvm.intr.gcroot` 가 LLVM 백엔드에 [shadow stack](../000-meta/007-glossary.md#shadow-stack) 등록 신호를 전달합니다.

이후 LLVM 백엔드는 함수 진입 시점과 종료 시점에 shadow stack 의 push 와 pop 코드를 자동으로 삽입해 줍니다.

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

작은 객체의 할당과 채우기는 대략 10 ~ 20 명령어 정도가 소요됩니다. 이 정도의 비용이 자체 GC 와 동시 수집의 일반적인 오버헤드라고 보면 됩니다.

이에 더해 LLVM 의 최적화 (인라인, mem2reg, simplification) 가 일부 명령어를 추가로 제거해 줍니다.
