# 2.22. `encoding::`: hex와 base64

`rok::core::encoding`은 바이트를 텍스트로 바꾸고 다시 되돌립니다. 인코딩은 RFC 4648 가운데 프로그램이 가장 자주 만나는 두 가지, hex(base16)와 base64입니다. 순수 계산이고 모든 타깃에서 똑같이 동작하므로 `rok::core`(0.1)에 둡니다. 인코딩마다 모듈이 하나씩 있고(`encoding::hex`, `encoding::base64`), 두 모듈은 같은 네 함수를 가집니다. 둘 다 던지는 에러는 `encoding::DecodeError`입니다.

```rok
// rok::core::encoding::hex
enum Case { Lower, Upper }
fn encode<B: AsBytes>(data: B, case: Case = Case::Lower) -> str;
fn decode<B: AsBytes>(text: B) throws (DecodeError) -> List<u8>;
fn encode_into<B: AsBytes>(data: B, out: SliceMut<u8>, case: Case = Case::Lower) -> usize;
fn decode_into<B: AsBytes>(text: B, out: SliceMut<u8>) throws (DecodeError) -> usize;

// rok::core::encoding::base64
fn encode<B: AsBytes>(data: B, codec: Codec = STANDARD) -> str;
fn decode<B: AsBytes>(text: B, codec: Codec = STANDARD) throws (DecodeError) -> List<u8>;
fn encode_into<B: AsBytes>(data: B, out: SliceMut<u8>, codec: Codec = STANDARD) -> usize;
fn decode_into<B: AsBytes>(text: B, out: SliceMut<u8>, codec: Codec = STANDARD) throws (DecodeError) -> usize;
const fn encoded_len(n: usize, codec: Codec = STANDARD) -> usize;
const fn decoded_len_max(n: usize) -> usize;
```

