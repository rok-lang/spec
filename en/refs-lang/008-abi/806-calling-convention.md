# 8.6. Calling convention

This document defines the register and stack conventions used for function calls. It follows the platform C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) by default and states only the parts that differ.

## 8.6.1. Default: platform C ABI

Every function declared as an [Extern "C" symbol](805-extern-symbols.md) uses the platform's C ABI. Calls between rok functions use the same registers, but pass aggregates differently (8.6.6).

| Platform                 | ABI                |
|--------------------------|--------------------|
| Linux x86_64             | System V AMD64     |
| Linux aarch64            | AAPCS64            |
| Linux i686               | System V i386      |
| Linux armv7              | AAPCS              |
| macOS (x86_64 / aarch64) | System V / AAPCS64 |
| Windows x86_64           | MS x64             |
| Windows aarch64          | AAPCS64 (MS variant) |
| WASM                     | clang's WebAssembly C ABI |

The LLVM backend handles register assignment of scalar arguments, but it does not classify how aggregates (struct / tuple / array) are passed across the C boundary. That classification is **the frontend's (rokc's) job**; rokc classifies by the same rules as clang and hands the result to LLVM (8.6.3). This specification defines only the remaining variations and additions. Under the WebAssembly C ABI an aggregate with a single scalar is passed as that scalar, and any other aggregate by pointer to a caller copy (results through `sret`); runtime `str` arguments are passed by reference.

## 8.6.2. Register arguments / return

The default argument and return registers defined by each ABI are used as is.

| Platform       | Argument registers (integer) | Return (integer) |
|----------------|----------------------------|-------------|
| Linux x86_64   | rdi, rsi, rdx, rcx, r8, r9 | rax, rdx    |
| Windows x86_64 | rcx, rdx, r8, r9           | rax         |
| aarch64        | x0 ~ x7                    | x0, x1      |
| i686           | (stack only)               | eax, edx    |
| armv7          | r0 ~ r3                    | r0, r1      |

Floating-point and vector values likewise use the standard registers defined by each ABI (xmm0-7, vN, sN, etc.).

## 8.6.3. `struct` arguments / return: small `struct` rule

When an aggregate is passed by value across the C boundary (`extern "C"` declarations and definitions), rokc classifies it according to the target ABI by the same rules as clang and coerces it into a form LLVM can move in registers. An HFA fits in up to four float / double registers.

| Target                                    | Classification                                                                            |
|-------------------------------------------|-------------------------------------------------------------------------------------------|
| AAPCS64 (Linux / Apple / Windows arm64)   | An HFA (1 ~ 4 elements of the same floating-point type) becomes `[n x f32]` / `[n x f64]`. Other aggregates of 16 bytes or less become `i64` / `[2 x i64]` (return values of 8 bytes or less become `iN`). Larger ones: arguments are passed as a pointer to a caller copy, returns use `sret` |
| System V x86_64                           | Each eightbyte is classified INTEGER / SSE and placed in integer / SSE registers. If registers run out, the whole aggregate is passed `byval`. Above 16 bytes: arguments `byval`, returns `sret` |
| Windows x64                               | Aggregates of size 1 / 2 / 4 / 8 bytes become `iN`. Others: arguments by reference (pointer to a caller copy), returns `sret` |
| Other targets                             | Passing an aggregate value across the C boundary is a compile error                       |

- `signext` / `zeroext` on narrow integer parameters and returns (`i8` / `i16` / `u8` / `u16` / `bool` / `c_char`) follow the platform C ABI: x86_64 SysV and Apple arm64 extend `i8` / `i16` / `bool`; Linux AAPCS64 extends none; Windows x64 extends only `bool`.
- Every function carries an inline stack probe, so a large frame touches the guard page instead of skipping it.
- The same classification applies to `extern "C" fn` **definitions**. A definition has a separate C-ABI entry point called from C; calls from rok keep using the rok convention.
- On System V i386, the in-aggregate alignment of `i64` / `u64` / `f64` is 4 bytes (same as C).
- This classification does not apply to calls between rok functions (8.6.6). It is needed only where rok meets C, including runtime functions taking or returning `str` (a 16-byte struct; by reference on Windows x64, [8.5](805-extern-symbols.md)).

## 8.6.4. GC object pointers: as ordinary pointers

GC object pointers are passed with the same ABI as the platform's ordinary pointers. No special registers or tagged pointers are used.

