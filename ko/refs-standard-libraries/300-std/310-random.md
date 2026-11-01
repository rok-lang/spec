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

중단된 호출은 다시 시도합니다. 원천의 실패는 복구할 수 없습니다. 이때 `fill`, `bytes`, `secret`은 예측 가능한 바이트를 돌려주지 않고 panic을 일으킵니다(Go의 `crypto/rand`와 같음). 알고리듬은 `rok::core::crypto`(2.21)에 있고, 무작위 입력(키, seed, nonce, salt)을 인자로 받습니다. `net::tls`(3.2.5)는 이 입력을 모두 여기서 얻습니다.
