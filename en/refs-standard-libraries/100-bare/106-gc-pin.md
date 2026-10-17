# 1.6. `Pinned<T>` — GC pinning

The `rok::bare::gc` module defines a mechanism for temporarily fixing an object on the GC heap so that it cannot move. When a raw `Ptr<T>` points to a GC object and the object moves, the pointer becomes invalid; this chapter covers how that conflict is resolved.

This is the most important chapter of rok's unsafe tier: it explains head-on how the conflict between pointers and the GC is reconciled.

## 1.6.1. Motivation — why pinning is needed

The Immix mark-region GC of [refs-lang 7.3 GC algorithm](../../refs-lang/007-runtime/703-gc-algorithm.md) performs **evacuation** (moving objects by copying). An object's address can therefore change freely between GC cycles.

```
  GC cycle start:    obj at 0x1000
  GC evacuation:     obj at 0x2000  (copied, then forwarded)
  GC cycle end:      obj at 0x2000
```

For managed code (`GcPtr<T>`), the GC traces roots and updates every ptr automatically. A raw `Ptr<T>`, however, is invisible to the GC, so it is not updated and becomes a **stale ptr**.

```rok
let xs: List<i32> = [1, 2, 3];                 // GcPtr (updated automatically)
let raw: Ptr<i32> = rok::bare::ptr::list_data(xs); // raw ptr
                                                // ↓ a GC cycle may occur
gc::collect();
unsafe { raw.read() };                          // UB — raw is stale
```

The solution is **pinning**: explicitly telling the GC "do not move this object".

## 1.6.2. `Pinned<T>` guard

```rok
struct Pinned<T> {
    obj: GcPtr<T>;
    // internal: registered in the GC's pin set
}

fn pin<T>(obj: GcPtr<T>) -> Pinned<T>;

impl<T> Pinned<T> {
    fn as_ptr(self) -> Ptr<T>;
    fn as_const_ptr(self) -> ConstPtr<T>;
    fn obj(self) -> GcPtr<T>;
}

impl<T> Drop for Pinned<T> {
    fn drop(ref self) {
        // removed from the GC pin set
    }
}
```

It is a guard bound with `using`. While the `Pinned<T>` returned by `pin()` is inside a `using` block, the GC does not move the object; on leaving the block `Pinned::drop` is called and the object is unpinned. A `Pinned<T>` obtained without `using` is not released automatically.

### Example — FFI call

```rok
let buf: List<u8> = read_input();

using pin = rok::bare::gc::pin(buf) {
    let raw: Ptr<u8> = pin.as_ptr().cast::<u8>();
    unsafe {
        ffi::process(raw, buf.len());          // FFI uses buf's data
    }
}     // ← automatic unpin at end of scope
```

The `using` block here follows the RAII form defined by [refs-lang 3.17 using expression](../../refs-lang/003-expressions/317-using-expression.md).

## 1.6.3. `GcPtr<T>` vs `Ptr<T>` — separation at the type level

```rok
struct GcPtr<T> { /* compiler-managed — roots traced automatically */ }
struct Ptr<T>   { /* raw — no root tracking */ }
```

| Aspect     | `GcPtr<T>`                    | `Ptr<T>`                                         |
|------------|-------------------------------|--------------------------------------------------|
| GC root    | **Traced automatically**      | **Not traced**                                   |
| On move    | **Updated automatically**     | **Stale**                                        |
| Used in    | Ordinary code (`List`/`Dict`/...) | unsafe + FFI only                            |
| Created by | GC alloc (`List::new()` etc.) | `gc::pin().as_ptr()` or `alloc::alloc(layout)`   |
| Valid for  | As long as the object is reachable | Only within the pin guard's scope           |

Because the type system separates the two kinds, user code normally deals only with `GcPtr` (invisibly wrapped in `List<T>` etc.). `Ptr<T>` appears explicitly only after entering unsafe.

### Internal implementation — the form of GcPtr

```rok
struct GcPtr<T> {
    // internal: usually a plain `*T` — the GC traces it externally via the root set
    // for a local variable, kept in a shadow stack frame slot emitted by the compiler
}
```

rok's GC traces the root set through a **per-thread shadow stack** emitted directly by the compiler ([refs-lang 7.4.1](../../refs-lang/007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)).

