# 1.6. `Pinned<T>`: GC pinning

`rok::bare::gc` 모듈은 GC 힙의 객체를 잠시 움직이지 못하게 고정하는 메커니즘을
정의합니다. 원시 `Ptr<T>`가 GC 객체를 가리키는 동안 객체가 이동하면 포인터가
무효가 됩니다. 이 장은 그 충돌을 푸는 방법을 다룹니다.

Rok unsafe tier에서 가장 중요한 장입니다. 포인터와 GC의 충돌을 어떻게 조정하는지
정면으로 설명합니다.

## 1.6.1. 동기: pinning이 필요한 이유

[refs-lang 7.3 GC 알고리듬](../../refs-lang/007-runtime/703-gc-algorithm.md)의 Immix mark-region GC는 **evacuation**(객체를
복사해 옮기기)을 합니다. 따라서 객체의 주소는 GC 주기 사이에 얼마든지 바뀔
수 있습니다.

```
  GC cycle 시작:    obj at 0x1000
  GC evacuation:    obj at 0x2000  (복사 후 forward)
  GC cycle 끝:      obj at 0x2000
```

관리 코드(`GcPtr<T>`)에서는 GC가 루트를 추적하면서 모든 포인터를 자동으로
갱신합니다. 그러나 GC는 원시 `Ptr<T>`를 알지 못하므로 갱신하지 않습니다.
그 포인터는 **낡은 포인터**(stale pointer)가 됩니다.

```rok
use rok::bare::gc;
use rok::bare::ptr::Ptr;

let xs: List<i32> = [1, 2, 3];                 // GcPtr (자동 갱신)
let raw: Ptr<i32> = Ptr::dangling();
using pin = gc::pin(xs) {
    raw = pin.as_data_ptr();                    // raw ptr, pin 안에서만 유효
}                                               // 여기서 unpin
gc::collect();                                  // xs 가 이동할 수 있음
unsafe { raw.read() };                          // UB: raw 가 stale
```

해법은 **pinning**입니다. GC에 "이 객체를 옮기지 말라"고 명시적으로 알립니다.

## 1.6.2. `Pinned<T>` guard

```rok
struct Pinned<T> {
    obj: T;
    // 내부: GC 의 pin set 등록
}

trait Pinnable {
    // 값의 data 를 담은 GC 객체; 움직일 것이 없으면 none
    fn gc_object(self) -> GcBuf<u8> | none;
}
impl<T> Pinnable for List<T> { /* list 의 buffer; 비어 있으면 none */ }
impl Pinnable for str { /* none (2.15.2) */ }
impl<T> Pinnable for GcPtr<T> { /* 그 cell */ }

fn pin<T: Pinnable>(obj: T) -> Pinned<T>;
fn collect();                           // 지금 전체 collection; pin 되지 않은 객체는 모두 이동 가능

impl<T> Pinned<T> {
    fn obj(self) -> T;
}

impl<T> Drop for Pinned<T> {
    fn drop(ref self) {
        // GC pin set 에서 제거
    }
}
```

