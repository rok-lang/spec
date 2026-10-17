# 8.10. Lowering: 코루틴과 제네릭

이 절에서는 [제너레이터](../004-functions/411-generators.md), [비동기](../004-functions/412-async.md), [스트림](../004-functions/413-stream.md) 에 해당하는 코루틴 lowering 과 [제네릭](../002-types/210-generics.md), [const 제네릭](../002-types/211-const-generics.md) 의 단형화 lowering 을 정의합니다.

## 8.10.1. 상태 기계

[코루틴](../007-runtime/708-coroutine.md) 은 컴파일러가 만든 상태 기계로 lower 합니다. `llvm.coro.*` 는 쓰지 않습니다.

1. HIR: `yield` / `async` / `async yield` 블록을 클로저처럼 body 함수로 들어 올립니다. 블록 자리에는 frame 을 만드는 식이 남습니다.
2. MIR: body 함수를 resume 함수로 바꿉니다. suspend 지점마다 state 번호를 매기고, 그 지점을 넘어 살아 있는 지역을 frame 필드로 옮깁니다.
3. frame 타입은 `(i32 state, locals..)` 이며 ScanTable 을 가집니다. frame 은 일반 할당 경로로 만듭니다 ([8.9.9](809-lowering-control.md#899-gc-alloc-fast--slow-path)).
4. resume 함수의 입구는 `state` 로 분기 (`switch`) 합니다.

## 8.10.2. Generator (`yield fn`)

```rok
yield fn count_up_to(n: i32) -> i32 {
    let i = 0;
    while i < n {
        yield i;
        i = i + 1;
    }
}
```

개념적인 resume 함수 (`next`) 는 다음과 같습니다.

```
next(frame) -> i32 | none:
    switch frame.state:
      0: frame.i = 0; goto loop
      1: frame.i = frame.i + 1; goto loop
      2: return none
    loop:
      if frame.i < frame.n:
          frame.state = 1
          return frame.i          // yield i
      frame.state = 2
      return none
```

`count_up_to(n)` 은 frame 을 할당하고 `state = 0` 으로 둔 래퍼 (`GenIter<i32>`) 를 돌려줍니다.

## 8.10.3. Async (`async fn`)

```rok
async fn fetch() -> Data {
    let r = await op1();
    process(r)
}
```

```
poll(frame, cx) -> Poll<Data>:
    switch frame.state:
      0: frame.fut = op1(); frame.state = 1; goto await1
      1: goto await1
    await1:
      match frame.fut.poll(cx):
        Pending  => return Pending
        Ready(r) => frame.state = 2; return Ready(process(r))
```

`await` 는 대상 future 를 frame 에 두고 poll 합니다. Pending 이면 state 를 남긴 채 Pending 을 돌려주고, 다음 poll 이 같은 지점에서 이어 갑니다.

## 8.10.4. Stream (`async yield fn`)

[스트림](../004-functions/413-stream.md) 은 둘을 합칩니다. 한 frame 의 state 번호가 `yield` 지점과 `await` 지점을 함께 셉니다. `poll_next` 는 `yield` 에서 `Ready(값)`, `await` 의 Pending 에서 `Pending`, 끝에서 `Ready(none)` 을 돌려줍니다.

## 8.10.5. frame 할당

frame 은 코루틴을 부를 때마다 할당합니다. 할당을 생략하는 최적화는 없습니다.

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

```rok
fn process<T: Describe>(x: T) -> str {
    x.describe()
}
```

호출 시점에 구체 타입 T 가 결정되므로, `T = Point` 인 경우라면 다음과 같이 lowering 됩니다.

```
func.func @process_Point(%x: !Point_ptr) -> !str_ptr {
  ; Point 의 Describe::describe 를 직접 호출
  %r = func.call @Point_Describe_describe(%x) : (!Point_ptr) -> !str_ptr
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

코루틴은 컴파일러가 직접 상태 기계로 lower 하며 (8.10.1), `llvm.coro.*` 는 쓰지 않습니다. 제네릭 lowering 은 단형화를 통해 각 구체 타입마다 별개의 코드를 생성하며, vtable 은 사용하지 않습니다.

이 두 메커니즘이 본 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 블록에서 가장 복잡한 lowering 에 해당하며, 나머지 부분인 [Lowering: 데이터 타입과 표현식](808-lowering-types.md) 이나 [Lowering: 제어 흐름과 GC 통합](809-lowering-control.md) 은 표준 다이얼렉트로의 직접 매핑에 가깝습니다.

이로써 8장 (런타임 ABI / MLIR lowering / codegen) 을 마무리합니다.
