# 2.22. `encoding::` - hex and base64

`rok::core::encoding` turns bytes into text and back with the two encodings of RFC 4648 that programs meet most: hex (base16) and base64. They are pure computation and behave identically on every target, so they live in `rok::core` (0.1). Each encoding is a module, `encoding::hex` and `encoding::base64`, with the same four functions; `encoding::DecodeError` is the error both throw.

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

Byte arguments are `AsBytes` ([2.11.13](211-conversion.md#21113-asbytes-trait---bytes-as-a-view)): a list, a view, an array or a string's UTF-8 bytes. `encode` returns a new `str` and `decode` a new `List<u8>`. The `_into` forms write to the front of `out`, a writable view of a list the caller sized ([2.9.7](209-collections.md)), and return how many bytes they wrote; they allocate nothing. Decoded bytes that are UTF-8 text become a `str` through `str::from_utf8` ([2.15.6](215-string.md#2156-utf-8-invariant---every-in-process-string)).

```rok
encoding::hex::encode("hi");                   //→ "6869"
encoding::hex::decode("6869");                 //→ [104, 105]
encoding::base64::encode("foob");              //→ "Zm9vYg=="
encoding::base64::decode("Zm9vYg");            //→ [102, 111, 111, 98]
```

`use rok::core::encoding::base64;` shortens the calls to `base64::encode(x)`.

## 2.22.1. Hex

`hex::encode` writes each byte as two digits, high half first (`[0x00, 0xff]` → `"00ff"`); the digits `a` to `f` are lowercase, or uppercase with `Case::Upper`. `hex::decode` reads two digits per byte and accepts both cases (`"DeadBeef"`). The text must have an even length; nothing else is accepted: no `0x` prefix, no spaces or separators.

`hex::encode_into` needs `2 * data.len()` bytes of `out` and `hex::decode_into` needs `text.len() / 2`; a shorter `out` is a bug in the caller and panics.

## 2.22.2. Base64

`base64::encode` takes each 3 bytes as four 6-bit values and writes one character per value. A final group of 1 or 2 bytes gives 2 or 3 characters, which `=` may pad to 4. A `Codec` picks the alphabet and the padding:

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

`Alphabet::Standard` is RFC 4648 section 4 (`A`-`Z`, `a`-`z`, `0`-`9`, `+`, `/`); `Alphabet::UrlSafe` is the URL and file name safe alphabet of section 5, which has `-` and `_` in place of `+` and `/`. The padding names what `encode` writes and what `decode` takes:

| `Padding`         | `encode` writes `=` | `decode` takes          |
|-------------------|---------------------|-------------------------|
| `Present`         | yes                 | padded text only        |
| `Absent`          | no                  | unpadded text only      |
| `PresentOptional` | yes                 | padded or unpadded text |
| `AbsentOptional`  | no                  | padded or unpadded text |

`STANDARD` is the default argument: it pads on encode and takes either form on decode. A strict decoder is one codec away: `Codec::new(Alphabet::Standard, Padding::Present)`.

| Input      | `STANDARD`     | `STANDARD_NO_PAD` |
|------------|----------------|-------------------|
| `""`       | `""`           | `""`              |
| `"f"`      | `"Zg=="`       | `"Zg"`            |
| `"fo"`     | `"Zm8="`       | `"Zm8"`           |
| `"foo"`    | `"Zm9v"`       | `"Zm9v"`          |
| `"foobar"` | `"Zm9vYmFy"`   | `"Zm9vYmFy"`      |

Padding that is present must be correct: one or two `=` at the end that bring the length to a multiple of 4. Bits left over in the last character are ignored. Whitespace and line breaks are not skipped: text wrapped in lines (PEM, MIME) is joined first, for example with `split_whitespace` ([2.15.2](215-string.md#2152-str---immutable-utf-8-string)).

```rok
let raw: List<u8> = [0xfb, 0xff, 0xbf];
base64::encode(raw);                           //→ "+/+/"
base64::encode(raw, base64::URL_SAFE);         //→ "-_-_"
base64::encode(raw, base64::STANDARD_NO_PAD);  //→ "+/+/"
base64::decode("-_-_", base64::URL_SAFE);      //→ [251, 255, 191]
```

`encoded_len(n, codec)` is the length `encode` gives for `n` bytes, and `encode_into` needs that much of `out`. `decoded_len_max(n)` is the most bytes `n` characters decode to; an `out` of that length is always enough for `decode_into`, which panics only if `out` is shorter than the text, less its padding, can need. A short `out` is a bug in the caller, not a decode error.

```rok
// A 32-byte key as base64 into bytes 4.. of a frame.
let end = 4 + base64::encoded_len(32);
let n = base64::encode_into(key, frame[4:end]);   // n == 44
```

## 2.22.3. Decode errors

```rok
// rok::core::encoding
struct DecodeError { .. }
impl DecodeError {
    fn kind(self) -> DecodeErrorKind;
    fn position(self) -> usize;        // byte offset in the text
    fn message(self) -> str;           // the Display text
}
#[non_exhaustive]
enum DecodeErrorKind { InvalidByte, InvalidLength, InvalidPadding }   // Copy, Eq, Hash, Debug
impl Display for DecodeError; impl Debug for DecodeError; impl Error for DecodeError;
```

A decoder throws `DecodeError` at the first problem. `kind` says what kind of problem it is and `position` the byte offset in `text` where it is. The message starts with the encoding, names the offset and says what to write instead:

| Problem                                   | `kind`           | Example message                                                                 |
|-------------------------------------------|------------------|---------------------------------------------------------------------------------|
| hex of odd length                         | `InvalidLength`  | `hex: odd length 3; every byte takes two digits, so write "0f" rather than "f"` |
| not a hex digit                           | `InvalidByte`    | `hex: 'g' at byte 1 is not a hex digit; use 0-9, a-f or A-F`                    |
| a character of the other base64 alphabet  | `InvalidByte`    | `base64: '-' at byte 2 is url-safe base64; decode with base64::URL_SAFE`        |
| not a base64 character                    | `InvalidByte`    | `base64: '*' at byte 1 is not base64; use A-Z, a-z, 0-9, + and /`               |
| `=` before the end                        | `InvalidPadding` | `base64: '=' at byte 1: '=' pads only the end of the text`                      |
| padding that does not end a group of 4    | `InvalidPadding` | `base64: padded text must be a multiple of 4 characters long, not 7; fix the '=' count or drop the padding` |
| padding the codec does not take           | `InvalidPadding` | `base64: '=' at byte 2, but this codec takes no padding; drop the '=' or decode with base64::STANDARD` |
| no padding where the codec requires it    | `InvalidPadding` | `base64: text of 2 characters has no '=' padding, which this codec requires; pad it to a multiple of 4 or decode with base64::STANDARD` |
| one character left in the last group      | `InvalidLength`  | `base64: text is cut short at byte 4; a last group needs 2 to 4 characters`     |

The position of an odd hex length is its last digit; of missing padding, the end of the text; of padding of the wrong length, its first `=`. A byte outside printable ASCII is shown by its value (`byte 0x0a`). Character errors are reported before length and padding errors, scanning from the start.

## 2.22.4. Comparison with other languages

| Language | Hex                          | Base64                                                    |
|----------|------------------------------|-----------------------------------------------------------|
| Rust     | third-party `hex` crate      | third-party `base64` crate (`STANDARD`, `URL_SAFE_NO_PAD`, ... engines) |
| Python   | `bytes.hex`, `bytes.fromhex` | `base64.b64encode`, `base64.urlsafe_b64encode`, ...       |
| Go       | `encoding/hex`               | `encoding/base64` (`StdEncoding`, `RawURLEncoding`, ...)  |
| Zig      | `std.fmt.hex`                | `std.base64` (`standard`, `url_safe_no_pad`, ...)         |
| Rok      | `hex::encode`, `hex::decode` | `base64::encode`, `base64::decode` (`STANDARD`, `URL_SAFE_NO_PAD`, ... codecs) |

Rok follows Go and Zig: one module per encoding, the variants as codec values rather than flags, a `str` form for the common case and a buffer form (`_into`) that allocates nothing. The padding policy names both sides, as Kotlin's `PaddingOption` does, so `STANDARD` writes padding and still accepts text without it.
