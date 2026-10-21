# 3.10. `crypto::` — the OS random source

## 3.10.1. `random`

```rok
use rok::std::crypto::random;

mod random {
    fn fill(buf: List<u8>);          // overwrite every byte with random bytes
    fn bytes(n: usize) -> List<u8>;
}
```

Bytes come from the OS cryptographically secure generator: `getrandom` (Linux), `getentropy` (macOS), `BCryptGenRandom` (Windows), `random_get` (WASI; `crypto.getRandomValues` in the browser). A failing source cannot be recovered from: it panics. The algorithms are `rok::core::crypto` (2.21).
