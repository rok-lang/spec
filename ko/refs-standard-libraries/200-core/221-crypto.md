# 2.21. `crypto::` — 암호 알고리즘

`rok::core::crypto` 는 TLS 스택에 필요한 알고리즘을 순수 계산으로 제공합니다. 모든 target 에서 동일하게 동작하므로 `rok::core` (0.1) 에 둡니다. 난수는 OS 가 필요하므로 `rok::std::crypto::random` (3.10) 에 있으며, 무작위 입력이 필요한 함수 (키, nonce, encapsulation) 는 그 입력을 인자로 받습니다. 비밀 데이터에 대한 연산은 2.21.5 가 정의하는 대로 constant time 으로 수행됩니다.

아래의 모든 항목은 `rok::core::crypto::` 에서 쓸 수 있으며, 주석은 그것을 함께 export 하는 하위 모듈을 가리킵니다.

바이트 인자는 `Bytes` 를 받습니다.

```rok
pub type Bytes = List<u8> | Slice<u8> | str | Secret;   // str 은 UTF-8 바이트를, Secret 은 제자리의 바이트를 줌
struct Secret {                                     // crypto::secret
    fn zeroed(len: usize) -> Secret;
    fn new(bytes: Bytes) -> Secret;                 // 사본
    fn len(self) -> usize;
    fn as_bytes(self) -> Slice<u8>;                 // 사본이 아니라 제자리의 바이트
    fn wipe(self);                                  // 지금 0 으로 덮어씀
}
impl Index<usize> for Secret; impl IndexSet<usize> for Secret;   // s[i], s[i] = b (범위 검사)
impl Sliceable<Range<usize>> for Secret;           // s[a:b]: 제자리의 view
impl Drop for Secret;                               // using 해제가 지움
impl Debug for Secret;                              // "Secret(n bytes)", 바이트는 결코 보이지 않음
fn constant_time_eq(a: Bytes, b: Bytes) -> bool;   // 시간은 길이에만 의존
```

