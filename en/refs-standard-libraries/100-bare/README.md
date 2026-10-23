# 1. Unsafe tier

The unsafe tier defines the standard library items that are low-level or dangerous. Tools that operate outside safe abstractions, such as raw pointers, transmute, raw allocation, GC pinning, FFI primitives and direct access to MLIR dialects, are all gathered here.

## Dependencies

Upward, it depends only on the `extern "C"` symbols exposed by the runtime. Downward, `rok`, `std` and user code are its dependents, but only through explicit imports.

## User import policy

```rok
use rok::bare::ptr::Ptr;                              // explicit import required - danger signal
use rok::bare::alloc::{alloc, Layout};

let layout = try! Layout::array::<u8>(1024);
let p: Ptr<u8> = unsafe { alloc(layout) };
```

The implicitly applied `use rok::core::**;` does not include unsafe items. `rok::bare::*` therefore requires a separate explicit import, which makes visible in the code the point where the user enters dangerous territory.

## Module contents

### Pointer family (5 kinds)

- [1.1. `Ptr<T>` / `ConstPtr<T>` / `ManyPtr<T>` / `CPtr<T>` - Zig-style arity-encoded](101-ptr.md)
- [1.2. cast operations - `ptr_cast`, `align_cast`, `const_cast`, `reinterpret`](102-cast.md)

### Memory / conversion

- [1.3. `MaybeUninit<T>` / `transmute_unchecked` - bit-level conversion](103-mem.md)
- [1.4. raw allocation (`alloc`, `dealloc`, `realloc`)](104-alloc.md)
- [1.5. volatile memory access - `volatile::read` / `volatile::write`](105-volatile.md)

### GC integration

- [1.6. `Pinned<T>` / `gc::pin` - keeping the GC from moving an object](106-gc-pin.md)

### FFI

- [1.7. extern "C" / `CStr` / `CString` / `CPtr` - C ABI compatibility](107-ffi.md)
- [1.8. WASM Component Model interface (`ffi.wit`)](108-wit.md)

### MLIR backend

- [1.9. `rok::bare::mlir` - direct access to MLIR dialects](109-mlir.md)
- [1.10. `rok::bare::mlir::plugin` - registering user-defined dialects](110-mlir-plugin.md)

## Core design principles

### 1. Strict provenance - adopting the Rust 2024 model

`Ptr<T>` is an (address, provenance) pair, not a plain integer. A `usize ↔ Ptr<T>` round-trip therefore loses provenance and is allowed only through explicit escape hatches.

```rok
let p: Ptr<i32> = ...;
let addr: usize = p.addr();              // safe - address only
let q: Ptr<i32> = p.with_addr(addr + 4); // safe - provenance kept, only the address changes
let r: Ptr<i32> = rok::bare::ptr::from_exposed_provenance(addr);  // unsafe - fresh provenance
```

`as` casts can never be used on pointers. The authority for this direction is [Ralf Jung "Pointers Are Complicated"](https://www.ralfj.de/blog/2022/04/11/provenance-exposed.html), and it is isomorphic to the model stabilized in Rust 1.84 (Dec 2024).

### 2. Arity is type-encoded - the Zig model

| Type          | Meaning                                  | C equivalent    |
|---------------|------------------------------------------|-----------------|
| `Ptr<T>`      | Exactly **one** T                        | `T*` (single)   |
| `ConstPtr<T>` | Exactly **one** const T                  | `const T*`      |
| `ManyPtr<T>`  | **N contiguous** Ts (length unknown, C-compatible) | `T*` (array) |
| `Slice<T>`    | **N contiguous** Ts (length included)    | (ptr, len)      |
| `CPtr<T>`     | C-compatible **nullable** pointer        | `T*` (nullable) |

`Slice<T>` is a safe view defined in [2.9 collections](../200-core/209-collections.md) and does not belong to the unsafe tier. This chapter covers only the paths that expose the underlying ptr of a `Slice` raw.

### 3. GC + raw pointer - two separate type levels

| Type       | Meaning                                    | When the GC moves the object |
|------------|--------------------------------------------|------------------------------|
| `GcPtr<T>` | Reference to an object on the GC heap (usually `T`) | **Updated automatically** |
| `Ptr<T>`   | raw pointer (unmanaged or pinned)          | **Invalid**                  |

This distinction follows Go 1.21's `runtime.Pinner` model. A raw `Ptr<T>` is valid only within a GC pin, so using it after the pin is released is UB.

```rok
let obj: List<i32> = [1, 2, 3];                // GcPtr (automatic)
using pin = rok::bare::gc::pin(obj) {           // pin begins
    let raw: Ptr<i32> = rok::bare::ptr::list_data(obj);
    ffi_function(raw, obj.len());
}                                              // pin released - raw invalid
```

### 4. Casts only as explicitly named operations

The pitfalls of `as` casts on pointers in C and Rust are the biggest hazard. rok therefore exposes every unsafe cast only as an explicitly named function.

| C/Rust notation   | rok notation                             | Meaning                        |
|-------------------|------------------------------------------|--------------------------------|
| `ptr as *U`       | `rok::bare::ptr_cast<U>(p)`               | Type change only (provenance kept) |
| `ptr as *Aligned` | `rok::bare::align_cast(p)`                | Alignment up-cast (checked in debug) |
| `*const → *mut`   | `rok::bare::const_cast(p)`                | Removes const                  |
| `transmute<A,B>`  | `rok::bare::reinterpret<A:Pod, B:Pod>(a)` | Bit reinterpretation (Pod enforced) |

### 5. RAII via `using` block - no reliance on finalizers

Every GC language has learned that finalizers are unreliable in a GC environment. External resources such as files, sockets and mutexes are therefore released by explicit `using` blocks. The GC never calls `drop` and there are no finalizers; an unreleased resource is reported by the `unclosed_resource` lint ([refs-lang 6.2.9](../../refs-lang/006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control)).

```rok
using f = rok::bare::ffi::open(path) {
    f.read(...);
}     // ← _guaranteed_ close at end of scope (RAII via Drop)

// ordinary GC objects need no using - the GC reclaims them
let xs: List<i32> = [1, 2, 3];     // GC automatic
```

The authority is the `Drop` trait of [2.13 marker](../200-core/213-marker.md); the syntax authority for the `using` block is [refs-lang 3.17 using expression](../../refs-lang/003-expressions/317-using-expression.md).
