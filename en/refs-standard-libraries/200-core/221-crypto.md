# 2.21. `crypto::` — cryptographic algorithms

`rok::core::crypto` provides the algorithms a TLS stack needs, as pure computation: they behave identically on every target, so they live in `rok::core` (0.1). Randomness needs the OS and is `rok::std::crypto::random` (3.10); functions that need random input (keys, nonces, encapsulation) take it as an argument. Operations on secret data run in constant time as 2.21.5 defines it.

Every item below is available at `rok::core::crypto::`; the comment names the submodule that also exports it.

Byte arguments take `Bytes`:

```rok
pub type Bytes = List<u8> | Slice<u8> | str | Secret;   // a str gives its UTF-8 bytes; a Secret its bytes in place
struct Secret {                                     // crypto::secret
    fn zeroed(len: usize) -> Secret;
    fn new(bytes: Bytes) -> Secret;                 // a copy
    fn len(self) -> usize;
    fn as_bytes(self) -> Slice<u8>;                 // the bytes in place, not a copy
    fn wipe(self);                                  // overwrite with zeros now
}
impl Index<usize> for Secret; impl IndexSet<usize> for Secret;   // s[i], s[i] = b (bounds-checked)
impl Sliceable<Range<usize>> for Secret;           // s[a:b]: a view in place
impl Drop for Secret;                               // using release wipes
impl Debug for Secret;                              // "Secret(n bytes)", never the bytes
fn constant_time_eq(a: Bytes, b: Bytes) -> bool;   // time depends only on the lengths
```

