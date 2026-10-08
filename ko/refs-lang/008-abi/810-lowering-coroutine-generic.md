# 8.10. Lowering: 코루틴과 제네릭

이 절에서는 [제너레이터](../004-functions/411-generators.md), [비동기](../004-functions/412-async.md), [스트림](../004-functions/413-stream.md) 에 해당하는 코루틴 lowering 과 [제네릭](../002-types/210-generics.md), [const 제네릭](../002-types/211-const-generics.md) 의 단형화 lowering 을 정의합니다.

## 8.10.1. LLVM Coroutine Intrinsic

[코루틴](../007-runtime/708-coroutine.md) 의 lowering 은 LLVM coroutine intrinsic 을 직접 사용합니다.

핵심 intrinsic 은 다음과 같습니다.

| Intrinsic           | 의미                      |
|---------------------|---------------------------|
| `llvm.coro.id`      | 이 함수가 코루틴임을 표시 |
| `llvm.coro.alloc`   | frame 할당 필요 여부      |
| `llvm.coro.size`    | frame 크기                |
| `llvm.coro.begin`   | frame 초기화              |
| `llvm.coro.suspend` | 일시 중단 지점            |
| `llvm.coro.end`     | 코루틴 종료               |
| `llvm.coro.resume`  | 코루틴 재개               |
| `llvm.coro.destroy` | frame 해제                |

컴파일러가 이 intrinsic들을 emit 하면, LLVM 백엔드가 이를 상태 머신으로 lower해 줍니다.

## 8.10.2. Generator (`yield fn`) lowering

다음과 같은 제너레이터가 있다고 가정해 보겠습니다.

```rok
yield fn count_up_to(n: i32) -> i32 {
    let i = 0;
    while i < n {
        yield i;
        i = i + 1;
    }
}
```

이는 개념적으로 다음과 같은 MLIR 로 lowering 됩니다.

