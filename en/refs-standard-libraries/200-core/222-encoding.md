# 2.22. `encoding::` - hex and base64

`rok::core::encoding` turns bytes into text and back with the two encodings of RFC 4648 that programs meet most: hex (base16) and base64. They are pure computation and behave identically on every target, so they live in `rok::core` (0.1), and every function below is a bare name in user code.

Byte arguments take a list, a view or a string's UTF-8 bytes:

```rok
fn hex_encode(data: List<u8> | Slice<u8> | str) -> str;
fn hex_decode(text: List<u8> | Slice<u8> | str) throws (ValueError) -> List<u8>;
fn base64_encode(data: List<u8> | Slice<u8> | str, url_safe: bool = false, pad: bool = true) -> str;
fn base64_decode(text: List<u8> | Slice<u8> | str, url_safe: bool = false) throws (ValueError) -> List<u8>;
```

Encoders return a new `str`; decoders return a new `List<u8>` and throw `ValueError` ([2.14.4](214-error.md#2144-built-in-error-types)) on text that is not in the encoding (2.22.3). Decoded bytes that are UTF-8 text become a `str` through `str::from_utf8` ([2.15.6](215-string.md#2156-utf-8-invariant---every-in-process-string)).

```rok
hex_encode("hi");                              //→ "6869"
hex_decode("6869");                            //→ [104, 105]
base64_encode("foob");                         //→ "Zm9vYg=="
base64_encode("foob", pad: false);             //→ "Zm9vYg"
base64_decode("Zm9vYg");                       //→ [102, 111, 111, 98]
```

## 2.22.1. Hex

`hex_encode` writes each byte as two lowercase digits, high half first (`[0x00, 0xff]` → `"00ff"`). `hex_decode` reads two digits per byte and accepts both cases (`"DeadBeef"`). The text must have an even length; nothing else is accepted: no `0x` prefix, no spaces or separators.

## 2.22.2. Base64

`base64_encode` takes each 3 bytes as four 6-bit values and writes one character per value. The alphabet is the standard one (RFC 4648 section 4: `A`-`Z`, `a`-`z`, `0`-`9`, `+`, `/`), or with `url_safe: true` the URL and file name safe one (section 5), which has `-` and `_` in place of `+` and `/`. A final group of 1 or 2 bytes gives 2 or 3 characters; with `pad: true` (the default) `=` fills the group to 4 characters, with `pad: false` nothing is added.

| Input      | `base64_encode` | `pad: false` |
|------------|-----------------|--------------|
| `""`       | `""`            | `""`         |
| `"f"`      | `"Zg=="`        | `"Zg"`       |
| `"fo"`     | `"Zm8="`        | `"Zm8"`      |
| `"foo"`    | `"Zm9v"`        | `"Zm9v"`     |
| `"foobar"` | `"Zm9vYmFy"`    | `"Zm9vYmFy"` |

`base64_decode` reads the alphabet `url_safe` selects, standard by default. It accepts padded and unpadded text alike, so it has no `pad` argument. Padding that is present must be correct: one or two `=` at the end that bring the length to a multiple of 4. Bits left over in the last character are ignored. Whitespace and line breaks are not skipped: text wrapped in lines (PEM, MIME) is joined first, for example with `split_whitespace` ([2.15.2](215-string.md#2152-str---immutable-utf-8-string)).

```rok
let raw: List<u8> = [0xfb, 0xff, 0xbf];
base64_encode(raw);                            //→ "+/+/"
base64_encode(raw, url_safe: true);            //→ "-_-_"
base64_decode("-_-_", url_safe: true);         //→ [251, 255, 191]
```

## 2.22.3. Decode errors

A decoder throws `ValueError` at the first problem. Its `msg` starts with the encoding, names the byte offset in `text` where the problem is, and says what to write instead:

| Problem                                    | Example `msg`                                                                   |
|--------------------------------------------|---------------------------------------------------------------------------------|
| hex of odd length                          | `hex: odd length 3; every byte takes two digits, so write "0f" rather than "f"` |
| not a hex digit                            | `hex: 'g' at byte 1 is not a hex digit; use 0-9, a-f or A-F`                    |
| a character of the other base64 alphabet   | `base64: '-' at byte 2 is url-safe base64; decode with url_safe: true`          |
| not a base64 character                     | `base64: '*' at byte 1 is not base64; use A-Z, a-z, 0-9, + and /`               |
| `=` before the end                         | `base64: '=' at byte 1: '=' pads only the end of the text`                      |
| padding that does not end a group of 4     | `base64: padded text must be a multiple of 4 characters long, not 7; fix the '=' count or drop the padding` |
| one character left in the last group       | `base64: text is cut short at byte 4; a last group needs 2 to 4 characters`     |

A byte outside printable ASCII is shown by its value (`byte 0x0a`). Character errors are reported before length and padding errors, scanning from the start.

## 2.22.4. Comparison with other languages

| Language | Hex                          | Base64                                                   |
|----------|------------------------------|----------------------------------------------------------|
| Rust     | third-party `hex` crate      | third-party `base64` crate (engine values)               |
| Python   | `bytes.hex`, `bytes.fromhex` | `base64.b64encode`, `base64.urlsafe_b64encode`, ...      |
| Go       | `encoding/hex`               | `encoding/base64` (`StdEncoding`, `RawURLEncoding`, ...) |
| Rok      | `hex_encode`, `hex_decode`   | `base64_encode`, `base64_decode` (`url_safe`, `pad`)     |

Rok puts the four base64 variants behind two named arguments rather than four encoding values, and its decoder takes padded and unpadded text, so the common case is one call with no options.
