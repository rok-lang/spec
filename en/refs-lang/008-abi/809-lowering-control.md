# 8.9. Lowering: control flow and GC integration

This section covers the lowering rules for [`if` expressions](../003-expressions/310-if-expression.md), [`match`](../003-expressions/312-match-expression.md), [loops](../003-expressions/313-loop-expressions.md) and [error control flow](../003-expressions/314-error-control-flow.md), and also defines the rules for emitting [GC](../000-meta/007-glossary.md#gc-garbage-collector) alloc / barrier / safepoint code.

The generated MLIR / LLVM module carries the target triple and data layout of the build target (`--target`).

## 8.9.1. `if` expressions

```rok
if cond { x } else { y }
```

This expression is lowered to MLIR using `scf.if` as follows.

```
%result = scf.if %cond -> (i32) {
  scf.yield %x : i32
} else {
  scf.yield %y : i32
}
```

An `if` with no `else` branch and a unit value:

```rok
if cond { do_thing(); }
```

lowers to the simpler form:

```
scf.if %cond {
  func.call @do_thing()
}
```

## 8.9.2. `if let`

Consider the `if let pattern = expr` form from [`if` expressions](../003-expressions/310-if-expression.md).

```rok
if let v: T = compute() { use(v) } else { fallback() }
```

This expression is lowered to MLIR in decision tree form:

```
%opt = func.call @compute() : () -> !sum_T_none
%tag = llvm.extractvalue %opt[0] : i32
%is_some = arith.cmpi "ne", %tag, %TAG_NONE : i32

scf.if %is_some {
  %payload = llvm.extractvalue %opt[1] : !T
  func.call @use(%payload)
} else {
  func.call @fallback()
}
```

## 8.9.3. `match` decision tree

The arms of a [`match`](../003-expressions/312-match-expression.md) are lowered to a decision tree.

```rok
match v {
    1 => a,
    2 => b,
    n if n > 100 => c,
    _ => d,
}
```

This `match` expression becomes the following MLIR.

```
cf.switch %v : i32, [
  default: ^check_guard,
  1: ^arm_a,
  2: ^arm_b
]

^check_guard:
  %is_big = arith.cmpi "sgt", %v, %100 : i32
  cf.cond_br %is_big, ^arm_c, ^arm_d

^arm_a: ; ... = a
  cf.br ^end(%a)

^arm_b: ; ... = b
  cf.br ^end(%b)

^arm_c: ; ... = c
  cf.br ^end(%c)

^arm_d: ; ... = d
  cf.br ^end(%d)

^end(%result: i32):
  ; ...
```

For complex patterns such as tuples, `struct`s, `enum` variants and ranges, each destructuring step is lowered to internal switch / cmp operations.

## 8.9.4. `loop` / `while` / `for`

### `loop`

`loop` is used as follows.

```rok
let v = loop { ...; if cond { break x; } }
```

It is lowered to MLIR using `scf.while` or expressed directly with `cf.br`.

```
%v = scf.while () : () -> (i32) {
  ; condition check (always true for loop)
  scf.condition (%true) ()
} do {
  ; body
  ; if cond { break x; } is handled separately (below)
  scf.yield
}
```

`break value` is hard to express inside `scf.while`, so it is spelled out with `scf.execute_region` or `cf.br`.

### `while`

`while` is used as follows.

```rok
while cond { body }
```

It is lowered to `scf.while`:

```
scf.while () : () -> () {
  ; condition
  %c = ...
  scf.condition (%c) ()
} do {
  ; body
  scf.yield
}
```

### `for`

`for` is used as follows.

```rok
for x in xs { body }
```

The desugaring from [Loop expressions](../003-expressions/313-loop-expressions.md) expands it to `let it = xs.into_iter(); loop { match it.next() { ... } }`, which then goes through the same lowering described above.

## 8.9.5. `break` / `continue`

`break` and `continue` inside a loop jump with `cf.br` to the start or end of the loop.

`scf.while` alone is awkward for this, so they are lowered to direct branches of the `cf` dialect.

Labels (`'outer:`) likewise jump to the appropriate [block](../000-meta/007-glossary.md#block).

## 8.9.6. `try` / `catch` / `finally`: value-based branching

Errors from [error control flow](../003-expressions/314-error-control-flow.md) are propagated through the `{ is_err, payload }` return value defined in [Calling convention](806-calling-convention.md#865-return-of-throwing-functions-value-based-error-propagation). `try` and `catch` are therefore both lowered to ordinary branches.

```rok
let v = try f(x);              // f: fn(X) throws (E) -> T
```

A call marked with `try` checks `is_err` of the return value: on success it extracts the payload as `T`; on error it branches to the error path.

```
%r = func.call @f(%x) : (!X) -> !Throws_T_E
%is_err = llvm.extractvalue %r[0] : !Throws_T_E
cf.cond_br %is_err, ^err, ^ok

^ok:
  %v = ; read payload as T
  ...

^err:
  %e = ; read payload as E
  ; if there is an enclosing catch, go to its dispatch block;
  ; otherwise build the enclosing function's return value { 1, e } and return it
  cf.br ^catch_dispatch(%e)
```

An expression with `catch` clauses has a single dispatch block where all error paths of the left-hand expression converge.

```rok
try body catch e: A { handler_a } catch _ { handler_any }
```

```
^catch_dispatch(%e: !E):
  ; test patterns in clause order (same scheme as the decision tree of 8.9.3)
  cf.cond_br %is_a, ^handler_a(%e), ^handler_any(%e)

^handler_a(%ea: !A):
  ; handler_a
  cf.br ^end(%result_a)

^handler_any(%ex: !E):
  ; handler_any
  cf.br ^end(%result_any)

^end(%result: ...):
  ...
```

If no clause matches, the dispatch block passes the error to the dispatch block of the outer `catch`, or builds the enclosing function's error return value and returns it. Type pattern tests on errors of untyped `throws` (`any Error`) are lowered to tag comparisons of [`any`](808-lowering-types.md).

`try?` produces `none` on the error path and goes to `^end`; `try!` calls `panic_fn` on the error path.

## 8.9.7. `throw`

`throw` is used as follows.

```rok
throw err;
```

`throw` is lowered to an unconditional branch to the error path. If the nearest `catch` is in the same function, it goes to that dispatch block; otherwise it builds the enclosing function's error return value and returns it.

```
; when a catch exists in the same function
cf.br ^catch_dispatch(%err)

; otherwise, error return from the function
%r0 = llvm.mlir.undef : !Throws_T_E
%r1 = llvm.insertvalue %true, %r0[0] : !Throws_T_E
%r2 = ; store %err in the payload
func.return %r2 : !Throws_T_E
```

The path returning an error out of the function is the same as an ordinary `return`, so epilogues such as the shadow stack pop run as usual. There is no runtime call.

## 8.9.8. `finally`

`finally` is used as follows.

```rok
try { body } finally { cleanup }
```

It is lowered so that every path leaving the left-hand expression passes through the cleanup block. Normal completion, completion after `catch` handling, propagation of an unhandled error, and each `return` / `break` / `continue` path branch to the cleanup block and then continue to their original destination. Error propagation is also an ordinary branch, so no separate unwind mechanism is needed.

```
^body_ok(%v):        cf.br ^cleanup(%k_ok, %v)
^body_err(%e):       cf.br ^cleanup(%k_err, %e)
^body_return(%rv):   cf.br ^cleanup(%k_ret, %rv)

^cleanup(%kind, %val):
  ; cleanup
  ; depending on %kind, branch to ^end / error propagation / function return / loop destination
```

If a new error is thrown inside the cleanup block, it replaces the original path. A panic terminates the process, so it does not pass through the cleanup block.

## 8.9.9. GC alloc: fast / slow path

Consider [Allocation](../007-runtime/702-allocation.md) together with this.

```rok
let p = Point { x: 1.0, y: 2.0 };
```

This is lowered to the following MLIR with the fast path inlined.

```
; fast path
%t = llvm.mlir.addressof @rok_tls : !llvm.ptr
%cursor_ptr = llvm.getelementptr %t[8] : (!llvm.ptr) -> !llvm.ptr, i8    ; tlab_cursor at p
%limit_ptr = llvm.getelementptr %t[16] : (!llvm.ptr) -> !llvm.ptr, i8    ; tlab_limit at 2p

%cursor = llvm.load %cursor_ptr : !llvm.ptr -> !llvm.ptr
%new_cursor = llvm.getelementptr %cursor[24] : !llvm.ptr
%limit = llvm.load %limit_ptr : !llvm.ptr -> !llvm.ptr

%overflow = llvm.icmp "ugt" %new_cursor, %limit : !llvm.ptr
cf.cond_br %overflow, ^slow, ^fast

^fast:
  llvm.store %new_cursor, %cursor_ptr : !llvm.ptr, !llvm.ptr
  cf.br ^cont(%cursor)

^slow:
  %p_slow = func.call @rok_gc_alloc_slow(%24, %8, %TYPE_ID_POINT) : (i64, i64, i32) -> !llvm.ptr
  cf.br ^cont(%p_slow)

^cont(%p: !llvm.ptr):
  ; %p is the start of the object header; fill header, then fields
  ; ...
```

In the fast path above, `%24` is the total object size, `%8` the alignment, and `%TYPE_ID_POINT` the type_id constant. The fast path bumps the TLAB pointer and then writes the header's `type_id` and `gc_word` (= size, [8.1.4](801-object-header-abi.md#814-gc_word-offset-8-u64-or-ptr)) inline.

The slow path calls `rok_gc_alloc_slow`.

### Filling the header

The object header and fields are filled as follows.

```
; fill type_id
%hdr_type_addr = llvm.getelementptr %p[0, 0] : ...
llvm.store %TYPE_ID_POINT, %hdr_type_addr : i32

; gc_flags initial value 0
; ... (may be filled by the slow path)

; gc_word = object size (8.1.4)
%hdr_word_addr = llvm.getelementptr %p[0, 2] : ...
llvm.store %24, %hdr_word_addr : i64

; fill fields
%x_addr = llvm.getelementptr %p[0, 3] : ...
llvm.store %1.0, %x_addr : f64
%y_addr = llvm.getelementptr %p[0, 4] : ...
llvm.store %2.0, %y_addr : f64
```

## 8.9.10. Write barrier

The object-remembering barrier from [Barriers and root tracking](../007-runtime/704-barriers-and-roots.md#742-write-barrier-incremental-update).

```rok
*p.field = q;       // write a GC pointer into a GC object
```

This is lowered to the following MLIR.

```
; 1. write barrier: checks only the header of the destination p (may precede the store)
%hdr_flags = llvm.load %p[1] : i32
%unlogged = llvm.and %hdr_flags, %UNLOGGED_BIT : i32
%need_log = arith.cmpi "ne", %unlogged, %0 : i32
cf.cond_br %need_log, ^log, ^store

^log:
  func.call @rok_gc_wb_slow(%p) : (!llvm.ptr) -> ()   ; atomically clear UNLOGGED and add to the remembered set
  cf.br ^store

^store:
  ; 2. ordinary store
  %addr = llvm.getelementptr %p[0, 4] : ...
  llvm.store %q, %addr : !llvm.ptr
```

- The header of the stored value `q` is not read. `q` may be an inline integer of an `any` slot or an interior pointer of a `str`, so reading it as a header is unsafe.
- The fast path is one load and one branch. In practice it checks `UNLOGGED | CARDED` together. If `CARDED` (a large old object), it marks dirty, without a call, the 512 B cards spanned by the written slot (`(slot - p) >> 9` ..= `(slot + len - 1 - p) >> 9`) in the card table (whose pointer is the word immediately before the header); if `UNLOGGED`, it calls `rok_gc_wb_slow` as above. The barrier therefore uses the slot address and the size of the stored value ([7.4.2](../007-runtime/704-barriers-and-roots.md#742-write-barrier-incremental-update)).
- `rok_gc_wb_slow` is not a safepoint. No collection intervenes between the barrier and the store.
- No barrier is inserted for initialization of a just-allocated object, stores into a `static`, raw `Ptr<T>` memory, or non-reference atomics.

## 8.9.11. Safepoint poll

Consider also the [Safepoint protocol](804-safepoint.md).

```
; safepoint poll
%t = llvm.mlir.addressof @rok_tls : !llvm.ptr
%flag_ptr = llvm.getelementptr %t[88] : (!llvm.ptr) -> !llvm.ptr, i8    ; yield_flag at 10p+8
%flag = llvm.load %flag_ptr : !llvm.ptr -> i8
%need_yield = arith.cmpi "ne", %flag, %0 : i8

cf.cond_br %need_yield, ^slow, ^cont

^slow:
  func.call @rok_runtime_yield() : () -> ()
  cf.br ^cont

^cont:
  ; ... continue ...
```

The compiler emits this code at function entry (or before the first safepoint of each path, [8.4.7](804-safepoint.md#847-safepoint-invariant)) and on loop back-edges.

## 8.9.12. Shadow stack frame

See also [Barriers and root tracking](../007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler). The compiler does not use LLVM's `gc "shadow-stack"` strategy or `llvm.gcroot`; it emits the link / unlink of frames directly. LLVM's chain head `llvm_gc_root_chain` is a single process-global and cannot form per-thread chains.

Example of a function with 2 GC reference words (`%x`, `%y`). Slots hold the address of each word, so a variable assignment is just an ordinary store into `%x`.

```
; entry: zero-initialize words, allocate and link the frame
%x = llvm.alloca %c1 x !llvm.ptr : (i64) -> !llvm.ptr
%y = llvm.alloca %c1 x !llvm.ptr : (i64) -> !llvm.ptr
llvm.store %null, %x : !llvm.ptr, !llvm.ptr
llvm.store %null, %y : !llvm.ptr, !llvm.ptr
%frame = llvm.alloca %c1 x !llvm.struct<(ptr, i64, array<2 x ptr>)> : (i64) -> !llvm.ptr
%head_ptr = llvm.mlir.addressof @rok_tls : !llvm.ptr             ; shadow stack head at offset 0
%head = llvm.load %head_ptr : !llvm.ptr -> !llvm.ptr
%prev_p = llvm.getelementptr %frame[0, 0] : (!llvm.ptr) -> !llvm.ptr
llvm.store %head, %prev_p : !llvm.ptr, !llvm.ptr
%count_p = llvm.getelementptr %frame[0, 1] : (!llvm.ptr) -> !llvm.ptr
llvm.store %c2_i64, %count_p : i64, !llvm.ptr          ; count: usize
%s0 = llvm.getelementptr %frame[0, 2, 0] : (!llvm.ptr) -> !llvm.ptr
llvm.store %x, %s0 : !llvm.ptr, !llvm.ptr          ; the address, not the value
%s1 = llvm.getelementptr %frame[0, 2, 1] : (!llvm.ptr) -> !llvm.ptr
llvm.store %y, %s1 : !llvm.ptr, !llvm.ptr
llvm.store %frame, %head_ptr : !llvm.ptr, !llvm.ptr

; variable assignment: no frame update
llvm.store %obj, %x : !llvm.ptr, !llvm.ptr

; immediately before every return (including error returns): unlink
%saved = llvm.load %prev_p : !llvm.ptr -> !llvm.ptr
llvm.store %saved, %head_ptr : !llvm.ptr, !llvm.ptr
```

`@rok_tls` is the thread-local block of [8.3.8](803-tls-slots.md#838-form-of-compiler-emission); the shadow stack head is its slot at offset 0. An error return is also an ordinary `return` ([8.9.6](#896-try--catch--finally-value-based-branching)), so it passes through the same unlink. The link may also come later than shown, before the first safepoint of each path ([8.4.7](804-safepoint.md#847-safepoint-invariant)); then only the returns of paths that linked unlink. A function needs a frame only if it holds GC references and can reach a safepoint ([8.4.2](804-safepoint.md#842-emission-sites)). Calls to functions that never return (panics) and to runtime functions that neither allocate nor block ([8.5.2](805-extern-symbols.md#functions-that-are-not-safepoints)) do not count.

### Module data layout

A module emitted by the compiler must set the `llvm.data_layout` attribute to match the target. Otherwise sizes obtained by folding `gep null[1]` (object sizes, frame sizes) may disagree with the target. On paths that do not set a data layout, sizes are computed as constants rather than with such folded `gep`s.

## 8.9.13. Indexing

Consider `arr[i]` from [Indexing and slicing](../003-expressions/306-indexing-and-slicing.md).

```rok
arr[5]
```

This expression becomes a call to the standard library function `Index::index(arr, 5)` and is lowered like an ordinary function call.

With the `Index` trait inlined, the result is:

```
%ptr = llvm.getelementptr %arr_data, %5 : ...
%val = llvm.load %ptr : ...
```

This result falls out automatically from generic monomorphization and inlining.

## 8.9.14. Summary: what the hot path looks like

The LLVM IR cost of the typical hot path, a [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) alloc plus field writes, is:

- TLAB cursor check: load / add / icmp / cond_br (4 instructions)
- cursor update: store (1 instruction)
- header fill: store \* 3 (type_id, gc_flags, gc_word)
- field fill: store \* N (number of fields)
- write barrier (if needed): load / and / icmp / cond_br + (slow) call (4 ~ 6 instructions)
- safepoint poll (if needed): load / icmp / cond_br (3 instructions)

Allocating and filling a small object takes roughly 10 ~ 20 instructions. LLVM optimizations (inlining, mem2reg, simplification) remove some of them.