```mlir
func.func @count_up_to(%n: i32) -> !llvm.ptr {
  %id = llvm.intr.coro.id(...) : ...
  %size = llvm.intr.coro.size : i64
  %frame = func.call @rok_coro_frame_alloc(%size, %TYPE_ID_COUNT_UP_TO_FRAME) : (i64, i32) -> !llvm.ptr
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

코드가 다소 복잡해 보이지만, 컴파일러는 intrinsic 만 emit 하면 LLVM 이 알아서 state machine 을 만들어 주기 때문에 실질적인 부담은 크지 않습니다. `yield i` 의 desugar 는 frame 의 `yield` 슬롯에 값을 저장한 뒤 suspend 하는 동작으로 풀립니다. 호출자가 (`for x in iter` 처럼) `iter.next()` 를 호출해 코루틴을 resume 하면, next 는 frame 의 `yield` 슬롯 값을 반환하게 됩니다.

## 8.10.3. Async (`async fn`) lowering

비동기 함수의 예를 살펴보겠습니다.

```rok
async fn fetch() -> Data {
    let r = await op1();
    process(r)
}
```

이는 개념적으로 다음과 같은 MLIR 로 변환됩니다.

```mlir
func.func @fetch() -> !llvm.ptr {
  %id = llvm.intr.coro.id(...) : ...
  %size = llvm.intr.coro.size : i64
  %frame = func.call @rok_coro_frame_alloc(%size, %TYPE_ID_FETCH_FRAME) : ...
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

호출자가 `await fetch()` 를 수행하면 Future 의 `poll` 이 호출되며, 결과가 ready 상태가 되었을 때 frame 의 result slot 값을 반환하게 됩니다.

## 8.10.4. Stream (`async yield fn`) lowering

[스트림](../004-functions/413-stream.md) 은 위의 두 메커니즘을 결합한 형태입니다.

```rok
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

즉 generator 와 `async` 의 결합으로, 하나의 frame 안에서 `yield` 에 의한 일시 중단과 `await` 에 의한 일시 중단이 공존하게 됩니다. state machine 은 어느 종류의 중단으로 들어왔는지를 함께 추적합니다.

## 8.10.5. CoroElide

LLVM 의 CoroElide 패스는 코루틴 호출이 `await` 또는 next 를 통해 직접 끝까지 진행되는 경우 frame 할당 자체를 생략해 줍니다.

조건은 다음과 같습니다.

- 호출자가 그 코루틴을 직접 `await` 하거나 next 로 진행해야 합니다.
- 코루틴의 result 가 escape 하지 않아야 합니다.

이러한 조건이 충족되면 `rok_coro_frame_alloc` 호출이 사라지고, frame 이 호출자의 지역 변수처럼 처리됩니다. 그 결과 자주 일어나는 짧은 `async` `fn` 호출의 비용은 사실상 0 에 수렴합니다.

## 8.10.6. 제네릭 단형화

[제네릭](../002-types/210-generics.md) 의 단형화는 호출 지점의 타입 튜플별로 별개의 인스턴스를 생성하는 방식으로 이루어집니다.

```rok
fn id<T>(x: T) -> T { x }

let a = id(42i32);     // id<i32>
let b = id(3.14f64);   // id<f64>
```

단형화를 거친 뒤의 MLIR 은 다음과 같은 모습이 됩니다.

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

이렇게 각 구체 타입 튜플마다 별개의 함수가 만들어집니다.

### 단형화의 시점

단형화는 다음과 같은 흐름으로 진행됩니다.

```
rok-hir → specialize(hir, type_tuple) → rok-mir → MLIR
```

[AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 모드에서는 모든 호출 지점을 수집한 뒤 일괄적으로 [specialize](../000-meta/007-glossary.md#specialize) 를 수행합니다. [JIT](../000-meta/007-glossary.md#jit-just-in-time) 모드에서는 첫 호출 시점에 specialize 를 수행하고 결과를 캐시합니다 ([실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 참고). 두 모드는 동일한 specialize 메커니즘을 공유합니다.

### 심볼 명명

[컴파일 파이프라인](../000-meta/002-pipeline.md) 에서 사용하는 심볼 맹글링 규칙은 다음과 같습니다.

```
_Y<pkg_hash_8>_<path>_<name>_<typehash_16>
```

예를 들어 `id<i32>` 는 `_Y3f8a1c2b_my_pkg_id_a7c91d8e44b2f190` (예시) 형태가 되며, 이 심볼이 곧 `func.func @...` 의 이름이 됩니다.

## 8.10.7. `const` 제네릭 단형화

[const 제네릭](../002-types/211-const-generics.md) 의 예를 보겠습니다.

```rok
fn dot<T: Num, const N: usize>(a: [T; N], b: [T; N]) -> T { ... }

let r = dot([1.0, 2.0, 3.0], [4.0, 5.0, 6.0]);  // T=f64, N=3
```

이는 다음과 같은 MLIR 로 단형화됩니다.

```
func.func @dot_f64_3(%a: !llvm.array<3 x f64>, %b: !llvm.array<3 x f64>) -> f64 {
  ; ... unroll 또는 일반 loop
}
```

`const` N=3 이 컴파일 타임 상수이므로, LLVM 의 unroll 이나 inlining 이 자유롭게 적용될 수 있습니다.

## 8.10.8. 제네릭 트레잇 디스패치

다음과 같은 제네릭 함수가 있다고 해 보겠습니다.

```rok
fn process<T: Display>(x: T) -> str {
    x.display()
}
```

호출 시점에 구체 타입 T 가 결정되므로, `T = Point` 인 경우라면 다음과 같이 lowering 됩니다.

```
func.func @process_Point(%x: !Point_ptr) -> !str_ptr {
  ; Point 의 Display::display 메서드를 *직접 호출*
  %r = func.call @Point_Display_display(%x) : (!Point_ptr) -> !str_ptr
  func.return %r
}
```

여기에는 vtable 을 통한 간접 호출이 전혀 없으며, 모두 직접 함수 호출로 처리됩니다. 이 모델은 [트레잇-as-타입 금지](../002-types/212-trait-as-type-banned.md) 결정과 결합되어 vtable 비용을 0으로 보장합니다.

## 8.10.9. 합 타입의 분기 디스패치

[합 타입](../002-types/206-sum-types.md) 에 대한 매칭은 `cf.switch` 기반의 decision tree 로 lowering 됩니다.

```rok
let v: i32 | str = ...;
match v {
    n: i32 => use_int(n),
    s: str => use_str(s),
}
```

이를 MLIR 로 옮기면 다음과 같은 모습이 됩니다.

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

태그 검사와 분기는 모두 정적인 코드로 처리됩니다.

## 8.10.10. 트레잇 메서드의 trait-bound dispatch

trait 바운드를 가진 함수의 예를 살펴보겠습니다.

```rok
fn add_all<T: Add<T, Output = T>>(items: List<T>) -> T {
    let r: T = ...;
    for x in items {
        r = r.add(x);
    }
    r
}
```

단형화 후에는 각 호출 지점의 T 마다 별개의 함수가 만들어집니다. `T = i32` 인 경우라면 `r.add(x)` 가 i32 의 `Add::add` 에 대한 직접 호출로 풀리며, 인라인 또한 가능합니다.

## 8.10.11. 정리

코루틴 lowering 은 LLVM intrinsic 에 위임되므로, 컴파일러는 intrinsic 만 emit 하면 됩니다. 제네릭 lowering 은 단형화를 통해 각 구체 타입마다 별개의 코드를 생성하며, vtable 은 사용하지 않습니다.

이 두 메커니즘이 본 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 블록에서 가장 복잡한 lowering 에 해당하며, 나머지 부분인 [Lowering: 데이터 타입과 표현식](808-lowering-types.md) 이나 [Lowering: 제어 흐름과 GC 통합](809-lowering-control.md) 은 표준 다이얼렉트로의 직접 매핑에 가깝습니다.

이로써 8장 (런타임 ABI / MLIR lowering / codegen) 을 마무리합니다.
