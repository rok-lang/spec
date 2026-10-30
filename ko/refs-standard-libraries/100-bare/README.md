# 1. Unsafe 계층

Unsafe tier는 저수준이거나 위험한 표준 라이브러리 항목을 정의합니다. 원시 포인터, transmute, 원시 할당, GC 고정(pinning), FFI 기본 요소, MLIR 다이얼렉트 직접 접근처럼 안전한 추상화 바깥에서 동작하는 도구는 모두 여기에 있습니다.

## 의존성

위쪽으로는 런타임이 노출하는 `extern "C"` 심볼에만 의존합니다. 아래쪽으로는 `rok`, `std`, 사용자 코드가 이 tier에 의존합니다. 단, 명시적으로 가져올 때만 의존할 수 있습니다.

## 사용자 import 정책

```rok
use rok::bare::ptr::Ptr;                              // 명시 import 필수 - 위험 시그널
use rok::bare::alloc::{alloc, Layout};

let layout = try! Layout::array::<u8>(1024);
let p: Ptr<u8> = unsafe { alloc(layout) };
```

묵시적으로 적용되는 `use rok::core::**;`에는 unsafe 항목이 들어 있지 않습니다. 따라서 `rok::bare::*`는 따로 명시적으로 가져와야 합니다. 그래서 사용자가 위험 영역에 들어가는 지점이 코드에 드러납니다.

## 모듈 목차

### 포인터 계열(5종)

- [1.1. `Ptr<T>` / `ConstPtr<T>` / `ManyPtr<T>` / `CPtr<T>` - Zig 식 arity-encoded](101-ptr.md)
- [1.2. cast 연산 - `ptr_cast`, `align_cast`, `const_cast`, `reinterpret`](102-cast.md)

### 메모리 / 변환

- [1.3. `MaybeUninit<T>` / `transmute_unchecked` - bit-level 변환](103-mem.md)
- [1.4. raw allocation (`alloc`, `dealloc`, `realloc`)](104-alloc.md)
- [1.5. volatile 메모리 접근 - `volatile::read` / `volatile::write`](105-volatile.md)

### GC 통합

- [1.6. `Pinned<T>` / `gc::pin` - GC가 객체를 이동 못 하게](106-gc-pin.md)

### FFI

- [1.7. extern "C" / `CStr` / `CString` / `CPtr` - C ABI 호환](107-ffi.md)
- [1.8. WASM Component Model 인터페이스 (`ffi.wit`)](108-wit.md)

### MLIR 백엔드

- [1.9. `rok::bare::mlir`: MLIR 다이얼렉트 직접 접근](109-mlir.md)
- [1.10. `rok::bare::mlir::plugin`: 사용자 정의 다이얼렉트 등록](110-mlir-plugin.md)

## 핵심 설계 원칙

### 1. Strict provenance: Rust 2024 모델 채택

`Ptr<T>`는 (주소, provenance) 쌍이고, 단순한 정수가 아닙니다. 그래서 `usize ↔ Ptr<T>` 왕복 변환은 provenance를 잃습니다. 이 변환은 명시적인 탈출구(escape hatch)를 거칠 때만 할 수 있습니다.

```rok
let p: Ptr<i32> = ...;
let addr: usize = p.addr();              // 안전 - 주소만
let q: Ptr<i32> = p.with_addr(addr + 4); // 안전 - provenance 보존, address 만 변경
let r: Ptr<i32> = rok::bare::ptr::from_exposed_provenance(addr);  // unsafe - provenance 새로
```

