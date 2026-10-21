# 2.21. `crypto::` — cryptographic algorithms

`rok::core::crypto` provides the algorithms a TLS stack needs, as pure computation: they behave identically on every target, so they live in `rok::core` (0.1). Randomness needs the OS and is `rok::std::crypto::random` (3.10); functions that need random input (keys, nonces, encapsulation) take it as an argument. Operations on secret data run in constant time: no branch and no memory index depends on a secret.

Byte arguments take `Bytes`:

```rok
pub type Bytes = List<u8> | Slice<u8> | str;   // a str gives its UTF-8 bytes
fn constant_time_eq(a: Bytes, b: Bytes) -> bool;   // time depends only on the lengths
```

## 2.21.1. Hashes, HMAC, HKDF

```rok
enum HashAlgorithm { Sha256, Sha384, Sha512 }       // crypto::sha2
impl HashAlgorithm {
    fn output_len(self) -> usize;
    fn block_len(self) -> usize;
    fn digest(self, data: Bytes) -> List<u8>;
}
struct Digest {                                     // a running hash; a copy forks it
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
    fn squeeze(ref self, n: usize) -> List<u8>;     // repeatable
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
    fn open(self, nonce: Bytes, aad: Bytes, sealed: Bytes) throws (ValueError) -> List<u8> | none;  // none: not authentic
}
```

Nonces are 12 bytes; a wrong key or nonce length is a `ValueError`. `open` checks the tag in constant time before releasing any plaintext.

## 2.21.3. Key exchange

```rok
struct X25519 {                                     // crypto::x25519 (RFC 7748)
    fn public_key(secret: Bytes) throws (ValueError) -> List<u8>;
    fn shared_secret(secret: Bytes, peer_public: Bytes) throws (ValueError) -> List<u8> | none;  // none: all-zero result
}
struct MlKem768 {                                   // crypto::mlkem (FIPS 203)
    const ENCAPSULATION_KEY_LEN: usize = 1184;
    const DECAPSULATION_KEY_LEN: usize = 2400;
    const CIPHERTEXT_LEN: usize = 1088;
    const SHARED_KEY_LEN: usize = 32;
    fn keygen(d: Bytes, z: Bytes) throws (ValueError) -> (List<u8>, List<u8>);       // (ek, dk) from 32-byte seeds
    fn encapsulate(ek: Bytes, m: Bytes) throws (ValueError) -> (List<u8>, List<u8>);  // (key, ciphertext) from 32-byte m
    fn decapsulate(dk: Bytes, c: Bytes) throws (ValueError) -> List<u8>;              // implicit rejection
}
```

`encapsulate` and `decapsulate` perform the input checks of FIPS 203 7.2 / 7.3 (`ValueError`).

## 2.21.4. Signatures

```rok
struct Ed25519 {                                    // crypto::ed25519 (RFC 8032)
    fn public_key(seed: Bytes) throws (ValueError) -> List<u8>;
    fn sign(seed: Bytes, msg: Bytes) throws (ValueError) -> List<u8>;
    fn verify(public_key: Bytes, msg: Bytes, sig: Bytes) -> bool;
}
enum EcCurve { P256, P384 }                         // crypto::ecdsa (FIPS 186-5)
struct Ecdsa {
    fn public_key(curve: EcCurve, private_key: Bytes) throws (ValueError) -> List<u8>;  // SEC1 uncompressed
    fn sign(curve: EcCurve, hash: HashAlgorithm, private_key: Bytes, msg: Bytes) throws (ValueError) -> List<u8>;  // DER, RFC 6979 nonce
    fn verify(curve: EcCurve, hash: HashAlgorithm, public_key: Bytes, msg: Bytes, signature: Bytes) -> bool;  // SEC1 uncompressed key, DER signature
}
struct RsaPublicKey {                               // crypto::rsa (RFC 8017), 2048..4096 bits
    fn new(n: Bytes, e: Bytes) throws (ValueError) -> RsaPublicKey;
    fn from_pkcs1_der(der: Bytes) throws (ValueError) -> RsaPublicKey;
    fn bits(self) -> usize;
    fn verify_pkcs1v15(self, hash: HashAlgorithm, msg: Bytes, signature: Bytes) -> bool;
    fn verify_pss(self, hash: HashAlgorithm, msg: Bytes, signature: Bytes) -> bool;  // MGF1(hash), salt length = hash length
}
struct RsaPrivateKey {
    fn from_pkcs1_der(der: Bytes) throws (ValueError) -> RsaPrivateKey;   // RSAPrivateKey, two primes
    fn public_key(self) -> RsaPublicKey;
    fn bits(self) -> usize;
    fn sign_pss(self, hash: HashAlgorithm, msg: Bytes, salt: Bytes) throws (ValueError) -> List<u8>;  // salt: fresh random, hash length
}
```

Ed25519 verification is cofactorless and rejects non-canonical encodings and S ≥ L. ECDSA and RSA verification are strict about encodings (minimal DER, exact signature length).

An ECDSA private key is a big-endian scalar `1 <= d < n` of at most the curve's size; the nonce is derived deterministically (RFC 6979, HMAC-DRBG with `hash`), so signing needs no random input. RSA signing uses the CRT with base blinding (the blinding factor is derived from `salt`, the message and the key) and checks the result against the public key before returning it; an inconsistent key is a `ValueError`. Signing runs in constant time like the other operations on secrets.
