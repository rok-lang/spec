# 2.15. 문자열: `str` / `char` / `c_char`

`rok::core::str` 모듈은 문자열과 문자 타입의 권위 있는 정의를 둡니다. Rok은 **UTF-8 우선** 정책을 따릅니다. 프로세스 안의 모든 문자열은 UTF-8로 다루고, `c_char`는 FFI 경계에서만 씁니다.

`str`, `char`, `c_char`는 struct가 아니라 컴파일러에 내장된 원시 타입입니다. 컴파일러는 이들의 lang slot(`str_type`, `char_type`, `cchar_type`)을 비워 둡니다([0.4](../000-meta/004-lang-slot-table.md)). 이 모듈은 그 메서드와 트레잇 구현을 정의합니다.

## 2.15.1. 문자열 모델: 불변 `str`, 가변 `StrBuf`

`str`은 불변 UTF-8 문자열 값입니다. 문자열을 만들어 가는 가변 버퍼는 별도 타입 `StrBuf`입니다.

| 타입     | 의미                                               | 메모리 표현                  | 가변성              |
|----------|----------------------------------------------------|------------------------------|---------------------|
| `str`    | UTF-8 문자열 값                                    | `(ptr, len)` 값, 바이트는 immortal 상수 또는 GC 객체  | **불변**  |
| `StrBuf` | 늘어나는 UTF-8 버퍼                                | GC 힙 객체                   | 가변                |
| `char`   | Unicode scalar value (언어 기본값)                  | 4B (u32)                     | (값, 변경 불가)     |
| `c_char` | C ABI 1바이트 문자 (FFI 원시 타입)                 | 1B (i8 또는 u8, 타깃 의존)   | (값)                |

### Rust / Python / Rok 모델 비교

세 언어가 문자열을 다루는 방식은 다음과 같습니다.

| 측면       | Rust             | Python         | Rok                      |
|------------|------------------|----------------|--------------------------|
| 값         | `&str`           | `str`          | `str` (불변 `(ptr, len)`) |
| 가변 버퍼  | `String`         | `io.StringIO` 등 | `StrBuf`               |
| 메모리     | borrow checker   | refcount       | GC                       |

