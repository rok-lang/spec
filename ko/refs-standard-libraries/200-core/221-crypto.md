# 2.21. `crypto::`: 암호 알고리듬

`rok::core::crypto`는 TLS 스택에 필요한 알고리듬을 순수 계산으로 구현합니다. 모든 타깃에서 똑같이 동작하므로 `rok::core`(0.1)에 둡니다. 난수는 OS가 필요하므로 `rok::std::crypto::random`(3.10)에 있습니다. 무작위 입력이 필요한 함수(키, nonce, 캡슐화)는 그 입력을 인자로 받습니다. 비밀 데이터를 다루는 연산은 2.21.5가 정의하는 대로 상수 시간(constant time)에 실행됩니다.

아래 항목은 모두 `rok::core::crypto::`에서 쓸 수 있습니다. 주석은 그 항목을 함께 내보내는 하위 모듈을 가리킵니다.

바이트 인자는 `Bytes`를 받습니다.

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

`Secret`은 비밀 저장소([2.21.5](#2215-보안-고려-사항))에 있는 고정 길이 바이트열입니다. 복사본은 `List`처럼 바이트를 공유하고, `Sync`가 아닙니다. `as_bytes`나 슬라이싱으로 얻은 뷰는 저장소 자체를 읽습니다. 그래서 이후의 쓰기가 보이고, secret이 지워지면 0을 읽습니다. `Secret`에는 `==`가 없으므로 `constant_time_eq`로 비교합니다.

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

`Digest`, `Hmac`, SHA-3 / SHAKE 상태는 값입니다. 복사본은 각자 독립적으로 이어 갑니다(중간에 읽는 transcript 해시). 연속된 `squeeze` 호출은 하나의 출력 스트림을 이어 갑니다. 첫 `squeeze` 뒤에 `update`를 부르면 panic이 일어납니다. `Hmac::verify`는 전체 길이의 tag를 상수 시간에 비교합니다. `Hkdf::extract`는 salt가 비어 있으면 0인 바이트 `output_len`개를 씁니다. `expand`는 `len > 255 * output_len`이면 `ValueError`를 던집니다.

## 2.21.2. AEAD

```rok
enum AeadAlgorithm { Aes128Gcm, Aes256Gcm, ChaCha20Poly1305 }   // crypto::aead
impl AeadAlgorithm { fn key_len(self) -> usize; fn nonce_len(self) -> usize; fn tag_len(self) -> usize; }
struct Aead {
    fn new(alg: AeadAlgorithm, key: Bytes) throws (ValueError) -> Aead;
    fn algorithm(self) -> AeadAlgorithm;
    fn seal(self, nonce: Bytes, aad: Bytes, plaintext: Bytes) throws (ValueError) -> List<u8>;   // ciphertext ‖ 16-byte tag
    fn open(self, nonce: Bytes, aad: Bytes, ciphertext_and_tag: Bytes) throws (ValueError) -> List<u8> | none;  // none: 인증 실패
    fn wipe(self);                                  // 키 자료를 0 으로 덮어씀
}
impl Drop for Aead;
```

`Aead`는 확장한 키(AES 라운드 키와 GHASH 키, 또는 ChaCha20 키)를 비밀 저장소에 둡니다. 복사본은 이 키를 공유하므로 `wipe`는 모든 복사본을 끝냅니다.

AES-GCM은 96비트 nonce를 쓰는 NIST SP 800-38D를 따르고, ChaCha20-Poly1305는 RFC 8439를 따릅니다. nonce는 12바이트, tag는 16바이트입니다. 키나 nonce의 길이가 틀리면 `ValueError`입니다. nonce 하나가 허용하는 것보다 긴 메시지도 마찬가지입니다. 32비트 블록 카운터가 한 바퀴 돌아 넘치면 안 되기 때문입니다(AES-GCM은 2^36 − 32바이트, ChaCha20-Poly1305는 2^38 − 64바이트). `open`은 tag보다 짧은 입력에 `none`을 돌려줍니다. 평문을 내놓기 전에 tag를 상수 시간에 검사합니다. 한 키 아래에서 nonce를 반복하면 안 됩니다. `Aead`는 nonce를 추적하지 않습니다.

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

X25519 키는 32바이트입니다(아니면 `ValueError`). secret은 clamp하고, 상대의 u 좌표는 RFC 7748 5절대로 받습니다(비트 255 무시, p 이상의 값은 법 p로 줄임). 그래서 small-order 점이나 twist 점은 에러가 아닙니다. `shared_secret`은 결과가 전부 0일 때만 `none`을 돌려줍니다(RFC 7748 6.1). ML-KEM 함수는 FIPS 203의 결정적 내부 형태입니다. 호출자가 32바이트 seed `d`, `z`, `m`을 난수 원천에서 꺼내 줍니다. `encapsulate`와 `decapsulate`는 FIPS 203 7.2 / 7.3의 입력 검사를 합니다(`ValueError`). 다시 암호화했을 때 자기 자신이 되지 않는 암호문은 에러가 아니라 implicit-rejection 키를 냅니다.

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

Ed25519 검증은 cofactorless입니다. `A`와 `R`의 정규형이 아닌 인코딩과 `S ≥ L`을 거부합니다. ECDSA 검증은 다음을 거부합니다. 곡선 위의 비압축 점이 아니거나 좌표가 p 이상인 공개 키, 그리고 최소 DER이 아니거나 `r` 또는 `s`가 `[1, n)` 밖인 서명입니다. RSA 검증은 modulus보다 작고 길이가 정확히 modulus 길이인 서명을 요구합니다. PKCS #1 v1.5는 다시 만든 인코딩 전체를 비교합니다. `RsaPublicKey::new`는 2048~4096비트의 홀수 modulus와 `3 ≤ e < 2^33`인 홀수 지수를 받습니다(앞의 0 바이트는 무시). 그 밖은 `ValueError`입니다. 검증은 지수 비트마다 모듈러 제곱이 한 번 들므로, 상대가 긴 지수를 고를 수 있으면 안 되기 때문입니다.

ECDSA 개인 키는 곡선의 크기 이하인 빅 엔디언 스칼라 `1 <= d < n`입니다. nonce는 결정적으로 유도하므로(RFC 6979, `hash`를 쓰는 HMAC-DRBG) 서명에 무작위 입력이 필요 없습니다. RSA 서명은 base blinding을 곁들인 CRT를 씁니다. blinding 인수는 키, 메시지, 그리고 PSS이면 `salt`에서 유도합니다. 그래서 PKCS #1 v1.5 서명에는 무작위 입력이 필요 없습니다. RSA 서명은 결과를 돌려주기 전에 공개 키로 검사합니다. 일관되지 않은 키는 `ValueError`입니다. PSS 검증은 modulus에 들어가는 salt 길이(`salt_len`)라면 무엇이든 받습니다. 서명은 해시 길이만큼의 salt를 씁니다.

## 2.21.5. 보안 고려 사항

**Constant time.** 비밀을 다루는 다음 연산에서는 어떤 분기도, 어떤 메모리 인덱스도 비밀 데이터에 의존하지 않습니다.

- AES(암호화와 키 스케줄), GHASH, ChaCha20, Poly1305, tag 비교
- X25519, Ed25519 서명, ECDSA 서명과 키 유도, RSA 개인 키 연산
- ML-KEM 키 생성과 decapsulation, HMAC, HKDF

구현 방식은 다음과 같습니다.

- AES는 bitslice로 구현합니다(테이블 조회 없음).
- GHASH는 성긴 피연산자의 정수 곱으로 곱합니다(테이블 없음).
- 스칼라 곱은 미리 계산한 테이블의 모든 항목을 훑어 읽습니다.
- 역원은 공개 지수의 거듭제곱(ECDSA, Ed25519)이나 고정 길이 binary GCD(RSA blinding)로 구합니다.
- RSA 거듭제곱은 소수의 전체 폭에 걸쳐 지수 비트마다 제곱과 곱셈을 합니다.
- ML-KEM은 나눗셈 대신 곱셈과 시프트로 나머지를 줄입니다.
- 조건부 선택은 마스크로 합니다. 컴파일러는 이 마스크가 0인지 모두 1인지 증명할 수 없으므로 다시 분기로 바꿀 수 없습니다.

실행 시간은 공개 값에만 의존합니다. 길이, 알고리듬, 곡선과 키 크기, 그리고 어차피 공개되는 결과(검증되지 않는 tag나 서명, 범위 밖의 RFC 6979 후보 nonce)입니다. ML-KEM decapsulation은 암호문이 다시 암호화되든 아니든 같은 경로를 밟습니다. implicit-rejection 키는 마스크로 고릅니다.

이 보장에는 구현이 밝히고 숨기지 않는 한계가 있습니다.
- 타깃의 32 × 32 → 64비트 정수 곱셈과 시프트, 회전이 상수 시간에 돈다고 가정합니다. 현재의 x86-64, AArch64, ARMv7-A, i686 코어에서는 참이고, 일부 마이크로컨트롤러와 옛 임베디드 코어에서는 거짓입니다. wasm 타깃에서는 엔진의 코드 생성만큼만 유효합니다.
- 제어 흐름과 메모리 주소로 드러나는 시간만 다룹니다. 전력, 전자기, fault injection 공격이나 투기 실행 부채널(side channel)은 다루지 않습니다.
- 파싱(DER, PEM, 키 파일)과 공개 데이터의 검증은 가변 시간입니다.
- 이 성질은 배포된 toolchain으로 컴파일한 구현의 성질이지 언어가 주는 보장이 아닙니다. 같은 방식으로 쓴 사용자 코드에는 해당하지 않습니다.

**메모리 안의 키 자료.** 비밀 바이트는 **비밀 저장소**에 있습니다. 비밀 저장소는 수집기가 large-object space([refs-lang 7.1.2](../../refs-lang/007-runtime/701-object-layout.md#712-메모리-공간))에 할당하는 버퍼입니다. 수집기는 이 버퍼를 옮기거나 복사하지 않고, 메모리를 놓기 전에 0으로 덮어씁니다. `Secret`(2.21)은 그 공개 형태입니다.

- `Secret`의 바이트는 힙에서 정확히 한 곳에만 있습니다. 이 바이트를 0으로 덮어쓰는 때는 세 가지입니다. `wipe`를 부를 때, `using` 바인딩이 `Secret`을 해제할 때(직접 또는 drop glue를 거쳐), 그리고 늦어도 그 버퍼를 해제하는 수집 때입니다. 0으로 지우는 쓰기는 최적화기가 없앨 수 없는 store입니다.
- 알고리듬은 키에 의존하는 상태를 비밀 저장소나 지역 변수에만 둡니다. 보통의 힙 객체에는 두지 않습니다. 그 상태는 다음과 같습니다. AES 라운드 키, GHASH 키, ChaCha20 키(`Aead`), HMAC과 HKDF(패딩한 키, chaining 값, 출력 블록), X25519와 ML-KEM의 공유 키, ML-KEM decapsulation 키와 그 비밀 다항식, seed, 메시지, 그리고 Ed25519의 확장 키와 nonce 해시입니다. 결과가 키 자료인 함수는 `Secret`을 돌려줍니다. 비밀 저장소의 임시 값은 함수가 반환하기 전에 지웁니다. 함수가 에러를 던지면 수집이 지웁니다.
- `net::tls`([3.2.5](../300-std/302-net.md#325-tls-13-nettls))는 traffic, handshake, resumption secret, record 키, 키 교환 secret, ticket PSK, Ed25519 / EC 개인 키를 비밀 저장소에 둡니다. 그리고 KeyUpdate 때, 핸드셰이크가 끝날 때, close 때 지웁니다.

이 보장에는 구현이 밝히고 숨기지 않는 한계가 있습니다.

- **레지스터와 스택.** 지역 변수는 레지스터와 스레드의 스택에 있고 지우지 않습니다. 고정 크기 배열, `Digest`, `Hmac`, SHA-3와 암호 상태 같은 struct 값, 그리고 X25519, Ed25519, ECDSA의 field element와 스칼라도 여기에 들어갑니다. 컴파일러가 만드는 spill도 지우지 않습니다. 슬롯은 스택을 다시 쓸 때만 덮어써집니다. 이들을 지우는 코드 생성은 없습니다. Rust의 `zeroize`, Go의 `crypto/subtle` 도우미, libsodium의 `sodium_memzero`도 마찬가지로 받은 메모리에만 닿고, 컴파일러가 만든 사본에는 닿지 않습니다.
- **보통의 객체.** 힙 객체에 저장한 값은 그 객체 안에 있습니다. 수집기는 evacuation 중에 그 값을 복사할 수 있고, 지우지 않은 채 해제합니다. `List` 원소, GC가 가진 struct의 필드, 클로저 환경, 중단 지점을 넘는 `async` / 제너레이터 프레임([refs-lang 7.8.2](../../refs-lang/007-runtime/708-coroutine.md#782-코루틴-프레임))이 여기에 해당합니다. `Secret`의 바이트를 꺼내 복사하면(`as_bytes().to_list()`, `str`, 포맷) 보통의 사본이 생깁니다.
- **보통의 입력과 출력.** 다음은 다루지 않습니다. `List<u8>`나 `str`로 넘긴 키, 평문과 암호문, `Hmac` tag, `Digest` 출력, 디코드한 키 파일입니다. PEM / DER 디코드는 `PrivateKey`가 키를 복사하기 전에 보통의 리스트를 만듭니다.
- **서명.** ECDSA와 RSA 서명은 다루지 않습니다. RFC 6979 상태와 `bignum` limb는 보통의 리스트입니다. `RsaPrivateKey`도 값을 보통의 리스트에 둡니다.
- **RAM에 고정하지 않음.** 비밀 저장소는 메모리에 고정하지 않습니다(`mlock` / `VirtualLock`). 스왑이나 최대 절전 파일에 쓰일 수 있고, 코어 덤프와 프로세스 스냅숏에 나타납니다. 고정은 페이지 단위이고, 프로세스마다 제한되며(RLIMIT_MEMLOCK), 페이지 단위로 풀립니다. 그래서 고정하려면 페이지 단위로 할당하는 자체 할당기가 필요합니다.
- **프로세스 종료와 wasm.** 프로세스가 끝나면서 놓이는 메모리는 지우지 않습니다. wasm 타깃에서는 호스트가 언제든 선형 메모리를 읽을 수 있습니다.

**난수.** 알고리듬은 모든 무작위 입력을 인자로 받습니다. 호출자는 무작위 입력을 OS의 CSPRNG인 `rok::std::crypto::random`(3.10)에서 꺼냅니다. ML-KEM seed, X25519와 Ed25519 secret, ECDSA 개인 키, PSS salt는 그런 원천의 새 출력이어야 합니다.
