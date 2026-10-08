# 8.10. Lowering: 코루틴과 제네릭

[제너레이터](../004-functions/411-generators.md) / [비동기](../004-functions/412-async.md) / [스트림](../004-functions/413-stream.md) 의 코루틴 lowering 과
[제네릭](../002-types/210-generics.md) / [const 제네릭](../002-types/211-const-generics.md) 의 단형화 lowering 을 정의한다.

## 8.10.1. LLVM Coroutine Intrinsic

[코루틴](../007-runtime/708-coroutine.md) 의 LLVM coroutine intrinsic 직접 사용.

핵심 intrinsic:

| Intrinsic           | 의미                      |
| ------------------- | ------------------------- |
| `llvm.coro.id`      | 이 함수가 코루틴임을 표시 |
| `llvm.coro.alloc`   | frame 할당 필요 여부      |
| `llvm.coro.size`    | frame 크기                |
| `llvm.coro.begin`   | frame 초기화              |
| `llvm.coro.suspend` | 일시 중단 지점            |
| `llvm.coro.end`     | 코루틴 종료               |
| `llvm.coro.resume`  | 코루틴 재개               |
| `llvm.coro.destroy` | frame 해제                |

이 intrinsic 들을 컴파일러가 emit 하면, LLVM 백엔드가 상태 머신으로 lower.

## 8.10.2. Generator (`yield fn`) lowering

```nok
yield fn count_up_to(n: i32) -> i32 {
    let i = 0;
    while i < n {
        yield i;
        i = i + 1;
    }
}
```

→ MLIR (개념적):

```mlir
func.func @count_up_to(%n: i32) -> !llvm.ptr {
  %id = llvm.intr.coro.id(...) : ...
  %size = llvm.intr.coro.size : i64
  %frame = func.call @nok_coro_frame_alloc(%size, %TYPE_ID_COUNT_UP_TO_FRAME) : (i64, i32) -> !llvm.ptr
  %hdl = llvm.intr.coro.begin(%id, %frame) : ...

  ; 본체 시작
  %i_ref = ; ... i 변수
  llvm.store %0, %i_ref : i32

  cf.br ^loop

^loop:
  %i = llvm.load %i_ref : i32
  %check = arith.cmpi "slt", %i, %n : i32
  cf.cond_br %check, ^body, ^done

^body:
  ; yield i;
  %i_val = llvm.load %i_ref : i32
  llvm.store %i_val, %frame_yield_slot : i32
  %suspend = llvm.intr.coro.suspend(...) : i8
  ; 분기: 0 = 재개, 1 = destroy, 그 외 = 정상 진행
  cf.switch %suspend, [
    default: ^continue,
    0: ^continue,        ; 재개
    1: ^cleanup          ; destroy
  ]

^continue:
  %i_inc = arith.addi %i, %1 : i32
  llvm.store %i_inc, %i_ref : i32
  cf.br ^loop

^done:
  cf.br ^cleanup

^cleanup:
  llvm.intr.coro.end(%hdl, ...) : ...
  func.return %hdl
}
```

복잡하지만 LLVM 이 자동 생성한다 (intrinsic 만 emit 하면 LLVM 이 state machine 을 만든다).

`yield i` 의 desugar 는 frame 의 `yield` 슬롯에 값을 저장 + suspend.

호출자 (`for x in iter`) 가 `iter.next()` 로 코루틴 resume 하면 next 가 frame 의 `yield` 슬롯
값을 반환한다.

## 8.10.3. Async (`async fn`) lowering

```nok
async fn fetch() -> Data {
    let r = await op1();
    process(r)
}
```

→ MLIR (개념):

```mlir
func.func @fetch() -> !llvm.ptr {
  %id = llvm.intr.coro.id(...) : ...
  %size = llvm.intr.coro.size : i64
  %frame = func.call @nok_coro_frame_alloc(%size, %TYPE_ID_FETCH_FRAME) : ...
  %hdl = llvm.intr.coro.begin(%id, %frame) : ...

  ; let r = await op1();
  %op1_fut = func.call @op1() : () -> !llvm.ptr

  ; await — Future poll 시작
  %suspend1 = llvm.intr.coro.suspend(%op1_fut, ...) : i8
  cf.switch %suspend1, [...]

^after_await1:
  %r = ; op1 결과 ...
  ; process(r)
  %result = func.call @process(%r) : ...

  ; 결과를 frame 에 저장
  llvm.store %result, %frame_result_slot : ...

  llvm.intr.coro.end(%hdl, ...) : ...
  func.return %hdl
}
```

호출자가 `await fetch()` 시 Future 의 `poll` 호출 후 결과 ready 면 frame 의 result slot 값
을 반환한다.

## 8.10.4. Stream (`async yield fn`) lowering

[스트림](../004-functions/413-stream.md) 의 두 메커니즘 결합.

```nok
async yield fn read_lines(file: str) -> str {
    let f = await open(file);
    loop {
        let line = await f.read_line();
        match line {
            l: str => yield l,
            none   => break,
        }
    }
}
```

→ 위 generator + `async` 의 결합. 한 frame 안에 `yield` 일시 중단과 `await` 일시 중단이
공존한다.

state machine 이 어느 종류로 일시 중단되었는지를 추적한다.

## 8.10.5. CoroElide

LLVM 의 CoroElide 패스는 코루틴 호출이 `await` / next 로 직접 끝까지 진행되면 frame 할당을
생략한다.

조건:

