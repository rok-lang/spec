# 3.10. `crypto::` - OS 난수 원천

## 3.10.1. `random`

```rok
use rok::std::crypto::random;

mod random {
    fn fill(buf: List<u8>);          // 모든 바이트를 난수로 덮어씀
    fn bytes(n: usize) -> List<u8>;
    fn secret(n: usize) -> Secret;   // 비밀 저장소의 n 개 난수 바이트 (키, seed): 다른 사본 없음
}
```

바이트는 OS 의 암호학적으로 안전한 생성기에서 호출마다 읽습니다 (사용자 공간 pool 이 없으므로, fork 한 프로세스나 복원한 snapshot 이 `random` 이 buffer 한 출력을 반복하지 않음):

| Target | 원천 |
|---|---|
| Linux (gnu, musl; x86-64, AArch64, i686, ARMv7) | flag 없는 `getrandom(2)`: 커널 pool 이 초기화될 때까지 block 하고, 그 뒤로는 block 하지 않음 |
| macOS | `getentropy(2)`, 호출마다 최대 256 바이트 |
| Windows | `BCRYPT_USE_SYSTEM_PREFERRED_RNG` 를 준 `BCryptGenRandom` |
| wasm32-wasi, wasm32-wasi-threads | WASI `random_get` (host 의 CSPRNG) |
| wasm32-unknown-unknown | JavaScript host 의 `crypto.getRandomValues` (host 가 주는 `random_get` import 를 통해) |

중단된 호출은 다시 시도합니다. 원천의 실패는 복구할 수 없습니다: `fill`, `bytes`, `secret` 은 예측 가능한 바이트를 돌려주는 대신 panic 합니다. 알고리즘은 `rok::core::crypto` (2.21) 이며 무작위 입력 (키, seed, nonce, salt) 을 인자로 받습니다. `net::tls` (3.2.5) 는 그 모두를 여기서 꺼냅니다.