Rok의 `str`은 수명을 추적하지 않아도 된다는 점에서 Python의 불변 `str`에 가깝습니다. 표현은 Rust의 `&str`처럼 `(ptr, len)` 쌍입니다. 런타임에 만든 문자열의 바이트는 GC 객체(type id `1`, [refs-lang 8.2.12](../../refs-lang/008-abi/802-scan-table.md#8212-런타임-예약-type-id))에 있습니다.

## 2.15.2. `str`: 불변 UTF-8 문자열

`str`은 컴파일러에 내장된 원시 타입이라 선언도, struct도, `#[keyword]`도 필요하지 않습니다. 값은 바이트의 주소와 바이트 길이(char 개수가 아님)로 된 `(ptr, len)` 쌍입니다.

`str`은 값으로 전달되는 `(ptr, len)` 쌍이며, 가리키는 바이트는 바뀌지 않습니다. 문자열을 바꾸는 연산은 모두 새 `str`을 돌려줍니다.

### 핵심 메서드

```rok
impl str {
    fn new() -> str;                        // ""
    fn from_utf8(bytes: List<u8> | Slice<u8>) throws (Utf8Error) -> str;
    fn from_utf8_lossy(bytes: List<u8> | Slice<u8>) -> str;   // 잘못된 시퀀스마다 → U+FFFD

    fn len(self) -> usize;                  // byte 길이
    fn char_count(self) -> usize;           // char 개수, chars().count() 와 같음
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
    fn splitn(self, n: usize, sep: str) -> SplitNIter;    // 최대 n 조각, 마지막은 나누지 않은 나머지
    fn rsplit(self, sep: str) -> RSplitIter;              // split 의 조각을 끝에서부터
    fn rsplitn(self, n: usize, sep: str) -> RSplitNIter;  // 끝에서부터 최대 n 조각
    fn match_indices(self, pat: str) -> MatchIndicesIter; // (byte offset, 일치한 str), 겹치지 않음

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
    fn trim_matches(self, pat: str) -> str;           // 앞뒤에서 반복되는 pat 을 모두 뗌
    fn trim_start_matches(self, pat: str) -> str;
    fn trim_end_matches(self, pat: str) -> str;

    fn to_lowercase(self) -> str;           // 새 str (Unicode 전체 매핑: 1:N, 문맥 규칙 포함)
    fn to_uppercase(self) -> str;
    fn replace(self, from: str, to: str) -> str;   // 빈 from 은 모든 char 경계에서 맞음: "ab".replace("", "-") == "-a-b-"

    fn parse<T: FromStr>(self) throws (T::Err) -> T;
}

// UTF-8 안전 slicing: s[a:b] 가 여기로 dispatch 되며 byte boundary 를 검증
impl Sliceable<Range<usize>> for str { type Output = str; }   // 바이트 공유
```

`splitn`과 `rsplitn`은 조각을 최대 `n`개 냅니다. 마지막 조각은 나누지 않은 문자열의 나머지이며, `n == 0`이면 아무것도 내지 않습니다. `rsplitn`은 끝에서부터 찾으므로 `"a.b.c".rsplitn(2, ".")`는 `"c"`, 그다음 `"a.b"`를 냅니다. `split`처럼 빈 `sep`은 문자열 전체를 한 조각으로 냅니다. `match_indices`는 일치할 때마다 `(usize, str)` 쌍을 왼쪽에서 오른쪽으로 냅니다. 일치한 뒤에는 그 뒤부터 다시 찾으므로 `"aaaaa".match_indices("aa")`는 오프셋 0과 2를 냅니다. 빈 `pat`은 아무것과도 일치하지 않습니다. `trim_matches`는 양 끝에서 `pat`의 온전한 반복을 뗍니다(`"--x--".trim_matches("-") == "x"`, `"aaa".trim_matches("aa") == "a"`). 빈 `pat`이면 문자열은 그대로입니다.

```rok
"key=a=b".splitn(2, "=");               //→ "key", "a=b"
"x.tar.gz".rsplitn(2, ".");             //→ "gz", "x.tar"
"ab-ab".match_indices("ab");            //→ (0, "ab"), (3, "ab")
"--x--".trim_start_matches("-");        //→ "x--"
```

`as_ptr`는 문자열 바이트가 있는 자리를 가리키는 포인터를 돌려줍니다. 포인터 값이 나올 수 있는 함수에서는, 그 `str`을 담은 변수가 스코프 안에 있는 동안 GC가 문자열을 옮기지 않고 포인터도 유효합니다. 포인터 값이 나올 수 있는 함수란 변수나 식의 타입에 `rok::bare::ptr` 타입이 이름으로 나오는 함수입니다. 타입 인자(`List<ConstPtr<u8>>`)나 함수 타입 안에 나와도 됩니다. 제너레이터나 `async` 본문에서는 `yield` / `await`를 지나서도 마찬가지입니다. 그 밖에서는 변수가 문자열을 마지막 사용까지만 붙듭니다. 다른 객체를 거쳐서만 닿는 문자열은 고정(pin)되지 않으므로 GC가 옮길 수 있습니다([stdlib 1.6](../100-bare/106-gc-pin.md)). 바이트를 더 오래 그 자리에 두려면 문자열을 고정하세요([1.6.6](../100-bare/106-gc-pin.md#166-추출-메서드)).

### `StrBuf`: 가변 버퍼

문자열을 조금씩 만들어 갈 때는 `StrBuf`를 씁니다.

```rok
struct StrBuf { /* GC heap 핸들: len / capacity / bytes */ }

impl StrBuf {
    fn new() -> StrBuf;                     // capacity 16
    fn with_capacity(cap: usize) -> StrBuf;
    fn len(self) -> usize;                  // byte 길이
    fn is_empty(self) -> bool;
    fn capacity(self) -> usize;             // byte 단위
    fn clear(self);                         // 길이 0, capacity 유지
    fn reserve(self, additional: usize);    // `additional` 바이트를 더 담을 자리 확보 (두 배씩 증가)
    fn push(self, c: char);                 // char 1 개 추가 (UTF-8 인코딩)
    unsafe fn push_byte(self, b: u8);       // 내용이 UTF-8 로 남도록 호출자가 보장. 안전한 바이트 경로는 str::from_utf8
    fn push_str(self, s: str);
    fn to_str(self) -> str;                 // 현재 내용의 불변 사본
}

impl !Sync for StrBuf {}
```

`StrBuf`는 컬렉션처럼 참조 핸들이므로 메서드가 `self`를 받습니다([2.9](209-collections.md)). 핸들의 사본이 버퍼 하나를 공유하므로 `Sync`가 아닙니다([2.13.9](213-marker.md#2139-send--sync-스레드-경계-마커)). `Display`는 현재 내용을 그대로 출력합니다.

```rok
let buf = StrBuf::new();
for x in xs {
    buf.push_str(f"{x},");
}
let s: str = buf.to_str();
```

### 문자열 리터럴: `"hello"`

```rok
let s: str = "hello";        // immortal 상수 바이트를 가리키는 str
```

컴파일러는 `"hello"` 리터럴을 immortal 영역의 상수 바이트를 가리키는 `str` 값으로 낮춥니다. `str`이 불변이므로 복사나 copy-on-write가 없습니다.

### 트레잇 구현

| 트레잇                      | impl                           | 동작                                       |
|-----------------------------|--------------------------------|--------------------------------------------|
| `Display`                   | ✓                              | self 그대로                                |
| `Debug`                     | ✓                              | `"..."` 이스케이프                         |
| `Clone`                     | ✓                              | 같은 값 (불변이라 바이트 공유)             |
| `Copy`                      | ✓                              | `(ptr, len)` 값 복사                       |
| `PartialEq`                 | ✓                              | 내용 비교 (바이트 단위)                    |
| `Eq`                        | ✓                              |                                            |
| `PartialOrd`                | ✓                              | 바이트 사전식 비교                         |
| `Ord`                       | ✓                              |                                            |
| `Hash`                      | ✓                              |                                            |
| `IntoIterator`              | **없음**                       | `.chars()` 명시, 바이트와 char의 모호성 회피 |
| `Add<str>` (`s + t`)        | ✓ → str (concat, 새 str)       |                                            |
| `AddAssign<str>` (`s += t`) | ✓ → `s = s + t` (새 str로 교체) |                                          |
| `FromIterator<char>`        | ✓                              | `(chars).collect::<str>()`                 |
| `FromIterator<str>`         | ✓                              | `(strs).collect::<str>()`                  |

> [!NOTE]
> `str`은 불변 값이라 `let t = s;`는 `(ptr, len)`을 복사할 뿐이며, 어느 쪽도 내용을 바꿀 수 없습니다.

## 2.15.3. `s[a:b]` 슬라이싱: UTF-8 안전

```rok
let s: str = "hello, 세계!";
s.len();                       //→ 14  (byte 길이: "hello, " 7 + 한국어 3바이트 × 2 + "!" 1)
s.chars().count();             //→ 10  (char 개수)
s.char_count();                //→ 10  (같은 값, iterator 없이)

let s2 = s[7:13];              //→ "세계"  (byte boundary OK)
let s3 = s[8:9];               // panic: UTF-8 sequence 중간 byte
let s4 = s[7:100];             //→ "세계!" (끝은 길이로 잘라 맞춤)
let s5 = s[-1:];               //→ "!"
```

`str` / `StrBuf`의 경계는 바이트 오프셋입니다. 경계는 먼저 Python처럼 범위 안으로 잘라 맞춥니다(음수는 끝에서부터, [refs-lang 3.6.2](../../refs-lang/003-expressions/306-indexing-and-slicing.md#362-슬라이싱-arrstartend)). 그 위치가 UTF-8 시퀀스 중간이면 panic이 일어납니다. 결과는 원래 바이트를 공유하는 `str`입니다(아래 참고). `str` 슬라이스에는 step이 없으며 `s[::2]`는 타입 에러입니다.

### 직접 인덱싱 거부

```rok
let s: str = "hello";
let c = s[0];                  // ERROR: `Index<usize>` 정의 안 함
                               //         이유: byte vs char 모호 + UTF-8 안전성
```

대신 `s.chars().next()`나 `s.byte(0)`처럼 의도를 명시해야 합니다.

### 부분 문자열은 복사하지 않음

`str` 값 `s`의 `s[a:b]`, `lines`, `split*`, `rsplit*`, `split_whitespace`, `split_once`, `match_indices`, `trim*`, `strip_*`의 결과는 원래 바이트를 공유하는 `str`이며 O(1)입니다([refs-lang 8.8.1](../../refs-lang/008-abi/808-lowering-types.md#881-원시-정수--부동소수)). 그래서 작은 조각이 큰 원본을 살려 둡니다. 조각을 원본에서 떼어 내는 API는 없습니다. `as_bytes()`도 복사 없는 읽기 전용 `Slice<u8>` 뷰입니다. `StrBuf`의 슬라이스는 그 범위의 사본을 담은 새 `str`이며 O(b − a)입니다. `str`은 불변이므로 아직 바뀔 수 있는 버퍼를 공유할 수 없기 때문입니다.

## 2.15.4. `char`: Unicode scalar value

`char`는 언어 내장 타입입니다([2.1](201-numbers.md)). struct가 아닌 4바이트 Unicode scalar value이며, 이 모듈이 그 메서드를 정의합니다.

Unicode scalar value의 범위는 0..=0xD7FF와 0xE000..=0x10FFFF의 합집합입니다. 서로게이트 범위인 0xD800..=0xDFFF는 **자동 niche**로 씁니다. 그래서 `char | none`을 4B로 인코딩할 수 있습니다.

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
    unsafe fn from_u32_unchecked(n: u32) -> char;        // 검사 없음, 호출자가 scalar value 를 보장
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
    fn to_lowercase_str(self) -> str;    // 1:N 을 포함한 전체 매핑
    fn to_uppercase_str(self) -> str;

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

`char`의 `to_lowercase` / `to_uppercase`는 한 문자를 돌려주므로 1:1 매핑만 적용합니다. 한 문자가 여러 문자로 바뀌는 경우는 그대로 둡니다(`'ß'.to_uppercase() == 'ß'`). 전체 매핑은 `to_lowercase_str` / `to_uppercase_str`가 돌려줍니다(`'ß'.to_uppercase_str() == "SS"`). `str`의 `to_lowercase` / `to_uppercase`는 전체 매핑을 적용합니다. 1:N 매핑과 문맥 규칙(그리스어 끝 시그마 `Σ` → `ς`)도 포함합니다(`"ß".to_uppercase() == "SS"`).

Unicode 속성과 대소문자 표는 UCD 18.0.0입니다.

### 변환

```rok
char::from_u32(0x41);                //→ 'A' (char | none; surrogate 는 none)
char::from_digit(5, 10);             //→ '5'

let n: u32 = c as u32;               //→ 65 (lossless)
let c2: char = n as char;            //→ ERROR: u8 만 `as char` 가능
let b: u8 = 0x41;
let c4: char = b as char;            //→ 'A' (u8 은 늘 scalar value)
let c3 = try char::try_from(n);      //→ char (TryFrom<u32>: from_u32 를 써서, none 이면 OverflowError)
```

### 트레잇 구현

`char`는 모든 cmp / fmt 트레잇과 함께 `Copy`, `Clone`, `Hash`를 구현합니다. niche는 자동으로 적용되므로 `char | none`의 크기는 4B입니다.

## 2.15.5. `c_char`: C ABI 1바이트(언어 내장)

`c_char`는 struct가 아니라 컴파일러에 내장된 FFI 원시 타입입니다. 크기는 1바이트입니다. 부호가 있는지(`i8`) 없는지(`u8`)는 타깃(OS / 아키텍처)에 따라 다릅니다. 이 모듈은 비교, 순서, 해시를 그 바이트(`c as u8`) 기준으로 정의합니다.

C의 `char`와 ABI가 호환됩니다. UTF-8 인코딩 단위가 아니라 날 바이트(raw byte)입니다. 용도는 FFI 전용입니다.

```rok
extern "C" {
    fn strlen(s: ConstPtr<c_char>) -> usize;
    fn strcmp(a: ConstPtr<c_char>, b: ConstPtr<c_char>) -> i32;
}
```

### 리터럴

```rok
let c: c_char = c'A';        //→ 65 (i8/u8, target dep)
let n: c_char = c'\n';       //→ 10
```

접두사가 붙은 `c'...'` 형태로 char와 구분합니다.

### char와 c_char의 차이

두 타입을 비교하면 차이가 분명합니다.

| 측면       | `char`                              | `c_char`                      |
|------------|-------------------------------------|-------------------------------|
| 크기       | 4바이트                             | 1바이트                       |
| 의미       | Unicode scalar (code point)         | C ABI byte                    |
| 범위       | 0..=0x10FFFF (서로게이트 제외)      | 0..=255 또는 -128..=127       |
| 멀티바이트 | char 하나로 BMP / SMP 모두 표현     | 멀티바이트 문자 = 바이트 시퀀스 |
| 리터럴     | `'A'`                               | `c'A'`                        |
| 등록       | 언어 기본값                         | 언어 기본값                   |
| 사용처     | 일반 문자열 처리                    | FFI / 바이트 단위 처리        |

그래서 표준 라이브러리 함수는 **`char` 우선**을 원칙으로 하며, `c_char`는 명시적인 FFI 경계에서만 씁니다.

## 2.15.6. UTF-8 불변식: 프로세스 안의 모든 문자열

`str` 값은 항상 올바른 UTF-8임이 보장됩니다. 근거는 다음과 같습니다. 생성자가 검증하므로 `str::from_utf8(bytes)`는 잘못된 바이트열이면 `Utf8Error`를 던집니다(`throws (Utf8Error)`). 슬라이싱은 바이트 경계를 검증해 멀티바이트 시퀀스 중간을 거부합니다. `push`나 `insert` 같은 제자리 수정도 UTF-8 무결성을 지킵니다. 검증을 우회할 수 있는 것은 `rok::bare::str_from_bytes_unchecked`뿐이며, 이 함수는 unsafe 계층에 속합니다.

### 검증 우회를 제한하는 이유

UTF-8 불변식이 **깨지면** 문제가 생깁니다. `chars()`가 `IndexError`를 던지거나 무한 루프에 빠질 수 있습니다. 잘린 멀티바이트 시퀀스가 깨져 표시될 수 있고, overlong encoding 같은 보안 문제가 생길 수도 있습니다.

표준 라이브러리가 생성 시점에 검증을 보장하므로 사용자 코드에서 `str.chars()` 등은 항상 안전하게 동작합니다. unsafe 코드에 들어가지 않고는 불변식을 깰 수 없습니다.

### byte ↔ str 변환

```rok
let bytes: List<u8> = [0x68, 0x65, 0x6C, 0x6C, 0x6F];
let s: str = try str::from_utf8(bytes);
                                          //→ "hello"

let invalid: List<u8> = [0xFF, 0xFE];
let t: str = try str::from_utf8(invalid);
                                          //→ throw Utf8Error { valid_up_to: 0 }
```

```rok
struct Utf8Error {
    pub valid_up_to: usize;   // 유효한 UTF-8 접두부의 byte 길이
}
```

`Utf8Error`는 `Error`를 구현하며, `Display`는 `invalid UTF-8 at byte {valid_up_to}`입니다.

`str::from_utf8_lossy(bytes)`는 에러를 던지지 않습니다. 잘못된 시퀀스의 maximal subpart 하나하나가 U+FFFD 하나가 됩니다(Unicode §3.9, U+FFFD Substitution of Maximal Subparts, WHATWG UTF-8 decoder의 동작).

## 2.15.7. `FromStr` 트레잇: 파싱

```rok
trait FromStr {
    type Err;
    fn from_str(s: str) throws (Err) -> Self;
}
```

`str.parse::<T>()`는 `throws (T::Err)`이며 이 트레잇으로 디스패치됩니다. 숫자 타입과 `bool`의 `Err`는 core의 `ParseError { input, pos, reason }`([2.14.4](214-error.md#2144-내장-에러-타입))입니다. 숫자 파싱은 런타임 helper `rok_str_parse_int` / `rok_str_parse_float` 위에 구현됩니다([refs-lang 8.5](../../refs-lang/008-abi/805-extern-symbols.md#문자열-헬퍼)). 실수의 `FromStr`은 유한한 숫자가 타입의 범위를 넘으면 `number out of range`로 실패합니다. `inf` / `NaN` 텍스트는 파싱됩니다.

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

`FromStr`은 [2.11 conversion](211-conversion.md)의 `TryFrom<str>`와 **별개**입니다. `str → T` 변환만 다루고, 실패할 수 있다는 점을 명시적으로 드러냅니다.

## 2.15.8. 모듈 매핑

세 타입의 등록 방식과 구현 위치입니다.

| 타입     | 등록          | 구현                               |
|----------|---------------|------------------------------------|
| `str`    | 언어 기본값   | 컴파일러 (메서드: 표준 라이브러리) |
| `StrBuf` | 일반 struct   | 표준 라이브러리                    |
| `c_char` | 언어 기본값   | 컴파일러 (메서드: 표준 라이브러리) |
| `char`   | 언어 기본값   | 컴파일러 (메서드: 표준 라이브러리) |

`str`과 `char`의 메서드 본문은 `rok::core::rt`에 `extern "rok"` 함수로 선언된 런타임 helper를 호출합니다. `rok_str_*`는 길이, 비교, 검색, 슬라이싱, 대소문자 변환, 공백 제거, 파싱을 맡고, `rok_char_*`는 Unicode 속성과 대소문자를 맡습니다([refs-lang 8.5](../../refs-lang/008-abi/805-extern-symbols.md#문자열-헬퍼)). 이 helper에는 `#[intrinsic]` 속성이 붙지 않습니다.

## 2.15.9. 다른 언어와 비교

| 측면         | Rust              | Python 3                | Java                    | Rok                                |
|--------------|-------------------|-------------------------|-------------------------|------------------------------------|
| 타입 분리    | `&str` + `String` | `str` (단일)            | `String`                | `str` + `StrBuf`                   |
| 가변성       | `String`만 가변  | **불변**                | **불변**                | `str` 불변, `StrBuf` 가변          |
| 인코딩       | UTF-8             | 정하지 않음 (구현 의존) | UTF-16                  | UTF-8                              |
| char         | `char` (4B)       | `str` (single-char)     | `char` (2B = code unit) | `char` (4B Unicode scalar)         |
| FFI byte     | `c_char` / `u8`   | `bytes`                 | `byte`                  | `c_char` (1B target dep)           |
| len 의미     | byte len          | char count              | UTF-16 code unit        | byte len (`.char_count()` 명시)    |
| index `s[i]` | **없음**          | char (Python str)       | UTF-16 code unit        | **없음** (chars/bytes 명시)        |

`str`은 불변 `(ptr, len)` 값이고, 가변 버퍼는 `StrBuf`로 분리했습니다. GC가 있으므로 Rust와 달리 수명을 구분하지 않아도 됩니다. `len()`은 Rust와 같이 **바이트 길이**를 뜻하며, char 개수는 `char_count()`(또는 `chars().count()`)로 따로 얻습니다. `s[i]`를 **두지 않아** Rust와 맞추고 바이트와 char 사이의 모호성을 피합니다. `char`를 4B Unicode scalar로 정의해 Java의 2B UTF-16 code unit이 일으키는 서로게이트 함정을 피합니다. 마지막으로 `c_char`를 별도 타입으로 두어 `char`와 ABI를 분명히 구분합니다.
