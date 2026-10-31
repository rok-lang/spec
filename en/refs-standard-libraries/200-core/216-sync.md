# 2.16. sync - Concurrency primitives

The `rok::core::sync` module defines the primitive traits and types for safe access to shared state. They are used to share data between multiple tasks or threads under Rok's GC and M:N scheduler.

Because `use rok::core::**;` applies implicitly, `Atomic`, `Mutex`, `RwLock` and `Cell` are usable without any import. Where concurrent code should be visibly marked as such, a narrow import like the following is recommended.

```rok
use rok::core::sync::{Atomic, Mutex, RwLock, Once, Condvar, Cell};
```

## 2.16.1. Overview of the four primitives

Each of the four primitives covers a different concurrency scenario. The following table summarizes each type's meaning and where it fits.

| Type        | Meaning                                    | Where used                     |
|-------------|--------------------------------------------|--------------------------------|
| `Atomic<T>` | lock-free atomic operations (T = primitive) | counter, flag, sequence number |
| `Mutex<T>`  | exclusive lock (single writer)             | shared mutable state           |
| `RwLock<T>` | multiple readers or a single writer        | read-heavy state               |
| `Cell<T>`   | thread-local interior mutability (no lock) | single-threaded mutable        |

## 2.16.2. `Atomic<T>` - lock-free atomics

`Atomic<T>` is the lightest primitive. Internally it is implemented with per-T LLVM atomic operations and has the following shape.

```rok
struct Atomic<T> {
    // internal: a one-element cell driven by per-T LLVM atomic ops
}

trait AtomicPrimitive { /* marker: i8/i16/i32/i64/isize, u8/u16/u32/u64/usize, bool, Ptr<T> */ }

impl<T: AtomicPrimitive + PartialEq> Atomic<T> {
    const fn new(value: T) -> Atomic<T>;

    fn load(self, ordering: AtomicOrdering = SeqCst) -> T;
    fn store(self, value: T, ordering: AtomicOrdering = SeqCst);
    fn swap(self, value: T, ordering: AtomicOrdering = SeqCst) -> T;
    fn compare_exchange(
        self, current: T, new: T,
        success: AtomicOrdering, failure: AtomicOrdering,
    ) -> Result<T, T>;                  // the failure result is a value too, not an error
    fn compare_exchange_weak(
        self, current: T, new: T,
        success: AtomicOrdering, failure: AtomicOrdering,
    ) -> Result<T, T>;                  // the failure result is a value too, not an error

    // integers only
    fn fetch_add(self, value: T, ordering: AtomicOrdering = SeqCst) -> T;
    fn fetch_sub(self, value: T, ordering: AtomicOrdering = SeqCst) -> T;
    fn fetch_and(self, value: T, ordering: AtomicOrdering = SeqCst) -> T;
    fn fetch_or(self, value: T, ordering: AtomicOrdering = SeqCst) -> T;
    fn fetch_xor(self, value: T, ordering: AtomicOrdering = SeqCst) -> T;
}

impl Atomic<i32> {
    fn wait(self, expected: i32, timeout_ns: i64 = -1);   // waits while the value is expected, up to timeout_ns (< 0 means forever)
    fn wait_for(self, expected: i32, timeout: Duration);   // wait, the limit a Duration (negative: forever)
    fn wake(self);                                         // wakes every thread waiting on this cell
}
```

`wait` may return spuriously, so the caller re-checks the condition (futex semantics).

`load` and the `failure` ordering of `compare_exchange` / `compare_exchange_weak` are load orderings: passing `Release` or `AcqRel` there panics. `compare_exchange_weak` never fails spuriously in this implementation; it behaves as `compare_exchange`. `PartialEq` is required because `compare_exchange` compares the old value with `current`; `Ptr<T>` compares by address.

### `AtomicOrdering`

Atomic operations come in five memory-ordering strengths, with the following meanings.

```rok
enum AtomicOrdering {
    Relaxed,        // no synchronization (counters etc.)
    Acquire,        // synchronization guaranteed after the load
    Release,        // synchronization guaranteed before the store
    AcqRel,         // both directions for RMW
    SeqCst,         // global sequential consistency (default, safe)
}
```

