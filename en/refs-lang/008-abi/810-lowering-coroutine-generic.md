# 8.10. Lowering: coroutines and generics

This section defines coroutine lowering for [generators](../004-functions/411-generators.md), [async](../004-functions/412-async.md) and [streams](../004-functions/413-stream.md), and monomorphization lowering for [generics](../002-types/210-generics.md) and [const generics](../002-types/211-const-generics.md).

## 8.10.1. State machines

[Coroutines](../007-runtime/708-coroutine.md) are lowered to compiler-built state machines. `llvm.coro.*` is not used.

1. HIR: `yield` / `async` / `async yield` blocks are lifted into body functions, like closures. An expression that creates the frame remains in place of the block.
2. MIR: the body function is turned into a resume function. Each suspend point gets a state number, and locals live across that point are moved into frame fields.
3. The frame type is `(i32 state, locals..)` and has a ScanTable. Frames are created through the ordinary allocation path ([8.9.9](809-lowering-control.md#899-gc-alloc-fast--slow-path)).
4. The entry of the resume function dispatches on `state`: a chain of two-way branches comparing it with each state number (8.9.3), which LLVM folds into a `switch` at `-O1` and above.

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

A frame is allocated on every coroutine call. There is no dedicated elision for a coroutine run to completion by its caller; a frame that never escapes may be removed like any allocation ([7.2.12](../007-runtime/702-allocation.md#7212-allocation-elision)).

## 8.10.6. Generic monomorphization

Monomorphization of [generics](../002-types/210-generics.md) generates a separate instance for each type tuple at call sites.

```rok
fn id<T>(x: T) -> T { x }

let a = id(42i32);     // id<i32>
let b = id(3.14f64);   // id<f64>
```

After monomorphization, the MLIR looks like this.

```
func.func @id<i32>(%x: i32, %tls: !llvm.ptr) -> i32 {
  func.return %x : i32
}

func.func @id<f64>(%x: f64, %tls: !llvm.ptr) -> f64 {
  func.return %x : f64
}

func.func @main(%tls: !llvm.ptr) {
  %a = func.call @id<i32>(%c42, %tls) : (i32, !llvm.ptr) -> i32
  %b = func.call @id<f64>(%c3_14, %tls) : (f64, !llvm.ptr) -> f64
}
```

A separate function is created for each concrete type tuple (`@id<i32>` stands for its mangled symbol, below).

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

The package is the hash, so a function at the package root has an empty `<path>`: `id<i32>` becomes something like `_Y3f8a1c2b__id_a7c91d8e44b2f190` (illustrative), and this symbol is the name of the `func.func @...`. A non-generic item has an empty `<typehash>` (`_Y3f8a1c2b__main_`). A literal `_` in an identifier is escaped as `_0` ([9.5](../009-tools/905-symbol-mangling.md)).

## 8.10.7. `const` generic monomorphization

An example of [const generics](../002-types/211-const-generics.md):

```rok
fn dot<T: Num, const N: usize>(a: [T; N], b: [T; N]) -> T { ... }

let r = dot([1.0, 2.0, 3.0], [4.0, 5.0, 6.0]);  // T=f64, N=3
```

This is monomorphized to the following MLIR.

```
func.func @dot<f64, 3>(%a: !llvm.array<3 x f64>, %b: !llvm.array<3 x f64>, %tls: !llvm.ptr) -> f64 {
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
func.func @process<Point>(%x: !llvm.struct<(f64, f64)>, %tls: !llvm.ptr) -> !llvm.struct<(ptr, i64)> {
  ; call Point's Describe::describe directly
  %r = func.call @Point.describe(%x, %tls) : (!llvm.struct<(f64, f64)>, !llvm.ptr) -> !llvm.struct<(ptr, i64)>
  func.return %r : !llvm.struct<(ptr, i64)>
}
```

`Point` is passed as its value struct (8.8.3), `str` as its `(ptr, i64)` pair (8.8.1). There are no indirect calls through a vtable; everything is a direct function call. Combined with the [trait-as-type ban](../002-types/212-trait-as-type-banned.md), this model guarantees zero vtable cost.

## 8.10.9. Branch dispatch on sum types

Matching on [sum types](../002-types/206-sum-types.md) is lowered to tag tests, a chain of `cf.cond_br` (8.9.3).

```rok
let v: i32 | str = ...;
match v {
    n: i32 => use_int(n),
    s: str => use_str(s),
}
```

`i32 | str` is `!llvm.struct<(i32, !llvm.array<2 x i64>)>` on a 64-bit target: tag 0 is `i32`, tag 1 is `str`, and the payload is two words because `str` is the two-word pair holding a GC reference (8.8.6). In MLIR:

```
%tag = llvm.extractvalue %v[0] : !llvm.struct<(i32, !llvm.array<2 x i64>)>
%p   = llvm.extractvalue %v[1] : !llvm.struct<(i32, !llvm.array<2 x i64>)>
%is_i32 = arith.cmpi eq, %tag, %c0 : i32
cf.cond_br %is_i32, ^use_int, ^test_str
^test_str:
%is_str = arith.cmpi eq, %tag, %c1 : i32
cf.cond_br %is_str, ^use_str, ^unreachable

^use_int:
  ; payload reinterpreted as i32
  func.call @use_int(%n, %tls) : (i32, !llvm.ptr) -> ()
  cf.br ^end

^use_str:
  ; payload reinterpreted as !llvm.struct<(ptr, i64)>
  func.call @use_str(%s, %tls) : (!llvm.struct<(ptr, i64)>, !llvm.ptr) -> ()
  cf.br ^end

^end:
  ...
```

The match is exhaustive, so `^unreachable` is never reached. Tag checks and branches are all static code.

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
