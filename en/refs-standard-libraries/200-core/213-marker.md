# 2.13. marker - `Clone` / `Copy` / `Drop`

Duplicating values and cleaning up resources that are no longer needed are unavoidable concerns in any language. The `rok::core::marker` module expresses these two concerns with three traits. `Clone` defines how to make an independent copy of a value; `Copy` is a marker stating that a plain bit copy suffices for a type; and `Drop` defines cleanup code that must run when a value leaves scope. The module also holds `NoneSentinel`, the library name of the `none` literal's type (2.13.8), and `Handoff`, a wrapper for values handed between threads (2.13.9).

Rok's use of a GC directly affects the meaning of these traits. The GC is responsible for reclaiming memory, so `Drop` is not tied to memory deallocation. `Drop` is limited to finalization work that cleans up resources living outside the language, such as file handles and sockets. This places Rok's marker traits apart from both Rust and Java/Python.

## 2.13.1. Memory model - Clone/Drop in a language with a GC

In Rok's runtime, the [Immix + Sticky Mark-Bit Generational GC](../../refs-lang/007-runtime/703-gc-algorithm.md) manages the lifetime of heap objects automatically. The user does not decide when an object disappears from memory. Cleanup of external resources, however, must happen deterministically, so Rok's `Drop` operates as a mechanism separate from the GC. Compared with Rust:

| Aspect               | Rust (no GC)                 | Rok (GC)                                          |
|----------------------|------------------------------|---------------------------------------------------|
| Memory reclamation   | `Drop` is responsible        | GC automatically - `Drop` **does not**            |
| Meaning of `Drop::drop` | release resource + free memory | **external resource cleanup only** (file handle, socket, ...) |
| When called          | deterministic (scope exit)   | **only on leaving a `using` block** (unrelated to GC collection) |
| Effect if omitted    | resource leak + memory leak  | resource leak only (the GC eventually reclaims memory) |

> [!NOTE]
> Ordinary bindings have no automatic drop at scope exit. `Drop::drop` is called only for bindings bound with `using` ([refs-lang 3.17 `using`](../../refs-lang/003-expressions/317-using-expression.md)). The GC does not guarantee finalization of other objects and never calls `drop`.

Because memory reclamation and resource cleanup are separate, ordinary objects need not care about `Drop` - the GC reclaims them - and only types that handle external resources implement `Drop` to obtain a deterministic cleanup point.

## 2.13.2. `Clone` - deep copy

`Clone::clone` is the method called when an independent copy of a value is needed. Its signature is simple:

```rok
#[lang("clone_trait")]
trait Clone {
    fn clone(self) -> Self;
}
```

`clone` performs a deep copy. Even if the value holds references internally, the data they point to is copied too, so the two sides can be modified independently afterwards. Shared data that the GC guarantees to be immutable may still be shared as is.

### Meaning of `Clone`

The following example shows `Clone` in its most direct form.

```rok
let a: List<i32> = [1, 2, 3];
let b = a.clone();           // new List, with a new internal buffer
b.push(4);                   // a is unaffected
a;                           //→ [1, 2, 3]
b;                           //→ [1, 2, 3, 4]
```

"Deep" here means precisely this: `Clone` preserves logical equality while guaranteeing mutation independence between the values. Two properties follow. Immediately after `let b = a.clone();`, `a == b` holds if `PartialEq` is defined. After that, no change to `a` is reflected in `b`, and vice versa.

### No automatic calls - `clone_trait` lang slot

In some languages, writing `let y = x;` makes the compiler call `clone` automatically. Rok chooses the explicit side here.

```rok
let xs: List<i32> = [1, 2, 3];
let ys = xs;          // GC heap object → reference shared (no clone)
```

`let y = x;` shares the reference for a GC heap object and performs a bit copy for a `Copy` type. `Clone::clone` is therefore never dispatched automatically; it runs only when the user calls it explicitly. For this reason `Clone` is registered not as a method slot but as the trait slot `clone_trait`, and the compiler looks this slot up only in the `#[derive(Clone)]` expansion.

### Default bodies

The `clone` of a primitive type amounts to a bit copy, so it maps directly to an intrinsic; the `clone` of a collection allocates a new internal buffer and clones the elements recursively.