`as` 캐스트는 포인터에 어떤 경우에도 쓸 수 없습니다. 이 방향의 근거는 [Ralf Jung "Pointers Are Complicated"](https://www.ralfj.de/blog/2022/04/11/provenance-exposed.html)입니다. Rust 1.84(2024년 12월)가 안정화한 모델과 동형입니다.

### 2. 타입에 담긴 arity: Zig 모델

| 타입          | 의미                               | C 대응          |
|---------------|------------------------------------|-----------------|
| `Ptr<T>`      | 정확히 **하나**의 T               | `T*` (single)   |
| `ConstPtr<T>` | 정확히 **하나**의 const T         | `const T*`      |
| `ManyPtr<T>`  | **연속 N개** T (길이 모름, C 호환) | `T*` (array)    |
| `Slice<T>`    | **연속 N개** T (길이 포함)         | (ptr, len)      |
| `CPtr<T>`     | C 호환 **nullable** 포인터         | `T*` (nullable) |

`Slice<T>`는 [2.9 collections](../200-core/209-collections.md)에 정의된 안전한 뷰이므로 unsafe tier에 속하지 않습니다. 이 장은 `Slice` 안의 포인터를 원시 포인터로 노출하는 경로만 다룹니다.

### 3. GC와 원시 포인터: 두 타입 수준 분리

| type       | 의미                                | GC 이동 시    |
|------------|-------------------------------------|---------------|
| `GcPtr<T>` | GC 힙의 객체 참조 (보통 `T`)       | **자동 갱신** |
| `Ptr<T>`   | 원시 포인터 (비관리 또는 고정됨)    | **무효**      |

이 구분은 Go 1.21의 `runtime.Pinner` 모델을 따릅니다. 원시 `Ptr<T>`는 GC pin 안에서만 유효합니다. pin이 해제된 뒤에 쓰면 UB입니다.

```rok
let obj: List<i32> = [1, 2, 3];                // GcPtr (자동)
using pin = rok::bare::gc::pin(obj) {           // pin 시작
    let raw: Ptr<i32> = rok::bare::ptr::list_data(obj);
    ffi_function(raw, obj.len());
}                                              // pin 해제 - raw 무효
```

### 4. 캐스트는 이름 있는 명시적 연산으로만

C와 Rust에서 포인터에 쓰는 `as` 캐스트의 함정은 가장 큰 위험 요소입니다. 그래서 Rok은 모든 unsafe 캐스트를 이름 붙은 함수로만 노출합니다.

| C/Rust 표기       | Rok 표기                                 | 의미                           |
|-------------------|------------------------------------------|--------------------------------|
| `ptr as *U`       | `rok::bare::ptr_cast<U>(p)`               | 타입만 변경 (provenance 보존) |
| `ptr as *Aligned` | `rok::bare::align_cast(p)`                | alignment up-cast (debug 검사) |
| `*const → *mut`   | `rok::bare::const_cast(p)`                | const 제거                     |
| `transmute<A,B>`  | `rok::bare::reinterpret<A:Pod, B:Pod>(a)` | bit reinterpret (Pod 강제)     |

### 5. `using` 블록으로 RAII: finalizer에 의존하지 않음

GC 환경에서는 finalizer를 믿을 수 없습니다. 모든 GC 언어가 겪어서 배운 사실입니다. 따라서 파일, 소켓, 뮤텍스 같은 외부 자원은 명시적인 `using` 블록으로 해제합니다. GC는 `drop`을 호출하지 않고, finalizer도 없습니다. 해제하지 않은 자원은 `unclosed_resource` 린트가 알립니다([refs-lang 6.2.9](../../refs-lang/006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)).

```rok
using f = rok::bare::ffi::open(path) {
    f.read(...);
}     // ← scope 끝에서 _보장된_ close (RAII via Drop)

// 일반 GC 객체는 using 불필요 - GC 가 자동 회수
let xs: List<i32> = [1, 2, 3];     // GC 자동
```

기준 정의는 [2.13 marker](../200-core/213-marker.md)의 `Drop` 트레잇입니다. `using` 블록 문법의 기준 정의는 [refs-lang 3.17 using 표현식](../../refs-lang/003-expressions/317-using-expression.md)에 있습니다.