## 1.6.4. GC-side mechanism of pin / unpin

```
GC pin set structure.
  - hash table: Ptr<Object> → pin count
  - during evacuation the GC _skips_ objects with pin count > 0

pin(obj):
  1. check the pin bit in obj's header (or register in the pin set)
  2. count++
  3. return Pinned<T>

drop(Pinned):
  1. count--
  2. if count == 0, clear the pin bit
```

This structure is effectively isomorphic to Go 1.21's `runtime.Pinner`. See [pkg.go.dev/runtime#Pinner](https://pkg.go.dev/runtime#Pinner) for details.

### overhead

The costs are:

- One pin is just an O(1) hash insert
- One unpin is likewise an O(1) hash remove
- A pin set check is added within a GC cycle, but its cost is small

In most cases this is not on a hot path, because FFI calls themselves are not that frequent.

## 1.6.5. Limits — where pinning is unnecessary

Pinning is **unnecessary** in the following situations.

| Place            | Reason                                  |
|------------------|-----------------------------------------|
| Ordinary method calls | GcPtr is updated automatically — no raw ptr is created |
| Operations between GcPtrs | All traced by the GC           |
| Short FFI (no GC) | When the GC is guaranteed not to run during the FFI |
| Immortal const   | Already does not move (string literals, etc.) |
| Non-moving regions | LOS (large object space), pinned region |

Short FFI calls get automatic treatment: if the compiler can guarantee that the GC does not run during the FFI call, the pin is omitted automatically. In the general case, however, an **explicit pin** is recommended.

## 1.6.6. Extraction methods besides `as_ptr`

```rok
impl<T> Pinned<T> {
    fn as_ptr(self) -> Ptr<T>;
    fn as_const_ptr(self) -> ConstPtr<T>;
}

impl<T> Pinned<List<T>> {
    fn as_data_ptr(self) -> Ptr<T>;          // start of the List's buffer
    fn as_data_many_ptr(self) -> ManyPtr<T>; // many ptr
}

impl Pinned<str> {
    fn as_bytes_ptr(self) -> ConstPtr<u8>;   // UTF-8 bytes
    fn as_c_str(self) -> ConstPtr<c_char>;   // null termination not guaranteed — convert separately
}
```

Each container gets dedicated extraction methods, so data can be taken out in a form matching each container's layout, such as the buffer start or a null-terminated form.

## 1.6.7. GC runtime linkage

Each item maps to GC runtime calls as follows.

| Item             | Runtime call                   |
|------------------|--------------------------------|
| `Pinned<T>`      | `rok_gc_pin` / `rok_gc_unpin`  |
| `pin<T>()`       | (same as above)                |
| `GcPtr<T>`       | shadow stack frame slot (compiler-emitted) |
| `as_data_ptr` etc. | depends on each container's layout |

## 1.6.8. Comparison with other languages

| Aspect           | Java JNI                       | .NET fixed             | Go 1.21 Pinner           | rok                       |
|------------------|--------------------------------|------------------------|--------------------------|---------------------------|
| pin API          | `Get*Critical`                 | `fixed (T* p = ...)`   | `runtime.Pinner`         | `rok::bare::gc::pin(obj)`  |
| RAII guard       | explicit `Release`             | block scope            | `defer pin.Unpin()`      | `using` block (automatic drop) |
| Type separation  | **none** (all `jobject`)       | **none**               | **none** (all ptrs alike) | `GcPtr<T>` vs `Ptr<T>`   |
| Expiry check after pin | **none**                 | compiler (block scope) | **none** (manual)        | type system + scope       |
| Multiple pins    | overlapping OK                 | **restricted**         | overlapping OK           | overlapping OK (count)    |

rok diverges from other languages at two main points.

1. The type system enforces it. `GcPtr<T>` and `Ptr<T>` are distinct types and automatic conversion is rejected; the only way across is through a pin guard.
2. Release is based on the `using` block. `Pinned<T>` is dropped on leaving the `using` block, so an unpin is never missed. This is safer than Go's manual `defer` approach.

> This policy is the backbone of rok's pointer model: the point where the two domains, unsafe and the GC, meet safely.
