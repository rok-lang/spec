# 2.15. 문자열 — `str` / `char` / `c_char`

`rok::core::str` 모듈은 문자열과 문자 타입의 권위 정의를 제공합니다. rok 은 **UTF-8 우선** 정책을 채택하고 있어서, 모든 in-process 문자열은 UTF-8 로 다루며, FFI 경계에서만 `c_char` 를 사용합니다.

이 장은 48 개의 lang slot 중 세 개를 정의합니다. 첫째로 `str_type` 은 `rok::core::str::str` 에 연결되며 언어 기본값에 해당하는 lowercase struct 입니다. 둘째로 `char_type` 은 내장 `char` 이며, 이 장이 그 메서드를 정의합니다. 셋째로 `cchar_type` 은 `rok::core::str::c_char` 에 연결되며 마찬가지로 언어 기본값입니다.

## 2.15.1. 문자열 모델 — 불변 `str` 과 가변 `StrBuf`

`str` 은 불변 UTF-8 문자열 값이고, 문자열을 만들어 가는 가변 buffer 는 별도 타입 `StrBuf` 입니다.

| 타입     | 의미                                               | 메모리 표현                  | mutability          |
|----------|----------------------------------------------------|------------------------------|---------------------|
| `str`    | UTF-8 문자열 값                                    | `(ptr, len)` 값 — 바이트는 immortal 상수 또는 GC 객체 | **불변**  |
| `StrBuf` | 늘어나는 UTF-8 buffer                              | GC heap 객체                 | 가변                |
| `char`   | Unicode scalar value (PascalCase struct + keyword) | 4B (u32)                     | (값 — 변경 X)       |
| `c_char` | C ABI 1-byte char (FFI primitive)                  | 1B (i8 또는 u8 — target dep) | (값)                |

### Rust / Python / rok 모델 비교

세 언어가 문자열을 어떻게 다루는지 비교하면 다음과 같습니다.

| 측면       | Rust             | Python         | rok                      |
|------------|------------------|----------------|--------------------------|
| 값         | `&str`           | `str`          | `str` (불변 `(ptr, len)`) |
| 가변 buffer | `String`        | `io.StringIO` 등 | `StrBuf`               |
| 메모리     | borrow checker   | refcount       | GC                       |

