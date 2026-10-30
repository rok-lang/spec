# 8.10. Lowering: 코루틴과 제네릭

이 절은 [제너레이터](../004-functions/411-generators.md), [비동기](../004-functions/412-async.md), [스트림](../004-functions/413-stream.md)을 코루틴으로 낮추는 규칙과 [제네릭](../002-types/210-generics.md), [const 제네릭](../002-types/211-const-generics.md)을 단형화로 낮추는 규칙을 정의합니다.

## 8.10.1. 상태 기계

[코루틴](../007-runtime/708-coroutine.md)은 컴파일러가 만든 상태 기계로 낮춥니다. `llvm.coro.*`는 쓰지 않습니다.

1. HIR: `yield` / `async` / `async yield` 블록을 클로저처럼 body 함수로 끌어올립니다. 블록 자리에는 프레임을 만드는 식이 남습니다.
2. MIR: body 함수를 resume 함수로 바꿉니다. suspend 지점마다 state 번호를 매깁니다. 그 지점을 넘어서도 살아 있는 지역 변수는 프레임 필드로 옮깁니다.
3. 프레임 타입은 `(i32 state, locals..)`이며 ScanTable이 있습니다. 프레임은 일반 할당 경로로 만듭니다([8.9.9](809-lowering-control.md#899-gc-할당-fast--slow-path)).
4. resume 함수의 입구는 `state`로 분기(`switch`)합니다.

## 8.10.2. 제너레이터(`yield fn`)

```rok
yield fn count_up_to(n: i32) -> i32 {
    let i = 0;
    while i < n {
        yield i;
        i = i + 1;
    }
}
```

개념상 resume 함수(`next`)는 다음과 같습니다.

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

`count_up_to(n)`은 프레임을 할당하고, `state = 0`으로 둔 래퍼(`GenIter<i32>`)를 돌려줍니다.

## 8.10.3. 비동기(`async fn`)

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

`await`는 대상 future를 프레임에 두고 폴링합니다. 결과가 Pending이면 state를 그대로 둔 채 Pending을 돌려줍니다. 다음 폴링은 같은 지점에서 이어 갑니다.

## 8.10.4. 스트림(`async yield fn`)

[스트림](../004-functions/413-stream.md)은 둘을 합칩니다. 한 프레임의 state 번호가 `yield` 지점과 `await` 지점을 함께 셉니다. `poll_next`는 `yield`에서 `Ready(값)`을, `await`가 Pending이면 `Pending`을, 끝에서 `Ready(none)`을 돌려줍니다.

## 8.10.5. 프레임 할당

프레임은 코루틴을 호출할 때마다 할당합니다. 호출자가 끝까지 실행하는 코루틴이라고 따로 할당을 생략하지는 않습니다. 밖으로 나가지 않는 프레임은 다른 할당처럼 없어질 수 있습니다([7.2.12](../007-runtime/702-allocation.md#7212-할당-생략)).

## 8.10.6. 제네릭 단형화

[제네릭](../002-types/210-generics.md)을 단형화하면 호출 지점의 타입 튜플마다 별개의 인스턴스가 생깁니다.

```rok
fn id<T>(x: T) -> T { x }

let a = id(42i32);     // id<i32>
let b = id(3.14f64);   // id<f64>
```

단형화를 거친 MLIR은 다음과 같습니다.

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

구체 타입 튜플마다 별개의 함수가 만들어집니다.

### 단형화의 시점

단형화는 다음 흐름으로 진행됩니다.

```
rok-hir → specialize(hir, type_tuple) → rok-mir → MLIR
```

[AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 모드에서는 모든 호출 지점을 모은 뒤 한꺼번에 특수화([specialize](../000-meta/007-glossary.md#specialize))합니다. [JIT](../000-meta/007-glossary.md#jit-just-in-time) 모드에서는 첫 호출 때 특수화하고 결과를 캐시합니다([실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 참고). 두 모드는 같은 특수화 메커니즘을 씁니다.

### 심볼 명명

[컴파일 파이프라인](../000-meta/002-pipeline.md)에서 쓰는 심볼 맹글링 규칙은 다음과 같습니다.

```
_Y<pkg_hash_8>_<path>_<name>_<typehash_16>
```

예를 들어 `id<i32>`는 `_Y3f8a1c2b_my_pkg_id_a7c91d8e44b2f190`(예시) 같은 형태가 됩니다. 이 심볼이 `func.func @...`의 이름입니다.

## 8.10.7. `const` 제네릭 단형화

[const 제네릭](../002-types/211-const-generics.md)의 예입니다.

```rok
fn dot<T: Num, const N: usize>(a: [T; N], b: [T; N]) -> T { ... }

let r = dot([1.0, 2.0, 3.0], [4.0, 5.0, 6.0]);  // T=f64, N=3
```

이 코드는 다음 MLIR로 단형화됩니다.

```
func.func @dot_f64_3(%a: !llvm.array<3 x f64>, %b: !llvm.array<3 x f64>) -> f64 {
  ; ... unroll 또는 일반 loop
}
```

`const` N=3이 컴파일 타임 상수이므로 LLVM이 루프 펼치기(unroll)와 인라인을 자유롭게 적용할 수 있습니다.

## 8.10.8. 제네릭 트레잇 디스패치

```rok
fn process<T: Describe>(x: T) -> str {
    x.describe()
}
```

구체 타입 T는 호출 지점에서 정해집니다. `T = Point`이면 다음과 같이 낮춥니다.

```
func.func @process_Point(%x: !Point_ptr) -> !str_ptr {
  ; Point 의 Describe::describe 를 직접 호출
  %r = func.call @Point_Describe_describe(%x) : (!Point_ptr) -> !str_ptr
  func.return %r
}
```

vtable을 거치는 간접 호출은 없고 모두 직접 함수 호출입니다. 이 모델과 [트레잇-as-타입 금지](../002-types/212-trait-as-type-banned.md)가 함께 vtable 비용이 0임을 보장합니다.

## 8.10.9. 합 타입의 분기 디스패치

[합 타입](../002-types/206-sum-types.md)의 매칭은 `cf.switch` 기반 decision tree로 낮춥니다.

```rok
let v: i32 | str = ...;
match v {
    n: i32 => use_int(n),
    s: str => use_str(s),
}
```

MLIR로는 다음과 같습니다.

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

tag 검사와 분기는 모두 정적인 코드입니다.

## 8.10.10. 트레잇 메서드의 트레잇 바운드 디스패치

```rok
fn add_all<T: Add<T, Output = T>>(items: List<T>) -> T {
    let r: T = ...;
    for x in items {
        r = r.add(x);
    }
    r
}
```

단형화 뒤에는 호출 지점의 T마다 별개의 함수가 만들어집니다. `T = i32`이면 `r.add(x)`는 i32의 `Add::add`를 직접 호출하는 코드로 풀립니다. 이 호출은 인라인할 수도 있습니다.

## 8.10.11. 정리

컴파일러는 코루틴을 직접 상태 기계로 낮추고(8.10.1) `llvm.coro.*`는 쓰지 않습니다. 제네릭은 단형화로 구체 타입마다 별개의 코드를 만들어 낮추고, vtable은 쓰지 않습니다.

이 두 메커니즘이 이 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 블록에서 가장 복잡한 lowering입니다. 나머지인 [Lowering: 데이터 타입과 표현식](808-lowering-types.md)과 [Lowering: 제어 흐름과 GC 통합](809-lowering-control.md)은 표준 다이얼렉트에 거의 그대로 대응합니다.

8장(런타임 ABI / MLIR lowering / codegen)은 여기까지입니다.
