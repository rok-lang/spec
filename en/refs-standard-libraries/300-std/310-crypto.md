# 3.10. `crypto::` - the OS random source

## 3.10.1. `random`

```rok
use rok::std::crypto::random;

mod random {
    fn fill(buf: List<u8>);          // overwrite every byte with random bytes
    fn bytes(n: usize) -> List<u8>;
    fn secret(n: usize) -> Secret;   // n random bytes in secret storage (keys, seeds): no other copy
}
```

Bytes come from the operating system's cryptographically secure generator, read on every call (no user-space pool, so a forked process or a restored snapshot does not repeat output that `random` buffered):

| Target | Source |
|---|---|
| Linux (gnu, musl; x86-64, AArch64, i686, ARMv7) | `getrandom(2)` with no flags: blocks until the kernel pool is initialized, then never blocks |
| macOS | `getentropy(2)`, at most 256 bytes per call |
| Windows | `BCryptGenRandom` with `BCRYPT_USE_SYSTEM_PREFERRED_RNG` |
| wasm32-wasi, wasm32-wasi-threads | WASI `random_get` (the host's CSPRNG) |
| wasm32-unknown-unknown | the JavaScript host's `crypto.getRandomValues` (through the `random_get` import the host provides) |

An interrupted call is retried. A failing source cannot be recovered from: `fill` and `bytes` panic rather than return predictable bytes. The algorithms are `rok::core::crypto` (2.21), which take their random inputs (keys, seeds, nonces, salts) as arguments; `net::tls` (3.2.5) draws all of its from here.
