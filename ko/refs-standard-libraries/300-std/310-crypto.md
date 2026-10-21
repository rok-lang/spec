# 3.10. `crypto::` — OS 난수 원천

## 3.10.1. `random`

```rok
use rok::std::crypto::random;

mod random {
    fn fill(buf: List<u8>);          // 모든 바이트를 난수로 덮어씀
    fn bytes(n: usize) -> List<u8>;
}
```

바이트는 OS 의 암호학적으로 안전한 생성기에서 옵니다: `getrandom` (Linux), `getentropy` (macOS), `BCryptGenRandom` (Windows), `random_get` (WASI; 브라우저에서는 `crypto.getRandomValues`). 원천이 실패하면 복구할 수 없으므로 panic 합니다. 알고리즘은 `rok::core::crypto` (2.21) 에 있습니다.
