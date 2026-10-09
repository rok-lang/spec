# 1. Unsafe tier

Unsafe tier 는 저수준이거나 위험한 영역에 속하는 표준 라이브러리 항목을 정의하는 자리입니다. raw pointer, transmute, raw allocation, GC pinning, FFI primitive, MLIR 다이얼렉트 직접 접근과 같이 안전 추상화 바깥에서 동작하는 도구들이 모두 여기에 모입니다.

## 의존성

위쪽으로는 runtime 이 노출하는 `extern "C"` 심볼에만 의존합니다. 한편 아래쪽으로는 `rok`, `std`, 그리고 사용자 코드가 의존자가 되지만, 어디까지나 명시 import 가 있을 때에만 그렇습니다.

## 사용자 import 정책

```rok
use rok::bare::ptr::Ptr;                              // 명시 import 필수 — 위험 시그널
use rok::bare::alloc::{alloc, Layout};

let layout = try! Layout::array::<u8>(1024);
let p: Ptr<u8> = unsafe { alloc(layout) };
```

묵시적으로 적용되는 `use rok::core::**;` 는 unsafe 항목을 포함하지 않습니다. 따라서 `rok::bare::*` 는 별도의 명시 import 를 강제하며, 이로써 사용자가 위험 영역으로 진입하는 순간이 코드 위에 시각적으로 드러납니다.

## 모듈 목차

### Pointer 계열 (5 종)

- [1.1. `Ptr<T>` / `ConstPtr<T>` / `ManyPtr<T>` / `CPtr<T>` — Zig 식 arity-encoded](101-ptr.md)
- [1.2. cast 연산 — `ptr_cast`, `align_cast`, `const_cast`, `reinterpret`](102-cast.md)

### 메모리 / 변환

- [1.3. `MaybeUninit<T>` / `transmute_unchecked` — bit-level 변환](103-mem.md)
- [1.4. raw allocation (`alloc`, `dealloc`, `realloc`)](104-alloc.md)
- [1.5. volatile 메모리 접근 — `volatile_read` / `volatile_write`](105-volatile.md)

### GC 통합

- [1.6. `Pinned<T>` / `gc::pin` — GC 가 객체를 이동 못 하게](106-gc-pin.md)

### FFI

- [1.7. extern "C" / `CStr` / `CString` / `CPtr` — C ABI 호환](107-ffi.md)
- [1.8. WASM Component Model 인터페이스 (`ffi.wit`)](108-wit.md)

### MLIR backend

- [1.9. `rok::bare::mlir` — MLIR 다이얼렉트 직접 접근](109-mlir.md)
- [1.10. `rok::bare::mlir::plugin` — 사용자 정의 다이얼렉트 등록](110-mlir-plugin.md)

## 핵심 디자인 원칙

### 1. Strict provenance — Rust 2024 모델 채택

`Ptr<T>` 는 (address, provenance) 의 pair 이며 단순한 정수와 다릅니다. 그래서 `usize ↔ Ptr<T>` round-trip 은 provenance 를 잃어버리는 동작이 되고, 명시적인 escape hatch 를 거치는 경로에서만 허용됩니다.

```rok
let p: Ptr<i32> = ...;
let addr: usize = p.addr();              // 안전 — 주소만
let q: Ptr<i32> = p.with_addr(addr + 4); // 안전 — provenance 보존, address 만 변경
let r: Ptr<i32> = rok::bare::ptr::from_exposed_provenance(addr);  // unsafe — provenance 새로
```

`as` cast 는 pointer 에 절대 사용할 수 없습니다. 이 방향성은 [Ralf Jung "Pointers Are Complicated"](https://www.ralfj.de/blog/2022/04/11/provenance-exposed.html) 가 권위로서 자리잡고 있고, Rust 1.84 (Dec 2024) 가 안정화한 모델과 동형입니다.

### 2. Arity 가 type encoded — Zig 모델

| 타입          | 의미                               | C 대응          |
|---------------|------------------------------------|-----------------|
| `Ptr<T>`      | 정확히 **하나** 의 T               | `T*` (single)   |
| `ConstPtr<T>` | 정확히 **하나** 의 const T         | `const T*`      |
| `ManyPtr<T>`  | **연속 N개** T (길이 모름, C 호환) | `T*` (array)    |
| `Slice<T>`    | **연속 N개** T (길이 포함)         | (ptr, len)      |
| `CPtr<T>`     | C 호환 **nullable** pointer        | `T*` (nullable) |

`Slice<T>` 는 [2.9 collections](../200-core/209-collections.md) 에 정의된 안전한 view 이므로 unsafe tier 에 속하지 않습니다. 따라서 본 장에서는 `Slice` 의 underlying ptr 을 raw 로 노출하는 경로만을 다룹니다.

### 3. GC + raw pointer — 두 type level 분리

| type       | 의미                                | GC 이동 시    |
|------------|-------------------------------------|---------------|
| `GcPtr<T>` | GC heap 의 객체 ref (보통 `T`)      | **자동 갱신** |
| `Ptr<T>`   | raw pointer (unmanaged 또는 pinned) | **무효**      |

이 구분은 Go 1.21 의 `runtime.Pinner` 모델을 따릅니다. raw `Ptr<T>` 는 GC pin 안에서만 유효하며, 따라서 pinning 이 해제된 뒤에도 사용하면 UB 가 됩니다.

```rok
let obj: List<i32> = [1, 2, 3];                // GcPtr (자동)
using pin = rok::bare::gc::pin(obj) {           // pin 시작
    let raw: Ptr<i32> = rok::bare::ptr::list_data(obj);
    ffi_function(raw, obj.len());
}                                              // pin 해제 — raw 무효
```

### 4. Cast 는 명시 named operation 만

C 와 Rust 에서 `as` cast 가 pointer 위에서 일으키는 함정은 가장 큰 위험 요소입니다. 그래서 rok 은 모든 unsafe cast 를 명시적으로 이름 붙여진 함수로만 노출합니다.

| C/Rust 표기       | rok 표기                                 | 의미                           |
|-------------------|------------------------------------------|--------------------------------|
| `ptr as *U`       | `rok::bare::ptr_cast<U>(p)`               | type 만 변경 (provenance 보존) |
| `ptr as *Aligned` | `rok::bare::align_cast(p)`                | alignment up-cast (debug 검증) |
| `*const → *mut`   | `rok::bare::const_cast(p)`                | const 제거                     |
| `transmute<A,B>`  | `rok::bare::reinterpret<A:Pod, B:Pod>(a)` | bit reinterpret (Pod 강제)     |

### 5. RAII via `using` block — finalizer 에 의존하지 않습니다

GC 환경에서 finalizer 의 신뢰성이 부족하다는 사실은 모든 GC 언어가 학습한 경험입니다. 따라서 file, socket, mutex 같은 외부 자원은 명시적인 `using` 블록을 primary 로 삼고, finalizer 는 backstop 의 자리에만 둡니다.

```rok
using f = rok::bare::ffi::open(path) {
    f.read(...);
}     // ← scope 끝에서 _보장된_ close (RAII via Drop)

// 일반 GC 객체는 using 불필요 — GC 가 자동 회수
let xs: List<i32> = [1, 2, 3];     // GC 자동
```

권위는 [2.13 marker](../200-core/213-marker.md) 의 `Drop` 트레잇이 가지며, `using` 블록의 syntax 권위는 [refs-lang 3.17 using 표현식](../../refs-lang/003-expressions/317-using-expression.md) 에 있습니다.
