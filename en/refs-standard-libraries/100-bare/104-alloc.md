# 1.4. raw allocation — `alloc` / `dealloc` / `realloc`

The `rok::bare::alloc` module provides an API for allocating and freeing memory directly in a region outside the GC's control. Ordinary rok code is managed by the GC, but this module is used where the GC is not a good fit: handing memory to the C side across an FFI boundary, large buffers, mmap and the like.

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

// convenience functions — typed
unsafe fn alloc_one<T>() -> CPtr<T>;
unsafe fn alloc_array<T>(n: usize) -> CPtr<T>;
unsafe fn dealloc_one<T>(ptr: Ptr<T>);
unsafe fn dealloc_array<T>(ptr: Ptr<T>, n: usize);
```

If `alloc` fails it returns `CPtr::null()`; it does not abort, so the caller must check and handle the result.

## 1.4.2. Example

```rok
let layout = try Layout::array::<u32>(1000);
let p: CPtr<u8> = unsafe { rok::bare::alloc::alloc(layout) };

match p.as_ptr() {
    raw: Ptr<u8> => {
        let typed: Ptr<u32> = rok::bare::ptr_cast::<u32>(raw);
        // use
        unsafe {
            for i in 0..1000 { typed.add(i).write(i as u32); }
        }
        unsafe { rok::bare::alloc::dealloc(raw, layout) };
    }
    none => panic("alloc failed"),
}
```

With a `using` block, cleanup takes RAII form.

```rok
using buf = try rok::bare::alloc::Allocation::<u32>::new(1000) {
    // buf.as_ptr() is usable
}     // ← automatic dealloc
```

Here `Allocation<T>` is the RAII wrapper included in this module.

## 1.4.3. Relationship with the GC

| Region                   | Allocation / deallocation   | GC-managed?        |
|--------------------------|-----------------------------|--------------------|
| Ordinary objects (`List<i32>`) | GC TLAB bump          | ✓                  |
| `rok::bare::alloc::alloc` | system malloc               | ✗ (manual)         |
| FFI `malloc`             | C `malloc`                  | ✗                  |
| Large object (≥8KB)      | LOS — non-moving mark-sweep | ✓ (but non-moving) |

If a GC-managed raw buffer is needed, use `rok::core::collections::raw::GcBuf<T>` ([2.9.12](../200-core/209-collections.md#2912-gcbuft--raw-gc-buffer)) instead of this module.

Regions outside the GC are not traced by it, so cleanup is the user's responsibility. Since objects there do not move, however, a `Ptr<T>` stays valid without a separate pin guard.

## 1.4.4. Runtime linkage

The body of `rok::bare::alloc::alloc` lowers to a wrapper the rok runtime provides over system malloc, ultimately calling `libc::malloc` / `libc::free` / `libc::realloc`.