A `Secret` is a fixed-length byte string in secret storage ([2.21.5](#2215-security-considerations)). Copies share the bytes, as with `List`, and it is not `Sync`. A view from `as_bytes` or slicing reads the storage itself: it sees later writes and reads zeros once the secret is wiped. `Secret` has no `==`; compare with `constant_time_eq`.

## 2.21.1. Hashes, HMAC, HKDF

```rok
enum HashAlgorithm { Sha256, Sha384, Sha512 }       // crypto::sha2 (FIPS 180-4)
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
    const OUTPUT_LEN: usize;                        // 32 / 64
    fn new() -> Self; fn update(ref self, data: Bytes); fn finish(self) -> List<u8>;
    fn digest(data: Bytes) -> List<u8>;
}
struct Shake128 / Shake256 {
    fn new() -> Self; fn update(ref self, data: Bytes);
    fn squeeze(ref self, n: usize) -> List<u8>;     // the next n output bytes
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

`Digest`, `Hmac` and the SHA-3 / SHAKE states are values: a copy continues independently (a transcript hash read mid-way). Successive `squeeze` calls continue one output stream; `update` after the first `squeeze` panics. `Hmac::verify` compares the full-length tag in constant time. `Hkdf::extract` with an empty salt uses `output_len` zero bytes; `expand` throws `ValueError` for `len > 255 * output_len`.

## 2.21.2. AEAD

```rok
enum AeadAlgorithm { Aes128Gcm, Aes256Gcm, ChaCha20Poly1305 }   // crypto::aead
impl AeadAlgorithm { fn key_len(self) -> usize; fn nonce_len(self) -> usize; fn tag_len(self) -> usize; }
struct Aead {
    fn new(alg: AeadAlgorithm, key: Bytes) throws (ValueError) -> Aead;
    fn algorithm(self) -> AeadAlgorithm;
    fn seal(self, nonce: Bytes, aad: Bytes, plaintext: Bytes) throws (ValueError) -> List<u8>;   // ciphertext ‖ 16-byte tag
    fn open(self, nonce: Bytes, aad: Bytes, sealed: Bytes) throws (ValueError) -> List<u8> | none;  // none: not authentic
    fn wipe(self);                                  // overwrite the key material with zeros
}
impl Drop for Aead;
```

`Aead` keeps its expanded key (AES round keys and the GHASH key, or the ChaCha20 key) in secret storage; copies share it, so `wipe` ends every copy.

AES-GCM follows NIST SP 800-38D with 96-bit nonces, ChaCha20-Poly1305 RFC 8439. Nonces are 12 bytes and tags 16; a wrong key or nonce length is a `ValueError`, and so is a message longer than one nonce allows (the 32-bit block counter must not wrap: 2^36 − 32 bytes for AES-GCM, 2^38 − 64 for ChaCha20-Poly1305). `open` returns `none` for input shorter than a tag and checks the tag in constant time before releasing any plaintext. A nonce must never repeat under one key; `Aead` does not track nonces.

## 2.21.3. Key exchange

```rok
struct X25519 {                                     // crypto::x25519 (RFC 7748)
    fn public_key(secret: Bytes) throws (ValueError) -> List<u8>;
    fn shared_secret(secret: Bytes, peer_public: Bytes) throws (ValueError) -> Secret | none;  // none: all-zero result
}
struct MlKem768 {                                   // crypto::mlkem (FIPS 203)
    const ENCAPSULATION_KEY_LEN: usize = 1184;
    const DECAPSULATION_KEY_LEN: usize = 2400;
    const CIPHERTEXT_LEN: usize = 1088;
    const SHARED_KEY_LEN: usize = 32;
    fn keygen(d: Bytes, z: Bytes) throws (ValueError) -> (List<u8>, Secret);       // (ek, dk) from 32-byte seeds
    fn encapsulate(ek: Bytes, m: Bytes) throws (ValueError) -> (Secret, List<u8>);  // (key, ciphertext) from 32-byte m
    fn decapsulate(dk: Bytes, c: Bytes) throws (ValueError) -> Secret;              // implicit rejection
}
```

X25519 keys are 32 bytes (else `ValueError`); the secret is clamped and the peer's u-coordinate is taken as RFC 7748 5 says (bit 255 ignored, values ≥ p reduced), so a small-order or twist point is not an error: `shared_secret` returns `none` exactly when the result is all zero (RFC 7748 6.1). The ML-KEM functions are FIPS 203's deterministic internal forms (the caller supplies the 32-byte seeds `d`, `z` and `m` from a random source). `encapsulate` and `decapsulate` perform the input checks of FIPS 203 7.2 / 7.3 (`ValueError`); a ciphertext that does not re-encrypt to itself yields the implicit-rejection key, not an error.

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
    fn modulus(self) -> List<u8>;                   // big-endian, no leading zero bytes
    fn exponent(self) -> List<u8>;
    fn verify_pkcs1v15(self, hash: HashAlgorithm, msg: Bytes, signature: Bytes) -> bool;
    fn verify_pss(self, hash: HashAlgorithm, msg: Bytes, signature: Bytes, salt_len: usize | none = none) -> bool;  // MGF1(hash); none: salt length = hash length
}
struct RsaPrivateKey {
    fn from_pkcs1_der(der: Bytes) throws (ValueError) -> RsaPrivateKey;   // RSAPrivateKey, two primes
    fn public_key(self) -> RsaPublicKey;
    fn bits(self) -> usize;
    fn sign_pkcs1v15(self, hash: HashAlgorithm, msg: Bytes) throws (ValueError) -> List<u8>;  // deterministic
    fn sign_pss(self, hash: HashAlgorithm, msg: Bytes, salt: Bytes) throws (ValueError) -> List<u8>;  // salt: fresh random, hash length
}
```

Ed25519 verification is cofactorless and rejects non-canonical encodings of `A` and `R` and `S ≥ L`. ECDSA verification rejects a public key that is not an uncompressed point on the curve with coordinates below p, and signatures that are not minimal DER or have `r` or `s` outside `[1, n)`. RSA verification requires a signature of exactly the modulus length below the modulus; PKCS #1 v1.5 compares the whole rebuilt encoding. `RsaPublicKey::new` takes an odd modulus of 2048 to 4096 bits and an odd exponent with `3 ≤ e < 2^33` (leading zero bytes ignored); anything else is a `ValueError` (verification costs one modular squaring per exponent bit, so a peer must not choose a long one).

An ECDSA private key is a big-endian scalar `1 <= d < n` of at most the curve's size; the nonce is derived deterministically (RFC 6979, HMAC-DRBG with `hash`), so signing needs no random input. RSA signing uses the CRT with base blinding (the blinding factor is derived from the key, the message and, for PSS, `salt`, so PKCS #1 v1.5 signing needs no random input) and checks the result against the public key before returning it; an inconsistent key is a `ValueError`. PSS verification accepts any salt length that fits the modulus (`salt_len`); signing uses a salt as long as the hash.

## 2.21.5. Security considerations

**Constant time.** For the operations on secrets — AES (encryption and key schedule), GHASH, ChaCha20, Poly1305, tag comparison, X25519, Ed25519 signing, ECDSA signing and key derivation, RSA private-key operations, ML-KEM key generation and decapsulation, HMAC and HKDF — no branch and no memory index depends on secret data. AES is bitsliced (no table lookups), GHASH multiplies with integer products of sparse operands (no tables), scalar multiplications read their precomputed tables by scanning every entry, inversions are exponentiations with a public exponent (ECDSA, Ed25519) or a fixed-length binary GCD (RSA blinding), RSA exponentiation squares and multiplies for every exponent bit over the prime's full width, ML-KEM reduces by multiply-and-shift instead of division, and conditional selections are masks the compiler cannot prove to be 0 or all ones, so it cannot turn them back into branches. Running time depends on public values only: lengths, the algorithm, curve and key sizes, and outcomes that are public anyway (a tag or signature that does not verify, an RFC 6979 candidate nonce out of range). ML-KEM decapsulation takes the same path whether or not the ciphertext re-encrypts; the implicit-rejection key is selected by mask.

The guarantee has limits an implementation states and does not hide:
- it assumes the target's 32 × 32 → 64-bit integer multiply and its shifts and rotates run in constant time (true of current x86-64, AArch64, ARMv7-A and i686 cores; not of some microcontrollers and older embedded cores), and on wasm targets it is only as good as the engine's code generation;
- it covers timing visible through control flow and memory addresses, not power, electromagnetic or fault-injection attacks, nor speculative-execution side channels;
- parsing (DER, PEM, key files) and verification of public data are variable time;
- it is a property of the implementation as compiled by the shipped toolchain, not a guarantee the language gives: user code written in the same style is not covered.

**Key material in memory.** Secret bytes live in *secret storage*: a buffer the collector allocates in the large-object space ([refs-lang 7.1.2](../../refs-lang/007-runtime/701-object-layout.md#712-memory-spaces)), never moves or copies, and overwrites with zeros before it releases the memory. `Secret` (2.21) is its public form.

- The bytes of a `Secret` exist in exactly one place in the heap. They are overwritten with zeros by `wipe`, when a `using` binding releases the `Secret` (directly or through drop glue), and at the latest by the collection that frees the buffer. The zeroing uses stores the optimizer may not remove.
- The algorithms keep key-dependent state only in secret storage or in locals, never in ordinary heap objects. That state is the AES round keys, GHASH key and ChaCha20 key (`Aead`), HMAC and HKDF (padded key, chaining values, output blocks), the X25519 and ML-KEM shared keys, the ML-KEM decapsulation key with its secret polynomials, seeds and messages, and Ed25519's expanded key and nonce hash. Functions whose result is key material return a `Secret`. Temporaries in secret storage are wiped before the function returns, unless it throws; then the collection wipes them.
- `net::tls` ([3.2.5](../300-std/302-net.md#325-tls-13--nettls)) keeps its traffic, handshake and resumption secrets, record keys, key-exchange secrets, ticket PSKs and Ed25519 / EC private keys in secret storage, and wipes them on KeyUpdate, at the end of the handshake and on close.

The guarantee has limits an implementation states and does not hide:

- **Registers and the stack.** Locals live in registers and on the thread's stack and are not cleared. This includes fixed-size arrays and struct values such as `Digest`, `Hmac`, the SHA-3 and cipher states, and the field elements and scalars of X25519, Ed25519 and ECDSA. Spills the compiler makes are not cleared either; a slot is overwritten only when the stack is reused. There is no code generation that clears them, and Rust's `zeroize`, Go's `crypto/subtle` helpers and libsodium's `sodium_memzero` likewise reach only the memory they are given, not compiler-made copies.
- **Ordinary objects.** A value stored in a heap object lives in that object: the collector may copy it during evacuation and frees it without clearing. This covers a `List` element, a field of a GC-held struct, a closure environment, and an `async` / generator frame across a suspension point ([refs-lang 7.8.2](../../refs-lang/007-runtime/708-coroutine.md#782-coroutine-frame)). Copying a `Secret`'s bytes out (`as_bytes().to_list()`, a `str`, formatting) makes an ordinary copy.
- **Ordinary inputs and outputs.** Not covered: keys passed as `List<u8>` or `str`, plaintext and ciphertext, `Hmac` tags, `Digest` outputs, and decoded key files (PEM / DER decoding produces ordinary lists before `PrivateKey` copies the key).
- **Signing.** ECDSA and RSA signing are not covered: the RFC 6979 state and `bignum` limbs are ordinary lists, and `RsaPrivateKey` holds its values in ordinary lists.
- **Not locked in RAM.** Secret storage is not locked (`mlock` / `VirtualLock`). It may be written to swap or a hibernation file and appears in core dumps and process snapshots. Locking is per page, limited per process (RLIMIT_MEMLOCK), and undone per page, so it would need a page-granular allocator of its own.
- **Process exit and wasm.** Memory released by process exit is not wiped. On wasm targets the host can read linear memory at any time.

**Randomness.** The algorithms take every random input as an argument; callers draw it from `rok::std::crypto::random` (3.10), the operating system's CSPRNG. The ML-KEM seeds, X25519 and Ed25519 secrets, ECDSA private keys and PSS salts must be fresh output of such a source.
