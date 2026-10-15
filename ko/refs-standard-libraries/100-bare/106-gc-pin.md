# 1.6. `Pinned<T>` — GC pinning

`rok::bare::gc` 모듈은 GC heap 위에 놓인 객체를 일시적으로 이동 불가능한 상태로
고정하는 메커니즘을 정의합니다. raw `Ptr<T>` 가 GC 객체를 가리키고 있을 때 객체가
이동하면 포인터가 무효가 되는데, 이 장은 바로 그 충돌을 해소하는 방법을 다룹니다.

이 장은 rok unsafe tier 의 가장 중요한 장으로, pointer 와 GC 의 충돌을 어떻게
조정하는지를 정면에서 설명합니다.

## 1.6.1. 동기 — pinning 이 필요한 이유

[refs-lang 7.3 GC 알고리듬](../../refs-lang/007-runtime/703-gc-algorithm.md) 의 Immix mark-region GC 는 **evacuation** (객체
복사 이동) 을 수행합니다. 이로 인해 객체의 주소는 GC cycle 사이에 얼마든지 바뀔
수 있습니다.

```
  GC cycle 시작:    obj at 0x1000
  GC evacuation:    obj at 0x2000  (복사 후 forward)
  GC cycle 끝:      obj at 0x2000
```

managed code (`GcPtr<T>`) 의 경우 GC 가 root 를 추적하면서 모든 ptr 을 자동으로
갱신합니다. 그러나 raw `Ptr<T>` 는 GC 가 인지하지 못하기 때문에 갱신 대상에서
빠지고, 결과적으로 **stale ptr** 이 되어버립니다.

```rok
let xs: List<i32> = [1, 2, 3];                 // GcPtr (자동 갱신)
let raw: Ptr<i32> = rok::bare::ptr::list_data(xs); // raw ptr
                                                // ↓ GC cycle 발생 가능
gc::collect();
unsafe { raw.read() };                          // UB — raw 가 stale
```

해법은 **pinning** 입니다. GC 에게 "이 객체는 이동시키지 말라"는 신호를 명시적으로
전달하는 방식입니다.

## 1.6.2. `Pinned<T>` guard

```rok
struct Pinned<T> {
    obj: GcPtr<T>;
    // 내부: GC 의 pin set 등록
}

fn pin<T>(obj: GcPtr<T>) -> Pinned<T>;

impl<T> Pinned<T> {
    fn as_ptr(self) -> Ptr<T>;
    fn as_const_ptr(self) -> ConstPtr<T>;
    fn obj(self) -> GcPtr<T>;
}

impl<T> Drop for Pinned<T> {
    fn drop(ref self) {
        // GC pin set 에서 제거
    }
}
```

`using` 으로 묶는 guard 입니다. `pin()` 이 반환한 `Pinned<T>` 가 `using` 블록 안에 있는
동안 GC 는 해당 객체를 이동시키지 않으며, 블록을 빠져나갈 때 `Pinned::drop` 이 불려 unpin
됩니다. `using` 없이 받은 `Pinned<T>` 는 자동으로 풀리지 않습니다.

### 사용 예 — FFI 호출

```rok
let buf: List<u8> = read_input();

using pin = rok::bare::gc::pin(buf) {
    let raw: Ptr<u8> = pin.as_ptr().cast::<u8>();
    unsafe {
        ffi::process(raw, buf.len());          // FFI 가 buf 데이터 사용
    }
}     // ← scope 끝에서 자동 unpin
```

여기서의 `using` 블록은 [refs-lang 3.17 using 표현식](../../refs-lang/003-expressions/317-using-expression.md) 이 정의하는 RAII 형태를
그대로 따릅니다.

## 1.6.3. `GcPtr<T>` vs `Ptr<T>` — type level 분리

```rok
struct GcPtr<T> { /* compiler-managed — root 자동 추적 */ }
struct Ptr<T>   { /* raw — root 추적 X */ }
```

| 측면       | `GcPtr<T>`                    | `Ptr<T>`                                         |
|------------|-------------------------------|--------------------------------------------------|
| GC root    | **자동 추적**                 | **추적하지 않음**                                |
| 이동 시    | **자동 갱신**                 | **stale**                                        |
| 사용처     | 일반 코드 (`List`/`Dict`/...) | unsafe + FFI 만                                  |
| 생성       | GC alloc (`List::new()` 등)   | `gc::pin().as_ptr()` 또는 `alloc::alloc(layout)` |
| valid 기간 | 객체 reachable 동안           | pin guard scope 안만                             |

타입 시스템 차원에서 두 종류를 분리해 두었기 때문에, 사용자 코드는 일반적으로
`GcPtr` 만 다루게 됩니다 (보이지 않게 `List<T>` 등으로 wrap 되어 있기 때문입니다).
`Ptr<T>` 는 unsafe 진입 이후에 비로소 명시적으로 모습을 드러냅니다.

### 내부 구현 — GcPtr 의 형태

```rok
struct GcPtr<T> {
    // 내부: 보통 단순 `*T` — GC 가 외부에서 root set 으로 추적
    // 지역 변수면 컴파일러가 emit 하는 shadow stack frame 슬롯에 둠
}
```

rok 의 GC 는 컴파일러가 직접 emit 하는 **thread 별 shadow stack** 으로 root set 을 추적합니다 ([refs-lang 7.4.1](../../refs-lang/007-runtime/704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-emit-하는-shadow-stack)).

