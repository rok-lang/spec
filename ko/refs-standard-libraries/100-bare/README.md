# 1. Unsafe tier

Unsafe tier는 저수준이거나 위험한 표준 라이브러리 항목을 정의합니다. raw pointer, transmute, raw allocation, GC pinning, FFI primitive, MLIR 다이얼렉트 직접 접근처럼 안전한 추상화 바깥에서 동작하는 도구는 모두 여기에 있습니다.

## 의존성

위쪽으로는 runtime이 runtime ABI `extern "rok"`로 노출하는 심볼에만 의존합니다. 아래쪽으로는 `rok`, `std`, 사용자 코드가 이 tier에 의존하지만, 명시 import를 거칠 때만 의존할 수 있습니다.

## 사용자 import 정책

```rok
use rok::bare::ptr::CPtr;                             // 명시 import 필수 (위험 시그널)
use rok::bare::alloc::{alloc, Layout};

let layout = try! Layout::array::<u8>(1024);
let p: CPtr<u8> = unsafe { alloc(layout) };    // 실패 시 null
```

묵시적으로 적용되는 `use rok::core::**;`에는 unsafe 항목이 들어 있지 않습니다. 따라서 `rok::bare::*`는 따로 명시 import해야 하고, 사용자가 위험 영역에 들어가는 지점이 코드에 드러납니다.

## 모듈 목차

### Pointer 계열 (5 종)

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

### MLIR backend

- [1.9. `rok::bare::mlir` - MLIR 다이얼렉트 직접 접근](109-mlir.md)
- [1.10. MLIR 다이얼렉트 plugin - 사용자 정의 다이얼렉트를 `rokc`에 로드](110-mlir-plugin.md)

## 핵심 디자인 원칙

### 1. Strict provenance - Rust 2024 모델 채택

`Ptr<T>`는 (address, provenance) 쌍이며 단순한 정수가 아닙니다. 그래서 `usize ↔ Ptr<T>` round-trip은 provenance를 잃고, 명시적인 escape hatch를 거칠 때만 허용됩니다.

```rok
let p: Ptr<i32> = ...;
let addr: usize = p.addr();              // 안전: 주소만
let q: Ptr<i32> = p.with_addr(addr + 4); // 안전: provenance 보존, address 만 변경
let r: Ptr<i32> = rok::bare::ptr::from_exposed_provenance(addr);  // unsafe: provenance 새로
```

`as` cast는 pointer에 절대 사용할 수 없습니다. 이 방향의 근거는 [Ralf Jung "Pointers Are Complicated"](https://www.ralfj.de/blog/2022/04/11/provenance-exposed.html)이며, Rust 1.84(Dec 2024)가 안정화한 모델과 동형입니다.

### 2. Arity 가 type encoded - Zig 모델

| 타입          | 의미                               | C 대응          |
|---------------|------------------------------------|-----------------|
| `Ptr<T>`      | 정확히 **하나**의 T               | `T*` (single)   |
| `ConstPtr<T>` | 정확히 **하나**의 const T         | `const T*`      |
| `ManyPtr<T>`  | **연속 N개** T (길이 모름, C 호환) | `T*` (array)    |
| `Slice<T>`    | **연속 N개** T (길이 포함)         | (ptr, len)      |
| `CPtr<T>`     | C 호환 **nullable** pointer        | `T*` (nullable) |

`Slice<T>`는 [2.9 collections](../200-core/209-collections.md)에 정의된 안전한 view이므로 unsafe tier에 속하지 않습니다. 이 장은 `Slice`와 pointer 사이의 raw 경로만 다룹니다. `ManyPtr::slice`, 그리고 list의 data pointer를 얻는 pin입니다.

### 3. GC + raw pointer - 두 type level 분리

| type       | 의미                                | GC 이동 시    |
|------------|-------------------------------------|---------------|
| `GcPtr<T>` | GC heap의 객체 ref (보통 `T`)      | **자동 갱신** |
| `Ptr<T>`   | raw pointer (unmanaged 또는 pinned) | **무효**      |

이 구분은 Go 1.21의 `runtime.Pinner` 모델을 따릅니다. raw `Ptr<T>`는 GC pin 안에서만 유효하므로, pin이 해제된 뒤에 사용하면 UB입니다.

```rok
let obj: List<i32> = [1, 2, 3];                // GcPtr (자동)
using pin = rok::bare::gc::pin(obj) {          // pin 시작
    let raw: Ptr<i32> = pin.as_data_ptr();
    ffi_function(raw, obj.len());
}                                              // pin 해제, raw 무효
```

### 4. Cast 는 명시 named operation 만

C와 Rust에서 pointer에 쓰는 `as` cast의 함정은 가장 큰 위험 요소입니다. 그래서 Rok은 모든 unsafe cast를 이름이 명시된 함수로만 노출합니다.

| C/Rust 표기       | Rok 표기                                 | 의미                           |
|-------------------|------------------------------------------|--------------------------------|
| `ptr as *U`       | `rok::bare::ptr_cast<U>(p)`               | type만 변경 (provenance 보존) |
| `ptr as *Aligned` | `rok::bare::align_cast(p)`                | alignment up-cast (어긋나면 `none`) |
| `*const → *mut`   | `rok::bare::const_cast(p)`                | const 제거                     |
| `transmute<A,B>`  | `rok::bare::reinterpret<A:Pod, B:Pod>(a)` | bit reinterpret (Pod 강제)     |

### 5. RAII via `using` block - finalizer 에 의존하지 않습니다

GC 환경에서 finalizer를 믿을 수 없다는 것은 모든 GC 언어가 겪어서 배운 사실입니다. 따라서 file, socket, mutex 같은 외부 자원은 명시적인 `using` 블록으로 해제합니다. GC는 `drop`을 호출하지 않으며 finalizer는 없습니다. 해제하지 않은 자원은 `unclosed_resource` lint가 알립니다 ([refs-lang 6.2.9](../../refs-lang/006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)).

```rok
use rok::bare::ptr::{ConstPtr, CPtr, Ptr};

extern "C" {
    fn fopen(path: ConstPtr<c_char>, mode: ConstPtr<c_char>) -> CPtr<u8>;
    fn fclose(f: Ptr<u8>) -> i32;
}

// 사용자가 쓴 wrapper: Drop 이 C handle 을 닫음
struct File { raw: Ptr<u8>; }

impl Drop for File {
    fn drop(ref self) { unsafe { fclose(self.raw) }; }
}

using f = File { raw: opened } {   // opened: fopen 이 준 Ptr<u8>
    // ... f.raw 사용 ...
}     // ← scope 끝에서 _보장된_ fclose (RAII via Drop)

// 일반 GC 객체는 using 불필요. GC 가 자동 회수
let xs: List<i32> = [1, 2, 3];     // GC 자동
```

기준 정의는 [2.13 marker](../200-core/213-marker.md)의 `Drop` 트레잇이며, `using` 블록 문법의 기준 정의는 [refs-lang 3.17 using 표현식](../../refs-lang/003-expressions/317-using-expression.md)에 있습니다.
