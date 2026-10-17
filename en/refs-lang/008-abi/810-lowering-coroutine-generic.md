# 8.10. Lowering: coroutines and generics

This section defines coroutine lowering for [generators](../004-functions/411-generators.md), [async](../004-functions/412-async.md) and [streams](../004-functions/413-stream.md), and monomorphization lowering for [generics](../002-types/210-generics.md) and [const generics](../002-types/211-const-generics.md).

## 8.10.1. State machines

[Coroutines](../007-runtime/708-coroutine.md) are lowered to compiler-built state machines. `llvm.coro.*` is not used.

1. HIR: `yield` / `async` / `async yield` blocks are lifted into body functions, like closures. An expression that creates the frame remains in place of the block.
2. MIR: the body function is turned into a resume function. Each suspend point gets a state number, and locals live across that point are moved into frame fields.
3. The frame type is `(i32 state, locals..)` and has a ScanTable. Frames are created through the ordinary allocation path ([8.9.9](809-lowering-control.md#899-gc-alloc-fast--slow-path)).
4. The entry of the resume function branches (`switch`) on `state`.

## 8.10.2. Generator (`yield fn`)

```rok
yield fn count_up_to(n: i32) -> i32 {
    let i = 0;
    while i < n {
        yield i;
        i = i + 1;
    }
}
```

The conceptual resume function (`next`) is:

```
next(frame) -> i32 | none:
    switch frame.state:
      0: frame.i = 0; goto loop
      1: frame.i = frame.i + 1; goto loop
      2: return none
    loop:
      if frame.i < frame.n:
          frame.state = 1
          return frame.i          // yield i
      frame.state = 2
      return none
```

`count_up_to(n)` allocates the frame and returns a wrapper (`GenIter<i32>`) with `state = 0`.

## 8.10.3. Async (`async fn`)

```rok
async fn fetch() -> Data {
    let r = await op1();
    process(r)
}
```

```
poll(frame, cx) -> Poll<Data>:
    switch frame.state:
      0: frame.fut = op1(); frame.state = 1; goto await1
      1: goto await1
    await1:
      match frame.fut.poll(cx):
        Pending  => return Pending
        Ready(r) => frame.state = 2; return Ready(process(r))
```

`await` stores the target future in the frame and polls it. If Pending, it returns Pending, leaving the state in place, and the next poll continues from the same point.

## 8.10.4. Stream (`async yield fn`)

[Streams](../004-functions/413-stream.md) combine the two. The state numbers of a single frame count `yield` points and `await` points together. `poll_next` returns `Ready(value)` at a `yield`, `Pending` on a Pending `await`, and `Ready(none)` at the end.

## 8.10.5. Frame allocation

A frame is allocated on every coroutine call. There is no optimization that elides the allocation.

## 8.10.6. Generic monomorphization

Monomorphization of [generics](../002-types/210-generics.md) generates a separate instance for each type tuple at call sites.

```rok
fn id<T>(x: T) -> T { x }

let a = id(42i32);     // id<i32>
let b = id(3.14f64);   // id<f64>
```

After monomorphization, the MLIR looks like this.

```
func.func @id_i32(%x: i32) -> i32 {
  func.return %x : i32
}

func.func @id_f64(%x: f64) -> f64 {
  func.return %x : f64
}

func.func @main() {
  %a = func.call @id_i32(%42) : (i32) -> i32
  %b = func.call @id_f64(%3.14) : (f64) -> f64
}
```

A separate function is created for each concrete type tuple.

### When monomorphization happens

Monomorphization proceeds as follows.

```
rok-hir → specialize(hir, type_tuple) → rok-mir → MLIR
```

In [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) mode, all call sites are collected and then [specialize](../000-meta/007-glossary.md#specialize) runs in bulk. In [JIT](../000-meta/007-glossary.md#jit-just-in-time) mode, specialize runs at the first call and the result is cached (see [Execution engines and user surface](../000-meta/001-execution-modes.md)). Both modes share the same specialize mechanism.

### Symbol naming

The symbol mangling rule used by the [compilation pipeline](../000-meta/002-pipeline.md) is:

```
_Y<pkg_hash_8>_<path>_<name>_<typehash_16>
```

For example, `id<i32>` becomes something like `_Y3f8a1c2b_my_pkg_id_a7c91d8e44b2f190` (illustrative), and this symbol is the name of the `func.func @...`.

## 8.10.7. `const` generic monomorphization

An example of [const generics](../002-types/211-const-generics.md):

```rok
fn dot<T: Num, const N: usize>(a: [T; N], b: [T; N]) -> T { ... }

let r = dot([1.0, 2.0, 3.0], [4.0, 5.0, 6.0]);  // T=f64, N=3
```

This is monomorphized to the following MLIR.

```
func.func @dot_f64_3(%a: !llvm.array<3 x f64>, %b: !llvm.array<3 x f64>) -> f64 {
  ; ... unrolled or an ordinary loop
}
```

Because `const` N=3 is a compile-time constant, LLVM's unrolling and inlining can apply freely.

## 8.10.8. Generic trait dispatch

```rok
fn process<T: Describe>(x: T) -> str {
    x.describe()
}
```

The concrete type T is determined at the call site, so for `T = Point` the lowering is:

```
func.func @process_Point(%x: !Point_ptr) -> !str_ptr {
  ; call Point's Describe::describe directly
  %r = func.call @Point_Describe_describe(%x) : (!Point_ptr) -> !str_ptr
  func.return %r
}
```

There are no indirect calls through a vtable; everything is a direct function call. Combined with the [trait-as-type ban](../002-types/212-trait-as-type-banned.md), this model guarantees zero vtable cost.

## 8.10.9. Branch dispatch on sum types

Matching on [sum types](../002-types/206-sum-types.md) is lowered to a `cf.switch`-based decision tree.

```rok
let v: i32 | str = ...;
match v {
    n: i32 => use_int(n),
    s: str => use_str(s),
}
```

In MLIR:

```
%tag = llvm.extractvalue %v[0] : i32

cf.switch %tag, [
  default: ^trap,
  TAG_I32: ^use_int,
  TAG_STR: ^use_str
]

^use_int:
  %n = llvm.extractvalue %v[1] : i64
  %n_i32 = llvm.trunc %n : i64 to i32
  func.call @use_int(%n_i32) : ...
  cf.br ^end

^use_str:
  %s = llvm.extractvalue %v[1] : i64
  %s_ptr = llvm.inttoptr %s : i64 to !llvm.ptr
  func.call @use_str(%s_ptr) : ...
  cf.br ^end

^end:
  ...
```

Tag checks and branches are all static code.

## 8.10.10. Trait-bound dispatch of trait methods

```rok
fn add_all<T: Add<T, Output = T>>(items: List<T>) -> T {
    let r: T = ...;
    for x in items {
        r = r.add(x);
    }
    r
}
```

After monomorphization, a separate function is created for each T at each call site. For `T = i32`, `r.add(x)` resolves to a direct call to i32's `Add::add`, which can also be inlined.

## 8.10.11. Summary

The compiler lowers coroutines itself into state machines (8.10.1); no `llvm.coro.*` intrinsics are used. Generic lowering generates separate code for each concrete type through monomorphization and uses no vtables.

These two mechanisms are the most complex lowerings in this [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) block; the rest, [Lowering: data types and expressions](808-lowering-types.md) and [Lowering: control flow and GC integration](809-lowering-control.md), are close to direct mappings onto standard dialects.

This concludes chapter 8 (runtime ABI / MLIR lowering / codegen).