This model is isomorphic to the atomic ordering model of C++ and Rust, and the default is the safe `SeqCst`. The name `AtomicOrdering` avoids a clash with the comparison `cmp::Ordering` under the implicit core glob; the variants (`Relaxed`, `Acquire`, `Release`, `AcqRel`, `SeqCst`) can be written without a prefix. The ordering argument must be a compile-time constant. Weak orderings should be used carefully and only where performance is critical.

### Example

The most common use, incrementing a counter, is written as follows.

```rok
static COUNTER: Atomic<i32> = Atomic::new(0);

task::spawn(async { COUNTER.fetch_add(1, Relaxed); });
task::spawn(async { COUNTER.fetch_add(1, Relaxed); });
// result: COUNTER == 2 (no race)

let v = COUNTER.load(SeqCst);     //→ 2
```

### CAS (compare-and-swap)

Updates that atomics do not offer directly are built on compare-and-swap loops. The following keeps the largest value ever stored.

```rok
fn store_max(cell: Atomic<i64>, x: i64) {
    let cur = cell.load(Acquire);
    loop {
        if cur >= x { break; }
        match cell.compare_exchange(cur, x, AcqRel, Acquire) {
            Ok(_)       => break,
            Err(actual) => cur = actual,          // another task got there first; retry with its value
        }
    }
}
```

The element type must be an `AtomicPrimitive`: a collection such as `List<T>` cannot be held in an `Atomic`. Shared collections go in a `Mutex<T>` (2.16.3).

## 2.16.3. `Mutex<T>` - exclusive lock

Shared state that is hard to express with atomics alone uses `Mutex<T>`. Internally it consists of a futex and the payload; a guard bound with `using` releases the lock.

```rok
struct Mutex<T> {
    // internal: futex + payload
}

impl<T> Mutex<T> {
    fn new(value: T) -> Mutex<T>;
    unsafe fn new_unshared(value: T) -> Mutex<T>;   // new without the hand-off copy

    fn lock(self) -> MutexGuard<T>;            // blocks the thread
    fn lock_async(self) -> LockFuture<T>;      // await m.lock_async(): suspends only the task (Future<Output = MutexGuard<T>>)
    fn try_lock(self) -> MutexGuard<T> | none; // non-blocking
}

struct MutexGuard<T> { /* RAII guard */ }

impl<T> Drop for MutexGuard<T> {
    fn drop(ref self) {
        // futex_wake + release the lock
    }
}

impl<T> MutexGuard<T> {
    fn get(self) -> T;                  // the protected value
    fn set(self, value: T);             // replaces the value
    fn update(self, f: fn(T) -> T);     // reads, transforms and writes back
}
```

A guard does not forward the methods of the value it protects: `g.push(1)` on a `MutexGuard<List<i32>>` is an error. Reach the value through `get`, `set` and `update`.

### Example

