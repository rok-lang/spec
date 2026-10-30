# 2.22. `encoding::`: hex와 base64

`rok::core::encoding`은 바이트를 텍스트로 바꾸고 다시 되돌립니다. 인코딩은 RFC 4648 가운데 프로그램이 가장 자주 만나는 두 가지, hex(base16)와 base64입니다. 순수 계산이고 모든 타깃에서 똑같이 동작하므로 `rok::core`(0.1)에 둡니다. 아래 함수는 모두 사용자 코드에서 이름만으로 쓸 수 있습니다.

바이트 인자는 리스트, view, 또는 문자열의 UTF-8 바이트를 받습니다.

```rok
fn hex_encode(data: List<u8> | Slice<u8> | str) -> str;
fn hex_decode(text: List<u8> | Slice<u8> | str) throws (ValueError) -> List<u8>;
fn base64_encode(data: List<u8> | Slice<u8> | str, url_safe: bool = false, pad: bool = true) -> str;
fn base64_decode(text: List<u8> | Slice<u8> | str, url_safe: bool = false) throws (ValueError) -> List<u8>;
```

인코더는 새 `str`을 돌려줍니다. 디코더는 새 `List<u8>`을 돌려주고, 인코딩에 맞지 않는 텍스트에는 `ValueError`([2.14.4](214-error.md#2144-내장-에러-타입))를 던집니다(2.22.3). 디코딩한 바이트가 UTF-8 텍스트이면 `str::from_utf8`([2.15.6](215-string.md#2156-utf-8-불변식-프로세스-안의-모든-문자열))로 `str`이 됩니다.

```rok
hex_encode("hi");                              //→ "6869"
hex_decode("6869");                            //→ [104, 105]
base64_encode("foob");                         //→ "Zm9vYg=="
base64_encode("foob", pad: false);             //→ "Zm9vYg"
base64_decode("Zm9vYg");                       //→ [102, 111, 111, 98]
```

## 2.22.1. Hex

`hex_encode`는 각 바이트를 상위 4비트부터 소문자 두 자리로 씁니다(`[0x00, 0xff]` → `"00ff"`). `hex_decode`는 바이트마다 두 자리를 읽고 대소문자를 모두 받습니다(`"DeadBeef"`). 텍스트의 길이는 짝수여야 합니다. 그 밖의 것은 받지 않습니다. `0x` 접두, 공백, 구분자 모두 받지 않습니다.

## 2.22.2. Base64

`base64_encode`는 3바이트씩을 6비트 값 네 개로 보고 값마다 한 글자를 씁니다. 알파벳은 표준 알파벳(RFC 4648 4절의 `A`-`Z`, `a`-`z`, `0`-`9`, `+`, `/`)입니다. `url_safe: true`이면 URL과 파일 이름에 안전한 알파벳(5절)을 씁니다. 이 알파벳은 `+`와 `/` 대신 `-`와 `_`를 씁니다. 마지막 묶음이 1바이트나 2바이트이면 2글자나 3글자가 됩니다. `pad: true`(기본값)이면 `=`로 묶음을 4글자로 채우고, `pad: false`이면 아무것도 덧붙이지 않습니다.

| 입력       | `base64_encode` | `pad: false` |
|------------|-----------------|--------------|
| `""`       | `""`            | `""`         |
| `"f"`      | `"Zg=="`        | `"Zg"`       |
| `"fo"`     | `"Zm8="`        | `"Zm8"`      |
| `"foo"`    | `"Zm9v"`        | `"Zm9v"`     |
| `"foobar"` | `"Zm9vYmFy"`    | `"Zm9vYmFy"` |

`base64_decode`는 `url_safe`가 고르는 알파벳을 읽습니다. 기본은 표준 알파벳입니다. 패딩이 있는 텍스트와 없는 텍스트를 똑같이 받으므로 `pad` 인자가 없습니다. 패딩이 있다면 올발라야 합니다. 끝에 `=`가 하나나 둘 있어서 길이를 4의 배수로 맞춰야 합니다. 마지막 글자에 남는 비트는 무시합니다. 공백과 줄바꿈은 건너뛰지 않습니다. 여러 줄로 나뉜 텍스트(PEM, MIME)는 먼저 이어 붙입니다. 예를 들어 `split_whitespace`([2.15.2](215-string.md#2152-str-불변-utf-8-문자열))로 이어 붙일 수 있습니다.

```rok
let raw: List<u8> = [0xfb, 0xff, 0xbf];
base64_encode(raw);                            //→ "+/+/"
base64_encode(raw, url_safe: true);            //→ "-_-_"
base64_decode("-_-_", url_safe: true);         //→ [251, 255, 191]
```

## 2.22.3. 디코딩 에러

디코더는 처음 만난 문제에서 `ValueError`를 던집니다. `msg`는 인코딩 이름으로 시작합니다. 이어서 문제가 있는 `text` 안의 바이트 오프셋을 밝히고, 대신 무엇을 쓸지 말합니다.

| 문제                                     | `msg` 예                                                                        |
|------------------------------------------|---------------------------------------------------------------------------------|
| 길이가 홀수인 hex                        | `hex: odd length 3; every byte takes two digits, so write "0f" rather than "f"` |
| hex 숫자가 아님                          | `hex: 'g' at byte 1 is not a hex digit; use 0-9, a-f or A-F`                    |
| 다른 base64 알파벳의 글자                | `base64: '-' at byte 2 is url-safe base64; decode with url_safe: true`          |
| base64 글자가 아님                       | `base64: '*' at byte 1 is not base64; use A-Z, a-z, 0-9, + and /`               |
| 끝이 아닌 곳의 `=`                       | `base64: '=' at byte 1: '=' pads only the end of the text`                      |
| 4글자 묶음을 끝맺지 않는 패딩            | `base64: padded text must be a multiple of 4 characters long, not 7; fix the '=' count or drop the padding` |
| 마지막 묶음에 글자 하나만 남음           | `base64: text is cut short at byte 4; a last group needs 2 to 4 characters`     |

출력 가능한 ASCII 밖의 바이트는 그 값으로 보입니다(`byte 0x0a`). 디코더는 처음부터 훑으며 글자 에러를 길이나 패딩 에러보다 먼저 보고합니다.

## 2.22.4. 다른 언어와 비교

| 언어   | Hex                          | Base64                                                   |
|--------|------------------------------|----------------------------------------------------------|
| Rust   | 서드파티 `hex` 크레이트      | 서드파티 `base64` 크레이트(engine 값)                    |
| Python | `bytes.hex`, `bytes.fromhex` | `base64.b64encode`, `base64.urlsafe_b64encode`, ...      |
| Go     | `encoding/hex`               | `encoding/base64` (`StdEncoding`, `RawURLEncoding`, ...) |
| Rok    | `hex_encode`, `hex_decode`   | `base64_encode`, `base64_decode` (`url_safe`, `pad`)     |

Rok은 base64의 네 가지 변형을 인코딩 값 네 개가 아니라 이름 있는 인자 두 개로 고릅니다. 디코더는 패딩이 있는 텍스트와 없는 텍스트를 모두 받습니다. 그래서 흔한 경우는 옵션 없는 호출 하나로 끝납니다.