## 1.6.4. pin / unpin 의 GC 측 메커니즘

```
GC pin set 구조.
  - 해시 테이블: Ptr<Object> → pin count
  - GC 가 evacuation 시 pin count > 0 인 객체 _건너뜀_

pin(obj):
  1. obj 의 헤더에서 pin bit 확인 (또는 pin set 에 등록)
  2. count++
  3. Pinned<T> 반환

drop(Pinned):
  1. count--
  2. count == 0 면 pin bit 해제
```

이 구조는 Go 1.21 의 `runtime.Pinner` 와 사실상 동형입니다. 자세한 내용은
[pkg.go.dev/runtime#Pinner](https://pkg.go.dev/runtime#Pinner) 를 참고합니다.

### overhead

비용은 다음과 같이 정리됩니다.

- pin 한 번은 O(1) hash insert 에 그칩니다
- unpin 한 번 또한 O(1) hash remove 입니다
- GC cycle 안에서 pin set 검사가 추가되긴 하지만, 그 비용은 작습니다

대부분의 경우 hot path 에 들어가지 않습니다. FFI 호출 자체가 그렇게 빈번한
연산이 아니기 때문입니다.

## 1.6.5. 한계 — pin 안 해도 될 자리

다음과 같은 상황에서는 pin 이 **불필요**합니다.

| 자리             | 이유                                    |
|------------------|-----------------------------------------|
| 일반 method 호출 | GcPtr 자동 갱신 — raw ptr 안 만듬       |
| GcPtr 끼리 연산  | 모두 GC 가 추적                         |
| 짧은 FFI (no GC) | FFI 안에서 GC 안 도는 보장 시           |
| Immortal const   | 이미 이동하지 않음 (string literal 등)  |
| Non-moving 영역  | LOS (large object space), pinned region |

짧은 FFI 자리에는 자동 판단이 적용됩니다. 컴파일러가 "이 FFI 호출 동안 GC 가 돌지 않는다"는 사실을 보장할 수 있는 경우 pin 이 자동으로 생략됩니다. 다만 일반적인 경우에는 **명시 pin** 을 권장합니다.

## 1.6.6. `as_ptr` 외 추출 메서드

```rok
impl<T> Pinned<T> {
    fn as_ptr(self) -> Ptr<T>;
    fn as_const_ptr(self) -> ConstPtr<T>;
}

impl<T> Pinned<List<T>> {
    fn as_data_ptr(self) -> Ptr<T>;          // List 의 buffer 시작
    fn as_data_many_ptr(self) -> ManyPtr<T>; // 다수 ptr
}

impl Pinned<str> {
    fn as_bytes_ptr(self) -> ConstPtr<u8>;   // UTF-8 bytes
    fn as_c_str(self) -> ConstPtr<c_char>;   // null-terminated 보장 안 함 — 별도 변환
}
```

container 별로 전용 추출 메서드가 마련됩니다. buffer 시작이나
null-terminated 형태처럼, 각 컨테이너의 layout 에 맞춘 형태로 꺼내 쓸 수 있도록
한 것입니다.

## 1.6.7. GC runtime 연계

각 항목이 GC 런타임의 어떤 호출과 연결되는지는 다음과 같습니다.

| 항목             | runtime 호출                   |
|------------------|--------------------------------|
| `Pinned<T>`      | `rok_gc_pin` / `rok_gc_unpin`  |
| `pin<T>()`       | (위와 동일)                    |
| `GcPtr<T>`       | shadow stack frame 슬롯 (컴파일러 emit) |
| `as_data_ptr` 등 | container 별 layout 의존       |

## 1.6.8. 다른 언어와의 비교

| 측면             | Java JNI                       | .NET fixed             | Go 1.21 Pinner           | rok                       |
|------------------|--------------------------------|------------------------|--------------------------|---------------------------|
| pin API          | `Get*Critical`                 | `fixed (T* p = ...)`   | `runtime.Pinner`         | `rok::bare::gc::pin(obj)`  |
| RAII guard       | `Release` 명시                 | block scope            | `defer pin.Unpin()`      | `using` block (자동 drop) |
| 타입 분리        | **없음** (`jobject` 모두 동일) | **없음**               | **없음** (모든 ptr 같음) | `GcPtr<T>` vs `Ptr<T>`    |
| pin 후 만료 검증 | **없음**                       | 컴파일러 (block scope) | **없음** (수동)          | type system + scope       |
| 다중 pin         | overlapping OK                 | **제한**               | overlapping OK           | overlapping OK (count)    |

rok 이 다른 언어와 갈라지는 가장 큰 지점은 두 가지입니다.

1. 타입 시스템이 강제한다는 점입니다. `GcPtr<T>` 와 `Ptr<T>` 는 서로 다른
   타입이며, 자동 변환은 거부됩니다. 오직 pin guard 를 통해서만 건너갈 수 있습니다.
2. `using` block 기반의 해제입니다. `using` 블록을 빠져나갈 때 `Pinned<T>` 가 drop 되기 때문에 unpin
   누락이 일어나지 않습니다. Go 의 수동 `defer` 방식보다 한층 안전한
   설계입니다.

> 이 정책은 rok pointer 모델의 backbone 입니다. unsafe 와 GC, 두 영역이 안전하게
> 만나는 접점입니다.
