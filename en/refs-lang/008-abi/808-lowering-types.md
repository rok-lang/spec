# 8.8. Lowering: data types and expressions

This section defines how rok values, types and expressions are lowered to standard MLIR dialects.

## 8.8.1. Primitive integers / floating-point

Primitive numeric types map to MLIR types as follows.

| rok type          | MLIR type                                                                                   |
|-------------------|---------------------------------------------------------------------------------------------|
| `i8` ~ `i64`      | `i8` ~ `i64`                                                                                |
| `u8` ~ `u64`      | `i8` ~ `i64` (sign determined by the op)                                                    |
| `usize` / `isize` | `index` or target-specific `iN`                                                             |
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
!Point = !llvm.struct<"Point", (f64, f64)>
```

A struct value lives in SSA registers, in stack slots, or inline in its container: a field of another struct, an element of a `List` buffer, a slot of a closure environment or coroutine frame. Assignment and argument passing copy the fields. The compiler may box a value internally only where the box cannot be observed.

### Field access

`p.x` on an SSA value is an `extractvalue`; on a stack slot or an inline location it is a load through a field address.

```
%val  = llvm.extractvalue %p[0] : !Point
%addr = llvm.getelementptr %slot[0, 0] : (!llvm.ptr) -> !llvm.ptr, !Point
%val2 = llvm.load %addr : !llvm.ptr -> f64
```

### Field write

`p.x = 1.0f64` updates the variable's own copy: `insertvalue` on an SSA value, or a store to the field address of its slot.

```
%p2 = llvm.insertvalue %one, %p[0] : !Point
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

is lowered to `!llvm.struct<(i32, !llvm.ptr, i1)>`. Like a `struct`, it is a value with no header.

## 8.8.6. Representation of `enum`

An `enum` from [`enum`](../002-types/204-enum.md) is lowered as a tagged `union`.

```rok
enum Result {
    Ok(i32),
    Err(str),
    Pending,
}
```

It is represented as:

```
!Result = !llvm.struct<"Result", (
    i32,                                // type_id
    i32,                                // gc_flags
    i64,                                // gc_word
    i32,                                // discriminant (tag)
    !llvm.struct<(i64)>                 // payload of the largest variant (8 bytes)
)>
```

The discriminant holds the variant identifier (`Ok`=0, `Err`=1, `Pending`=2), and the payload is the `union` of all variants, with the size of the largest variant.

### Variant construction

Constructing a variant:

```rok
Result::Ok(42)
```

is lowered to the following MLIR sequence.

```
; allocate struct + fill header + fill tag + fill payload
%obj = call @rok_gc_alloc_slow(...)
; ... header ...
%tag_addr = llvm.getelementptr %obj [0, 3] : ...
llvm.store i32 0, %tag_addr  ; Ok = 0
%payload_addr = llvm.getelementptr %obj [0, 4] : ...
llvm.store %42_i32, %payload_addr
```

### Variant matching

Matching variants:

```rok
match v {
    Ok(n) => ...,
    Err(s) => ...,
    Pending => ...,
}
```

is lowered to MLIR in decision tree form.

```
%tag = llvm.load %v[3] : i32

cf.switch %tag, [
  default: ^unreachable,    ; every variant has a case, so default is never reached
  0: ^ok,
  1: ^err,
  2: ^pending
]

^ok:
  %payload = llvm.load %v[4] : i32
  ; ... used as n ...
  br ^end

^err:
  %payload_str = llvm.load %v[4] : !llvm.ptr
  ; ... used as s ...
  br ^end

^pending:
  ; ...
  br ^end
```

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

This is represented as `!llvm.struct<(i32, i64)>`, a combination of tag (i32) and payload (i64, the size of the largest variant).

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

It is lowered to an `(i32 tag, usize payload)` pair, where the tag is the type_id. `any Tr` and bare `any` have the same representation.

```
!Any = !llvm.struct<(i32, iN)>      ; iN = target pointer width (usize)
```

Number, `bool` and `char` values no wider than a pointer are stored directly in the payload. Other values are boxes on the GC heap, and the payload is the pointer to the box. Whether the payload is a pointer is therefore known only from the tag, and the GC must look at the tag when scanning an `any` slot.

### Dynamic dispatch on `any`

Consider:

```rok
v + 1i32     // v: any
```

It is lowered to the following MLIR with runtime checks.

```
%tag = llvm.extractvalue %v[0] : i32

cf.switch %tag, [
  default: ^trap,
  TAG_I32: ^add_i32,
  TAG_F64: ^add_f64,       ; other operand type mismatches go to ^trap (panic, exit 101)
  ...
]

^add_i32:
  %payload = llvm.extractvalue %v[1] : i64     ; 64-bit target
  %i = llvm.trunc %payload : i64 to i32
  %sum = arith.addi %i, %1_const : i32
  ; wrap the result back into any
  ...
```

This branching structure, which must check every possible type at runtime, is the cost of `any` slots.

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

It is lowered to a function pointer of type `(i32, i32) -> i32`, i.e. `!llvm.ptr` or `!llvm.func<...>`. Function pointer values can be stored in variables as values.

## 8.8.12. Representation of closures

Consider a capturing [closure](../004-functions/405-closure.md).

```rok
let n = 10;
let f = |x| x + n;       // captures n
```

It is lowered to the following MLIR.

```
; closure object = environment + code pointer
!Closure = !llvm.struct<(
    i32, i32, i64,                ; GC header
    !llvm.ptr,                    ; function pointer (code)
    i32                            ; environment: copy of n
)>

; on call
%env = llvm.getelementptr %clo[0, 4] : ...
%fn_ptr = llvm.extractvalue %clo[0, 3] : !llvm.ptr
%result = llvm.call %fn_ptr(%env, %x) : ...
```

On call, the environment is passed as an extra first argument to the function.

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
