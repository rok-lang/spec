# 8.8. Lowering: data types and expressions

This section defines how Rok values, types and expressions are lowered to standard MLIR dialects.

## 8.8.1. Primitive integers / floating-point

Primitive numeric types map to MLIR types as follows.

| Rok type          | MLIR type                                                                                   |
|-------------------|---------------------------------------------------------------------------------------------|
| `i8` ~ `i64`      | `i8` ~ `i64`                                                                                |
| `u8` ~ `u64`      | `i8` ~ `i64` (sign determined by the op)                                                    |
| `usize` / `isize` | `iN`, N = the target's pointer width (never `index`)                                        |
| `f32` / `f64`     | `f32` / `f64`                                                                               |
| `bool`            | `i1` (or i8, depending on the [ABI](../000-meta/007-glossary.md#abi-application-binary-interface)) |
| `char`            | `i32` (Unicode code point)                                                                  |
| `c_char`          | `i8`                                                                                        |

`str` is an immutable pair passed by value as `!llvm.struct<(ptr, iN)>` (a pointer and a `usize` byte length). Literals point to immortal constants; strings created at runtime point to the payload of a GC object (type id `1`, [8.2.12](802-scan-table.md#8212-runtime-reserved-type-ids)).

- `s[a:b]` is `(ptr + a, b - a)` and shares the original bytes (O(1)).
- `ptr` may point to any byte of a GC object (including the trailing NUL) and keeps the whole object alive. The GC treats it as an interior pointer, and evacuation rewrites it preserving the offset.
- Literals and strings newly created by the runtime have a NUL after their bytes. Slices may not.

Note that MLIR integer types themselves carry no sign; signedness is determined by the operator (`addi`, `subi`, etc.).

## 8.8.2. Arithmetic operations

Arithmetic expressions are lowered to `arith` dialect operations as follows.

```
a + b            →   arith.addi %a, %b : i32
a - b            →   arith.subi %a, %b : i32
a * b            →   arith.muli %a, %b : i32
a / b            →   arith.divsi %a, %b : i32  (signed) / divui (unsigned)
a % b            →   arith.remsi / remui

a + b (f64)      →   arith.addf %a, %b : f64
a < b            →   arith.cmpi "slt", %a, %b : i32  (signed less than)
                       or arith.cmpf "olt", %a, %b : f64
```

The trait mapping of [operator dispatch](../003-expressions/301-operator-dispatch.md) is lowered as above for primitive types. User-defined trait implementations are lowered to function calls after generic monomorphization.

## 8.8.3. Representation of `struct`

A `struct` is a value type ([2.3.8](../002-types/203-struct.md#238-value-semantics)) and has no object header.

```rok
struct Point { x: f64; y: f64; }
```

It is lowered to an LLVM struct of its fields, in the layout of [2.3.6](../002-types/203-struct.md#236-memory-layout).

```
!llvm.struct<(f64, f64)>
```

Struct types are literal (unnamed) LLVM structs; two structs with the same fields have the same MLIR type. A struct whose layout LLVM would not reproduce (an explicit `#[repr]` alignment or offset) is a packed struct with explicit `!llvm.array<k x i8>` padding, so LLVM's offsets and size are those of 2.3.6.

A struct value lives in SSA registers, in stack slots, or inline in its container: a field of another struct, an element of a `List` buffer, a slot of a closure environment or coroutine frame. Assignment and argument passing copy the fields. The compiler may box a value internally only where the box cannot be observed.

### Field access

`p.x` on an SSA value is an `extractvalue`; on a stack slot or an inline location it is a load through a field address.

```
%val  = llvm.extractvalue %p[0] : !llvm.struct<(f64, f64)>
%addr = llvm.getelementptr %slot[0, 0] : (!llvm.ptr) -> !llvm.ptr, !llvm.struct<(f64, f64)>
%val2 = llvm.load %addr : !llvm.ptr -> f64
```

### Field write

`p.x = 1.0f64` updates the variable's own copy: `insertvalue` on an SSA value, or a store to the field address of its slot.

```
%p2 = llvm.insertvalue %one, %p[0] : !llvm.struct<(f64, f64)>
```

When the struct is stored inline in a GC heap object and the field holds a GC reference, the store goes through the write barrier ([Barriers and roots](../007-runtime/704-barriers-and-roots.md)). Pointer fields of a struct in a stack slot are GC roots ([7.4](../007-runtime/704-barriers-and-roots.md)).

## 8.8.4. Representation of tuple `struct`

Consider the tuple `struct`:

```rok
struct Pair(i32, i32);
```

It is represented the same as an ordinary `struct`, except that fields are accessed by positional index instead of by name.

## 8.8.5. Tuple types

The tuple type

```rok
(i32, str, bool)
```

is lowered to `!llvm.struct<(i32, !llvm.struct<(ptr, i64)>, i1)>` on a 64-bit target: `str` is its `(ptr, iN)` pair (8.8.1) wherever it appears. Like a `struct`, it is a value with no header.

## 8.8.6. Representation of `enum`

An `enum` from [`enum`](../002-types/204-enum.md) is a value with no object header, lowered as a tagged `union`: the same `(i32 tag, payload)` layout as a sum type (8.8.8).

```rok
enum Result {
    Ok(i32),
    Err(str),
    Pending,
}
```

It is represented as:

```
!llvm.struct<(i32, !llvm.array<2 x i64>)>    ; 64-bit target
;            tag   payload: 16 bytes, 8-aligned
```

The tag holds the variant's index (`Ok`=0, `Err`=1, `Pending`=2). The payload slot holds every variant with data:

- variants with the same payload type share it (`enum E { A(i64), B(i64) }` is `(i32, i64)`);
- scalar payloads of different types share an unsigned integer as wide as the widest (`A(i32)`, `B(f64)` share an `i64`);
- any other mix (aggregates, `str`) shares a byte-array `!llvm.array<k x iN>`, as large and as aligned as the largest variant;
- a payload that can hold a GC reference is `!llvm.array<k x iN>` of pointer-width words, which the collector scans per the tag ([8.2](802-scan-table.md), `SUM_PAYLOAD`).

A fieldless enum is its tag alone: a bare `i32`, or the integer of its `#[repr(..)]`.

### Variant construction

`Result::Ok(42)` builds the value in registers or in the destination's slot: no allocation.

```
%t = llvm.insertvalue %c0_i32, %undef[0] : !llvm.struct<(i32, !llvm.array<2 x i64>)>
; the payload is written as its own type and reinterpreted as the slot's
%v = llvm.insertvalue %payload, %t[1] : !llvm.struct<(i32, !llvm.array<2 x i64>)>
```

A union value built from a shorter variant's payload holds zero in the bytes past that payload, so a root word copied from it is null or a reference, never stale stack ([8.9.12](809-lowering-control.md#8912-shadow-stack-frame)).

### Variant matching

Matching variants:

```rok
match v {
    Ok(n) => ...,
    Err(s) => ...,
    Pending => ...,
}
```

compares the tag with each arm's variant index in arm order, a chain of `cf.cond_br` (8.9.3); a value no arm matches reaches an unreachable block.

```
%tag = llvm.extractvalue %v[0] : !llvm.struct<(i32, !llvm.array<2 x i64>)>
%is_ok = arith.cmpi eq, %tag, %c0 : i32
cf.cond_br %is_ok, ^ok, ^next1
^next1:
%is_err = arith.cmpi eq, %tag, %c1 : i32
cf.cond_br %is_err, ^err, ^pending
^ok:
  ; payload reinterpreted as i32 -> n
```

At `-O1` and above LLVM turns the chain into one `switch` on the tag, a jump table when the cases are dense (8.7.3).

## 8.8.7. Representation of (untagged) `union`

Consider a `union` from [`union`](../002-types/205-union.md).

```rok
union EpollData {
    ptr: usize;
    fd: i32;
    val: u64;
}
```

It is lowered to `!llvm.struct<(i64)>`, the size of its largest field. Each field access interprets the same memory as a different type through bitcast or getelementptr. Field reads must be inside `unsafe` in the source ([2.5](../002-types/205-union.md)).

## 8.8.8. Representation of sum types (`A | B`)

Sum types from [Sum types](../002-types/206-sum-types.md) are handled as follows.

```rok
type X = i32 | f64
```

This is represented as `!llvm.struct<(i32, i64)>`, a combination of tag (i32) and payload slot, built as for an `enum` (8.8.6): here the two scalar payloads share an `i64`. A `none` member is tag 0 and carries no payload, so zeroed memory reads back as `none`.

If niche optimization ([niche optimization](../002-types/207-niche-optimization.md)) applies, the tag slot disappears and only the union bit pattern remains.

```rok
Ptr<u8> | none
```

This type is lowered to `!llvm.ptr` (a single pointer; an automatic niche in which null means `none`).

## 8.8.9. Representation of `any` slots

Consider the dynamic slot from [`any` slot](../002-types/202-any-slot.md).

```rok
let x: any = ...
```

It is lowered to an `(i32 tag, payload)` pair. `any Tr` and bare `any` have the same representation.

```
!llvm.struct<(i32, !llvm.ptr)>
```

The low 31 bits of the tag identify the value's type: a 31-bit FNV-1a hash of the type's name, so separately compiled packages agree (a collision in one build takes the next free value). Number, `bool` and `char` values no wider than a pointer are stored directly in the payload word, as its bits (`inttoptr` / `ptrtoint`). Other values are boxes on the GC heap, the payload is the box, and bit 31 of the tag (`ANY_BOXED`) is set; the GC reads that bit when scanning an `any` slot ([8.2.7](802-scan-table.md)).

### Type tests on `any`

A type pattern on an `any` value

```rok
match v {
    n: i32 => n + 1,
    s: str => s.len() as i32,
    _ => 0,
}
```

masks off `ANY_BOXED` and compares the tag with each pattern type's tag, a chain of `cf.cond_br`:

```
%tag = llvm.extractvalue %v[0] : !llvm.struct<(i32, !llvm.ptr)>
%t   = arith.andi %tag, %c0x7fffffff : i32
%is  = arith.cmpi eq, %t, %TAG_I32 : i32
cf.cond_br %is, ^as_i32, ^next
^as_i32:
  %w = llvm.extractvalue %v[1] : !llvm.struct<(i32, !llvm.ptr)>
  %i = llvm.ptrtoint %w : !llvm.ptr to i64
  %n = arith.trunci %i : i64 to i32
^next:
  ; boxed `str`: the pair is loaded from the box payload (offset 16)
```

These tests at run time are the cost of `any` slots.

## 8.8.10. Array types: `[T; N]`

Consider a fixed-size array from [Type expressions](../002-types/201-type-expressions.md).

```rok
[i32; 8]
```

It is lowered to `!llvm.array<8 x i32>`. Inside a GC object, a 16-byte header precedes the array.

## 8.8.11. Function types: `fn(...) -> ...`

Consider a [function type](../004-functions/404-function-type.md).

```rok
fn(i32, i32) -> i32
```

Every function value, a named function or a closure, is a `{code, env}` pair:

```
!llvm.struct<(ptr, ptr)>      ; code, env
```

`code` takes `env` as its first argument, then the declared arguments (and, like every Rok function, the thread's TLS base last, [8.6](806-calling-convention.md)). A named function used as a value has a null `env` and a code pointer to a thunk that drops it. Only a C function type (`extern "C" fn(..)`) is a bare code pointer `!llvm.ptr`.

## 8.8.12. Representation of closures

Consider a capturing [closure](../004-functions/405-closure.md).

```rok
let n = 10;
let f = |x| x + n;       // captures n
```

It is lowered to a function value (8.8.11) whose `env` is a GC object holding the captures, one field each, in a struct after the 16-byte header:

```
; env object: header + !llvm.struct<(i32)>   ; copy of n
%f = !llvm.struct<(ptr, ptr)>                ; {code of the closure body, env}

; on call
%code = llvm.extractvalue %f[0] : !llvm.struct<(ptr, ptr)>
%env  = llvm.extractvalue %f[1] : !llvm.struct<(ptr, ptr)>
%r    = llvm.call %code(%env, %x, %tls) : !llvm.ptr, (!llvm.ptr, i32, !llvm.ptr) -> i32
```

A closure that captures nothing has a null `env` and allocates nothing.

## 8.8.13. Identifier / variable access

Consider local variables from [Variable declarations](../005-modules/501-variable-declaration.md).

```rok
let x = 42i32;
let y = x + 1i32;
```

They are lowered to LLVM SSA form.

```
%x = arith.constant 42 : i32
%y = arith.addi %x, %1 : i32
```

Locals are lowered to SSA; locals that need a stack slot use `llvm.alloca` and load / store (same in debug and release).

```rok
let counter = 0i32;
counter = counter + 1i32;
```

This is lowered to:

```
%counter_ref = llvm.alloca %c1 x i32 : (i64) -> !llvm.ptr
llvm.store %0, %counter_ref : i32, !llvm.ptr

%loaded = llvm.load %counter_ref : !llvm.ptr -> i32
%new = arith.addi %loaded, %1 : i32
llvm.store %new, %counter_ref : i32, !llvm.ptr
```

In optimized builds, LLVM's mem2reg promotes this back to SSA.

## 8.8.14. Literal desugaring

Consider a collection literal from [`#[literal]` magic](../006-meta-system/605-literal-magic.md).

```rok
[1, 2, 3]
```

It desugars to a call to the standard library function `list_literal(1, 2, 3)` and follows ordinary function call lowering.

String and numeric literals are handled as compile-time `const`s stored in .rodata and are then used as pointers or values.