They must, however, be registered in the [shadow stack](../000-meta/007-glossary.md#shadow-stack) of [Barriers and root tracking](../007-runtime/704-barriers-and-roots.md) before and after the call.

This registration is done by the link / unlink of the shadow stack frame emitted by the compiler ([8.9.12](809-lowering-control.md#8912-shadow-stack-frame)).

## 8.6.5. Return of throwing functions: value-based error propagation

A function with a [`throws` signature](../004-functions/414-throws.md) returns its error as part of the return value. No separate unwinding mechanism or runtime symbol is used.

```
fn f(args) throws (E) -> T
```

The ABI return type of this function is a `struct` of two fields:

```mlir
!Throws_T_E = !llvm.struct<(i1, !Payload_T_E)>
; field 0: is_err - 0 on success, 1 on error
; field 1: payload - union(T, E). Read as T if is_err is 0, as E if 1
```

The payload is a union with the size and alignment of the larger of `T` and `E`. If `T` is unit, the payload holds only `E`. The error channel is distinguished by the signature, so `T` and `E` may be the same type. For untyped `throws` (`throws (any Error)`), `E` is an `any Error` value in the same tag + payload representation as [`any`](../002-types/202-any-slot.md) ([8.8](808-lowering-types.md)).

Field 0 may be stored wider than `i1` (an `i32` when the payload holds GC references, so the collector reads it as a sum tag); only bit 0 is significant, and when the payload is at least 4-aligned the offsets, size and registers are those of `{i1, payload}`. Values of `ref` parameters ([4.7](../004-functions/407-ref-keyword.md)) of a throwing function travel beside the carrier as an implementation-defined part of the return value.

This `struct` is passed through the platform C ABI like any other return value: in registers within the limits of 8.6.3, via sret beyond them. A `try` at the call site checks field 0 and branches; the detailed lowering is defined in [8.9.6](809-lowering-control.md#896-try--catch--finally-value-based-branching).

This scheme was chosen for the following reasons.

- It works the same on every platform (including WASM; no DWARF or platform EH infrastructure is needed).
- It does not conflict with the [shadow stack](../000-meta/007-glossary.md#shadow-stack). Errors also leave frames through the ordinary return path, so each frame pops its shadow stack normally.
- The cost on the no-error path is a single branch after return; entering `try` or `catch` costs nothing.

## 8.6.6. rok-to-rok calling convention

Calls between rok functions pass scalars in the registers of 8.6.2. An aggregate (struct, tuple, array, `str`) over 64 bytes is passed by address and returned through a leading result pointer. The callee copies a by-address argument on entry, so the caller's value is not modified. The callee may write the result through the result pointer at any point before it returns, after copying its by-address arguments. The result pointer may therefore point at a by-address argument, but at no other memory the callee reads during the call. This holds for every rok-to-rok call, including calls into prebuilt code, exported functions and function values. A direct call also passes the caller's `rok_tls` block address ([8.3.8](803-tls-slots.md#838-form-of-compiler-emission)) as a hidden last pointer argument, after the declared ones; a function value, an `#[export_name]` rok function and the `any` dispatcher take none (8.6.11). Smaller aggregates are passed and returned as first-class aggregates, lowered the way LLVM's SelectionDAG lowers them on the target, with one exception: on 32-bit Arm, an array whose elements are 1 or 2 bytes is passed and returned by address, like an aggregate over 64 bytes, because LLVM's own lowering of such stack arguments differs between caller and callee. Both backends produce exactly this lowering, so code from either calls the other. It differs from the C ABI classification of 8.6.3. The rules below apply to aggregates of 64 bytes or less.

> [!NOTE]
> A value passed by address is moved in memory (`memcpy`), never as a first-class aggregate.

**Arguments**

- An aggregate is split into its scalar leaves, each taking the next register of its class; when registers run out, the remaining leaves go on the stack. A `str` may therefore be split between the last argument register and the stack.
- AArch64: an array of `i64` / `ptr` / `f32` / `f64` takes consecutive registers of its class. If it does not fit in the remaining registers, it goes on the stack whole and the rest of that register class is marked used. Arrays of `i8` / `i16` / `i32` are split like structs.
- Stack slots: on Apple targets, top-level scalar arguments take their natural size and alignment, and `i1` / `i8` / `i16` leaves of an aggregate take 4-byte slots. On other targets every stack slot is 8 bytes.

**Results**

- AArch64: up to 8 integer (`x0`–`x7`) and 8 floating-point (`v0`–`v7`) registers. A larger result is returned through an sret pointer passed in `x8` (not returned).
- x86-64: up to three integer registers (`rax`, `rdx`, `rcx`); floating-point results use `xmm0` / `xmm1`, then x87 `st0` / `st1`. A result needing a 4th integer or 5th floating-point register is returned through an sret pointer passed in `rdi` and returned in `rax`.

**Windows**

- Windows x64: integer and floating-point arguments share four register positions (`rcx` `rdx` `r8` `r9` / `xmm0`–`xmm3`); each argument takes the next position, in the register of its class. Stack slots are 8 bytes, after the caller's 32-byte home area. The sret pointer is passed in `rcx`; results are otherwise as on x86-64.
- Windows arm64: AAPCS64 as above. A C variadic call passes every floating-point argument in integer registers.

**Narrow integers**

A callee must not rely on the upper bits of narrow integer or `bool` arguments and results; a caller may extend them. The native backend masks an incoming `bool` to bit 0 and re-extends narrow values.

A function no other module can reach (internal, never used as a value) may use an implementation-defined convention.

Calls to `extern "C"` and `extern "rok"` runtime functions use the C ABI rules of 8.6.1–8.6.5.

## 8.6.7. Tail calls

LLVM's `tail call` optimization applies automatically to ordinary functions. It is not applied to coroutines because of their state machine structure.

## 8.6.8. ABI of inline functions

The effect of `#[inline]` ([built-in attributes](../006-meta-system/602-built-in-attributes.md)) is that the call itself may disappear. In that case the calling convention has no meaning.

`#[inline(never)]` functions are always called and follow the ordinary ABI.

## 8.6.9. Evaluation order of argument passing

The left-to-right evaluation order of [parameters](../004-functions/402-parameters.md) is orthogonal to the calling convention.

Left-to-right evaluation means each argument expression is evaluated in order starting from the left. Which register or stack slot receives each result is a separate question decided by the ABI.

Some C and C++ ABIs favor right-to-left evaluation because of stack push order. rok, however, guarantees left-to-right semantics, and the compiler reconciles the two requirements using temporaries and registers as needed.

## 8.6.10. Variadic functions

The `...T` variadic of [parameters](../004-functions/402-parameters.md) is a language-level variadic that bundles the arguments into a single slice view (`Slice<T>`) (usually the same ABI as a struct argument). It is a separate concept from the platform's C variadic (`...` variable arguments).

C variadic functions (e.g. `printf`) can be called directly through `extern "C"`, in which case the ABI follows the platform's C variadic convention.

```
extern "C" {
    fn printf(fmt: ConstPtr<c_char>, ...);
}
```

- `...` may appear only in declarations in an `extern` block, and only as the last parameter. rok function definitions cannot have C variadics.
- C default argument promotion applies to arguments in variadic positions: `bool`, `i8`, `i16`, `u8`, `u16` and `c_char` are passed as `c_int` (`i32`), `f32` as `f64`, and `str` as `char*` (a NUL-terminated byte pointer).
- Aggregates (struct / tuple / array) cannot be passed as variadic arguments (compile error).

## 8.6.11. Function pointer ABI

`fn(...) -> ...` from [Function types](../004-functions/404-function-type.md) uses the same calling convention for indirect calls.

```mlir
%fn_ptr = ; ... function pointer ...
%result = call i32 %fn_ptr(...)
```

Arguments and returns have the same ABI, so direct and indirect calls share the same interface, except for the hidden `rok_tls` address of 8.6.6: a function value takes none. Its code pointer is a thunk that reads the address itself and calls the function with it.

## 8.6.12. Closure calls

A closure as described in [Closures](../004-functions/405-closure.md) is a single unit bundling an environment and a code pointer.

A call works as follows.

```
The environment pointer as the first argument, followed by the ordinary arguments.
```

This environment argument is how the closure body accesses captured variables.

LLVM compiles it like an ordinary function (the environment is just the first argument).

## 8.6.13. Async function calls

An `async fn` from [Async](../004-functions/412-async.md) returns a Future ptr immediately when called, following the ordinary function call ABI.

```
let f = async_fn(args);     // returns immediately (Future ptr)
let r = await f;             // coroutine mechanism
```

`await` consists of ordinary calls (polling the target future) and returns of the state machine ([8.10](810-lowering-coroutine-generic.md)), and works on top of the ordinary ABI.

## 8.6.14. Stability

This calling convention follows the platform C ABI, so it has no separate stability decision. rok stays compatible within the range the platform guarantees compatibility.

The return value layout of throwing functions (8.6.5) is part of the runtime ABI; any change follows the stability policy in the [overview](README.md).
