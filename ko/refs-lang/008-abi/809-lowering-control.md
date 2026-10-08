# 8.9. Lowering: 제어 흐름과 GC 통합

[`if` 표현식](../003-expressions/310-if-expression.md) / [`match`](../003-expressions/312-match-expression.md) / [루프](../003-expressions/313-loop-expressions.md) / [에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의
lowering, 그리고 [GC](../000-meta/007-glossary.md#gc-garbage-collector) alloc / barrier / safepoint 의 emit 규칙을 정의한다.

## 8.9.1. `if` 표현식

```nok
if cond { x } else { y }
```

→ MLIR (`scf.if`):

```
%result = scf.if %cond -> (i32) {
  scf.yield %x : i32
} else {
  scf.yield %y : i32
}
```

`else` 가 없는 `if` (값 unit):

```nok
if cond { do_thing(); }
```

→

```
scf.if %cond {
  func.call @do_thing()
}
```

## 8.9.2. `if let`

[`if` 표현식](../003-expressions/310-if-expression.md) 의 `if let pattern = expr`.

```nok
if let v: T = compute() { use(v) } else { fallback() }
```

→ MLIR (decision tree):

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

[`match`](../003-expressions/312-match-expression.md) 의 가지들은 decision tree 로 lower.

```nok
match v {
    1 => a,
    2 => b,
    n if n > 100 => c,
    _ => d,
}
```

→ MLIR:

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

복잡한 패턴 (튜플, `struct`, `enum` 변형, range 등) 은 각각의 분해를 내부적인 switch / cmp 로
lower 한다.

## 8.9.4. `loop` / `while` / `for`

### `loop`

```nok
let v = loop { ...; if cond { break x; } }
```

→ MLIR (`scf.while`) 또는 `cf.br` 으로 직접 표현.

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

`break value` 는 `scf.while` 안에서 표현하기 어렵다. `scf.execute_region` 또는 `cf.br` 로
풀어쓴다.

### `while`

```nok
while cond { body }
```

→ `scf.while`:

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

```nok
for x in xs { body }
```

[루프 표현식](../003-expressions/313-loop-expressions.md) 의 desugar 는 `let it = xs.into_iter(); loop { match it.next() { ... } }` 형태로 풀린 뒤
위와 같은 lowering 을 거친다.

## 8.9.5. `break` / `continue`

루프 안의 `break` / `continue` 는 `cf.br` 로 루프 시작 / 끝으로 점프한다.

`scf.while` 만으로는 표현이 불편하다. `cf` 다이얼렉트의 직접 분기로 lower.

라벨 (`'outer:`) 도 적절한 [block](../000-meta/007-glossary.md#block-블록) 에 점프한다.

## 8.9.6. `try` / `catch`: setjmp / longjmp

[에러 제어 흐름](../003-expressions/314-error-control-flow.md) / [Calling convention](806-calling-convention.md) 의 EH 모델.

```nok
try { body } catch e: T { handler }
```

→ MLIR:

```
%jbuf = llvm.alloca i32 (sizeof_jmp_buf) : !llvm.ptr
%r = llvm.call @setjmp(%jbuf) : (!llvm.ptr) -> i32
%is_normal = arith.cmpi "eq", %r, %0 : i32

cf.cond_br %is_normal, ^try_body, ^catch

^try_body:
  ; body
  ; ... throw 가 있으면 nok_throw 호출 (longjmp)
  cf.br ^end(%result_normal)

^catch:
  ; e 를 받아 (현재 던져진 예외 가져오기)
  %exc = func.call @nok_exception_current() : () -> !llvm.ptr
  ; 타입 검사 후 handler 실행
  ; ...
  cf.br ^end(%result_catch)

^end(%result: ...):
  ...
```

## 8.9.7. `throw`

```nok
throw err;
```

→ MLIR:

```
func.call @nok_throw(%err) : (!llvm.ptr) -> ()
; longjmp 가 가장 가까운 jmp_buf 로 점프 — return 안 함
llvm.unreachable
```

`nok_throw` 는 `nokr` 가 제공한다. setjmp 가 저장한 jmp_buf 로 longjmp.

## 8.9.8. `finally`

```nok
try { body } finally { cleanup }
```

→ try_body 와 `catch` (있으면) 양쪽 마지막에 cleanup 코드 emit. `nok_throw` 가 unwind 할
때도 cleanup 호출.

자세한 unwind 메커니즘은 `nokr` 의 `nok_throw` 구현.

## 8.9.9. GC alloc: fast / slow path

[할당](../007-runtime/702-allocation.md).

```nok
let p = Point { x: 1.0, y: 2.0 };
```

→ MLIR (fast path 인라인):

```
; fast path
%cursor_ptr = llvm.mlir.addressof @nok_tls_tlab_cursor
%limit_ptr = llvm.mlir.addressof @nok_tls_tlab_limit

%cursor = llvm.load %cursor_ptr : !llvm.ptr -> !llvm.ptr
%new_cursor = llvm.getelementptr %cursor[24] : !llvm.ptr
%limit = llvm.load %limit_ptr : !llvm.ptr -> !llvm.ptr

%overflow = llvm.icmp "ugt" %new_cursor, %limit : !llvm.ptr
cf.cond_br %overflow, ^slow, ^fast

^fast:
  llvm.store %new_cursor, %cursor_ptr : !llvm.ptr, !llvm.ptr
  cf.br ^cont(%cursor)

^slow:
  %p_slow = func.call @nok_gc_alloc_slow(%24, %8, %TYPE_ID_POINT) : (i64, i64, i32) -> !llvm.ptr
  cf.br ^cont(%p_slow)

^cont(%p: !llvm.ptr):
  ; %p 가 객체 헤더의 시작 — 헤더 채우고 필드 채우기
  ; ...
```

fast path 의 `%24` 는 객체 총 크기, `%8` 은 정렬, `%TYPE_ID_POINT` 는 type_id 상수.

slow path 는 `nok_gc_alloc_slow` 호출.

### 헤더 채우기

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

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md) 의 incremental-update + SATB.

```nok
*p.field = q;       // GC 객체에 GC 포인터 쓰기
```

→ MLIR:

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
  func.call @nok_gc_wb_slow(%p) : (!llvm.ptr) -> ()
  cf.br ^cont

^cont:
  ; ...
```

barrier 가 모든 GC 포인터 쓰기에 추가된다. fast path 는 분기 1 ~ 2 개 + load 1 개.

write barrier 의 세부는 GC 구현. 일부 변형 (Yuasa SATB, Dijkstra-style) 가능.

## 8.9.11. Safepoint poll

[Safepoint 프로토콜](804-safepoint.md).

```
; safepoint poll
%flag_ptr = llvm.mlir.addressof @nok_tls_yield_flag
%flag = llvm.load %flag_ptr : !llvm.ptr -> i8
%need_yield = arith.cmpi "ne", %flag, %0 : i8

cf.cond_br %need_yield, ^slow, ^cont

^slow:
  func.call @nok_runtime_yield() : () -> ()
  cf.br ^cont

^cont:
  ; ... 계속 ...
```

컴파일러는 함수 진입과 루프 back-edge 에 위 코드를 emit 한다.

## 8.9.12. llvm.gcroot

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md).

```
; 변수 선언 시점
%root = llvm.alloca i32 1 x !llvm.ptr : !llvm.ptr
llvm.intr.gcroot %root, %null

; 변수에 GC 객체 저장 시
llvm.store %obj, %root : !llvm.ptr, !llvm.ptr
```

`llvm.intr.gcroot` 가 LLVM 백엔드에 [shadow stack](../000-meta/007-glossary.md#shadow-stack) 등록 신호를 준다.

LLVM 백엔드가 함수 진입 / 종료 시점에 shadow stack push / pop 코드를 자동 삽입한다.

## 8.9.13. Indexing

[인덱싱과 슬라이싱](../003-expressions/306-indexing-and-slicing.md) 의 `arr[i]`.

```nok
arr[5]
```

→ 표준 라이브러리의 `Index::index(arr, 5)` 함수 호출. 일반 함수 호출 lowering.

`Index` 트레잇이 인라인되어 결과적으로:

```
%ptr = llvm.getelementptr %arr_data, %5 : ...
%val = llvm.load %ptr : ...
```

같은 결과가 제네릭 단형화 + 인라이닝으로 자동 도출된다.

## 8.9.14. 종합: Hot path 의 모습

대표적인 hot path ([TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) alloc + 필드 쓰기) 의 LLVM IR 비용:

- TLAB cursor 검사: load / add / icmp / cond_br (4 명령어)
- cursor 갱신: store (1 명령어)
- 헤더 채우기: store \* 3 (type_id, gc_flags, gc_word)
- 필드 채우기: store \* N (필드 수)
- write barrier (필요 시): load / and / icmp / cond_br + (slow) call (4 ~ 6 명령어)
- safepoint poll (필요 시): load / icmp / cond_br (3 명령어)

작은 객체 할당과 채우기가 10 ~ 20 명령어. 이 비용이 자체 GC + 동시 수집의 일반적
오버헤드다.

LLVM 의 최적화 (인라인, mem2reg, simplification) 가 추가로 일부 명령어를 제거한다.
