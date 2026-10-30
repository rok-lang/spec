# 1.4. 원시 할당: `alloc` / `dealloc` / `realloc`

`rok::bare::alloc` 모듈은 GC 바깥 영역에서 메모리를 직접 할당하고 해제하는 API입니다. Rok의 일반 코드는 GC가 관리합니다. 하지만 GC에 맞지 않는 곳에서는 이 모듈을 씁니다. FFI 경계에서 C 쪽에 메모리를 넘길 때, 큰 버퍼, mmap 같은 곳입니다.

## 1.4.1. API

```rok
struct Layout {
    size: usize;
    align: usize;
}

impl Layout {
    fn new<T>() -> Layout;                     // size_of<T>(), align_of<T>()
    fn array<T>(n: usize) throws (LayoutError) -> Layout;
    fn from_size_align(size: usize, align: usize) throws (LayoutError) -> Layout;
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

`alloc`이 실패하면 `CPtr::null()`을 반환합니다. 프로세스를 중단(abort)하지 않으므로 호출자가 결과를 확인하고 처리해야 합니다.

## 1.4.2. 예

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

`using` 블록을 쓰면 RAII 형태로 정리할 수 있습니다.

```rok
using buf = try rok::bare::alloc::Allocation::<u32>::new(1000) {
    // buf.as_ptr() 사용 가능
}     // ← 자동 dealloc
```

여기서 `Allocation<T>`는 이 모듈에 든 RAII 래퍼입니다.

## 1.4.3. GC와 관계

| 영역                     | 할당 / 해제                 | GC 관리            |
|--------------------------|-----------------------------|--------------------|
| 일반 객체 (`List<i32>`)  | GC TLAB bump                | ✓                  |
| `rok::bare::alloc::alloc` | 시스템 malloc               | ✗ (수동)           |
| FFI `malloc`             | C `malloc`                  | ✗                  |
| 큰 객체 (8 KB 이상)      | LOS (이동 없는 mark-sweep)  | ✓ (단, 이동 없음)  |

GC가 관리하는 원시 버퍼가 필요하면 이 모듈 대신 `rok::core::collections::raw::GcBuf<T>`([2.9.12](../200-core/209-collections.md#2912-gcbuft-원시-gc-버퍼))를 씁니다.

GC는 바깥 영역을 추적하지 않으므로 정리는 사용자 책임입니다. 대신 이 영역의 객체는 이동하지 않습니다. 그래서 `Ptr<T>`는 별도의 pin guard 없이도 유효합니다.

## 1.4.4. 런타임 연계

컴파일러는 `rok::bare::alloc::alloc`의 본문을 런타임 래퍼로 낮춥니다. 이 래퍼는 Rok 런타임이 시스템 malloc 위에 둔 것입니다. 최종적으로 `libc::malloc` / `libc::free` / `libc::realloc`을 호출합니다.
