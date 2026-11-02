# 3.10. `random::`: OS 난수 원천

## 3.10.1. `random`

```rok
use rok::std::random;

// rok::std::random
fn fill(buf: SliceMut<u8>);      // 모든 바이트를 난수로 덮어씀
fn bytes(n: usize) -> List<u8>;  // n 개 난수 바이트
fn secret(n: usize) -> Secret;   // 비밀 저장소의 n 개 난수 바이트 (키, seed): 다른 사본 없음
```

`fill`은 저수준 형태입니다. 호출자가 가진 버퍼, 곧 그 버퍼의 쓰기 가능한 뷰([2.9.7](../200-core/209-collections.md))에 씁니다. `fill(buf)`는 `List<u8>` 전체를, `fill(buf[4:20])`은 그 바이트만 채웁니다. `bytes`와 `secret`은 버퍼를 직접 만들어 주는 편의 형태입니다.

바이트는 호출할 때마다 OS의 암호학적으로 안전한 생성기에서 읽습니다. 사용자 공간 풀이 없습니다. 그래서 fork로 만든 프로세스나 복원한 스냅숏이 `random`이 버퍼에 담아 둔 출력을 반복하지 않습니다.

| Target | 원천 |
|---|---|
| Linux (gnu, musl; x86-64, AArch64, i686, ARMv7) | 플래그 없는 `getrandom(2)`: 커널 풀이 초기화될 때까지 막고, 그 뒤로는 막지 않음 |
| macOS | `getentropy(2)`, 호출마다 최대 256바이트 |
| Windows | `BCRYPT_USE_SYSTEM_PREFERRED_RNG`를 준 `BCryptGenRandom` |
| wasm32-wasip2, wasm32-wasip1-threads | WASI `random_get` (호스트의 CSPRNG) |
| wasm32-unknown-unknown | JavaScript 호스트의 `crypto.getRandomValues` (호스트가 주는 `random_get` import로) |

중단된 호출은 다시 시도합니다. 원천의 실패는 복구할 수 없습니다. 이때 `fill`, `bytes`, `secret`은 예측 가능한 바이트를 돌려주지 않고 panic을 일으킵니다(Go의 `crypto/rand`와 같음). 메시지에는 호출한 함수와 OS 에러를 적습니다. `random::fill: the OS random source failed (error N)`(`random::bytes: ..`, `random::secret: ..`). 알고리듬은 `rok::core::crypto`(2.21)에 있고, 무작위 입력(키, seed, nonce, salt)을 인자로 받습니다. `net::tls`(3.2.5)는 이 입력을 모두 여기서 얻습니다.

## 3.10.2. 생성기: `Rng`

```rok
pub trait Rng {
    fn next_u64(ref self) -> u64;                        // 유일한 필수 메서드
    fn f64(ref self) -> f64;                             // [0, 1): (next_u64() >> 11) * 2^-53
    fn below(ref self, n: u64) -> u64;                   // [0, n), 치우침 없음. n == 0이면 패닉
    fn range(ref self, r: Range<i64>) -> i64;            // a..b, 비면 패닉
    fn range_inclusive(ref self, r: RangeInclusive<i64>) -> i64;
    fn uniform(ref self, lo: f64, hi: f64) -> f64;       // lo + (hi - lo) * f64()
    fn bool(ref self, p: f64 = 0.5) -> bool;             // f64() < p
    fn normal(ref self, mean: f64 = 0.0, std: f64 = 1.0) -> f64;
    fn shuffle<T>(ref self, xs: List<T>);                // 제자리
    fn choice<T>(ref self, xs: List<T>) -> T | none;
    fn sample<T>(ref self, xs: List<T>, k: usize) -> List<T>;
}
```

파생 메서드는 이 절이 정하므로, 시드를 준 생성기는 모든 타깃과 릴리스에서 같은 값을 냅니다. `below(n)`은 `next_u64()`가 `(2^64 - n) mod n` 이상이 될 때까지 뽑고 그 값의 `mod n`을 돌려줍니다. `range(a..b)`는 `a + below(b - a)`(2의 보수 폭), `range_inclusive(a..=b)`는 `a + below(b - a + 1)`입니다(폭 전체 범위는 `next_u64()`를 그대로 씀). `normal`은 `u1 = 1 - f64()`, 다음 `u2 = f64()`로 하는 Box-Muller입니다: `mean + std * sqrt(-2 ln u1) * cos(2π u2)`. `shuffle`은 뒤에서부터의 Fisher-Yates입니다(위치 `i`와 `below(i + 1)`을 바꿈). `choice`는 `xs[below(len)]`입니다. `sample`은 `0..k`의 `i`마다 위치 `i`와 `i + below(len - i)`를 바꾸고 그 순서로 항목을 가져옵니다.

**`Xoshiro256PlusPlus`**(Blackman, Vigna. Julia의 기본값)는 `Rng`를 구현합니다. `Xoshiro256PlusPlus::seed(n: u64)`는 `n`에서 시작한 SplitMix64 출력 네 개를 상태 워드로 씁니다. 한 시드의 수열은 바뀌지 않으며 새 알고리즘은 새 타입이 됩니다. `from_os()`는 OS 원천으로 시드를 정합니다. `split(ref self) -> Xoshiro256PlusPlus`는 생성기의 복사본을 돌려주고 원본을 2^128걸음 앞으로 건너뜁니다(Vigna의 `jump`). 그래서 병렬 작업용으로 겹치지 않는 수열을 주며, 수열은 시드와 `split` 호출 순서에만 달려 있습니다. 비밀 값에는 쓰지 않습니다.

**`OsRng`**는 OS 원천 위의 `Rng`입니다(`next_u64`는 8바이트를 빅 엔디언으로 읽음). 결코 재현되지 않습니다. `random::rng()`는 `Xoshiro256PlusPlus::from_os()`입니다. 한 번 호출 형태 `random::f64()`, `random::range(r)`, `random::shuffle(xs)`, `random::choice(xs)`는 `OsRng`를 씁니다(매번 시스템 호출). 값이 많이 필요하면 생성기를 하나 만드세요.

```rok
use rok::std::random::{Rng, Xoshiro256PlusPlus};

let rng = Xoshiro256PlusPlus::seed(42);
let u = rng.f64();
let k = rng.range(0..10);
rng.shuffle(xs);
let worker = rng.split();
```