A guard bound with `using` is `drop`ped on leaving the block, releasing the lock. A guard received without `using` is not released automatically, so it must always be bound with `using`. A guard is received from a call, a method call or `await`. A received guard may only initialize a binding (`let`, a pattern, `using`), be assigned to a variable, or be handed to the caller (`return`, the function's value). Passing it to a function, calling a method on it, storing it in a field or a collection, or discarding it is a compile error. A guard bound with `using` may be passed to functions; a guard bound without `using` is not released by passing it on. Binding a guard with a plain `let` (or `if let`) that is never released by `using`, or discarding a guard returned by `lock`, is a compile error.

`Mutex::new(value)` takes `value` over: using the caller's handle afterwards is a compile error, and so is using what `get` returned or `set` gave after the guard's `using` block ends, handing over what `get` returned while the guard holds it (until `set` gives the lock a new value), or returning it, yielding it, making it the block's value or giving it to a coroutine run after the block ([2.13.9](213-marker.md#2139-send--sync---thread-boundary-markers)). Work on it inside the `using` block, or keep a `clone()`. A guard is `!Sync` and good only inside its `using` block: using a copy of it (in a collection, a closure, a field) after the block ends is a compile error; to use the value in a closure that runs on another thread, read it first (`let v = g.get();`) and capture `v`. At run time `new`, `set` and `update` store a copy of the collections the value brings in (2.13.9); `new_unshared` skips the copy, the caller guaranteeing that nothing else reaches `value`'s collections (a value just made).

```rok
static STATE: Mutex<List<i32>> = Mutex::new([]);

task::spawn(async {
    using g = STATE.lock() {
        g.update(|xs| { xs.push(42); xs });
    }   // ← g.drop() → lock released ([2.13 marker](213-marker.md))
});
```

### Avoiding deadlock - `try_lock`

To avoid deadlock, a non-blocking attempt can be made with `try_lock` instead of blocking.

```rok
match STATE.try_lock() {
    v: MutexGuard<List<i32>> => using g = v { g.update(|xs| { xs.push(1); xs }); },
    none => { log("locked, skip"); },
}
```

### Poisoning - **none**

Rok's Mutex differs from Rust's PoisonError model. Even if a panic occurs, the lock is released automatically and the next task can enter normally. There are two reasons for this design.

1. A panic cannot be caught and ends the process, so no other task ever observes the lock state mid-panic. When an error propagates via `throw` out of a `using` block, [`using` calls Drop](213-marker.md#2134-drop---cleanup-on-using-release), so the `MutexGuard`'s drop is responsible for releasing the lock.
2. Poisoning looks conservative but burdens users heavily and is a false positive in most cases.

Where consistency really must be restored before leaving with an error, handle it explicitly:

```rok
using g = MUTEX.lock() {
    try unsafe_op(g) catch e {
        g.set(known_good_state());
        throw e;
    };
}
```

## 2.16.4. `RwLock<T>` - read-write split

Where reads vastly outnumber writes, `RwLock<T>` fits better. Separating multiple readers from a single writer raises throughput.

```rok
struct RwLock<T> { /* ... */ }

impl<T> RwLock<T> {
    fn new(value: T) -> RwLock<T>;

    fn read(self) -> RwLockReadGuard<T>;        // multiple readers OK
    fn write(self) -> RwLockWriteGuard<T>;      // single writer

    fn try_read(self) -> RwLockReadGuard<T> | none;
    fn try_write(self) -> RwLockWriteGuard<T> | none;

    fn read_async(self) -> ReadFuture<T>;       // await l.read_async(): suspends only the task
    fn write_async(self) -> WriteFuture<T>;
}
```

It suits read-dominated state such as config or caches, and for workloads with rare writes its throughput exceeds that of Mutex.

```rok
static CONFIG: RwLock<Config> = RwLock::new(Config::default());

using c = CONFIG.read() {        // concurrent with other readers OK
    process(c);
}   // ← read guard drop, lock released

using w = CONFIG.write() {       // briefly blocks all readers
    w.set(Config::load_from_file());
}
```

`RwLock<T>` is `Sync` only when `T` is `Send + Sync` ([2.16.5](#send--sync-markers)), so `Config` here must be `Sync`.

As for `Mutex`, `RwLock::new(value)` takes `value` over: using the caller's handle afterwards is a compile error, and so is using what a guard's `get` returned or `set` gave after its `using` block ends, handing over what `get` returned while the guard holds it (until `set` gives the lock a new value), or returning it from inside the block ([2.13.9](213-marker.md#2139-send--sync---thread-boundary-markers)). Work on it inside the `using` block, or keep a `clone()`. As for `Mutex`, `new` and the write guard's `set` and `update` store a copy.

The same rule holds for `RwLock` guards: binding the guard of `read` / `write` with a plain `let` (or `if let`) that `using` never releases, or discarding it, is a compile error.

## 2.16.4a. `Once` / `Condvar`

Two primitives shaped as in Rust. There is no poisoning.

```rok
struct Once { /* ... */ }
impl Once {
    const fn new() -> Once;
    fn call_once(self, f: fn());          // runs f only the first time; concurrent callers wait until it finishes
    fn is_completed(self) -> bool;        // true once a call_once has finished
}

struct Condvar { /* ... */ }
impl Condvar {
    const fn new() -> Condvar;
    fn wait<T>(self, guard: MutexGuard<T>) -> MutexGuard<T>;   // releases the lock and waits, relocks on wakeup
    fn wait_async<T>(self, guard: MutexGuard<T>) -> CondvarWait<T>;   // g = await cv.wait_async(g): suspends only the task
    fn notify_one(self);
    fn notify_all(self);
}
```

`wait` can wake spuriously, so the condition is re-checked in a loop. `notify_one` wakes at least one waiter, and may wake more than one async waiter.

### Async locks and cancellation

The futures of `lock_async` / `read_async` / `write_async` / `wait_async` do not take the lock on a cancelled poll ([2.18.4](218-task.md#2184-cooperative-cancellation---taskcancel)). A cancelled `wait_async` ends as follows.

- If the mutex is free, it re-acquires it, so the caller's `using` guard releases it as usual.
- If another task holds it, it leaves without acquiring, and the discarded guard does not release that mutex.

## 2.16.4b. `Frozen<T>` - read-only sharing

A value made once and only read after (configuration, trust anchors, tables) is shared through `Frozen<T>`. Any thread may read it without a lock.

```rok
struct Frozen<T> { /* ... */ }   // Sync when T: Send

impl<T> Frozen<T> {
    const fn new(value: T) -> Frozen<T>;   // keeps a copy no one else reaches
    fn get(self) -> T;                       // the reader's own copy
}
```

`new` keeps a copy of the collections `value` reaches, as a hand-off does ([2.13.9](213-marker.md#2139-send--sync---thread-boundary-markers)), so nothing outside reaches what it holds. `get` gives each reader a copy of its own: changing it changes nothing the other readers see. `Sync` parts (a `Mutex`, a cache that locks itself, a value of an immutable `Sync` type) are not copied and stay shared, so a `get` costs a copy of the value's other collections only. Keep big read-only parts out of that copy: hold them as a `Sync` type, or in a `Frozen` field of their own that is read where needed.

```rok
struct Settings {
    name: str;
    limits: List<i64>;               // copied by each get: small
    table: Frozen<List<Entry>>;      // shared: read with table.get() where needed
}

static SETTINGS: Frozen<Settings> = Frozen::new(load());
```

The standard library's TLS `Certificate` is such a type: never changed once parsed (its accessors give copies), so a `Frozen<ClientConfig>`'s `get` shares its trust anchors.

## 2.16.5. `Cell<T>` - single-threaded interior mutability

Finally, `Cell<T>` provides interior mutability that works only within a single task.

```rok
struct Cell<T> { /* ... */ }

impl<T> Cell<T> {
    fn new(value: T) -> Cell<T>;
    fn get(self) -> T            where T: Copy;
    fn set(ref self, value: T);
    fn replace(ref self, value: T) -> T;
    fn take(ref self) -> T       where T: Default;
}
```

`Cell<T>` means thread-local interior mutability, so it is not safe across threads. It can be shared only within a single task, and having no lock makes it very fast.

```rok
let c: Cell<i32> = Cell::new(0);
c.set(42);
let v = c.get();          //→ 42

// sharing across tasks: compile error (Cell has no Send / Sync)
task::spawn(async { c.set(100) });       // ERROR: Cell is not cross-task safe
```

### `Send` / `Sync` markers

`Send` / `Sync` are the automatic marker traits of [2.13.9](213-marker.md#2139-send--sync---thread-boundary-markers). The types in this chapter follow this policy:

- `Atomic<T>`: Send + Sync, `Atomic<Ptr<T>>` included (the cell holds an address)
- `Mutex<T>`: Send + Sync when `T: Send`
- `RwLock<T>`: Send when `T: Send`; Sync when `T: Send + Sync`. Its readers share the value at once, so an `RwLock` of a collection (`!Sync`) cannot be shared between threads; use `Mutex<T>`.
- `MutexGuard<T>`, `RwLockReadGuard<T>`, `RwLockWriteGuard<T>`: Send, `!Sync`. A guard is good only inside its `using` block
- `Frozen<T>`: Sync when `T: Send` (2.16.4b)
- `Cell<T>`: neither (single-task only). The `task::spawn` above is an error because the `async` block captures the `Cell` and is therefore not `Send`

## 2.16.6. Module mapping

The implementation of each primitive maps as follows.

| Type         | Module                    | Internal structure               |
|--------------|---------------------------|----------------------------------|
| `Atomic<T>`  | `rok::core::sync::atomic` | LLVM atomic ops (`rok.atomic.*`) |
| `Mutex<T>`   | `rok::core::sync::mutex`  | futex / wait queue (per target)  |
| `RwLock<T>`  | `rok::core::sync::rwlock` | futex + reader counter           |
| `Cell<T>`    | `rok::core::sync::cell`   | plain wrapper, no lock           |
| `MutexGuard` | `rok::core::sync::mutex`  | RAII via Drop                    |
