# 3.10. `random::` - the OS random source

## 3.10.1. `random`

```rok
use rok::std::random;

// rok::std::random
fn fill(buf: SliceMut<u8>);      // overwrite every byte with random bytes
fn bytes(n: usize) -> List<u8>;  // n random bytes
fn secret(n: usize) -> Secret;   // n random bytes in secret storage (keys, seeds): no other copy
```

`fill` is the low-level form: it writes into a buffer the caller owns, any writable view of one ([2.9.7](../200-core/209-collections.md)): `fill(buf)` fills a whole `List<u8>`, `fill(buf[4:20])` only those bytes. `bytes` and `secret` are the convenient forms that allocate the buffer.

Bytes come from the operating system's cryptographically secure generator, read on every call (no user-space pool, so a forked process or a restored snapshot does not repeat output that `random` buffered):

| Target | Source |
|---|---|
| Linux (gnu, musl; x86-64, AArch64, i686, ARMv7) | `getrandom(2)` with no flags: blocks until the kernel pool is initialized, then never blocks |
| macOS | `getentropy(2)`, at most 256 bytes per call |
| Windows | `BCryptGenRandom` with `BCRYPT_USE_SYSTEM_PREFERRED_RNG` |
| wasm32-wasip2, wasm32-wasip1-threads | WASI `random_get` (the host's CSPRNG) |
| wasm32-unknown-unknown | the JavaScript host's `crypto.getRandomValues` (through the `random_get` import the host provides) |

An interrupted call is retried. A failing source cannot be recovered from: `fill`, `bytes` and `secret` panic rather than return predictable bytes (as Go's `crypto/rand`). The message names the called function and the OS error: `random::fill: the OS random source failed (error N)` (`random::bytes: ..`, `random::secret: ..`). The algorithms are `rok::core::crypto` (2.21), which take their random inputs (keys, seeds, nonces, salts) as arguments; `net::tls` (3.2.5) draws all of its from here.

## 3.10.2. Generators - `Rng`

```rok
pub trait Rng {
    fn next_u64(ref self) -> u64;                        // the only required method
    fn f64(ref self) -> f64;                             // [0, 1): (next_u64() >> 11) * 2^-53
    fn below(ref self, n: u64) -> u64;                   // [0, n), unbiased; n == 0 panics
    fn range(ref self, r: Range<i64>) -> i64;            // a..b; empty panics
    fn range_inclusive(ref self, r: RangeInclusive<i64>) -> i64;
    fn uniform(ref self, lo: f64, hi: f64) -> f64;       // lo + (hi - lo) * f64()
    fn bool(ref self, p: f64 = 0.5) -> bool;             // f64() < p
    fn normal(ref self, mean: f64 = 0.0, std: f64 = 1.0) -> f64;
    fn shuffle<T>(ref self, xs: List<T>);                // in place
    fn choice<T>(ref self, xs: List<T>) -> T | none;
    fn sample<T>(ref self, xs: List<T>, k: usize) -> List<T>;
}
```

The derived methods are fixed by this section so that a seeded generator gives the same values on every target and release. `below(n)` draws `next_u64()` until it is at least `(2^64 - n) mod n` and returns it `mod n`. `range(a..b)` is `a + below(b - a)` (two's complement width), `range_inclusive(a..=b)` `a + below(b - a + 1)` (a full-width range takes `next_u64()` as is). `normal` is Box-Muller on `u1 = 1 - f64()` then `u2 = f64()`: `mean + std * sqrt(-2 ln u1) * cos(2π u2)`. `shuffle` is Fisher-Yates from the back (position `i` swaps with `below(i + 1)`); `choice` is `xs[below(len)]`; `sample` swaps position `i` with `i + below(len - i)` for `i` in `0..k` and takes the items in that order.

**`Xoshiro256PlusPlus`** (Blackman and Vigna; Julia's default) implements `Rng`. `Xoshiro256PlusPlus::seed(n: u64)` sets the four state words to four successive SplitMix64 outputs from `n`; the stream of a seed never changes, and a new algorithm gets a new type. `from_os()` seeds it from the OS source. `split(ref self) -> Xoshiro256PlusPlus` returns a copy of the generator and jumps the original 2^128 steps ahead (Vigna's `jump`), giving non-overlapping streams for parallel work that depend only on the seed and the order of `split` calls. It is not for secrets.

**`OsRng`** implements `Rng` over the OS source (`next_u64` reads 8 bytes, big-endian), never reproducible. `random::rng()` is `Xoshiro256PlusPlus::from_os()`. The one-call forms `random::f64()`, `random::range(r)`, `random::shuffle(xs)` and `random::choice(xs)` use `OsRng` (a system call each); for many values make one generator.

```rok
use rok::std::random::{Rng, Xoshiro256PlusPlus};

let rng = Xoshiro256PlusPlus::seed(42);
let u = rng.f64();
let k = rng.range(0..10);
rng.shuffle(xs);
let worker = rng.split();
```
