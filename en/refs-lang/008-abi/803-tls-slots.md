# 8.3. Mutator TLS slot layout

Each OS thread (= scheduler worker = mutator) holds runtime information at fixed locations in thread-local storage (TLS). Code emitted by the compiler and the runtime read and write the same TLS slots, so this document defines the offset and type of each slot.

## 8.3.1. TLS slot structure

The TLS slots are one `thread_local` block, `rok_tls`, defined and exported by the runtime; each slot is at a fixed offset in it. Compiled code takes the block's address once per function and reaches each slot at its offset.

`p` is the pointer width in bytes.

| Offset   | Slot                        | Type | Meaning                                 |
|----------|-----------------------------|------|-----------------------------------------|
| 0        | `rok_tls_shadow_stack_top`  | ptr  | Head of the shadow stack chain          |
| p        | `rok_tls_tlab_cursor`       | ptr  | Next allocation position in the TLAB    |
| 2p       | `rok_tls_tlab_limit`        | ptr  | End of the TLAB                         |
| 3p       | `rok_tls_wb_log_cursor`     | ptr  | Position in the write barrier log       |
| 4p       | `rok_tls_wb_log_limit`      | ptr  | End of the write barrier log            |
| 5p       | `rok_tls_satb_cursor`       | ptr  | Position in the SATB buffer (during concurrent mark) |
| 6p       | `rok_tls_satb_limit`        | ptr  | End of the SATB buffer                  |
| 7p       | `rok_tls_task_worker`       | ptr  | This thread's executor worker (GC object, 0 if none) |
| 8p       | `rok_tls_stack_guard`       | [usize; 2] | `[lo, hi)` — a fault in this range is a stack overflow of this thread (`[0, 0)` means unknown, [8.5.7](805-extern-symbols.md#stack-overflow)) |
| 10p      | `rok_tls_worker_id`         | u64  | Worker id                               |
| 10p+8    | `rok_tls_yield_flag`        | u8   | Whether to `yield` at a safepoint       |
| 10p+9    | `rok_tls_mutator_state`     | u8   | Mutator state                           |

The **ABI contract** is the block symbol `rok_tls` and each slot's offset, type and meaning. Slot names are for reference only; they are not symbols. The block is 8-aligned and at least 10p+10 bytes; implementation-defined slots may follow the last ABI slot.

**Implementation-defined slots**: the runtime may keep further thread-local state, after the ABI slots or elsewhere, for scheduling, statistics, profiling, debugging and so on. The compiler and external packages must not reference it.

> [!NOTE]
> rok's runtime places `task_lane` at 10p+16, `static_inits` at 11p+16 and `held_locks` (the locks this thread holds; in a session host, locks a thread that is gone held while blocked are released) at 12p+16.

## 8.3.2. Access mechanism

Each OS and architecture has its own TLS access mechanism.

| Platform       | Mechanism                             |
|----------------|---------------------------------------|
| Linux x86_64   | `%fs:offset` or `%gs:offset`          |
| Linux aarch64  | `tpidr_el0`                           |
| macOS x86_64   | `%gs:offset`                          |
| Windows x86_64 | TEB (Thread Environment Block) offset |
| `wasm32-wasi-threads` | `__tls_base` global + offset   |
| other WASM (no threads) | the block is a plain global  |

The compiler emits the appropriate instructions per target; LLVM's `__thread` and TLS models handle this automatically.

## 8.3.3. Meaning of each slot

### `rok_tls_yield_flag` (u8)

A flag the [GC](../000-meta/007-glossary.md#gc-garbage-collector) sets for a safepoint; mutators check it when polling. Bit 0 is the runtime's stop request, raised and cleared by the runtime; other bits belong to the embedding host (e.g. the REPL's interrupt) and are left alone by the runtime. A poll calls `rok_runtime_yield` whenever the flag is nonzero.

The detailed mechanism is defined under Safepoint Polling in [Barriers and roots](../007-runtime/704-barriers-and-roots.md).

### `rok_tls_tlab_cursor` (ptr)

Points to where the next allocation happens in the [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) of [Allocation](../007-runtime/702-allocation.md).

```
allocate(size):
    tmp = tlab_cursor
    new_cursor = tmp + size
    if new_cursor > tlab_limit:
        slow path
    tlab_cursor = new_cursor
    return tmp
```

These four instructions form the TLAB fast path, which the compiler emits inline.

### `rok_tls_tlab_limit` (ptr)

Points to the end of the TLAB. When the cursor passes this value, the slow path is entered.

### `rok_tls_shadow_stack_top` (ptr)

Head of the shadow stack chain. Defined in detail in 8.3.8.

### `rok_tls_satb_cursor` / `rok_tls_satb_limit` (ptr)

The SATB buffer recorded by the write barrier slow path during concurrent mark ([7.5](../007-runtime/705-concurrent-collection.md)). When full, it is handed to the collector. Not used outside mark.

### `rok_tls_wb_log_cursor` / `rok_tls_wb_log_limit` (ptr)

Cursor and limit of the write barrier log buffer from [Barriers and root tracking](../007-runtime/704-barriers-and-roots.md).

```
write_barrier(p, q):
    if !needs_log: return
    if wb_log_cursor >= wb_log_limit: flush
    *wb_log_cursor++ = p
```

Most operations are non-atomic and thread-local.

The two slots point to a per-thread 4 KiB log chunk. A full chunk is pushed onto a lock-free stack that the collector drains ([7.4.2](../007-runtime/704-barriers-and-roots.md#742-write-barrier-incremental-update)).

### `rok_tls_mutator_state` (u8)

The mutator's current state.

| Value | Meaning                       |
|-------|-------------------------------|
| `0`   | RUNNING. Normal execution     |
| `1`   | GC_COOPERATING. Cooperating with the GC |
| `2`   | PARKED. In sleep / wait       |
| `3` ~ | Reserved                      |

This value is used for GC cycle synchronization and safepoint cooperation.

### `rok_tls_worker_id` (u64)

The thread's worker id (used inside the scheduler), used for [work-stealing](../000-meta/007-glossary.md#work-stealing), statistics and the like.

## 8.3.4. Slot width and alignment

Each slot has the width of its type and lies at an offset aligned to it.

## 8.3.5. 32-bit vs. 64-bit

Pointer slots are p bytes (8 on 64-bit, 4 on 32-bit), so offsets differ between 32-bit and 64-bit as given by the table.

## 8.3.6. TLS slot initialization

When a worker starts, the slots are initialized as follows.

```
rok_thread_init():
  rok_tls_yield_flag = 0
  rok_tls_tlab_cursor = NULL          // filled by the slow path on first allocation
  rok_tls_tlab_limit = NULL
  rok_tls_shadow_stack_top = NULL     // empty shadow stack chain (8.3.8)
  rok_tls_wb_log_cursor = wb_log_base
  rok_tls_wb_log_limit = wb_log_base + LOG_SIZE
  rok_tls_mutator_state = RUNNING
  rok_tls_worker_id = (allocated)
  rok_tls_task_worker = 0
  rok_tls_stack_guard = [guard_lo, guard_hi)   // guard range below the stack
```

The scheduler calls `rok_thread_init` when a worker starts; this function is part of the [extern symbols](805-extern-symbols.md).

## 8.3.7. TLS slot teardown

When a worker exits, it goes through the following steps.

```
rok_thread_shutdown():
  // log flush
  // shadow stack walk (and a final mark, if possible)
  // return worker_id
```

## 8.3.8. Form of compiler emission

The block is a `thread_local` global defined and exported by the runtime. The compiler references it as an external declaration, takes its address once per function and loads slots at their offsets.

```mlir
; LLVM IR — defined by the runtime; compiler output has an external declaration
@rok_tls = external thread_local global [N x i8]

%t = call ptr @llvm.threadlocal.address(ptr @rok_tls)   ; once per function

; safepoint poll (yield_flag at 10p+8)
%flag_p = getelementptr i8, ptr %t, i64 88               ; 10p+8 with p = 8
%flag = load i8, ptr %flag_p
%check = icmp ne i8 %flag, 0
br i1 %check, label %slow, label %cont
```

The LLVM backend then lowers this to target-specific TLS instructions.

On ELF, `rok_tls` uses the initial-exec TLS model (the compiler emits access code in this model and does not rely on linker relaxation). The runtime is always statically linked into the executable, so this model suffices. Mach-O uses the platform's TLV descriptor scheme. An implementation whose toolchain cannot choose a TLS model may use general-dynamic instead.

### Shadow stack head

The slot at offset 0 of `rok_tls` (`rok_tls_shadow_stack_top`) is the head of the [shadow stack](../000-meta/007-glossary.md#shadow-stack) chain from [Barriers and root tracking](../007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler), i.e. the address of the innermost `ShadowFrame` (null if none). It is the only chain head. Function entry code emitted by the compiler links the frame and unlinks it immediately before every `return`.

LLVM's `llvm.gcroot` and `llvm_gc_root_chain` are not used. `llvm_gc_root_chain` is a single process-global and cannot express per-thread chains.

## 8.3.9. Stability

The offsets, types and meanings in the 8.3.1 table are part of the [ABI](../000-meta/007-glossary.md#abi-application-binary-interface); changes follow the stability policy in the [overview](README.md). Changing an offset is a major `break`; a new slot takes an offset after the existing ones, a minor compatible change. Changing the type or meaning of an existing slot is also a major `break`.

Implementation-defined slots are not part of the ABI and may change freely even within the same major version.
