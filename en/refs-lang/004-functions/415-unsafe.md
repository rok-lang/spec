# 4.15. `unsafe` - function modifier and block expression

The `unsafe` keyword appears in two forms that visibly confine regions whose safety the compiler does not guarantee: the **modifier on function / method declarations** and the **block expression**. The two work as a pair: a function declared `unsafe fn` must be called inside an `unsafe { ... }` block.

```
unsafe_fn        ::= "unsafe" fn_decl
unsafe_block     ::= "unsafe" block_expr
```

This model is isomorphic to Rust's `unsafe`; its purpose is to make dangerous regions explicit in the code.

## 4.15.1. `unsafe fn` - function modifier

Placing `unsafe` before a function or method declaration means that the function's body is outside the guarantees of the safe abstraction. The signature states directly that the caller is responsible for satisfying the preconditions.

```rok
impl<T> Ptr<T> {
    unsafe fn read(self) -> T;
    unsafe fn write(self, value: T);
}

extern "C" {
    unsafe fn strlen(s: ConstPtr<c_char>) -> usize;
}

unsafe fn transmute_unchecked<A, B>(a: A) -> B { ... }
```

Inside the body of an `unsafe fn` itself, unsafe calls are allowed without a separate `unsafe { ... }` block. Because the signature already declares unsafe, the whole body is treated as an unsafe scope.

```rok
unsafe fn copy_two<T>(p: Ptr<T>) -> (T, T) {
    let a = p.read();              // body is an unsafe scope: no explicit block needed
    let b = p.add(1).read();
    (a, b)
}
```

### With method receivers

Methods in `impl` follow the same form.

```rok
impl Buffer {
    unsafe fn raw_data(self) -> Ptr<u8>;
    unsafe fn raw_data_mut(ref self) -> Ptr<u8>;
}
```

`unsafe` does not conflict with receiver notations such as `ref self` / `self`, and it comes first in the function header.

### Inside `extern "C"` blocks

User-declared C ABI functions are all implicitly `unsafe`, because their definitions are outside the compiler's control ([extern blocks](../005-modules/509-extern-block.md)). Writing it explicitly (`unsafe fn`) is recommended; if omitted, it is treated as `unsafe` automatically.

As an exception, functions declared by the toolchain (symbols injected by the compiler or provided by the runtime, such as `rok_print_*` and `rok_panic_str`) may be called safely.

```rok
extern "C" {
    fn malloc(size: usize) -> CPtr<u8>;        // treated as unsafe automatically
    unsafe fn free(ptr: CPtr<u8>);             // explicit (recommended)
}
```

## 4.15.2. `unsafe { ... }` - block expression

Operations such as calling unsafe functions or dereferencing raw pointers are allowed only inside an `unsafe { ... }` block. Like an ordinary [block expression](../003-expressions/309-block-expressions.md), the block's result is the value of its last expression.

```rok
let n: usize = unsafe { strlen(cs.as_ptr()) };

let v: i32 = unsafe {
    let p = ptr.add(idx);
    p.read()
};

unsafe {
    free(ptr);
}
```

### Operations allowed in the block

Inside an `unsafe { ... }` block, the following operations are additionally allowed:

- Calling functions / methods declared `unsafe fn`
- Dereferencing raw pointers (`Ptr<T>`, `ConstPtr<T>`, `ManyPtr<T>`) (`p.read()`, `p.write(v)`)
- Directly calling dangerous functions of the [`rok::bare::*`](../../refs-standard-libraries/100-bare/README.md) modules
- Directly calling MLIR ops via [`rok::bare::mlir::__mlir_op`](../../refs-standard-libraries/100-bare/109-mlir.md)
- Reading `union` fields (bit reinterpretation, [union](../002-types/205-union.md)). Writing fields is allowed without `unsafe`.
- Using a value after it was handed to another owner ([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers)): the block's author guarantees no other thread uses it at the same time.

All other ordinary safe operations work the same inside an unsafe block: `unsafe` *adds capabilities*; it does not *change existing meaning*.

### Narrowing the scope

`unsafe` blocks should be kept as narrow as possible. Instead of wrapping a whole function in `unsafe { ... }`, wrapping only the one line that is actually dangerous makes the safety of the other lines visibly clear.

