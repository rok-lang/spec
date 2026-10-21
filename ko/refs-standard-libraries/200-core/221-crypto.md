# 2.21. `crypto::` — 암호 알고리즘

`rok::core::crypto` 는 TLS 스택에 필요한 알고리즘을 순수 계산으로 제공합니다. 모든 target 에서 동일하게 동작하므로 `rok::core` (0.1) 에 둡니다. 난수는 OS 가 필요하므로 `rok::std::crypto::random` (3.10) 에 있으며, 무작위 입력이 필요한 함수 (키, nonce, encapsulation) 는 그 입력을 인자로 받습니다. 비밀 데이터에 대한 연산은 constant time 으로 수행됩니다: 어떤 분기도, 어떤 메모리 인덱스도 비밀에 의존하지 않습니다.

바이트 인자는 `Bytes` 를 받습니다.

```rok
pub type Bytes = List<u8> | Slice<u8> | str;   // str 은 UTF-8 바이트를 줌
fn constant_time_eq(a: Bytes, b: Bytes) -> bool;   // 시간은 길이에만 의존
```

## 2.21.1. 해시, HMAC, HKDF

```rok
enum HashAlgorithm { Sha256, Sha384, Sha512 }       // crypto::sha2
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
    fn new() -> Self; fn update(ref self, data: Bytes); fn finish(self) -> List<u8>;
    fn digest(data: Bytes) -> List<u8>;
}
struct Shake128 / Shake256 {
    fn new() -> Self; fn update(ref self, data: Bytes);
    fn squeeze(ref self, n: usize) -> List<u8>;     // 반복 호출 가능
}
struct Hmac {                                       // crypto::hmac (RFC 2104)
    fn new(alg: HashAlgorithm, key: Bytes) -> Hmac;
    fn update(ref self, data: Bytes);
    fn finish(self) -> List<u8>;
    fn mac(alg: HashAlgorithm, key: Bytes, data: Bytes) -> List<u8>;
    fn verify(alg: HashAlgorithm, key: Bytes, data: Bytes, tag: Bytes) -> bool;   // constant time
}
struct Hkdf {                                       // RFC 5869
    fn extract(alg: HashAlgorithm, salt: Bytes, ikm: Bytes) -> List<u8>;
    fn expand(alg: HashAlgorithm, prk: Bytes, info: Bytes, len: usize) throws (ValueError) -> List<u8>;
}
```

## 2.21.2. AEAD

```rok
enum AeadAlgorithm { Aes128Gcm, Aes256Gcm, ChaCha20Poly1305 }   // crypto::aead
impl AeadAlgorithm { fn key_len(self) -> usize; fn nonce_len(self) -> usize; fn tag_len(self) -> usize; }
struct Aead {
    fn new(alg: AeadAlgorithm, key: Bytes) throws (ValueError) -> Aead;
    fn algorithm(self) -> AeadAlgorithm;
    fn seal(self, nonce: Bytes, aad: Bytes, plaintext: Bytes) throws (ValueError) -> List<u8>;   // ciphertext ‖ 16-byte tag
    fn open(self, nonce: Bytes, aad: Bytes, sealed: Bytes) throws (ValueError) -> List<u8> | none;  // none: 인증 실패
}
```

nonce 는 12 바이트이며, 키나 nonce 의 길이가 틀리면 `ValueError` 입니다. `open` 은 평문을 내놓기 전에 tag 를 constant time 으로 검사합니다.

## 2.21.3. 키 교환

```rok
struct X25519 {                                     // crypto::x25519 (RFC 7748)
    fn public_key(secret: Bytes) throws (ValueError) -> List<u8>;
    fn shared_secret(secret: Bytes, peer_public: Bytes) throws (ValueError) -> List<u8> | none;  // none: 결과가 전부 0
}
struct MlKem768 {                                   // crypto::mlkem (FIPS 203)
    const ENCAPSULATION_KEY_LEN: usize = 1184;
    const DECAPSULATION_KEY_LEN: usize = 2400;
    const CIPHERTEXT_LEN: usize = 1088;
    const SHARED_KEY_LEN: usize = 32;
    fn keygen(d: Bytes, z: Bytes) throws (ValueError) -> (List<u8>, List<u8>);       // 32-byte seed 로부터 (ek, dk)
    fn encapsulate(ek: Bytes, m: Bytes) throws (ValueError) -> (List<u8>, List<u8>);  // 32-byte m 으로부터 (key, ciphertext)
    fn decapsulate(dk: Bytes, c: Bytes) throws (ValueError) -> List<u8>;              // implicit rejection
}
```

`encapsulate` 와 `decapsulate` 는 FIPS 203 7.2 / 7.3 의 입력 검사를 수행합니다 (`ValueError`).

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
    fn verify_pkcs1v15(self, hash: HashAlgorithm, msg: Bytes, signature: Bytes) -> bool;
    fn verify_pss(self, hash: HashAlgorithm, msg: Bytes, signature: Bytes) -> bool;  // MGF1(hash), salt 길이 = 해시 길이
}
struct RsaPrivateKey {
    fn from_pkcs1_der(der: Bytes) throws (ValueError) -> RsaPrivateKey;   // RSAPrivateKey, 소수 두 개
    fn public_key(self) -> RsaPublicKey;
    fn bits(self) -> usize;
    fn sign_pss(self, hash: HashAlgorithm, msg: Bytes, salt: Bytes) throws (ValueError) -> List<u8>;  // salt: 새 난수, 해시 길이
}
```

Ed25519 검증은 cofactorless 이며, 정규형이 아닌 인코딩과 S ≥ L 을 거부합니다. ECDSA 와 RSA 검증은 인코딩에 엄격합니다 (최소 DER, 정확한 서명 길이).

ECDSA 개인 키는 곡선의 크기 이하인 big-endian 스칼라 `1 <= d < n` 입니다. nonce 는 결정적으로 유도되므로 (RFC 6979, `hash` 를 쓰는 HMAC-DRBG) 서명에 무작위 입력이 필요 없습니다. RSA 서명은 base blinding 을 곁들인 CRT 를 쓰며 (blinding 인수는 `salt`, 메시지, 키에서 유도), 결과를 돌려주기 전에 공개 키로 검사합니다. 일관되지 않은 키는 `ValueError` 입니다. 서명은 비밀에 대한 다른 연산처럼 constant time 으로 수행됩니다.