바이트 인자는 `AsBytes`([2.11.13](211-conversion.md#21113-asbytes-트레잇-바이트-뷰))입니다. 리스트, view, 배열, 또는 문자열의 UTF-8 바이트를 받습니다. `encode`는 새 `str`을, `decode`는 새 `List<u8>`을 돌려줍니다. `_into` 형태는 호출자가 크기를 맞춘 리스트의 쓰기 가능한 view인 `out`([2.9.7](209-collections.md))의 앞쪽에 쓰고, 쓴 바이트 수를 돌려줍니다. 메모리는 할당하지 않습니다. 디코딩한 바이트가 UTF-8 텍스트이면 `str::from_utf8`([2.15.6](215-string.md#2156-utf-8-불변식-프로세스-안의-모든-문자열))로 `str`이 됩니다.

```rok
encoding::hex::encode("hi");                   //→ "6869"
encoding::hex::decode("6869");                 //→ [104, 105]
encoding::base64::encode("foob");              //→ "Zm9vYg=="
encoding::base64::decode("Zm9vYg");            //→ [102, 111, 111, 98]
```

`use rok::core::encoding::base64;`를 쓰면 호출이 `base64::encode(x)`로 짧아집니다.

## 2.22.1. Hex

`hex::encode`는 각 바이트를 상위 4비트부터 두 자리로 씁니다(`[0x00, 0xff]` → `"00ff"`). `a`부터 `f`까지는 소문자로 쓰고, `Case::Upper`이면 대문자로 씁니다. `hex::decode`는 바이트마다 두 자리를 읽고 대소문자를 모두 받습니다(`"DeadBeef"`). 텍스트의 길이는 짝수여야 합니다. 그 밖의 것은 받지 않습니다. `0x` 접두, 공백, 구분자 모두 받지 않습니다.

`hex::encode_into`에는 `out`이 `2 * data.len()`바이트, `hex::decode_into`에는 `text.len() / 2`바이트 필요합니다. `out`이 이보다 짧으면 호출자의 버그이므로 패닉합니다.

## 2.22.2. Base64

`base64::encode`는 3바이트씩을 6비트 값 네 개로 보고 값마다 한 글자를 씁니다. 마지막 묶음이 1바이트나 2바이트이면 2글자나 3글자가 되고, `=`로 4글자까지 채울 수 있습니다. 알파벳과 패딩은 `Codec`이 고릅니다.

```rok
#[non_exhaustive]
enum Alphabet { Standard, UrlSafe }
enum Padding { Present, Absent, PresentOptional, AbsentOptional }
struct Codec { .. }                    // Copy, Eq, Hash, Debug
impl Codec {
    const fn new(alphabet: Alphabet, padding: Padding) -> Codec;
    const fn alphabet(self) -> Alphabet;
    const fn padding(self) -> Padding;
}
const STANDARD: Codec;                 // Standard, PresentOptional
const STANDARD_NO_PAD: Codec;          // Standard, Absent
const URL_SAFE: Codec;                 // UrlSafe, PresentOptional
const URL_SAFE_NO_PAD: Codec;          // UrlSafe, Absent
```

`Alphabet::Standard`는 RFC 4648 4절의 알파벳(`A`-`Z`, `a`-`z`, `0`-`9`, `+`, `/`)입니다. `Alphabet::UrlSafe`는 5절의 URL과 파일 이름에 안전한 알파벳으로, `+`와 `/` 대신 `-`와 `_`를 씁니다. 패딩은 `encode`가 무엇을 쓰고 `decode`가 무엇을 받는지 정합니다.

| `Padding`         | `encode`가 `=`를 씀 | `decode`가 받는 텍스트   |
|-------------------|---------------------|--------------------------|
| `Present`         | 예                  | 패딩 있는 텍스트만       |
| `Absent`          | 아니요              | 패딩 없는 텍스트만       |
| `PresentOptional` | 예                  | 패딩이 있든 없든         |
| `AbsentOptional`  | 아니요              | 패딩이 있든 없든         |

기본 인자는 `STANDARD`입니다. 인코딩할 때 패딩을 쓰고, 디코딩할 때는 두 형태를 모두 받습니다. 엄격한 디코더는 코덱 하나로 얻습니다. `Codec::new(Alphabet::Standard, Padding::Present)`입니다.

| 입력       | `STANDARD`     | `STANDARD_NO_PAD` |
|------------|----------------|-------------------|
| `""`       | `""`           | `""`              |
| `"f"`      | `"Zg=="`       | `"Zg"`            |
| `"fo"`     | `"Zm8="`       | `"Zm8"`           |
| `"foo"`    | `"Zm9v"`       | `"Zm9v"`          |
| `"foobar"` | `"Zm9vYmFy"`   | `"Zm9vYmFy"`      |

패딩이 있다면 올발라야 합니다. 끝에 `=`가 하나나 둘 있어서 길이를 4의 배수로 맞춰야 합니다. 마지막 글자에 남는 비트는 무시합니다. 공백과 줄바꿈은 건너뛰지 않습니다. 여러 줄로 나뉜 텍스트(PEM, MIME)는 먼저 이어 붙입니다. 예를 들어 `split_whitespace`([2.15.2](215-string.md#2152-str-불변-utf-8-문자열))로 이어 붙일 수 있습니다.

```rok
let raw: List<u8> = [0xfb, 0xff, 0xbf];
base64::encode(raw);                           //→ "+/+/"
base64::encode(raw, base64::URL_SAFE);         //→ "-_-_"
base64::encode(raw, base64::STANDARD_NO_PAD);  //→ "+/+/"
base64::decode("-_-_", base64::URL_SAFE);      //→ [251, 255, 191]
```

`encoded_len(n, codec)`은 `n`바이트를 `encode`한 길이이고, `encode_into`에는 `out`이 그만큼 필요합니다. `decoded_len_max(n)`은 `n`글자가 디코딩될 수 있는 최대 바이트 수입니다. `out`이 이 길이이면 `decode_into`에 언제나 충분합니다. `decode_into`는 `out`이 패딩을 뺀 텍스트에 필요할 수 있는 길이보다 짧을 때만 패닉합니다. `out`이 짧은 것은 디코딩 에러가 아니라 호출자의 버그입니다.

```rok
// 32바이트 키를 base64로 frame의 4번 바이트부터 씀.
let end = 4 + base64::encoded_len(32);
let n = base64::encode_into(key, frame[4:end]);   // n == 44
```

## 2.22.3. 디코딩 에러

```rok
// rok::core::encoding
struct DecodeError { .. }
impl DecodeError {
    fn kind(self) -> DecodeErrorKind;
    fn position(self) -> usize;        // 텍스트 안의 바이트 오프셋
    fn message(self) -> str;           // Display 텍스트
}
#[non_exhaustive]
enum DecodeErrorKind { InvalidByte, InvalidLength, InvalidPadding }   // Copy, Eq, Hash, Debug
impl Display for DecodeError; impl Debug for DecodeError; impl Error for DecodeError;
```

디코더는 처음 만난 문제에서 `DecodeError`를 던집니다. `kind`는 문제의 종류를, `position`은 문제가 있는 `text` 안의 바이트 오프셋을 알려 줍니다. 메시지는 인코딩 이름으로 시작하고, 오프셋을 밝히고, 대신 무엇을 쓸지 말합니다.

| 문제                                     | `kind`           | 메시지 예                                                                       |
|------------------------------------------|------------------|---------------------------------------------------------------------------------|
| 길이가 홀수인 hex                        | `InvalidLength`  | `hex: odd length 3; every byte takes two digits, so write "0f" rather than "f"` |
| hex 숫자가 아님                          | `InvalidByte`    | `hex: 'g' at byte 1 is not a hex digit; use 0-9, a-f or A-F`                    |
| 다른 base64 알파벳의 글자                | `InvalidByte`    | `base64: '-' at byte 2 is url-safe base64; decode with base64::URL_SAFE`        |
| base64 글자가 아님                       | `InvalidByte`    | `base64: '*' at byte 1 is not base64; use A-Z, a-z, 0-9, + and /`               |
| 끝이 아닌 곳의 `=`                       | `InvalidPadding` | `base64: '=' at byte 1: '=' pads only the end of the text`                      |
| 4글자 묶음을 끝맺지 않는 패딩            | `InvalidPadding` | `base64: padded text must be a multiple of 4 characters long, not 7; fix the '=' count or drop the padding` |
| 코덱이 받지 않는 패딩                    | `InvalidPadding` | `base64: '=' at byte 2, but this codec takes no padding; drop the '=' or decode with base64::STANDARD` |
| 코덱이 요구하는 패딩이 없음              | `InvalidPadding` | `base64: text of 2 characters has no '=' padding, which this codec requires; pad it to a multiple of 4 or decode with base64::STANDARD` |
| 마지막 묶음에 글자 하나만 남음           | `InvalidLength`  | `base64: text is cut short at byte 4; a last group needs 2 to 4 characters`     |

hex 길이가 홀수이면 위치는 마지막 숫자, 패딩이 없으면 텍스트의 끝, 패딩 길이가 틀리면 첫 `=`입니다. 출력 가능한 ASCII 밖의 바이트는 그 값으로 보입니다(`byte 0x0a`). 디코더는 처음부터 훑으며 글자 에러를 길이나 패딩 에러보다 먼저 보고합니다.

## 2.22.4. 다른 언어와 비교

| 언어   | Hex                          | Base64                                                    |
|--------|------------------------------|-----------------------------------------------------------|
| Rust   | 서드파티 `hex` 크레이트      | 서드파티 `base64` 크레이트(`STANDARD`, `URL_SAFE_NO_PAD` 등 engine) |
| Python | `bytes.hex`, `bytes.fromhex` | `base64.b64encode`, `base64.urlsafe_b64encode`, ...       |
| Go     | `encoding/hex`               | `encoding/base64` (`StdEncoding`, `RawURLEncoding`, ...)  |
| Zig    | `std.fmt.hex`                | `std.base64` (`standard`, `url_safe_no_pad`, ...)         |
| Rok    | `hex::encode`, `hex::decode` | `base64::encode`, `base64::decode` (`STANDARD`, `URL_SAFE_NO_PAD` 등 코덱) |

Rok은 Go와 Zig처럼 인코딩마다 모듈을 하나 두고, 변형을 플래그가 아니라 코덱 값으로 고릅니다. 흔한 경우를 위한 `str` 형태와, 할당하지 않는 버퍼 형태(`_into`)를 함께 둡니다. 패딩 정책은 Kotlin의 `PaddingOption`처럼 양쪽을 모두 정하므로, `STANDARD`는 패딩을 쓰면서도 패딩 없는 텍스트를 받습니다.
