# 1.5. volatile memory access

The `rok::bare::volatile` module provides memory reads / writes that the compiler does not optimize away. This channel is essential where every access must be executed as written, such as hardware registers, MMIO, signal handlers and DMA buffers.

## 1.5.1. Motivation

The compiler may freely remove, reorder or merge ordinary reads / writes. Consider the following code.

```rok
let p: Ptr<u32> = mmio_register;
unsafe {
    p.write(1);                // compiler: possible dead store (overwritten by the next write)
    p.write(2);                // the actual hardware must receive both, 1 → 2
}
```

The compiler may treat the first write as a dead store and delete it, yet some hardware works correctly only if it receives 1 and then 2. `volatile` rejects such optimizations and forces the accesses to happen exactly as written.

```rok
unsafe {
    rok::bare::volatile::write(p, 1);    // always writes 1 to the hardware
    rok::bare::volatile::write(p, 2);    // always writes 2
}
```

## 1.5.2. API

The API consists of just two functions, read and write.

```rok
// rok::bare::volatile
unsafe fn read<T>(src: ConstPtr<T>) -> T;
unsafe fn write<T>(dst: Ptr<T>, value: T);
```

The same operations are available as the methods `Ptr<T>::read_volatile` / `write_volatile`, which often read more naturally ([1.1.2](101-ptr.md#112-ptrt--strict-provenance-api)).

## 1.5.3. Uses

The places that need `volatile` generally fall into these patterns.

| Place                  | Reason                                |
|------------------------|---------------------------------------|
| MMIO register          | Hardware has side effects even on read |
| Memory-mapped device   | DMA, etc.                             |
| Signal handler variable | Changed inside a signal — must be visible outside |
| Spinlock retry loop    | Repeatedly reads a value changed by another thread |

## 1.5.4. Caution — not the same as atomic

A common misconception: `volatile` only blocks compiler optimizations and gives no atomicity guarantee.

- Race conditions still occur across threads
- No memory ordering is enforced

Variables shared between threads must therefore use [`Atomic<T>`](../200-core/216-sync.md#2162-atomict--lock-free-atomics), and `volatile` should be reserved as a tool for **hardware MMIO**.

## 1.5.5. lowering

`volatile::read` and `volatile::write` lower to LLVM's `load volatile` / `store volatile`; their intrinsic names are `rok.volatile.read.T` and `rok.volatile.write.T` respectively.

Atomics are covered in detail in [2.16 sync](../200-core/216-sync.md); this chapter is authoritative only for non-atomic cases.