- 호출자가 그 코루틴을 직접 `await` / next.
- 코루틴 result 가 escape 안 함.

이 경우 `nok_coro_frame_alloc` 호출이 사라지고 frame 이 호출자의 지역 변수처럼 처리된다.

자주 일어나는 짧은 `async` `fn` 호출은 비용 0.

## 8.10.6. 제네릭 단형화

[제네릭](../002-types/210-generics.md) 의 단형화는 호출 지점의 타입 튜플별로 별개 인스턴스를 생성한다.

```nok
fn id<T>(x: T) -> T { x }

let a = id(42i32);     // id<i32>
let b = id(3.14f64);   // id<f64>
```

→ MLIR (단형화 후):

```
func.func @id_i32(%x: i32) -> i32 {
  func.return %x : i32
}

func.func @id_f64(%x: f64) -> f64 {
  func.return %x : f64
}

func.func @main() {
  %a = func.call @id_i32(%42) : (i32) -> i32
  %b = func.call @id_f64(%3.14) : (f64) -> f64
}
```

각 구체 타입 튜플마다 별개 함수.

### 단형화의 시점

```
nok-hir → specialize(hir, type_tuple) → nok-mir → MLIR
```

- [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 모드: 모든 호출 지점을 수집해 일괄 [specialize](../000-meta/007-glossary.md#specialize).
- [JIT](../000-meta/007-glossary.md#jit-just-in-time) 모드: 첫 호출 시 specialize + 캐시 ([실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md)).

같은 specialize 메커니즘을 두 모드가 공유한다.

### 심볼 명명

[컴파일 파이프라인](../000-meta/002-pipeline.md) 의 심볼 맹글링.

```
_N<pkg_hash_8>_<path>_<name>_<typehash_16>
```

예: `id<i32>` 는 `_N3f8a1c2b_my_pkg_id_a7c91d8e44b2f190` (예시).

이 심볼이 `func.func @...` 의 이름이 된다.

## 8.10.7. `const` 제네릭 단형화

[const 제네릭](../002-types/211-const-generics.md).

```nok
fn dot<T: Num, const N: usize>(a: [T; N], b: [T; N]) -> T { ... }

let r = dot([1.0, 2.0, 3.0], [4.0, 5.0, 6.0]);  // T=f64, N=3
```

→ MLIR:

```
func.func @dot_f64_3(%a: !llvm.array<3 x f64>, %b: !llvm.array<3 x f64>) -> f64 {
  ; ... unroll 또는 일반 loop
}
```

`const` N=3 이 컴파일 타임 상수라서 LLVM 의 unroll / inlining 이 자유롭게 적용된다.

## 8.10.8. 제네릭 트레잇 디스패치

```nok
fn process<T: Display>(x: T) -> str {
    x.display()
}
```

→ 호출 시점의 구체 T 가 결정된다. `T = Point` 면:

```
func.func @process_Point(%x: !Point_ptr) -> !str_ptr {
  ; Point 의 Display::display 메서드를 *직접 호출*
  %r = func.call @Point_Display_display(%x) : (!Point_ptr) -> !str_ptr
  func.return %r
}
```

vtable 간접 호출 없음. 직접 함수 호출.

이 모델이 [트레잇-as-타입 금지](../002-types/212-trait-as-type-banned.md) 의 결정과 결합되어 vtable 비용 0 을 보장한다.

## 8.10.9. 합 타입의 분기 디스패치

[합 타입](../002-types/206-sum-types.md) 의 매칭은 `cf.switch` decision tree.

```nok
let v: i32 | str = ...;
match v {
    n: i32 => use_int(n),
    s: str => use_str(s),
}
```

→ MLIR:

```
%tag = llvm.extractvalue %v[0] : i32

cf.switch %tag, [
  default: ^trap,
  TAG_I32: ^use_int,
  TAG_STR: ^use_str
]

^use_int:
  %n = llvm.extractvalue %v[1] : i64
  %n_i32 = llvm.trunc %n : i64 to i32
  func.call @use_int(%n_i32) : ...
  cf.br ^end

^use_str:
  %s = llvm.extractvalue %v[1] : i64
  %s_ptr = llvm.inttoptr %s : i64 to !llvm.ptr
  func.call @use_str(%s_ptr) : ...
  cf.br ^end

^end:
  ...
```

태그 검사 + 분기는 정적 코드.

## 8.10.10. 트레잇 메서드의 trait-bound dispatch

```nok
fn add_all<T: Add<T, Output = T>>(items: List<T>) -> T {
    let r: T = ...;
    for x in items {
        r = r.add(x);
    }
    r
}
```

→ 단형화 후 각 호출 지점의 T 에 대해 별개 함수. `T = i32` 면 `r.add(x)` 가 i32 의 Add::add
직접 호출 (인라인 가능).

## 8.10.11. 정리

코루틴 lowering 은 LLVM intrinsic 에 위임한다. 컴파일러는 intrinsic 만 emit.
제네릭 lowering 은 단형화로 각 구체 타입마다 별개 코드를 만든다. vtable 없음.

이 두 메커니즘이 본 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 블록의 가장 복잡한 lowering. 다른 부분 ([Lowering: 데이터 타입과 표현식](808-lowering-types.md) / [Lowering: 제어 흐름과 GC 통합](809-lowering-control.md)) 은 표준
다이얼렉트의 직접 매핑이다.

이로써 8장 (런타임 ABI / MLIR lowering / codegen) 마무리.
