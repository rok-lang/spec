# 1.4. raw allocation - `alloc` / `dealloc` / `realloc`

`rok::bare::alloc` 모듈은 GC 가 관여하지 않는 외부 영역에서 메모리를 직접 할당하고 해제하기 위한 API 를 제공합니다. Rok 의 일반 코드는 GC 가 알아서 관리해 주지만, FFI 경계에서 C 측에 메모리를 넘겨야 하거나 large buffer, mmap 처럼 GC 가 다루기에 적절치 않은 자리에서는 이 모듈을 사용합니다.

## 1.4.1. API

```rok
struct LayoutError {
    pub reason: str;
}

struct Layout {
    size: usize;
    align: usize;
}

impl Layout {
    fn new<T>() -> Layout;                     // size_of<T>(), align_of<T>()
    fn array<T>(n: usize) throws (LayoutError) -> Layout;
    fn from_size_align(size: usize, align: usize) throws (LayoutError) -> Layout;
                                               // align 은 2 의 거듭제곱
    fn size(self) -> usize;
    fn align(self) -> usize;
}

unsafe fn alloc(layout: Layout) -> CPtr<u8>;
unsafe fn alloc_zeroed(layout: Layout) -> CPtr<u8>;
unsafe fn dealloc(ptr: Ptr<u8>, layout: Layout);
unsafe fn realloc(ptr: Ptr<u8>, old_layout: Layout, new_size: usize) -> CPtr<u8>;

// 편의 함수 - typed
unsafe fn alloc_one<T>() -> CPtr<T>;
unsafe fn alloc_array<T>(n: usize) -> CPtr<T>;
unsafe fn dealloc_one<T>(ptr: Ptr<T>);
unsafe fn dealloc_array<T>(ptr: Ptr<T>, n: usize);
```

`alloc` 이 실패하면 `CPtr::null()` 을 반환하며, abort 하지 않으니 호출자가 직접 결과를 살펴 처리해야 합니다. `Layout` 의 필드는 private 이므로 `size()` / `align()` 으로 읽습니다. `array` 는 `n` 개 원소가 `usize` 를 넘칠 때, `from_size_align` 은 `align` 이 2 의 거듭제곱이 아닐 때 `LayoutError` 를 던지며, `reason` 이 그 까닭을 알려 줍니다.

## 1.4.2. 사용 예

```rok
let layout = try Layout::array::<u32>(1000);
let p: CPtr<u8> = unsafe { rok::bare::alloc::alloc(layout) };

match p.as_ptr() {
    raw: Ptr<u8> => {
        let typed: Ptr<u32> = rok::bare::ptr_cast::<u32>(raw);
        // 사용
        unsafe {
            for i in 0u32..1000u32 { typed.add(i as usize).write(i); }
        }
        unsafe { rok::bare::alloc::dealloc(raw, layout) };
    }
    none => panic("alloc failed"),
}
```

`using` 블록을 활용하면 RAII 형태로 정리할 수 있습니다.

```rok
using buf = try rok::bare::alloc::Allocation::<u32>::new(1000) {
    // buf.as_ptr() 사용 가능
}     // ← 자동 dealloc
```

여기서 `Allocation<T>` 는 본 모듈에 포함된 RAII wrapper 입니다.

```rok
struct Allocation<T> { /* private: pointer + 길이 */ }

impl<T> Allocation<T> {
    fn new(n: usize) throws (LayoutError) -> Allocation<T>;   // 0 으로 채운 T n 개
    fn as_ptr(self) -> Ptr<T>;
    fn as_many(self) -> ManyPtr<T>;
    fn len(self) -> usize;
}

impl<T> Drop for Allocation<T> { /* block 을 해제 */ }
```

`new` 는 크기가 `usize` 를 넘치거나 시스템에 메모리가 없을 때 `LayoutError` 를 던집니다 (`reason` 은 `"out of memory"`). `Allocation` 의 복사본들은 block 하나를 공유하므로, 그것을 해제하는 `using` 블록 안에서만 씁니다.

## 1.4.3. GC 와의 관계

| 영역                     | 할당 / 해제                 | GC 관리?           |
|--------------------------|-----------------------------|--------------------|
| 일반 객체 (`List<i32>`)  | GC TLAB bump                | ✓                  |
| `rok::bare::alloc::alloc` | system malloc               | ✗ (수동)           |
| FFI `malloc`             | C `malloc`                  | ✗                  |
| Large object (≥8KB)      | LOS - non-moving mark-sweep | ✓ (단, non-moving) |

GC 가 관리하는 raw buffer 가 필요하면 이 모듈 대신 `rok::core::collections::raw::GcBuf<T>` ([2.9.12](../200-core/209-collections.md#2912-gcbuft---raw-gc-buffer)) 를 씁니다.

GC 외부 영역은 GC 의 추적 대상이 아니므로 정리 책임이 사용자에게 있습니다. 다만 객체가 이동하지 않는 영역인 만큼, `Ptr<T>` 는 별도의 pin guard 없이도 그대로 유효합니다.

## 1.4.4. runtime 연계

`rok::bare::alloc::alloc` 의 본문은 Rok 런타임이 system malloc 위에 제공하는 wrapper 로 lower 되며, 최종적으로 `libc::malloc` / `libc::free` / `libc::realloc` 을 호출하는 형태로 동작합니다.