`Secret` 은 비밀 저장소 ([2.21.5](#2215-보안-고려-사항)) 에 있는 고정 길이 바이트열입니다. 복사본은 `List` 처럼 바이트를 공유하며, `Sync` 가 아닙니다. `as_bytes` 나 slicing 으로 얻은 view 는 저장소 자체를 읽습니다: 이후의 쓰기를 보며, secret 이 지워지면 0 을 읽습니다. `Secret` 에는 `==` 가 없으므로 `constant_time_eq` 로 비교합니다.

## 2.21.1. 해시, HMAC, HKDF

```rok
enum HashAlgorithm { Sha256, Sha384, Sha512 }       // crypto::sha2 (FIPS 180-4)
impl HashAlgorithm {
    fn output_len(self) -> usize;
    fn block_len(self) -> usize;
    fn digest(self, data: Bytes) -> List<u8>;
}
struct Digest {                                     // 진행 중인 해시; 복사하면 분기
    fn new(alg: HashAlgorithm) -> Digest;
    fn algorithm(self) -> HashAlgorithm;
    fn update(ref self, data: Bytes);
    fn finish(self) -> List<u8>;
}
struct Sha3_256 / Sha3_512 {                        // crypto::sha3 (FIPS 202)
    const OUTPUT_LEN: usize;                        // 32 / 64
    fn new() -> Self; fn update(ref self, data: Bytes); fn finish(self) -> List<u8>;
    fn digest(data: Bytes) -> List<u8>;
}
struct Shake128 / Shake256 {
    fn new() -> Self; fn update(ref self, data: Bytes);
    fn squeeze(ref self, n: usize) -> List<u8>;     // 다음 n 바이트의 출력
}
struct Hmac {                                       // crypto::hmac (RFC 2104)
    fn new(alg: HashAlgorithm, key: Bytes) -> Hmac;
    fn update(ref self, data: Bytes);
    fn finish(self) -> List<u8>;
    fn mac(alg: HashAlgorithm, key: Bytes, data: Bytes) -> List<u8>;
    fn verify(alg: HashAlgorithm, key: Bytes, data: Bytes, tag: Bytes) -> bool;   // constant time
}
struct Hkdf {                                       // RFC 5869
    fn extract(alg: HashAlgorithm, salt: Bytes, ikm: Bytes) -> Secret;
    fn expand(alg: HashAlgorithm, prk: Bytes, info: Bytes, len: usize) throws (ValueError) -> Secret;
}
```

`Digest`, `Hmac`, SHA-3 / SHAKE 상태는 값입니다: 복사본은 독립적으로 이어 갑니다 (중간에 읽는 transcript 해시). 연속된 `squeeze` 호출은 하나의 출력 스트림을 이어 가며, 첫 `squeeze` 뒤의 `update` 는 panic 합니다. `Hmac::verify` 는 전체 길이의 tag 를 constant time 으로 비교합니다. `Hkdf::extract` 는 salt 가 비면 `output_len` 개의 0 바이트를 쓰며, `expand` 는 `len > 255 * output_len` 이면 `ValueError` 를 던집니다.

## 2.21.2. AEAD

```rok
enum AeadAlgorithm { Aes128Gcm, Aes256Gcm, ChaCha20Poly1305 }   // crypto::aead
impl AeadAlgorithm { fn key_len(self) -> usize; fn nonce_len(self) -> usize; fn tag_len(self) -> usize; }
struct Aead {
    fn new(alg: AeadAlgorithm, key: Bytes) throws (ValueError) -> Aead;
    fn algorithm(self) -> AeadAlgorithm;
    fn seal(self, nonce: Bytes, aad: Bytes, plaintext: Bytes) throws (ValueError) -> List<u8>;   // ciphertext ‖ 16-byte tag
    fn open(self, nonce: Bytes, aad: Bytes, sealed: Bytes) throws (ValueError) -> List<u8> | none;  // none: 인증 실패
    fn wipe(self);                                  // 키 자료를 0 으로 덮어씀
}
impl Drop for Aead;
```

`Aead` 는 확장한 키 (AES round key 와 GHASH 키, 또는 ChaCha20 키) 를 비밀 저장소에 둡니다. 복사본은 그것을 공유하므로 `wipe` 는 모든 복사본을 끝냅니다.

AES-GCM 은 96-bit nonce 의 NIST SP 800-38D 를, ChaCha20-Poly1305 는 RFC 8439 를 따릅니다. nonce 는 12 바이트, tag 는 16 바이트입니다. 키나 nonce 의 길이가 틀리면 `ValueError` 이며, nonce 하나가 허용하는 것보다 긴 메시지도 그렇습니다 (32-bit block counter 가 넘쳐서는 안 됨: AES-GCM 은 2^36 − 32 바이트, ChaCha20-Poly1305 는 2^38 − 64). `open` 은 tag 보다 짧은 입력에 `none` 을 돌려주며, 평문을 내놓기 전에 tag 를 constant time 으로 검사합니다. 한 키 아래에서 nonce 는 결코 반복되어서는 안 되며, `Aead` 는 nonce 를 추적하지 않습니다.

## 2.21.3. 키 교환

```rok
struct X25519 {                                     // crypto::x25519 (RFC 7748)
    fn public_key(secret: Bytes) throws (ValueError) -> List<u8>;
    fn shared_secret(secret: Bytes, peer_public: Bytes) throws (ValueError) -> Secret | none;  // none: 결과가 전부 0
}
struct MlKem768 {                                   // crypto::mlkem (FIPS 203)
    const ENCAPSULATION_KEY_LEN: usize = 1184;
    const DECAPSULATION_KEY_LEN: usize = 2400;
    const CIPHERTEXT_LEN: usize = 1088;
    const SHARED_KEY_LEN: usize = 32;
    fn keygen(d: Bytes, z: Bytes) throws (ValueError) -> (List<u8>, Secret);       // 32-byte seed 로부터 (ek, dk)
    fn encapsulate(ek: Bytes, m: Bytes) throws (ValueError) -> (Secret, List<u8>);  // 32-byte m 으로부터 (key, ciphertext)
    fn decapsulate(dk: Bytes, c: Bytes) throws (ValueError) -> Secret;              // implicit rejection
}
```

X25519 키는 32 바이트입니다 (아니면 `ValueError`). secret 은 clamp 하고 상대의 u 좌표는 RFC 7748 5 가 말하는 대로 받으므로 (bit 255 무시, p 이상의 값은 reduce), small-order 나 twist 점은 에러가 아닙니다: `shared_secret` 은 결과가 전부 0 일 때 정확히 `none` 을 돌려줍니다 (RFC 7748 6.1). ML-KEM 함수들은 FIPS 203 의 결정적 내부 형태입니다 (호출자가 32 바이트 seed `d`, `z`, `m` 을 난수 원천에서 줌). `encapsulate` 와 `decapsulate` 는 FIPS 203 7.2 / 7.3 의 입력 검사를 수행하며 (`ValueError`), 자기 자신으로 다시 암호화되지 않는 ciphertext 는 에러가 아니라 implicit-rejection 키를 냅니다.

## 2.21.4. 서명

```rok
struct Ed25519 {                                    // crypto::ed25519 (RFC 8032)
    fn public_key(seed: Bytes) throws (ValueError) -> List<u8>;
    fn sign(seed: Bytes, msg: Bytes) throws (ValueError) -> List<u8>;
    fn verify(public_key: Bytes, msg: Bytes, sig: Bytes) -> bool;
}
enum EcCurve { P256, P384 }                         // crypto::ecdsa (FIPS 186-5)
struct Ecdsa {
    fn public_key(curve: EcCurve, private_key: Bytes) throws (ValueError) -> List<u8>;  // SEC1 비압축
    fn sign(curve: EcCurve, hash: HashAlgorithm, private_key: Bytes, msg: Bytes) throws (ValueError) -> List<u8>;  // DER, RFC 6979 nonce
    fn verify(curve: EcCurve, hash: HashAlgorithm, public_key: Bytes, msg: Bytes, signature: Bytes) -> bool;  // SEC1 비압축 키, DER 서명
}
struct RsaPublicKey {                               // crypto::rsa (RFC 8017), 2048..4096 bits
    fn new(n: Bytes, e: Bytes) throws (ValueError) -> RsaPublicKey;
    fn from_pkcs1_der(der: Bytes) throws (ValueError) -> RsaPublicKey;
    fn bits(self) -> usize;
    fn modulus(self) -> List<u8>;                   // big-endian, 앞의 0 바이트 없음
    fn exponent(self) -> List<u8>;
    fn verify_pkcs1v15(self, hash: HashAlgorithm, msg: Bytes, signature: Bytes) -> bool;
    fn verify_pss(self, hash: HashAlgorithm, msg: Bytes, signature: Bytes, salt_len: usize | none = none) -> bool;  // MGF1(hash); none: salt 길이 = 해시 길이
}
struct RsaPrivateKey {
    fn from_pkcs1_der(der: Bytes) throws (ValueError) -> RsaPrivateKey;   // RSAPrivateKey, 소수 두 개
    fn public_key(self) -> RsaPublicKey;
    fn bits(self) -> usize;
    fn sign_pkcs1v15(self, hash: HashAlgorithm, msg: Bytes) throws (ValueError) -> List<u8>;  // 결정적
    fn sign_pss(self, hash: HashAlgorithm, msg: Bytes, salt: Bytes) throws (ValueError) -> List<u8>;  // salt: 새 난수, 해시 길이
}
```

Ed25519 검증은 cofactorless 이며, `A` 와 `R` 의 정규형이 아닌 인코딩과 `S ≥ L` 을 거부합니다. ECDSA 검증은 곡선 위의 비압축 점이 아니거나 좌표가 p 이상인 공개 키, 최소 DER 이 아니거나 `r` 또는 `s` 가 `[1, n)` 밖인 서명을 거부합니다. RSA 검증은 modulus 보다 작고 정확히 modulus 길이인 서명을 요구하며, PKCS #1 v1.5 는 다시 만든 인코딩 전체를 비교합니다. `RsaPublicKey::new` 는 2048 에서 4096 bit 의 홀수 modulus 와 `3 ≤ e < 2^33` 인 홀수 지수를 받으며 (앞의 0 바이트는 무시), 그 밖은 `ValueError` 입니다 (검증은 지수 bit 마다 modular 제곱 한 번이 들므로 상대가 긴 지수를 고르게 해서는 안 됨).

ECDSA 개인 키는 곡선의 크기 이하인 big-endian 스칼라 `1 <= d < n` 입니다. nonce 는 결정적으로 유도되므로 (RFC 6979, `hash` 를 쓰는 HMAC-DRBG) 서명에 무작위 입력이 필요 없습니다. RSA 서명은 base blinding 을 곁들인 CRT 를 쓰며 (blinding 인수는 키, 메시지, 그리고 PSS 이면 `salt` 에서 유도하므로 PKCS #1 v1.5 서명에는 무작위 입력이 필요 없음), 결과를 돌려주기 전에 공개 키로 검사합니다. 일관되지 않은 키는 `ValueError` 입니다. PSS 검증은 modulus 에 들어가는 어떤 salt 길이든 받으며 (`salt_len`), 서명은 해시 길이만큼의 salt 를 씁니다.

## 2.21.5. 보안 고려 사항

**Constant time.** 비밀에 대한 연산 — AES (암호화와 key schedule), GHASH, ChaCha20, Poly1305, tag 비교, X25519, Ed25519 서명, ECDSA 서명과 키 유도, RSA 개인 키 연산, ML-KEM 키 생성과 decapsulation, HMAC, HKDF — 에서는 어떤 분기도, 어떤 메모리 인덱스도 비밀 데이터에 의존하지 않습니다. AES 는 bitslice 로 구현하고 (table 조회 없음), GHASH 는 성긴 피연산자의 정수 곱으로 곱하며 (table 없음), 스칼라 곱은 미리 계산한 table 을 모든 항목을 훑어 읽고, 역원은 공개 지수의 거듭제곱 (ECDSA, Ed25519) 이나 고정 길이 binary GCD (RSA blinding) 로 구하며, RSA 거듭제곱은 소수의 전체 폭에 걸쳐 지수 bit 마다 제곱과 곱셈을 하고, ML-KEM 은 나눗셈 대신 곱하고 shift 해서 reduce 하며, 조건부 선택은 컴파일러가 0 이나 모두 1 임을 증명할 수 없는 mask 이므로 다시 분기로 바꿀 수 없습니다. 실행 시간은 공개 값에만 의존합니다: 길이, 알고리즘, 곡선과 키 크기, 그리고 어차피 공개되는 결과 (검증되지 않는 tag 나 서명, 범위 밖의 RFC 6979 후보 nonce). ML-KEM decapsulation 은 ciphertext 가 다시 암호화되든 아니든 같은 경로를 밟으며, implicit-rejection 키는 mask 로 고릅니다.

이 보장에는 구현이 밝히고 숨기지 않는 한계가 있습니다.
- target 의 32 × 32 → 64-bit 정수 곱셈과 shift, rotate 가 constant time 으로 돈다고 가정합니다 (현재의 x86-64, AArch64, ARMv7-A, i686 코어에서는 참이며, 일부 마이크로컨트롤러와 옛 임베디드 코어에서는 아님). wasm target 에서는 엔진의 코드 생성만큼만 유효합니다.
- 제어 흐름과 메모리 주소로 보이는 시간만 다루며, 전력, 전자기, fault injection 공격이나 투기 실행 side channel 은 다루지 않습니다.
- 파싱 (DER, PEM, 키 파일) 과 공개 데이터의 검증은 가변 시간입니다.
- 이것은 배포된 toolchain 으로 컴파일한 구현의 성질이지 언어가 주는 보장이 아닙니다: 같은 방식으로 쓴 사용자 코드는 포함되지 않습니다.

**메모리 안의 키 자료.** 비밀 바이트는 *비밀 저장소* 에 있습니다: 수집기가 large-object space ([refs-lang 7.1.2](../../refs-lang/007-runtime/701-object-layout.md#712-메모리-공간)) 에 할당하고, 결코 옮기거나 복사하지 않으며, 메모리를 놓기 전에 0 으로 덮어쓰는 buffer 입니다. `Secret` (2.21) 은 그 공개 형태입니다.

- `Secret` 의 바이트는 heap 에서 정확히 한 곳에만 있습니다. `wipe` 가, `using` 바인딩이 `Secret` 을 해제할 때 (직접 또는 drop glue 를 거쳐), 늦어도 그 buffer 를 해제하는 수집이 그것을 0 으로 덮어씁니다. 0 으로 지우는 쓰기는 최적화기가 없앨 수 없는 store 입니다.
- 알고리즘은 키에 의존하는 상태를 비밀 저장소나 지역 변수에만 두며, 보통의 heap 객체에는 두지 않습니다. 그 상태는 AES round key, GHASH 키, ChaCha20 키 (`Aead`), HMAC 과 HKDF (padding 한 키, chaining 값, 출력 block), X25519 와 ML-KEM 의 공유 키, ML-KEM decapsulation 키와 그 비밀 다항식, seed, 메시지, 그리고 Ed25519 의 확장 키와 nonce 해시입니다. 결과가 키 자료인 함수는 `Secret` 을 돌려줍니다. 비밀 저장소의 임시 값은 함수가 반환하기 전에 지웁니다. 함수가 throw 하면 수집이 지웁니다.
- `net::tls` ([3.2.5](../300-std/302-net.md#325-tls-13--nettls)) 는 traffic, handshake, resumption secret, record 키, 키 교환 secret, ticket PSK, Ed25519 / EC 개인 키를 비밀 저장소에 두며, KeyUpdate 때, 핸드셰이크가 끝날 때, close 때 지웁니다.

이 보장에는 구현이 밝히고 숨기지 않는 한계가 있습니다.

- **레지스터와 스택.** 지역 변수는 레지스터와 스레드의 스택에 있으며 지우지 않습니다. 고정 크기 배열과 `Digest`, `Hmac`, SHA-3 와 cipher 상태 같은 struct 값, 그리고 X25519, Ed25519, ECDSA 의 field element 와 스칼라도 포함됩니다. 컴파일러가 만드는 spill 도 지우지 않으며, slot 은 스택을 다시 쓸 때만 덮어써집니다. 그것을 지우는 코드 생성은 없으며, Rust 의 `zeroize`, Go 의 `crypto/subtle` helper, libsodium 의 `sodium_memzero` 도 마찬가지로 받은 메모리에만 닿고 컴파일러가 만든 사본에는 닿지 않습니다.
- **보통의 객체.** heap 객체에 저장한 값은 그 객체 안에 있습니다: 수집기는 evacuation 중에 그것을 복사할 수 있고 지우지 않은 채 해제합니다. `List` 원소, GC 가 가진 struct 의 필드, 클로저 환경, 중단 지점을 넘는 `async` / generator 프레임 ([refs-lang 7.8.2](../../refs-lang/007-runtime/708-coroutine.md#782-코루틴-프레임)) 이 해당합니다. `Secret` 의 바이트를 꺼내 복사하면 (`as_bytes().to_list()`, `str`, 포맷) 보통의 사본이 생깁니다.
- **보통의 입력과 출력.** 다루지 않는 것: `List<u8>` 나 `str` 로 넘긴 키, 평문과 암호문, `Hmac` tag, `Digest` 출력, 디코드한 키 파일 (PEM / DER 디코드는 `PrivateKey` 가 키를 복사하기 전에 보통의 리스트를 만듦).
- **서명.** ECDSA 와 RSA 서명은 다루지 않습니다: RFC 6979 상태와 `bignum` limb 는 보통의 리스트이며, `RsaPrivateKey` 는 값을 보통의 리스트에 둡니다.
- **RAM 에 고정하지 않음.** 비밀 저장소는 고정 (`mlock` / `VirtualLock`) 하지 않습니다. swap 이나 hibernation 파일에 쓰일 수 있고, core dump 와 프로세스 snapshot 에 나타납니다. 고정은 page 단위이고 프로세스마다 제한되며 (RLIMIT_MEMLOCK) page 단위로 풀리므로, page 단위의 자체 할당기가 필요할 것입니다.
- **프로세스 종료와 wasm.** 프로세스 종료로 놓이는 메모리는 지우지 않습니다. wasm target 에서는 host 가 언제든 linear memory 를 읽을 수 있습니다.

**난수.** 알고리즘은 모든 무작위 입력을 인자로 받습니다. 호출자는 그것을 OS 의 CSPRNG 인 `rok::std::crypto::random` (3.10) 에서 꺼냅니다. ML-KEM seed, X25519 와 Ed25519 secret, ECDSA 개인 키, PSS salt 는 그런 원천의 새 출력이어야 합니다.