rok 의 `str` 은 수명 추적이 필요 없다는 점에서 Python 의 불변 `str` 에 가깝고, 표현은 Rust 의 `&str` 처럼 `(ptr, len)` 쌍입니다. 런타임에 만들어지는 문자열의 바이트는 GC 객체 (type id `1`, [refs-lang 8.2.12](../../refs-lang/008-abi/802-scan-table.md#8212-런타임-예약-type-id)) 에 있습니다.

## 2.15.2. `str` — 불변 UTF-8 문자열

```rok
#[lang("str_type")]
struct str {
    ptr: ConstPtr<u8>;
    len: usize;            // byte length (NOT char count)
}
```

`str` 는 언어 기본값이라 컴파일러가 직접 인식하므로 `#[keyword]` 가 따로 필요하지 않습니다. lowercase struct 이름이 사용자 코드의 타입 표기와 동일합니다.

`str` 은 값으로 전달되는 `(ptr, len)` 쌍이며, 가리키는 바이트는 바뀌지 않습니다. 문자열을 바꾸는 연산은 모두 새 `str` 을 돌려줍니다.

### 핵심 메서드

```rok
impl str {
    fn from_utf8(bytes: List<u8> | Slice<u8>) throws (Utf8Error) -> str;

    fn len(self) -> usize;                  // byte 길이
    fn is_empty(self) -> bool;
    fn as_bytes(self) -> Slice<u8>;         // 복사 없는 읽기 전용 view
    fn byte(self, i: usize) -> u8;          // O(1), 범위 밖이면 panic
    fn as_ptr(self) -> ConstPtr<u8>;        // FFI

    // 읽기 메서드
    fn chars(self) -> CharsIter;            // Unicode scalar iterator
    fn bytes(self) -> BytesIter;            // u8 iterator
    fn char_indices(self) -> CharIndicesIter;
    fn lines(self) -> LinesIter;            // 끝의 \r 제거, 마지막 빈 줄은 내지 않음
    fn split(self, sep: str) -> SplitIter;  // 빈 조각도 유지 ("a,,b" → "a", "", "b")
    fn split_whitespace(self) -> SplitWhitespaceIter;   // Unicode White_Space 로 나눔 (trim 과 같은 집합)

    fn starts_with(self, pat: str) -> bool;
    fn ends_with(self, pat: str) -> bool;
    fn contains(self, pat: str) -> bool;
    fn find(self, pat: str) -> usize | none;
    fn rfind(self, pat: str) -> usize | none;

    fn strip_prefix(self, prefix: str) -> str | none;   // 접두가 있으면 뗀 나머지
    fn strip_suffix(self, suffix: str) -> str | none;
    fn split_once(self, sep: str) -> (str, str) | none; // 처음 sep 의 앞 / 뒤
    fn repeat(self, n: usize) -> str;

    fn trim(self) -> str;                   // Unicode White_Space 를 뗌
    fn trim_start(self) -> str;
    fn trim_end(self) -> str;

    fn to_lowercase(self) -> str;           // 새 str (Unicode 전체 매핑 — 1:N, 문맥 규칙 포함)
    fn to_uppercase(self) -> str;
    fn replace(self, from: str, to: str) -> str;   // 빈 from 은 모든 char 경계에서 맞음: "ab".replace("", "-") == "-a-b-"

    fn parse<T: FromStr>(self) throws (T::Err) -> T;

    // UTF-8 안전 slicing — byte boundary 검증
    // (s[a:b] desugar 가 본 메서드 dispatch)
    fn slice(self, range: Range<usize>) -> str;        // s[a:b] 와 같음, 바이트 공유
}
```

`as_ptr` 가 돌려주는 포인터는 그 `str` 을 살아 있는 지역 변수가 들고 있는 동안 유효합니다. 다른 객체를 통해서만 닿는 문자열은 GC 가 옮길 수 있으며, pin 되지 않습니다 ([stdlib 1.6](../100-bare/106-gc-pin.md)).

### `StrBuf` — 가변 buffer

문자열을 조금씩 만들어 갈 때는 `StrBuf` 를 씁니다.

```rok
struct StrBuf { /* GC heap — ptr / len / capacity */ }

impl StrBuf {
    fn new() -> StrBuf;
    fn push(self, c: char);                 // char 1 개 추가 (UTF-8 인코딩)
    unsafe fn push_byte(self, b: u8);       // 내용이 UTF-8 로 남도록 호출자가 보장. 안전한 바이트 경로는 str::from_utf8
    fn push_str(self, s: str);
    fn capacity(self) -> usize;
    fn to_str(self) -> str;                 // 현재 내용의 불변 str
}
```

`StrBuf` 는 컬렉션처럼 참조 핸들이므로 메서드가 `self` 를 받습니다 ([2.9](209-collections.md)).

```rok
let buf = StrBuf::new();
for x in xs {
    buf.push_str(f"{x},");
}
let s: str = buf.to_str();
```

### 문자열 리터럴 — `"hello"`

```rok
let s: str = "hello";        // immortal 상수 바이트를 가리키는 str
```

`"hello"` 리터럴은 immortal 영역의 상수 바이트를 가리키는 `str` 값으로 lower 됩니다. `str` 이 불변이므로 복사나 copy-on-write 가 없습니다.

### 트레잇 구현

| 트레잇                      | impl                           | 동작                                       |
|-----------------------------|--------------------------------|--------------------------------------------|
| `Display`                   | ✓                              | self 그대로                                |
| `Debug`                     | ✓                              | `"..."` 이스케이프                         |
| `Clone`                     | ✓                              | 같은 값 (불변이라 바이트 공유)             |
| `Copy`                      | ✓                              | `(ptr, len)` 값 복사                       |
| `PartialEq`                 | ✓                              | 내용 비교 (byte-wise)                      |
| `Eq`                        | ✓                              |                                            |
| `PartialOrd`                | ✓                              | lexicographic byte                         |
| `Ord`                       | ✓                              |                                            |
| `Hash`                      | ✓                              |                                            |
| `IntoIterator`              | **없음**                       | `.chars()` 명시 — byte vs char 모호성 회피 |
| `Add<str>` (`s + t`)        | ✓ → str (concat — 새 str)      |                                            |
| `AddAssign<str>` (`s += t`) | ✓ → `s = s + t` (새 str 로 교체) |                                          |
| `FromIterator<char>`        | ✓                              | `(chars).collect::<str>()`                 |
| `FromIterator<str>`         | ✓                              | `(strs).collect::<str>()`                  |

> [!NOTE]
> `str` 는 불변 값이라 `let t = s;` 는 `(ptr, len)` 을 복사할 뿐이며, 어느 쪽도 내용을 바꿀 수 없습니다.

## 2.15.3. `s[a:b]` 슬라이싱 — UTF-8 안전

```rok
let s: str = "hello, 세계!";
s.len();                       //→ 14  (byte 길이 — "hello, " 7 + 한국어 3바이트 × 2 + "!" 1)
s.chars().count();             //→ 10  (char 개수)

let s2 = s[7:13];              //→ "세계"  (byte boundary OK)
let s3 = s[8:9];               // panic — UTF-8 sequence 중간 byte
let s4 = s[7:100];             //→ "세계!" (끝은 길이로 잘라 맞춤)
let s5 = s[-1:];               //→ "!"
```

`str` / `StrBuf` 의 경계는 바이트 offset 입니다. 경계는 먼저 Python 처럼 잘라 맞추고 (음수는 끝에서부터, [refs-lang 3.6.2](../../refs-lang/003-expressions/306-indexing-and-slicing.md#362-슬라이싱-arrstartend)), 그 위치가 UTF-8 sequence 중간이면 panic 합니다. 결과는 원래 바이트를 공유하는 `str` 입니다 (아래). `str` 슬라이스에는 step 이 없으며 `s[::2]` 는 타입 에러입니다.

### 직접 indexing 거부

```rok
let s: str = "hello";
let c = s[0];                  // ERROR — `Index<usize>` 정의 안 함
                               //         이유: byte vs char 모호 + UTF-8 안전성
```

이 경우 `s.chars().next()` 나 `s.byte(0)` 처럼 의도를 명시해야 합니다.

### 부분 문자열은 복사하지 않음

`s[a:b]`, `lines`, `split*`, `split_whitespace`, `split_once`, `trim*`, `strip_*` 의 결과는 원래 바이트를 공유하는 `str` 이며 O(1) 입니다 ([refs-lang 8.8.1](../../refs-lang/008-abi/808-lowering-types.md#881-원시-정수--부동소수)). 그래서 작은 조각이 큰 원본을 살려 둡니다. 떼어 내는 API 는 없습니다. `as_bytes()` 도 복사 없는 읽기 전용 `Slice<u8>` view 입니다.

## 2.15.4. `char` — Unicode scalar value

`char` 는 언어 내장 타입입니다 ([2.1](201-numbers.md)): struct 가 아닌 4 바이트 Unicode scalar value 이며, 이 모듈이 그 메서드를 정의합니다.

Unicode scalar value 의 범위는 0..=0xD7FF 와 0xE000..=0x10FFFF 의 합집합입니다. Surrogate 범위인 0xD800..=0xDFFF 는 **자동 niche** 로 활용되어, `char | none` 이 4B 로 인코딩될 수 있습니다.

### 리터럴 / 메서드

```rok
let c: char = 'A';                   //→ U+0041
let k: char = '한';                  //→ U+D55C
let e: char = '😀';                  //→ U+1F600

c.to_lowercase();                    //→ 'a'
c.is_alphabetic();                   //→ true
c.is_ascii();                        //→ true
try '1'.to_digit(10);                //→ 1 (u32, 실패 시 ValueError)
c.len_utf8();                        //→ 1 (UTF-8 인코딩 시 byte 수)
e.len_utf8();                        //→ 4
```

```rok
impl char {
    fn from_u32(n: u32) -> char | none;                  // surrogate / 범위 밖이면 none
    unsafe fn from_u32_unchecked(n: u32) -> char;        // 검사 없음 — 호출자가 scalar value 를 보장
    fn from_digit(d: u32, radix: u32) -> char;           // d >= radix 나 radix 가 2..=36 밖이면 panic
    fn to_digit(self, radix: u32) throws (ValueError) -> u32;
    fn is_digit(self, radix: u32) -> bool;
    fn len_utf8(self) -> usize;
    fn len_utf16(self) -> usize;

    // Unicode 속성
    fn is_alphabetic(self) -> bool;
    fn is_lowercase(self) -> bool;
    fn is_uppercase(self) -> bool;
    fn is_whitespace(self) -> bool;
    fn is_numeric(self) -> bool;
    fn is_alphanumeric(self) -> bool;
    fn is_control(self) -> bool;
    fn to_lowercase(self) -> char;       // 1:1 매핑만
    fn to_uppercase(self) -> char;

    // ASCII
    fn is_ascii(self) -> bool;
    fn is_ascii_uppercase(self) -> bool;     fn is_ascii_lowercase(self) -> bool;
    fn is_ascii_alphabetic(self) -> bool;    fn is_ascii_digit(self) -> bool;
    fn is_ascii_alphanumeric(self) -> bool;  fn is_ascii_hexdigit(self) -> bool;
    fn is_ascii_whitespace(self) -> bool;    fn is_ascii_control(self) -> bool;
    fn is_ascii_graphic(self) -> bool;       fn is_ascii_punctuation(self) -> bool;
    fn to_ascii_uppercase(self) -> char;     fn to_ascii_lowercase(self) -> char;
    fn eq_ignore_ascii_case(self, other: char) -> bool;
}
```

`char` 의 `to_lowercase` / `to_uppercase` 는 한 문자를 돌려주므로 1:1 매핑만 적용합니다. 한 문자가 여러 문자로 바뀌는 경우는 그대로 둡니다 (`'ß'.to_uppercase() == 'ß'`). `str` 의 `to_lowercase` / `to_uppercase` 는 1:N 과 문맥 규칙 (그리스어 끝 시그마 `Σ` → `ς`) 까지 포함한 전체 매핑입니다 (`"ß".to_uppercase() == "SS"`).

Unicode 속성과 대소문자 표는 UCD 18.0.0 입니다.

### 변환

```rok
char::from_u32(0x41);                //→ 'A' (char | none — surrogate 는 none)
char::from_digit(5, 10);             //→ '5'

let n: u32 = c as u32;               //→ 65 (lossless)
let c2: char = n as char;            //→ ERROR — u8 만 `as char` 가능
let b: u8 = 0x41;
let c4: char = b as char;            //→ 'A' (u8 은 늘 scalar value)
let c3 = try char::try_from(n);      //→ char (TryFrom<u32> — from_u32 를 써서, none 이면 OverflowError)
```

### 트레잇 구현

`char` 는 모든 cmp / fmt 트레잇과 함께 `Copy`, `Clone`, `Hash` 를 구현합니다. niche 는 자동으로 적용되어 `char | none` 의 크기가 4B 가 됩니다.

## 2.15.5. `c_char` — C ABI 1-byte (언어 기본값)

```rok
#[lang("cchar_type")]
struct c_char {
    value: i8 또는 u8;     // target dep (signedness 가 OS / arch 별 다름)
}
```

`c_char` 는 언어 기본값에 해당하는 FFI primitive 로, 컴파일러가 직접 인식하는 lowercase struct 입니다.

C 의 `char` 와 ABI 호환을 이루며, UTF-8 인코딩 단위가 아니라 raw byte 라는 점에 유의해야 합니다. 용도는 다음과 같이 FFI 전용입니다.

```rok
extern "C" {
    fn strlen(s: ConstPtr<c_char>) -> usize;
    fn strcmp(a: ConstPtr<c_char>, b: ConstPtr<c_char>) -> i32;
}
```

### 리터럴

```rok
let c: c_char = c'A';        //→ 65 (i8/u8 — target dep)
let n: c_char = c'\n';       //→ 10
```

prefix `c'...'` 형태를 사용하여 char 와 분리합니다.

### char vs c_char 차이

두 타입을 비교하면 차이가 분명히 드러납니다.

| 측면       | `char`                              | `c_char`                      |
|------------|-------------------------------------|-------------------------------|
| 크기       | 4 bytes                             | 1 byte                        |
| 의미       | Unicode scalar (code point)         | C ABI byte                    |
| 범위       | 0..=0x10FFFF (surrogate 제외)       | 0..=255 또는 -128..=127       |
| 멀티바이트 | 단일 char 가 BMP / SMP 모두 표현    | 멀티바이트 char = byte 시퀀스 |
| 리터럴     | `'A'`                               | `c'A'`                        |
| 등록       | 표준 라이브러리 추가 (`#[keyword]`) | 언어 기본값                   |
| 사용처     | 일반 문자열 처리                    | FFI / 바이트 단위 처리        |

따라서 표준 라이브러리 함수에서는 **`char` 우선** 사용을 원칙으로 하며, `c_char` 는 명시적인 FFI 경계에서만 활용합니다.

## 2.15.6. UTF-8 invariant — 모든 in-process 문자열

`str` 는 항상 UTF-8 valid 임이 보장되며, 그 근거는 다음과 같습니다. 우선 생성자가 검증을 수행하므로 `str::from_utf8(bytes)` 는 잘못된 바이트열에 대해 `Utf8Error` 를 던집니다 (`throws (Utf8Error)`). 다음으로 슬라이싱이 byte boundary 를 검증하여 멀티바이트 시퀀스 중간을 거부합니다. 또한 `push` 나 `insert` 같은 in-place 수정 역시 UTF-8 무결성을 보장합니다. 마지막으로 `rok::bare::str_from_bytes_unchecked` 만이 검증을 우회할 수 있으며 이는 unsafe tier 에 속합니다.

### 검증 우회의 이유

UTF-8 invariant 가 **깨지면** 어떤 문제가 생기는지 살펴볼 필요가 있습니다. 우선 `chars()` 가 `IndexError` 를 던지거나 무한 루프에 빠질 수 있고, 멀티바이트 sequence 가 잘려서 표시가 깨질 수 있으며, 더 나아가 overlong encoding 같은 security 이슈가 발생할 수도 있습니다.

따라서 표준 라이브러리가 빌드 시점에 검증을 보장함으로써, 사용자 코드에서 `str.chars()` 등이 항상 안전하게 동작합니다. unsafe 진입 없이는 invariant 가 깨질 수 없습니다.

### byte ↔ str 변환

```rok
let bytes: List<u8> = [0x68, 0x65, 0x6C, 0x6C, 0x6F];
let s: str = try str::from_utf8(bytes);
                                          //→ "hello"

let invalid: List<u8> = [0xFF, 0xFE];
let s: str = try str::from_utf8(invalid);
                                          //→ throw Utf8Error { ... }
```

## 2.15.7. `FromStr` 트레잇 — 파싱

```rok
trait FromStr {
    type Err;
    fn from_str(s: str) throws (Err) -> Self;
}
```

`str.parse::<T>()` 는 `throws (T::Err)` 이며 이 트레잇을 통해 dispatch 됩니다. 숫자 타입과 `bool` 의 `Err` 는 core 의 `ParseError { input, pos, reason }` ([2.14.4](214-error.md#2144-내장-에러-타입)) 입니다. 숫자 파싱은 런타임 helper `rok_str_int_status` / `rok_str_to_i64` / `rok_str_to_u64` / `rok_str_float_ok` / `rok_str_to_f64` 위에 구현됩니다 ([refs-lang 8.5](../../refs-lang/008-abi/805-extern-symbols.md#문자열-helper)).

```rok
let n: i32 = try "42".parse();         // FromStr for i32
let f: f64 = try "3.14".parse();       // FromStr for f64
let b: bool = try "true".parse();      // FromStr for bool

// 사용자 정의
impl FromStr for Color {
    type Err = ParseColorError;
    fn from_str(s: str) throws (ParseColorError) -> Color { ... }
}
let c: Color = try "red".parse();
```

`FromStr` 은 [2.11 conversion](211-conversion.md) 의 `TryFrom<str>` 와 **별개** 입니다. `str → T` 변환만 다루며 fallible 함을 명시적으로 드러낸다는 차이가 있습니다.

## 2.15.8. 모듈 매핑

세 타입의 등록 방식과 구현 위치는 다음 표에 정리됩니다.

| 타입     | struct 이름          | 등록                 | 구현            |
|----------|----------------------|----------------------|-----------------|
| `str`    | `str` (lowercase)    | 언어 기본값          | 표준 라이브러리 |
| `StrBuf` | `StrBuf`             | 일반 타입            | 표준 라이브러리 |
| `c_char` | `c_char` (lowercase) | 언어 기본값          | 표준 라이브러리 |
| `char`   | `char` (내장)        | 언어 기본            | 컴파일러 (메서드: 표준 라이브러리) |

intrinsic 부착에 관해서는, `str` 메서드 본문 일부와 `char` 메서드 일부에 `#[intrinsic("rok.str.*")]` 또는 `#[intrinsic("rok.char.*")]` 가 부착됩니다.

## 2.15.9. 다른 언어와의 비교

| 측면         | Rust              | Python 3                | Java                    | rok                                |
|--------------|-------------------|-------------------------|-------------------------|------------------------------------|
| 타입 분리    | `&str` + `String` | `str` (단일)            | `String`                | `str` + `StrBuf`                   |
| mutability   | `String` 만 가변  | **불변**                | **불변**                | `str` 불변, `StrBuf` 가변          |
| 인코딩       | UTF-8             | unspecified (구현 의존) | UTF-16                  | UTF-8                              |
| char         | `char` (4B)       | `str` (single-char)     | `char` (2B = code unit) | `char` (4B Unicode scalar)         |
| FFI byte     | `c_char` / `u8`   | `bytes`                 | `byte`                  | `c_char` (1B target dep)           |
| len 의미     | byte len          | char count              | UTF-16 code unit        | byte len (`.chars().count()` 명시) |
| index `s[i]` | **없음**          | char (Python str)       | UTF-16 code unit        | **없음** (chars/bytes 명시)        |

먼저 `str` 을 불변 `(ptr, len)` 값으로 두고 가변 buffer 는 `StrBuf` 로 분리했습니다. GC 가 있으므로 Rust 와 달리 수명 구분은 필요 없습니다. 다음으로 `len()` 이 **byte length** 를 의미하도록 하여 Rust 와 동형으로 맞추었고, char 개수는 `chars().count()` 로 따로 얻습니다. 또한 `s[i]` 를 **제공하지 않음**으로써 Rust 와 동형을 유지하고 byte vs char 모호성을 회피합니다. `char` 를 4B Unicode scalar 로 정의함으로써 Java 의 2B UTF-16 code unit 이 야기하는 surrogate 함정을 피할 수 있게 되었습니다. 마지막으로 `c_char` 를 별도 타입으로 두어 `char` 와 ABI 를 명확히 구분합니다.
