# 2.21. `crypto::` - cryptographic algorithms

`rok::core::crypto` provides the algorithms a TLS stack needs, as pure computation: they behave identically on every target, so they live in `rok::core` (0.1). Randomness needs the OS and is `rok::std::random` (3.10); functions that need random input (keys, nonces, encapsulation) take it as an argument. Operations on secret data run in constant time as 2.21.5 defines it.

The names that are unique in `rok::core::crypto` (`Secret`, `constant_time_eq`, `HashAlgorithm`, `Digest`, the SHA-3 and SHAKE states, `Hmac`, `AeadAlgorithm`, `Aead`, `EcCurve`) are available at `rok::core::crypto::`; the comment names the submodule that also exports them. Keys and HKDF are reached through their algorithm module only (`crypto::ed25519::SigningKey`, `crypto::hkdf::extract`), since several modules have a `PublicKey`.

Every byte argument is generic over `AsBytes` ([2.11.13](211-conversion.md#21113-asbytes-trait---bytes-as-a-view)): a `str` gives its UTF-8 bytes, a `List<u8>`, `Slice<u8>` or `[u8; N]` its bytes, a `Secret` its bytes in place, and a type of the program's own whatever its `byte_slice` returns. Several byte arguments of one function each have their own type parameter, so a key and a message need not have the same type.

```rok
struct Secret {                                     // crypto::secret
    fn zeroed(len: usize) -> Secret;
    fn new<B: AsBytes>(bytes: B) -> Secret;                 // a copy
    fn len(self) -> usize;
    fn as_bytes(self) -> Slice<u8>;                 // the bytes in place, not a copy
    fn wipe(self);                                  // overwrite with zeros now
}
impl Index<usize> for Secret; impl IndexSet<usize> for Secret;   // s[i], s[i] = b (bounds-checked)
impl Sliceable<Range<usize>> for Secret;           // s[a:b]: a view in place
impl Drop for Secret;                               // using release wipes
impl Debug for Secret;                              // "Secret { len: n, .. }", never the bytes
impl AsBytes for Secret;                            // byte_slice: as_bytes (2.11.13)
fn constant_time_eq<A: AsBytes, B: AsBytes>(a: A, b: B) -> bool;   // time depends only on the lengths
```

A `Secret` is a fixed-length byte string in secret storage ([2.21.5](#2215-security-considerations)). Copies share the bytes, as with `List`, and it is not `Sync`. As for a collection, a hand-off to another thread gives the new owner its own copy, in secret storage, and `par` runs items that reach one `Secret` one at a time ([2.13.9](213-marker.md#2139-send--sync---thread-boundary-markers)). A view from `as_bytes` or slicing reads the storage itself: it sees later writes and reads zeros once the secret is wiped. `Secret` has no `==`; compare with `constant_time_eq`.

## 2.21.1. Hashes, HMAC, HKDF

```rok
#[non_exhaustive]
enum HashAlgorithm { Sha256, Sha384, Sha512, Sha3_256, Sha3_512 }   // crypto (FIPS 180-4, FIPS 202); Copy, Eq, Hash, Debug
impl HashAlgorithm {
    fn output_len(self) -> usize;                   // 32, 48, 64, 32, 64
    fn block_len(self) -> usize;                    // 64, 128, 128; SHA-3: the rate, 136 / 72
    fn digest<B: AsBytes>(self, data: B) -> List<u8>;
}
struct Digest {                                     // crypto::sha2; a running hash, a copy forks it
    fn new(alg: HashAlgorithm) -> Digest;
    fn algorithm(self) -> HashAlgorithm;
    fn update<B: AsBytes>(ref self, data: B);
    fn finish(self) -> List<u8>;
}
struct Sha3_256 / Sha3_512 {                        // crypto::sha3 (FIPS 202), the low-level form
    const OUTPUT_LEN: usize;                        // 32 / 64
    fn new() -> Self; fn update<B: AsBytes>(ref self, data: B); fn finish(self) -> List<u8>;
    fn digest<B: AsBytes>(data: B) -> List<u8>;
}
struct Shake128 / Shake256 {
    fn new() -> Self; fn update<B: AsBytes>(ref self, data: B);
    fn squeeze(ref self, n: usize) -> List<u8>;     // the next n output bytes
}
struct Hmac {                                       // crypto::hmac (RFC 2104)
    fn new<B: AsBytes>(alg: HashAlgorithm, key: B) -> Hmac;
    fn update<B: AsBytes>(ref self, data: B);
    fn finish(self) -> List<u8>;
    fn mac<K: AsBytes, D: AsBytes>(alg: HashAlgorithm, key: K, data: D) -> List<u8>;
    fn verify<K: AsBytes, D: AsBytes, T: AsBytes>(alg: HashAlgorithm, key: K, data: D, tag: T) -> bool;   // constant time
}
mod hkdf {                                          // crypto::hkdf (RFC 5869)
    fn extract<S: AsBytes, I: AsBytes>(alg: HashAlgorithm, salt: S, ikm: I) -> Secret;
    fn expand<P: AsBytes, I: AsBytes>(alg: HashAlgorithm, prk: P, info: I, len: usize) throws (ValueError) -> Secret;
}
```

`HashAlgorithm` names every hash function that takes part in the keyed and signing APIs: `Digest`, `Hmac`, `hkdf`, ECDSA and RSA (PKCS #1 v1.5 and PSS, MGF1 included) accept each variant. It is `#[non_exhaustive]` ([refs-lang 6.2.10](../../refs-lang/006-meta-system/602-built-in-attributes.md#6210-non_exhaustive-opting-out-of-external-exhaustiveness)): a `match` on it outside `rok::core` needs a `_` arm, and later releases may add functions. For a SHA-3 function HMAC's block is the sponge rate. `Sha3_256`, `Sha3_512`, `Shake128` and `Shake256` are the low-level form: one function fixed at compile time each (SHAKE has no `HashAlgorithm` variant, its output length being the caller's); `Sha3_256::digest(x)` and `HashAlgorithm::Sha3_256.digest(x)` compute the same function.

`Digest`, `Hmac` and the SHA-3 / SHAKE states are values: a copy continues independently (a transcript hash read mid-way). Successive `squeeze` calls continue one output stream; `update` after the first `squeeze` panics. `Hmac::verify` compares the full-length tag in constant time. `hkdf::extract` with an empty salt uses `output_len` zero bytes; `hkdf::expand` throws `ValueError` for `len > 255 * output_len`.

## 2.21.2. AEAD

```rok
#[non_exhaustive]
enum AeadAlgorithm { Aes128Gcm, Aes256Gcm, ChaCha20Poly1305 }   // crypto::aead; Copy, Eq, Hash, Debug
impl AeadAlgorithm { fn key_len(self) -> usize; fn nonce_len(self) -> usize; fn tag_len(self) -> usize; }
struct Aead {
    fn new<B: AsBytes>(alg: AeadAlgorithm, key: B) throws (ValueError) -> Aead;
    fn algorithm(self) -> AeadAlgorithm;
    fn seal<N: AsBytes, A: AsBytes, P: AsBytes>(self, nonce: N, aad: A, plaintext: P) throws (ValueError) -> List<u8>;   // ciphertext ‖ 16-byte tag
    fn open<N: AsBytes, A: AsBytes, C: AsBytes>(self, nonce: N, aad: A, ciphertext_and_tag: C) throws (ValueError) -> List<u8> | none;  // none: not authentic
    fn wipe(self);                                  // overwrite the key material with zeros
}
impl Drop for Aead;
```

`Aead` keeps its expanded key (AES round keys and the GHASH key, or the ChaCha20 key) in secret storage; copies share it, so `wipe` ends every copy.

AES-GCM follows NIST SP 800-38D with 96-bit nonces, ChaCha20-Poly1305 RFC 8439. Nonces are 12 bytes and tags 16; a wrong key or nonce length is a `ValueError`, and so is a message longer than one nonce allows (the 32-bit block counter must not wrap: 2^36 − 32 bytes for AES-GCM, 2^38 − 64 for ChaCha20-Poly1305). `open` returns `none` for input shorter than a tag and checks the tag in constant time before releasing any plaintext. A nonce must never repeat under one key; `Aead` does not track nonces.

## 2.21.3. Key exchange

```rok
mod x25519 {                                        // crypto::x25519 (RFC 7748)
    struct PrivateKey {                             // 32 bytes in secret storage
        fn from_bytes<B: AsBytes>(bytes: B) throws (ValueError) -> PrivateKey;
        fn to_secret(self) -> Secret;               // a copy of the 32 bytes
        fn public_key(self) -> PublicKey;
        fn diffie_hellman(self, peer: PublicKey) -> Secret | none;   // none: all-zero result
        fn wipe(self);
    }
    impl Drop for PrivateKey; impl Debug for PrivateKey;   // release wipes; Debug: PrivateKey { .. }
    struct PublicKey {                              // Copy, Eq, Hash, Debug
        fn from_bytes<B: AsBytes>(bytes: B) throws (ValueError) -> PublicKey;
        fn to_bytes(self) -> [u8; 32];
    }
}
mod mlkem {                                         // crypto::mlkem (FIPS 203)
    const SHARED_KEY_LEN: usize = 32;
    struct DecapsulationKey768 {                    // in secret storage
        fn from_seed<D: AsBytes, Z: AsBytes>(d: D, z: Z) throws (ValueError) -> DecapsulationKey768;   // 32-byte seeds
        fn from_expanded_bytes<B: AsBytes>(bytes: B) throws (ValueError) -> DecapsulationKey768;   // 2400 bytes
        fn to_expanded_bytes(self) -> Secret;
        fn encapsulation_key(self) -> EncapsulationKey768;
        fn decapsulate<C: AsBytes>(self, c: C) throws (ValueError) -> Secret;   // implicit rejection
        fn wipe(self);
    }
    impl Drop for DecapsulationKey768; impl Debug for DecapsulationKey768;   // DecapsulationKey768 { .. }
    struct EncapsulationKey768 {                    // Clone, Eq, Hash, Debug
        const LEN: usize = 1184;
        const CIPHERTEXT_LEN: usize = 1088;
        fn from_bytes<B: AsBytes>(bytes: B) throws (ValueError) -> EncapsulationKey768;
        fn to_bytes(self) -> List<u8>;
        fn encapsulate<M: AsBytes>(self, m: M) throws (ValueError) -> (Secret, List<u8>);   // (key, ciphertext) from 32-byte m
    }
}
```

X25519 keys are 32 bytes (else `ValueError`); the private key is clamped when used and the peer's u-coordinate is taken as RFC 7748 5 says (bit 255 ignored, values ≥ p reduced), so a small-order or twist point is not an error: `diffie_hellman` returns `none` exactly when the result is all zero (RFC 7748 6.1). The ML-KEM operations are FIPS 203's deterministic internal forms: the caller supplies the 32-byte seeds `d`, `z` and `m` from a random source. A decapsulation key is made from its seeds (`from_seed`) or, the low-level form, from its 2400-byte expanded encoding `dk_PKE ‖ ek ‖ H(ek) ‖ z` (`from_expanded_bytes`, FIPS 203 7.3), whose stored `H(ek)` must match (`ValueError`). `EncapsulationKey768::from_bytes` performs the input checks of FIPS 203 7.2 (length and modulus, `ValueError`), `decapsulate` the ciphertext length check of 7.3; a ciphertext that does not re-encrypt to itself yields the implicit-rejection key, not an error.

## 2.21.4. Signatures

```rok
mod ed25519 {                                       // crypto::ed25519 (RFC 8032)
    struct SigningKey {                             // the seed and its expansion in secret storage
        fn from_seed<B: AsBytes>(seed: B) throws (ValueError) -> SigningKey;   // 32 bytes
        fn seed(self) -> Secret;                    // a copy
        fn verifying_key(self) -> VerifyingKey;
        fn sign<B: AsBytes>(self, msg: B) -> List<u8>;   // 64 bytes, deterministic
        fn wipe(self);
    }
    impl Drop for SigningKey; impl Debug for SigningKey;   // SigningKey { verifying_key: .., .. }
    struct VerifyingKey {                           // Copy, Eq, Hash, Debug
        fn from_bytes<B: AsBytes>(bytes: B) throws (ValueError) -> VerifyingKey;   // 32 bytes, a canonical point
        fn to_bytes(self) -> [u8; 32];
        fn verify<M: AsBytes, S: AsBytes>(self, msg: M, sig: S) -> bool;
    }
}
#[non_exhaustive]
enum EcCurve { P256, P384 }                         // crypto::ecdsa (FIPS 186-5); Copy, Eq, Hash, Debug
mod ecdsa {
    struct SigningKey {                             // the scalar in secret storage
        fn from_bytes<B: AsBytes>(curve: EcCurve, bytes: B) throws (ValueError) -> SigningKey;   // big-endian scalar
        fn curve(self) -> EcCurve;
        fn verifying_key(self) -> VerifyingKey;
        fn sign<M: AsBytes>(self, hash: HashAlgorithm, msg: M) -> List<u8>;   // DER, RFC 6979 nonce
        fn wipe(self);
    }
    impl Drop for SigningKey; impl Debug for SigningKey;   // SigningKey { curve: P256, .. }
    struct VerifyingKey {                           // Copy, Eq, Hash, Debug
        fn from_sec1<B: AsBytes>(curve: EcCurve, bytes: B) throws (ValueError) -> VerifyingKey;   // SEC1 uncompressed
        fn curve(self) -> EcCurve;
        fn to_sec1(self) -> List<u8>;
        fn verify<M: AsBytes, S: AsBytes>(self, hash: HashAlgorithm, msg: M, signature: S) -> bool;   // DER signature
    }
}
mod rsa {                                           // crypto::rsa (RFC 8017), 2048..4096 bits
    struct PublicKey {                              // Clone, Eq, Hash, Debug
        fn new<N: AsBytes, E: AsBytes>(n: N, e: E) throws (ValueError) -> PublicKey;
        fn from_pkcs1_der<B: AsBytes>(der: B) throws (ValueError) -> PublicKey;
        fn bits(self) -> usize;
        fn modulus(self) -> List<u8>;               // big-endian, no leading zero bytes
        fn exponent(self) -> List<u8>;
        fn verify_pkcs1v15<M: AsBytes, S: AsBytes>(self, hash: HashAlgorithm, msg: M, signature: S) -> bool;
        fn verify_pss<M: AsBytes, S: AsBytes>(self, hash: HashAlgorithm, msg: M, signature: S, salt_len: usize | none = none) -> bool;  // MGF1(hash); none: salt length = hash length
    }
    struct PrivateKey {
        fn from_pkcs1_der<B: AsBytes>(der: B) throws (ValueError) -> PrivateKey;   // RSAPrivateKey, two primes
        fn public_key(self) -> PublicKey;
        fn bits(self) -> usize;
        fn sign_pkcs1v15<B: AsBytes>(self, hash: HashAlgorithm, msg: B) throws (ValueError) -> List<u8>;  // deterministic
        fn sign_pss<M: AsBytes, S: AsBytes>(self, hash: HashAlgorithm, msg: M, salt: S) throws (ValueError) -> List<u8>;  // salt: fresh random, hash length
        fn wipe(self);
    }
    impl Drop for PrivateKey; impl Debug for PrivateKey;   // PrivateKey { bits: 2048, .. }
}
```

The private and signing keys hold their secret material in secret storage ([2.21.5](#2215-security-considerations)): copies share it, they are not `Sync`, `wipe` and a `using` release overwrite it with zeros, and `Debug` shows no key bytes: it prints the type in the record form `Type { field: .., .. }` with only public facts (length, curve, size, the public key) and `..` for the rest ([0.5.12](../000-meta/005-notation.md#0512-trait-coverage-non_exhaustive-and-sealed)). A key is checked once, when it is made, so Ed25519 and ECDSA signing cannot fail. Keys are made from bytes; a source of randomness is the caller's (`SigningKey::from_seed(random::secret(32))`, 3.10).

Ed25519 verification is cofactorless and rejects a non-canonical encoding of `R` and `S ≥ L`; `ed25519::VerifyingKey::from_bytes` rejects a non-canonical `A` or one off the curve. `ecdsa::VerifyingKey::from_sec1` rejects a public key that is not an uncompressed point on the curve with coordinates below p, and ECDSA verification rejects signatures that are not minimal DER or have `r` or `s` outside `[1, n)`. RSA verification requires a signature of exactly the modulus length below the modulus; PKCS #1 v1.5 compares the whole rebuilt encoding. `rsa::PublicKey::new` takes an odd modulus of 2048 to 4096 bits and an odd exponent with `3 ≤ e < 2^33` (leading zero bytes ignored); anything else is a `ValueError` (verification costs one modular squaring per exponent bit, so a peer must not choose a long one).

An ECDSA signing key is a big-endian scalar `1 <= d < n` of at most the curve's size (else `ValueError`); the nonce is derived deterministically (RFC 6979, HMAC-DRBG with `hash`), so signing needs no random input. RSA signing uses the CRT with base blinding (the blinding factor is derived from the key, the message and, for PSS, `salt`, so PKCS #1 v1.5 signing needs no random input) and checks the result against the public key before returning it; an inconsistent key is a `ValueError`. PSS verification accepts any salt length that fits the modulus (`salt_len`); signing uses a salt as long as the hash.

## 2.21.5. Security considerations

**Constant time.** No branch and no memory index depends on secret data in the operations on secrets: AES (encryption and key schedule), GHASH, ChaCha20, Poly1305, tag comparison, X25519, Ed25519 signing, ECDSA signing and key derivation, RSA private-key operations, ML-KEM key generation and decapsulation, HMAC and HKDF. AES is bitsliced (no table lookups), GHASH multiplies with integer products of sparse operands (no tables), scalar multiplications read their precomputed tables by scanning every entry, inversions are exponentiations with a public exponent (ECDSA, Ed25519) or a fixed-length binary GCD (RSA blinding), RSA exponentiation squares and multiplies for every exponent bit over the prime's full width, ML-KEM reduces by multiply-and-shift instead of division, and conditional selections are masks the compiler cannot prove to be 0 or all ones, so it cannot turn them back into branches. Running time depends on public values only: lengths, the algorithm, curve and key sizes, and outcomes that are public anyway (a tag or signature that does not verify, an RFC 6979 candidate nonce out of range). ML-KEM decapsulation takes the same path whether or not the ciphertext re-encrypts; the implicit-rejection key is selected by mask.

The guarantee has limits an implementation states and does not hide:
- it assumes the target's 32 × 32 → 64-bit integer multiply and its shifts and rotates run in constant time (true of current x86-64, AArch64, ARMv7-A and i686 cores; not of some microcontrollers and older embedded cores), and on wasm targets it is only as good as the engine's code generation;
- it covers timing visible through control flow and memory addresses, not power, electromagnetic or fault-injection attacks, nor speculative-execution side channels;
- parsing (DER, PEM, key files) and verification of public data are variable time;
- it is a property of the implementation as compiled by the shipped toolchain, not a guarantee the language gives: user code written in the same style is not covered.

**Key material in memory.** Secret bytes live in *secret storage*: a buffer the collector allocates in the large-object space ([refs-lang 7.1.2](../../refs-lang/007-runtime/701-object-layout.md#712-memory-spaces)), never moves or copies, and overwrites with zeros before it releases the memory. `Secret` (2.21) is its public form.

- The bytes of a `Secret` exist in exactly one place in the heap; a hand-off to another thread (2.13.9) makes a second `Secret` in secret storage, wiped by the same rules. They are overwritten with zeros by `wipe`, when a `using` binding releases the `Secret` (directly or through drop glue), and at the latest by the collection that frees the buffer. The zeroing uses stores the optimizer may not remove.
- The algorithms keep key-dependent state only in secret storage or in locals, never in ordinary heap objects. That state is the AES round keys, GHASH key and ChaCha20 key (`Aead`), HMAC and HKDF (padded key, chaining values, output blocks), the X25519 private key and the X25519 and ML-KEM shared keys, the ML-KEM decapsulation key with its secret polynomials, seeds and messages, Ed25519's seed, expanded key and nonce hash, the ECDSA scalar and RSA's primes, CRT exponents and `q^-1`. Functions whose result is key material return a `Secret`. Temporaries in secret storage are wiped before the function returns, unless it throws; then the collection wipes them.
- `net::tls` ([3.2.5](../300-std/302-net.md#325-tls-13---nettls)) keeps its traffic, handshake and resumption secrets, record keys, key-exchange secrets, ticket PSKs and Ed25519 / EC private keys in secret storage, and wipes them on KeyUpdate, at the end of the handshake and on close.

The guarantee has limits an implementation states and does not hide:

- **Registers and the stack.** Locals live in registers and on the thread's stack and are not cleared. This includes fixed-size arrays and struct values such as `Digest`, `Hmac`, the SHA-3 and cipher states, and the field elements and scalars of X25519, Ed25519 and ECDSA. Spills the compiler makes are not cleared either; a slot is overwritten only when the stack is reused. There is no code generation that clears them, and Rust's `zeroize`, Go's `crypto/subtle` helpers and libsodium's `sodium_memzero` likewise reach only the memory they are given, not compiler-made copies.
- **Ordinary objects.** A value stored in a heap object lives in that object: the collector may copy it during evacuation and frees it without clearing. This covers a `List` element, a field of a GC-held struct, a closure environment, and an `async` / generator frame across a suspension point ([refs-lang 7.8.2](../../refs-lang/007-runtime/708-coroutine.md#782-coroutine-frame)). Copying a `Secret`'s bytes out (`as_bytes().to_list()`, a `str`, formatting) makes an ordinary copy.
- **Ordinary inputs and outputs.** Not covered: keys passed as `List<u8>` or `str`, plaintext and ciphertext, `Hmac` tags, `Digest` outputs, and decoded key files (PEM / DER decoding produces ordinary lists before `PrivateKey` copies the key).
- **Signing.** ECDSA and RSA signing are not covered: the RFC 6979 state and `bignum` limbs are ordinary lists, and an RSA operation computes on ordinary lists (`rsa::PrivateKey` keeps its primes, CRT exponents and `q^-1` in secret storage and rebuilds the moduli for each operation).
- **Not locked in RAM.** Secret storage is not locked (`mlock` / `VirtualLock`). It may be written to swap or a hibernation file and appears in core dumps and process snapshots. Locking is per page, limited per process (RLIMIT_MEMLOCK), and undone per page, so it would need a page-granular allocator of its own.
- **Process exit and wasm.** Memory released by process exit is not wiped. On wasm targets the host can read linear memory at any time.

**Randomness.** The algorithms take every random input as an argument; callers draw it from `rok::std::random` (3.10), the operating system's CSPRNG. The ML-KEM seeds, X25519 and Ed25519 secrets, ECDSA private keys and PSS salts must be fresh output of such a source.