`using`으로 묶는 guard입니다. `pin()`이 반환한 `Pinned<T>`가 `using` 블록 안에 있는
동안 GC는 그 객체를 옮기지 않습니다. 블록을 빠져나갈 때 `Pinned::drop`이 호출되어 고정이
풀립니다. `using` 없이 받은 `Pinned<T>`는 자동으로 풀리지 않습니다. pin은 횟수를 셉니다. 객체는 고정한 횟수만큼 풀어 준 뒤에야 다시 움직입니다. `pin`은 `gc_object`가 가리키는 객체를 고정합니다. `none`이 돌아오면 아무것도 고정하지 않습니다. `GcBuf<u8>`은 [1.3.9](103-mem.md#139-gcbuft와-buf_-기본-요소)에 나오는, 타입이 지워진 GC 버퍼입니다. 사용자 타입은 자기 데이터를 담은 GC 객체를 돌려주는 방식으로 `Pinnable`을 구현할 수 있습니다.

### 예: FFI 호출

```rok
let buf: List<u8> = read_input();

using pin = rok::bare::gc::pin(buf) {
    let raw: Ptr<u8> = pin.as_data_ptr();
    unsafe {
        ffi::process(raw, buf.len());          // FFI 가 buf 데이터 사용
    }
}     // ← scope 끝에서 자동 unpin
```

여기서 `using` 블록은 [refs-lang 3.17 using 표현식](../../refs-lang/003-expressions/317-using-expression.md)이 정의하는 RAII 형태를
따릅니다.

## 1.6.3. `GcPtr<T>`와 `Ptr<T>`: 타입 수준 분리

```rok
struct GcPtr<T> { /* compiler-managed; root 자동 추적 */ }
struct Ptr<T>   { /* raw; root 추적 X */ }
```

| 측면       | `GcPtr<T>`                    | `Ptr<T>`                                         |
|------------|-------------------------------|--------------------------------------------------|
| GC 루트    | **자동 추적**                 | **추적하지 않음**                                |
| 이동 시    | **자동 갱신**                 | **낡음**                                         |
| 사용처     | 일반 코드 (`List`/`Dict`/...) | unsafe + FFI만                                  |
| 생성       | GC 할당 (`List::new()` 등)    | `gc::pin().as_data_ptr()` 또는 `alloc::alloc(layout)` |
| 유효 기간  | 객체에 도달할 수 있는 동안    | pin guard 스코프 안에서만                        |

타입 시스템이 두 종류를 나누므로 사용자 코드는 보통 `GcPtr`만 다룹니다.
이 포인터는 `List<T>` 등의 안에 보이지 않게 감싸여 있습니다.
`Ptr<T>`는 unsafe에 들어간 뒤에야 명시적으로 나타납니다.

### 연산

```rok
impl<T> GcPtr<T> {
    fn new(value: T) -> GcPtr<T>;           // value 를 담은 새 GC cell
    fn get(self) -> T;
    fn set(self, value: T);                 // GcPtr 의 모든 복사본이 봄
    fn same(self, other: GcPtr<T>) -> bool; // 동일성: 같은 cell
}
impl<T> !Sync for GcPtr<T> {}
impl<T> Pinnable for GcPtr<T> {}
```

### 내부 구현: GcPtr의 형태

```rok
struct GcPtr<T> {
    // 내부: 보통 단순 `*T`. GC 가 외부에서 root set 으로 추적
    // 지역 변수면 컴파일러가 emit 하는 shadow stack frame 슬롯에 둠
}
```

Rok의 GC는 **스레드별 shadow stack**으로 루트 집합을 추적합니다. 이 shadow stack은 컴파일러가 직접 내보냅니다([refs-lang 7.4.1](../../refs-lang/007-runtime/704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-내보내는-shadow-stack)).

## 1.6.4. pin / unpin의 GC 측 메커니즘

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

이 구조는 Go 1.21의 `runtime.Pinner`와 거의 같습니다. 자세한 내용은
[pkg.go.dev/runtime#Pinner](https://pkg.go.dev/runtime#Pinner)에 있습니다.

### 비용

비용은 다음과 같습니다.

- pin 한 번은 O(1) 해시 삽입입니다.
- unpin 한 번도 O(1) 해시 삭제입니다.
- GC 주기 안에 pin 집합 검사가 추가되지만, 비용은 작습니다.

대부분 자주 실행되는 경로(hot path)에 들어가지 않습니다. FFI 호출 자체가 그리 잦지 않기 때문입니다.

## 1.6.5. 한계: pin 안 해도 될 자리

다음과 같은 상황에서는 pin이 **불필요**합니다.

| 자리             | 이유                                    |
|------------------|-----------------------------------------|
| 일반 메서드 호출 | GcPtr 자동 갱신, 원시 포인터 안 만듦   |
| GcPtr끼리 연산  | 모두 GC가 추적                         |
| 짧은 FFI (GC 없음) | FFI 안에서 GC가 돌지 않는다고 보장될 때 |
| Immortal const   | 이미 이동하지 않음 (문자열 리터럴 등)   |
| 이동 없는 영역   | LOS (large object space), 고정 영역     |

짧은 FFI 호출은 컴파일러가 자동으로 판단합니다. 이 FFI 호출 동안 GC가 돌지 않는다고 보장할 수 있으면 컴파일러가 pin을 자동으로 생략합니다. 다만 일반적인 경우에는 **명시 pin**을 권장합니다.

## 1.6.6. 추출 메서드

```rok
impl<T> Pinned<List<T>> {
    fn as_data_ptr(self) -> Ptr<T>;          // List 의 buffer 시작
    fn as_data_many_ptr(self) -> ManyPtr<T>; // 다수 ptr
}

impl<T> Pinned<GcPtr<T>> {
    fn as_value_ptr(self) -> Ptr<T>;         // cell 의 값
}

impl Pinned<str> {
    fn as_bytes_ptr(self) -> ConstPtr<u8>;   // UTF-8 bytes
    fn as_c_str(self) -> ConstPtr<c_char>;   // null-terminated 보장 안 함. 별도 변환
}
```

이 컨테이너 메서드가 추출 API입니다. 결과는 `Pinned`를 들고 있는 동안 유효합니다. `str::as_ptr`와 `CStr::as_ptr`는 pin 없이 주소를 주고, 더 좁은 [2.15.2](../200-core/215-string.md#2152-str-불변-utf-8-문자열)의 규칙을 따릅니다. 제네릭 `as_ptr`은 없습니다. `as_value_ptr`는 `List`의 `as_data_ptr`와 구별하려고 붙인 이름입니다. 컨테이너마다 전용 추출 메서드가 있습니다.
그래서 버퍼 시작이나 null 종료 형태처럼 각 컨테이너의 레이아웃에 맞는 형태로
데이터를 꺼낼 수 있습니다.

## 1.6.7. GC 런타임 연계

각 항목에 대응하는 GC 런타임 호출은 다음과 같습니다.

| 항목             | 런타임 호출                    |
|------------------|--------------------------------|
| `Pinned<T>`      | `rok_gc_pin` / `rok_gc_unpin`  |
| `pin<T>()`       | (위와 동일)                    |
| `collect()`      | `rok_gc_collect`               |
| `GcPtr<T>`       | shadow stack 프레임 슬롯 (컴파일러가 내보냄) |
| `as_data_ptr` 등 | 컨테이너별 레이아웃에 따름     |
| `Weak<T>`        | `rok_gc_weak_new` / `rok_gc_weak_get` (1.6.9) |

## 1.6.8. 다른 언어와 비교

| 측면             | Java JNI                       | .NET fixed             | Go 1.21 Pinner           | Rok                       |
|------------------|--------------------------------|------------------------|--------------------------|---------------------------|
| pin API          | `Get*Critical`                 | `fixed (T* p = ...)`   | `runtime.Pinner`         | `rok::bare::gc::pin(obj)`  |
| RAII guard       | `Release` 명시                 | block scope            | `defer pin.Unpin()`      | `using` block (자동 drop) |
| 타입 분리        | **없음** (`jobject` 모두 동일) | **없음**               | **없음** (모든 포인터 같음) | `GcPtr<T>` vs `Ptr<T>`    |
| pin 후 만료 검증 | **없음**                       | 컴파일러 (block scope) | **없음** (수동)          | 타입 시스템 + 스코프      |
| 다중 pin         | overlapping OK                 | **제한**               | overlapping OK           | overlapping OK (count)    |

Rok이 다른 언어와 크게 다른 점은 두 가지입니다.

1. 타입 시스템이 강제합니다. `GcPtr<T>`와 `Ptr<T>`는 서로 다른
   타입이고, 컴파일러는 자동 변환을 거부합니다. 둘 사이는 pin guard로만 건널 수 있습니다.
2. `using` 블록으로 해제합니다. `using` 블록을 빠져나갈 때 `Pinned<T>`가 해제되므로 unpin을
   빠뜨릴 수 없습니다. Go의 수동 `defer` 방식보다 안전한
   설계입니다.

> 이 정책은 Rok 포인터 모델의 뼈대입니다. unsafe와 GC, 두 영역이 안전하게
> 만나는 접점입니다.

## 1.6.9. `Weak<T>`: 약한 참조

```rok
// rok::bare::gc
struct Weak<T> { /* opaque */ }

impl<T> Weak<T> {
    fn new(target: GcPtr<T>) -> Weak<T>;
    fn get(self) -> GcPtr<T> | none;        // 대상이 해제되면 none
}
impl<T> !Sync for Weak<T> {}
impl<T> !Send for Weak<T> {}    // 넘김이 그것이 가리키는 것을 복사할 수 없음 (2.13.9)
```

약한 참조는 객체를 살려 두지 않습니다. 수집이 객체를 해제하면 그 객체를 가리키는 모든 `Weak`는 `none`을 읽습니다. 객체를 옮기는 수집은 `Weak`를 갱신합니다. 이 동작이 `WEAK_REF` 스캔 종류([refs-lang 8.2.8](../../refs-lang/008-abi/802-scan-table.md#828-weak_ref))의 라이브러리 인터페이스입니다.

런타임에서 `Weak`는 type id `3`([refs-lang 8.2.12](../../refs-lang/008-abi/802-scan-table.md#8212-런타임-예약-type-id))의 GC 셀이고, `WEAK_REF` 워드 하나를 담습니다. `rok_gc_weak_new`로 만들고 `rok_gc_weak_get`으로 읽습니다([refs-lang 8.5.12](../../refs-lang/008-abi/805-extern-symbols.md#8512-보조-심볼부록-비규범)).