```rok
// recommended: narrow unsafe scope
fn safe_wrapper(p: Ptr<i32>) throws -> i32 {
    let validated = try check_ptr(p);
    unsafe { validated.read() }    // only the dangerous operation is unsafe
}

// not recommended: wide unsafe scope
fn safe_wrapper(p: Ptr<i32>) throws -> i32 {
    unsafe {
        let validated = try check_ptr(p);
        validated.read()
    }
}
```

## 4.15.3. Call rules

Calls of `unsafe fn` follow these rules.

| Call position                  | Can call `unsafe fn`?        |
|--------------------------------|------------------------------|
| ordinary function body         | ✗ (compile error)            |
| `unsafe fn` body               | ✓ (automatic unsafe scope)   |
| `unsafe { ... }` block         | ✓                            |
| `extern "C"` block declarator  | ✓ (automatically unsafe)     |

Violating the rules is a compile error:

```rok
fn safe_caller(p: Ptr<i32>) -> i32 {
    p.read()    // ERROR: call to unsafe fn 'read' outside unsafe block
}
```

There are two fixes: wrap the call in `unsafe { ... }`, or mark the caller itself `unsafe fn`.

### `unsafe fn` types

When an `unsafe fn` or a C function is used as a value, its type is `unsafe fn(..) -> R` ([4.4](404-function-type.md)). Calling a value of this type requires `unsafe`. A safe `fn(..) -> R` value may be used in an `unsafe fn(..) -> R` position, but not the reverse.

```rok
let f: unsafe fn(usize) -> CPtr<u8> = malloc;   // C function value
let p = unsafe { f(16) };
let g: unsafe fn(i32) -> i32 = |x| x + 1;      // OK: safe → unsafe
let h: fn(usize) -> CPtr<u8> = f;              // ERROR: unsafe → safe
```

## 4.15.4. In traits

Trait methods may also use `unsafe fn`. The `impl` side must implement them with the same signature, and callers follow the same rules as for ordinary call positions.

```rok
trait RawAccess {
    unsafe fn raw_get(self, idx: usize) -> u8;
}

impl RawAccess for Buffer {
    unsafe fn raw_get(self, idx: usize) -> u8 {
        self.ptr.add(idx).read()
    }
}

let buf: Buffer = ...;
let b = unsafe { buf.raw_get(0) };
```

### `unsafe impl`

`unsafe impl` is used only to implement the auto marker traits `Send` / `Sync` by hand. It marks that the author guarantees the type is safe to cross thread boundaries. Using `unsafe impl` for any other trait is a compile error ([2.9.1](../002-types/209-impl.md#291-two-forms)).

```rok
unsafe impl Send for Handle { }
```

## 4.15.5. Responsibility model

`unsafe` does not mean the compiler *gives up* safety checking. It means that the following two responsibilities transfer to the caller:

1. **Memory safety**: the responsibility not to cause undefined behavior such as dangling pointers, misaligned access or data races.
2. **Preconditions**: the responsibility to satisfy the usage conditions promised by the `unsafe fn`'s documentation (e.g., that a pointer points to a valid object, that a length is within the buffer).

`unsafe { ... }` is the visible signal that the caller accepts these two responsibilities, so dangerous regions are apparent at a glance during code review.

## 4.15.6. Relationship to implicit unsafe

The following positions are treated as unsafe scopes automatically, without an explicit `unsafe`:

- Function declaration bodies in [`extern "C"` blocks](../005-modules/509-extern-block.md)
- The body of an `unsafe fn`

This automatic treatment reduces noisy repetition of `unsafe { ... }` in commonly occurring dangerous regions.

## 4.15.7. Comparison with other languages

| Language | unsafe notation model                                       |
|----------|-------------------------------------------------------------|
| Rust     | `unsafe fn` + `unsafe { ... }` (isomorphic)                 |
| C / C++  | no separate notation; all code is potentially unsafe        |
| Swift    | `@unsafe` annotation / `withUnsafePointer` functional pattern |
| Go       | `unsafe` package (separate import); no block notation       |
| Zig      | explicit builtins such as `@ptrCast`; no block notation     |
| **Rok**  | **adopts Rust-style `unsafe fn` + `unsafe { ... }`**        |

Rok follows Rust's model as-is. Visual separation of dangerous regions is marked consistently with a single keyword, which benefits both code review and static analysis.