```rok
impl Clone for i32 {
    #[intrinsic("rok.clone.copy")]    // bitwise copy
    fn clone(self) -> i32;
}

impl<T: Clone> Clone for List<T> {
    fn clone(self) -> List<T> {
        let r: List<T> = [];
        for x in self {
            r.push(x.clone());
        }
        r
    }
}
```

Cloning a GC object always allocates a new buffer and clones the elements recursively.

### `#[derive(...)]`

For most user-defined structs whose fields are all `Clone`, there is no reason to write the body by hand. `#[derive(Clone)]` does it instead.

```rok
#[derive(Clone)]
struct Config {
    name: str;
    timeout: u32;
    retries: u32;
}

let c1 = Config { name: "default", timeout: 30, retries: 3 };
let c2 = c1.clone();         // every field cloned recursively
```

The derived body calls `Clone::clone` on every field and returns the results in the same structure. Every field must therefore implement `Clone`, recursively.

## 2.13.3. `Copy` - marker

```rok
trait Copy: Clone {
    // marker - no methods
}
```

In Rok every value is copied by value. For collections and GC objects the handle is copied and points to the same object; primitive values, `str` (an immutable `(ptr, len)`), structs, enums and tuples are copied themselves ([refs-lang 2.3.8](../../refs-lang/002-types/203-struct.md#238-value-semantics)). `Copy` is therefore **only a marker** and does not change the meaning of assignment or argument passing. `Copy` is used only in generic bounds (`T: Copy`) to select types for which "a bit copy is an independent copy".

### impl policy

| Type                             | `Clone`     | `Copy`             | Notes                                     |
|----------------------------------|-------------|--------------------|-------------------------------------------|
| `i*`, `u*`, `f*`                 | ✓           | ✓                  | primitive numbers                         |
| `bool`, `char`, `c_char`         | ✓           | ✓                  |                                           |
| `str`                            | ✓           | ✓                  | immutable `(ptr, len)` value              |
| `int`, `float`                   | ✓           | ✓                  | immutable value - a heap BigInt is only shared |
| `List<T>`, `Dict<K,V>`, `Set<T>` | ✓ (T:Clone) | ✗                  | copying the handle shares - not an independent copy |
| tuples                           | ✓ (all)     | ✓ (if all Copy)    | per component                             |
| `Range<T>`                       | ✓ (T:Clone) | ✓ (T:Copy)         | when both start/end are Copy              |
| user struct / enum               | derive      | derive             | when every field implements the same trait |

A type that implements `Drop` cannot be `Copy`.

```rok
#[derive(Clone, Copy)]
struct Vec3 { x: f64; y: f64; z: f64; }
```

> [!IMPORTANT]
> Deriving `Copy` is allowed only when `Clone` is implemented as well, and only when every field is `Copy`.

## 2.13.4. `Drop` - cleanup on `using` release

A file that was opened must be closed; a mutex that was locked must be unlocked. Such external resource cleanup cannot be left to the GC. Holding a resource until an unpredictable sweep could exhaust file descriptors or leave other code stalled on a lock that was never released. The `Drop` trait and the `using` expression solve this.

```rok
trait Drop {
    #[lang("drop_method")]
    fn drop(ref self);
}
```

### When called - only for `using` bindings

Rok is a GC language, so `drop` is not called automatically when an ordinary binding leaves scope. `drop` is called only for bindings bound with `using`, on leaving the block ([refs-lang 3.17 `using`](../../refs-lang/003-expressions/317-using-expression.md)).

- It is called on normal completion, `return`, `break` and error propagation alike.
- Multiple bindings are dropped in reverse declaration order (LIFO).
- A panic ends the process, so `drop` is not called.
- Drop glue: after the value's own `drop`, fields that have `Drop` are cleaned up recursively in reverse declaration order. The same applies to tuple elements and to the payload currently held by an enum / sum type.

```rok
fn process(path: str) throws (IoError) {
    using lock = MUTEX.lock(), file = try File::open(path) {
        do_work(file);
    }   // ← on normal exit or error: file.drop() then lock.drop()
}

{
    let f = try! File::open("data.txt");
}   // ← no automatic drop - the GC reclaims f but File::drop is not called
```

Binding a type that does not implement `Drop` with `using` is a compile error stating that `Drop` is required.

### Relation to the GC

The GC never calls `drop`, and there are no finalizers. A resource that needs cleanup must be bound with `using` or closed directly with a method such as `close()`; the `unclosed_resource` lint warns otherwise ([refs-lang 6.2.9](../../refs-lang/006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control)).

### `Drop` implementation example

The most typical implementation is a wrapper around an OS resource.

```rok
struct File {
    handle: i32;
}

impl Drop for File {
    fn drop(ref self) {
        rok::bare::close_fd(self.handle);     // OS file descriptor close
    }
}
```

### Restrictions on `Drop`

`Drop` comes with several restrictions. First, a type that implements `Drop` cannot be `Copy`: multiplying copies freely by bit copy while guaranteeing deterministic cleanup is a contradiction.

```rok
#[derive(Copy)]              // ERROR - conflicts with Drop
struct File { handle: i32; }
impl Drop for File { ... }
```

Second, calling `drop` directly is a compile error. Correctness is guaranteed only if it is called exactly once, at `using` release.

```rok
let f = File::open(path);
f.drop();                    // ERROR - calling Drop::drop directly is forbidden
```

To close a resource early, expose a separate method such as `File::close()`.

The only exception is unsafe code inside `rok::bare`. For example, `unsafe fn drop_in_place` ([1.3 mem](../100-bare/103-mem.md)) is a function for manual memory management. It runs the drop glue: the type's own `drop`, then its fields' in reverse declaration order, as above.

Also, `drop` has no `throws` signature, so its body cannot throw an error. A panic inside `drop` ends the process like any other panic.

```rok
impl Drop for X {
    fn drop(ref self) {
        if some_cond { throw "oops" }    // ERROR - drop is not throws
    }
}
```

### derive policy

Unlike `Clone`, `Drop` cannot be derived. After the `drop` of a value bound with `using` finishes, the compiler calls `drop` on that struct's `Drop` fields in reverse declaration order. A user-written `Drop` body therefore handles only "what needs cleaning up beyond the fields".

```rok
struct Container {
    a: File;
    b: Socket;
}
// when Container is bound with using, release does b.drop() → a.drop()
// (because the fields themselves implement Drop)

impl Drop for Container {
    fn drop(ref self) {
        log("Container released");
        // field drops are called automatically by the compiler _after this method returns_
    }
}
```

## 2.13.5. lang slot mapping

This module connects to the compiler through four lang slots. The automatic call points follow the policies described above.

| slot          | Attached to                  | Automatic call point                       |
|---------------|------------------------------|--------------------------------------------|
| `clone_trait` | `Clone` trait                | none - used in the `#[derive(Clone)]` expansion |
| `drop_method` | `Drop::drop` method          | release of a `using` binding (and drop glue) |
| `send_trait`  | `Send` trait (2.13.9)        | implemented automatically, thread-boundary checks |
| `sync_trait`  | `Sync` trait (2.13.9)        | implemented automatically, thread-boundary checks |

For details see sections 0.4.2 / 0.4.3 / 0.4.5 of the [0.4 lang slot table](../000-meta/004-lang-slot-table.md).

## 2.13.6. Trait implementation matrix

How the main standard library types relate to the three traits:

| Type                             | Clone       | Copy           | Drop                            | Behavior                             |
|----------------------------------|-------------|----------------|---------------------------------|--------------------------------------|
| `i*`, `u*`, `f*`                 | ✓           | ✓              | ✗                               | `rok.clone.copy` (bitwise)           |
| `bool`, `char`, `c_char`         | ✓           | ✓              | ✗                               | bitwise                              |
| `str`                            | ✓           | ✓              | ✗                               | copies the immutable `(ptr, len)` value |
| `List<T>`, `Dict<K,V>`, `Set<T>` | ✓ (T:Clone) | ✗              | ✗                               | recursive clone                      |
| tuples                           | ✓ (all)     | ✓ (all Copy)   | ✗                               | per component                        |
| `Range<T>`                       | ✓ (T:Clone) | ✓ (T:Copy)     | ✗                               | derived automatically                |
| `File`                           | ✗           | ✗              | ✓                               | `rok::bare::close_fd`                 |
| `MutexGuard<T>`                  | ✗           | ✗              | ✓ (on `using` release)          | `rok::bare::unlock`                   |
| `Channel::Sender<T>`             | ✓           | ✗              | ✗ (closed with `close()`)       | (same as above)                      |

The principle behind the table is simple. Every primitive and internal GC type implements `Clone`, and only those for which a bit copy is semantically correct are `Copy`. The collections are `Clone` but not `Copy`, because a bit copy of a handle does not preserve the meaning of a deep copy. `str` is `Copy`: it is immutable, so copying its `(ptr, len)` is an independent copy. `Drop` is implemented only by wrappers around external resources such as `File`, `Socket` and `MutexGuard`. These types usually do not implement `Clone`, to prevent duplicate ownership of the resource.

## 2.13.7. Comparison with other languages

Comparing Rok's model with existing languages makes the design intent clearer.

| Aspect                  | Rust                     | Java/Python (GC)                | Rok (GC + Drop)          |
|-------------------------|--------------------------|---------------------------------|--------------------------|
| Memory reclamation      | `Drop` is responsible    | GC                              | GC                       |
| Deterministic finalize  | `Drop` (compiler-guaranteed) | **none** (finalizers nondeterministic) | `Drop` (scope-bound) |
| Meaning of Copy         | bitwise + `!Drop` enforced | **none** (every object is a ref) | bitwise + `!Drop` enforced |
| Meaning of Clone        | deep copy                | `clone()` (Object.clone)        | deep copy                |
| derive                  | `#[derive(Clone, Copy)]` | `Cloneable` interface           | `#[derive(Clone, Copy)]` |

What sets Rok apart is the coexistence of a GC with a deterministic `Drop`. Because the GC handles memory reclamation, `Drop` does not mean memory deallocation, so ordinary objects never need to implement `Drop`. Only types that need external resource cleanup implement `Drop`, and users bind them with `using`.

This model avoids both the nondeterminism of Java/Python finalizers and the costs Rust incurs from having no GC.

## 2.13.8. `NoneSentinel` - the type of the `none` literal

The `none` literal of [refs-lang 1.9.2](../../refs-lang/001-lexical/109-other-literals.md) has the compiler's built-in `none` type, a zero-sized type with a single value. `rok::core::marker` gives that type its library name `NoneSentinel` with a type alias; no lang slot is involved.

```rok
pub type NoneSentinel = none;
```

Users rarely write this name directly. In sum type notation such as `T | none` ([refs-lang 2.6](../../refs-lang/002-types/206-sum-types.md)), the `none` position is this type; the `none` literal, the `none` type notation and `NoneSentinel` all name the same built-in type.

## 2.13.9. `Send` / `Sync` - thread-boundary markers

```rok
#[lang("send_trait")]
trait Send { }      // safe to move to another thread

#[lang("sync_trait")]
trait Sync { }      // safe to read from multiple threads through shared references
```

Both traits are **automatic marker traits** with no methods; the compiler implements them structurally.

- Primitive numbers, `bool`, `char`, `c_char`, `str` and `none` are `Send + Sync`.
- A `struct`, `enum`, `union`, tuple, array or sum type is `Send` if all its fields (components) are `Send`, and `Sync` if all are `Sync`. `Send` for a generic container follows `T` by the same rule.
- Collections (`List`, `Dict`, `Set`, `StrBuf`, `Deque`, `BinaryHeap`, `OrderedDict`, `OrderedSet`, `Slice`) are `Send` and `!Sync`: they share a mutable handle. Across threads they are shared through `Mutex` / `RwLock` / channels.
- Rok has no move semantics. Handing a value over (`Mutex::new(xs)`, `RwLock::new(xs)`, `tx.send(xs)` / `try_send` / `send_async`, `Handoff::new(xs)`) leaves the sender's handle pointing at the object the new owner now shares with other threads. A handle taken out of a guard (`get`), given to it (`set`) or reached by `update`'s closure still points at the guarded object after the guard's `using` block ends. Using such a value afterwards, on any path, is a compile error (`E202`, [refs-lang 9.9.4](../../refs-lang/009-tools/909-debug-tools.md#994-diagnostics--error-messages)). The value is followed through:
  - copies (`let ys = xs`, a struct field or tuple element, a field assignment);
  - the fields it is passed by (`Mutex::new(s.items)`);
  - the objects that hold it (`all.push(xs)`, a closure that captures it), both ways: handing over `xs` leaves `all` unusable too;
  - the elements of a collection. What is stored into a collection and what is read out of it (`xs[i]`, `get`, iteration, a slice of it) are one value, so handing over either leaves the other unusable (`let x = xs[0]; tx.send(xs); x.len()`, `tx.send(xs[0]); xs[1].len()`). Only `len`, `is_empty`, `capacity`, `clear`, `truncate`, `reserve`, `shrink_to_fit`, `push` and `insert` (and `Deque`'s `push_back` / `push_front`) stay allowed on a collection whose elements were handed over; `clear` empties it, so it is fully usable again. A collection copied from another (`extend`, `+=`, `collect`, `par::filter`) shares the source's elements. An element taken out with `pop` or `remove` (`List`, `Deque`, `Dict`, `OrderedDict`, `BinaryHeap`) has left the collection: handing it over leaves the collection usable, unless the collection may hold that object twice - one value stored twice, two stored values that share an object, or elements of one collection stored into another in a loop (elements are not told apart by position);
  - the arguments of a direct call that hands something over: passing one object through two arguments, when the callee hands it over through one and uses it through the other (`f(tx, a, a)`, `h.go(tx, h.items)` where `go` uses `self.items`, a list it pops from and sends plus one of its elements), is a compile error. Two elements of one collection are taken to be distinct (`f(tx, rows[0], rows[1])` is accepted);
  - a `ref` parameter. When the callee hands a part of it over and then gives that part a new value on every path that hands it over (`self.buf = []` after `tx.send(self.buf)`), the caller's variable gets the new value back and stays usable; other handles to the old value do not. A by-value parameter is a copy: giving its field a new value does not reach the caller, which still holds what was handed over;
  - calls. A function that hands a parameter over, returns it or a part of it, stores it into another parameter's object or a `ref` parameter, captures it into the closure it returns, or returns what a guard parameter holds, does so for its caller too, also across a prebuilt library ([refs-lang 9.4.7](../../refs-lang/009-tools/904-rokhir-bytecode.md#947-body)).

  A call through a function value is taken to reach every function used as a value with its parameter types. Conditions are followed where they test a value the compiler knows on each path: a `bool` or a sum or enum variant set by a constant or copied (`let sent = false; if c { tx.send(xs); sent = true; } if !sent { xs.len() }`), and how a call ended. A function that hands a parameter over only when it returns without an error, or a value other than `none` / `false`, does not hand it over on its caller's `catch`, `try?`, `??`, `else` or `if` path for the other outcome; the same holds for the `catch` of `try await` on an `async fn`. Other conditions are not followed: a value handed over on one path is gone where that path joins others.

  To keep using a value, hand over a `clone()` (an independent value), or give the variable or field a new value first (for a struct copy, only that field); a `set` releases what an earlier `get` returned. Uses of `Sync` values, reading a value's tag (`x is none`, `match x`), and uses inside `unsafe` ([refs-lang 4.15.2](../../refs-lang/004-functions/415-unsafe.md#4152-unsafe------block-expression)) are exempt. A `Handoff` protocol ends inside the function that makes it: its callers do not see that hand-off. Captures of a spawned closure need no check: they must be `Send + Sync`. `task::spawn` hands its future over: the task's frame gets copies of what the future captured, and the future and what it captured are gone for the spawner, as after a send ([2.18.1](218-task.md#2181-core-api)).

  Also compile errors: handing over a value read out of a `Sync` object that is not itself `Sync` (a field of a user's `unsafe impl Sync` type, a `thread::local`'s value), which other code reaches without the hand-off - hand over a `clone()`; values a channel's `recv`, a task's result or a lock's guard give out are the receiver's and are not covered by this rule; handing over what a guard's `get` returned while the guard still holds it (until `set` gives the lock a new value); returning it from inside the `using` block, yielding it there (`yield g.get()` in a generator), making it the block's own value (`let x = using g = m.lock() { g.get() }`, `f(using g = m.lock() { g.get() })`), or giving it to a coroutine made inside the block and run after it (an `async fn`'s future, a generator: `fut = grow(g.get()[0])`): the block's end releases the lock before the value is used. A guard is good only inside its `using` block: a copy of it (pushed into a collection, captured by a closure, stored by a callee, bound by a second `using`, held by a coroutine: an `async fn`'s or generator's argument, an `async` block's capture) used after that block ends is a compile error; guards are `!Sync` ([2.16.3](216-sync.md#2163-mutext---exclusive-lock)). `par::for_each`, `par::map` and `par::filter` give each item to a worker, so two items must not reach one collection that is not `Sync`. Each call checks this at run time ([2.19.1](219-par.md#2191-core-api)): if two items reach one collection, the items run one at a time on the calling thread, with a warning once per call site. Slices only read, so items may share a list through slices of it, as long as no item holds the list itself.

  What remains outside these rules: code inside `unsafe`; a call through a function value or a closure, which skips the argument check above; and a generator that yields one object more than once, collected into a collection whose elements are then handed out one by one - such a collection is not seen as holding the object twice.

  At run time every hand-off also copies, so on the routes above leave out (and in code inside `unsafe`) no collection the handed value reaches by value ends up shared between the sender and the new owner (the exceptions are listed below). `Mutex::new`, `RwLock::new`, a guard's `set` and `update`, `send` / `try_send` / `send_async`, and the result of a task, a thread or a `par` scope job (and a `try_scope` error) give the new owner a copy of every collection the value reaches: through the parts of structs, tuples, arrays, enums and sums and through the elements of collections, to any depth, with a work list rather than stack per level. A collection reached more than once is copied once, so the copy keeps the value's sharing and its cycles. Copied are `List`, `Dict`, `Deque`, `StrBuf`, `Slice` and the collections built on them, and `GcPtr` ([1.6.3](../100-bare/106-gc-pin.md)). A coroutine future (`AsyncFuture`, `TryAsyncFuture`) gets a frame of its own, holding copies of what it captured made in the same copy; one that has started or finished, or whose frame (or a captured future's) has a `[ref x]` capture, cannot be handed over, and the hand-off panics ([2.18.1](218-task.md#2181-core-api)). A `Task` is not copied, but once it is handed over each `await` of it gets its own copy of the result ([2.18.2](218-task.md#2182-taskt-type)). A list and every view of its buffer keep sharing one copied buffer; a view of a `str`'s bytes copies only those bytes (they cannot change). `update` keeps what the guard already holds: only what the closure brings in is copied. The sender's handles are left as they were: one the rules above missed names an object no other thread uses. A send that throws carries back an equal value (the caller's or its copy). The copy takes time and memory in proportion to what it copies; a run of plain values (no GC reference) is one memory copy. `rok::core::marker::hand_over(x)` returns the same copy. Left shared: `Sync` parts, values of types with an `unsafe impl Send` (they manage their own sharing: tasks, files, sockets), and generators and streams (not `Send`: they are never handed over). `Handoff::new` and `Mutex::new_unshared` (2.16.3) do not copy. A handle taken out of a guard (`get`, `update`'s closure argument) is not copied at run time: the rules above follow it, and the guard itself, out of the `using` block through every copy, a `return`, a `yield`, the block's value and the frames of coroutines made inside it.
- The following are excluded: `Cell<T>`, raw pointers (`Ptr<T>`, `ConstPtr<T>`, `CPtr<T>`), bare `any` and `any Tr` whose bound lacks `Send` / `Sync` (an `any Tr` is `Send` only when its bound has both `Send` and `Sync`: copies of an `any` share its box, as a closure's captures are shared, so no collection may sit inside one that crosses threads), function types without a bound, and closures with a `[ref]` capture ([refs-lang 4.6.9](../../refs-lang/004-functions/406-capture-semantics.md#469-additional-constraints-in-parallel-and-concurrent-contexts)). Other exceptions are specified in each type's chapter.
- A function type has these traits only when the bound is written, as in `fn(..) -> R + Send` ([refs-lang 4.4](../../refs-lang/004-functions/404-function-type.md#send--sync-bounds)).

Types for which the automatic implementation is wrong declare it directly. A type that holds a raw pointer internally but wraps it thread-safely opts in with `unsafe impl` (safety is the author's responsibility); a type that is structurally auto-implemented but not thread-safe opts out with `impl !Send`. The exclusion of `Cell<T>` is declared this way.

```rok
struct Handle { raw: Ptr<u8>; }
unsafe impl Send for Handle { }
unsafe impl Sync for Handle { }

impl<T> !Send for Cell<T> { }
impl<T> !Sync for Cell<T> { }
impl<T> !Send for Ptr<T> { }
impl<T> !Sync for Ptr<T> { }
impl<T> !Send for CPtr<T> { }
impl<T> !Sync for CPtr<T> { }
```

APIs that pass values to threads or tasks (`thread::spawn`, `task::spawn`, `par::*`, `channel`) constrain their arguments with these two traits. The policy for synchronization primitives follows [2.16 sync](216-sync.md).

### `Handoff<T>` and the hand-off machinery

```rok
pub struct Handoff<T> { value: T; }

impl<T> Handoff<T> {
    pub const unsafe fn new(value: T) -> Handoff<T>;
    pub const fn get(self) -> T;
}

unsafe impl<T: Send> Send for Handoff<T> {}
unsafe impl<T: Send> Sync for Handoff<T> {}

pub fn hand_over<T>(x: T) -> T;
```

`Handoff<T>` carries a value between threads under an outside protocol (a join, a lock, a channel hand-off) that keeps any two threads from using it at once. It is `Send + Sync` whenever `T` is `Send`, even when `T` is not `Sync` (a collection). `new` is `unsafe`: the caller guarantees that no two threads use the value, or another copy of its handle, at the same time. `new` makes no copy (see above); `get` returns the wrapped value.

`hand_over(x)` returns what a hand-off gives the new owner: `x` with a copy of every collection it reaches, or `x` itself when it reaches none.

`Transfer` and `Memo` are public only so that the hand-off copy can reach the standard collections in other modules.

```rok
pub trait Transfer {
    fn transfer_copy(self, m: Memo) -> Self;
    fn transfer_keep(self, m: Memo) -> Self;
    fn transfer_visit(self, m: Memo) -> Self;
}

pub struct Memo { /* private fields */ }
```

The compiler calls `Transfer` only for its fixed list of standard handle types; an impl on any other type is ignored. `Memo` records the copies one hand-off has made, by original object, and the copied buffers still to fix. Its methods and the `transfer*` intrinsics beside it are `pub(pkg)`: they are internal to the library and not a stable API.

## 2.13.10. `Default` - default values

```rok
trait Default {
    fn default() -> Self;
}
```

| Type                                 | `default()`        |
|--------------------------------------|--------------------|
| integers, `int`                      | `0`                |
| floating-point, `float`              | `0.0`              |
| `bool`                               | `false`            |
| `char`                               | `'\0'`             |
| `str`                                | `""`               |
| collections such as `List`, `Dict`, `Set` | empty collection |
| `T \| none`                          | `none`             |

`Default` can be derived; it builds the value from every field's `default()`.

## 2.13.11. Derivable traits

The built-in traits that `#[derive(...)]` ([refs-lang 6.2.1](../../refs-lang/006-meta-system/602-built-in-attributes.md#621-derive-automatic-trait-implementation)) can produce are only these:

| Trait         | Generated implementation                      | Requirement                            |
|---------------|-----------------------------------------------|----------------------------------------|
| `Clone`       | per-field `clone`                             | every field `Clone`                    |
| `Copy`        | marker                                        | `Clone` also derived/implemented, not `Drop` |
| `Debug`       | field names and values in `f"{x:?}"` form     | every field `Debug`                    |
| `Default`     | per-field `default()`                         | every field `Default`                  |
| `PartialEq`   | AND of per-field `==`                         | every field `PartialEq`                |
| `Eq`          | marker                                        | `PartialEq` also derived/implemented   |
| `PartialOrd`  | lexicographic comparison in field declaration order | every field `PartialOrd`, `PartialEq` |
| `Ord`         | lexicographic comparison in field declaration order | `Eq`, `PartialOrd` also derived/implemented |
| `Hash`        | combination of per-field hashes (combining function unspecified) | every field `Hash` |

- `Display` cannot be derived ([2.10](210-fmt.md)).
- Deriving `Eq` / `Ord` without its supertrait is a compile error.
- Applying `#[derive]` to a `union` is a compile error.
