# 5.3. `static`

The `static` keyword (a contextual keyword that is a keyword only at the start of an item; the following `mut` is likewise a keyword only in the `static mut` position, [1.5.4](../001-lexical/105-keywords.md#154-contextual-keywords)) declares global memory that persists for the lifetime of the program.

```
static_decl     ::= "static" "mut"? identifier ":" type_expr "=" const_expr ";"
extern_static   ::= "static" "mut"? identifier ":" type_expr ";"   // in extern block
```

## 5.3.1. Basic form

```rok
static SHARED_COUNTER: Atomic<i32> = Atomic::new(0i32);
static DEFAULT_CONFIG: Config = Config::default();
static APP_NAME: str = "rok";
```

The basic form is `static`, a name, `:` with a type, and `=` with an initializer. The type must be written, and the initializer must be a `const` expression in the sense of [`const`](502-const.md), so it must be evaluable at compile time.

A `static` is reachable from every thread, so its type must be `Sync` ([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync--thread-boundary-markers)); this holds for lazily initialized statics too. `static XS: List<i32> = [];` is a compile error (collections, `StrBuf` and `Cell<T>` are `!Sync`); write `static XS: Mutex<List<i32>> = Mutex::new([]);`. Per-thread state uses `thread::local` ([stdlib 3.4.4](../../refs-standard-libraries/300-std/304-thread.md#344-thread-local-storage)), whose `ThreadLocal<T>` is `Sync`. Statics of an `extern` block ([5.3.10](#5310-extern-static-ffi)) are exempt.

## 5.3.2. Difference from `const`

```
const X: i32 = 42i32;              // inlined at each use (no address)
static X: i32 = 42i32;             // stable memory address (program lifetime)
```

| Aspect         | `const`        | `static`                                       |
|----------------|----------------|------------------------------------------------|
| Memory address | None           | Stable address                                 |
| At use site    | Inlined        | Memory reference                               |
| Lifetime       | Compile time   | Program lifetime                               |
| Mutability     | Always immutable | Immutable by default (mutability via a wrapper) |

## 5.3.3. Permitted positions

`static` may be declared at module level or in a block ([3.9.1](../003-expressions/309-block-expressions.md#391-form)), not in a trait or an `impl` ([5.3.8](#538-combination-with-methods--traits)).

```rok
static APP_NAME: str = "rok";                          // module level — OK

fn f() -> i32 {
    static CALLS: Atomic<i32> = Atomic::new(0i32);     // OK — one slot, visible in this block
    CALLS.fetch_add(1i32)
}
```

A `static` in a block is one slot for the whole program, initialized once, whatever instance of its function runs.

## 5.3.4. Mutable globals in ordinary user code: use a wrapper

```rok
static COUNTER: Atomic<i32> = Atomic::new(0i32);
static CACHE: Mutex<Dict<str, Data>> = Mutex::new(Dict::new());
```

A global that needs interior mutability is declared wrapped in a concurrency-safe wrapper.

- `Atomic<T>`: lock-free atomic operations (integers, `bool`, pointers).
- `Mutex<T>`: lock-based protection.
- `RwLock<T>`: read-write lock, for a `T` that is itself `Sync` ([stdlib 2.16.4](../../refs-standard-libraries/200-core/216-sync.md#2164-rwlockt--read-write-split)).

`Cell<T>` is single-threaded and `!Sync`, so it cannot be a `static`; a per-thread mutable global is `thread::local(..)`.

All are standard library types; their exact signatures are defined in the library specification.

## 5.3.5. `static mut`: [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) only

Only for interfacing directly with the C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface), `static mut` may be used, and only inside an `extern "C"` block.

```rok
extern "C" {
    static errno: i32;             // FFI global (read-only)
    static mut atexit_count: u64;  // FFI mutable global
}
```

- `static mut` is allowed only inside an `extern "C"` block.
- `static mut` in ordinary rok code is an ERROR.
- Assigning to a `static` that is not an extern `static mut` is a compile error (`COUNTER = ...`). Interior mutation goes through the methods of the wrappers above.

Shared mutable state is a direct cause of data races, hence this restriction. Ordinary code gets safety through wrappers; raw access is allowed only in the FFI domain.

## 5.3.6. Initialization order

Initialization order among multiple `static` declarations:

- Within the same module, declaration order.
- Across modules, dependency order: if one `static`'s initializer references another `static`, the latter is initialized first.
- A circular dependency is a compile error.

The ordering check for `const` initializers is performed at compile time. Non-`const` initializers are initialized at first access. Initialization is thread-safe; the initializer runs exactly once, and other threads accessing concurrently wait until it finishes. An initializer that reads its own `static` panics.

## 5.3.7. Visibility

Like other items, a `static` may carry a visibility keyword.

```rok
pub static VERSION: str = "1.0.0";
pub(pkg) static INTERNAL: i32 = 42i32;
static _local: bool = false;
```

The detailed visibility rules are defined in [visibility](506-visibility.md).

## 5.3.8. Combination with methods / traits

A `static` is a slot holding a value, not an item definer. It therefore does not appear in trait or `impl` block definitions; `const` or associated functions go there instead.

```rok
impl Counter {
    static GLOBAL: i32 = 0i32;     // ERROR — no static inside impl
    const ZERO: i32 = 0i32;        // OK — associated const
}
```

## 5.3.9. `const` restriction on initializers

If the initializer is a constant expression ([5.2.4](502-const.md#524-constant-expressions-and-the-evaluator)), it is initialized at compile time. Any other initializer is initialized once on first use (lazy initialization). Lazy initialization is thread-safe, and threads accessing concurrently wait until initialization finishes. Functions in the initializer need not be `const fn` (`Mutex::new`, etc.).

```rok
static PI: f64 = 3.14159f64;             // const — compile time
static START: Instant = Instant::now();   // initialized once at first access
static CACHE: Mutex<Dict<str, Data>> = Mutex::new(Dict::new());
```

## 5.3.10. `extern static`: FFI

A `static` in an `extern "C"` block, as shown above, references from rok a global defined externally; no initializer is written.

```rok
extern "C" {
    static errno: i32;             // externally defined — read-only from rok
}
```

The detailed rules are defined in [`extern` blocks](509-extern-block.md).

## 5.3.11. Evaluation / memory model

The memory of a `static` persists until termination (`const` initializers are initialized at startup, others at first access). Every `static` is registered as a GC root at program startup (`rok_gc_add_root`, [8.5](../008-abi/805-extern-symbols.md)), so the [GC](../000-meta/007-glossary.md#gc-garbage-collector) keeps objects referenced by a `static` alive.

GC and runtime behavior are covered in [7. Runtime model](../007-runtime/README.md) and [8. Runtime ABI](../008-abi/README.md).
