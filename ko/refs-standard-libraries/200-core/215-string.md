# 2.15. 문자열 — `str` / `char` / `c_char`

`rok::core::str` 모듈은 문자열과 문자 타입의 권위 정의를 제공합니다. rok 은 **UTF-8 우선** 정책을 채택하고 있어서, 모든 in-process 문자열은 UTF-8 로 다루며, FFI 경계에서만 `c_char` 를 사용합니다.

이 장은 48 개의 lang slot 중 세 개를 정의합니다. 첫째로 `str_type` 은 `rok::core::str::str` 에 연결되며 언어 기본값에 해당하는 lowercase struct 입니다. 둘째로 `char_type` 은 `rok::core::str::Char` 에 연결되며 표준 라이브러리 추가 타입으로 `#[keyword("char")]` 가 부착됩니다. 셋째로 `cchar_type` 은 `rok::core::str::c_char` 에 연결되며 마찬가지로 언어 기본값입니다.

## 2.15.1. 단일 가변 문자열 모델 — `str`

rok 은 GC 환경과 가변 컬렉션 일관성에 따라 String 별도 타입을 두지 않습니다. 모든 문자열 작업은 `str` 단일 타입으로 처리됩니다.

| 타입     | 의미                                               | 메모리 표현                  | mutability          |
|----------|----------------------------------------------------|------------------------------|---------------------|
| `str`    | UTF-8 문자열 (GC heap, growable)                   | GC heap (ptr/len/capacity)   | **가변** (in-place) |
| `char`   | Unicode scalar value (PascalCase struct + keyword) | 4B (u32)                     | (값 — 변경 X)       |
| `c_char` | C ABI 1-byte char (FFI primitive)                  | 1B (i8 또는 u8 — target dep) | (값)                |

### Rust / Python / rok 모델 비교

세 언어가 문자열을 어떻게 다루는지 비교하면 다음과 같습니다.

| 측면       | Rust             | Python         | rok            |
|------------|------------------|----------------|----------------|
| view       | `&str`           | **없음**       | **없음**       |
| owned      | `String`         | **기본 `str`** | **기본 `str`** |
| mutability | `String` 만 가변 | **불변**       | **`str` 가변** |
| 메모리     | borrow checker   | refcount       | GC             |

핵심 차이는 다음과 같이 정리할 수 있습니다. Rust 는 **borrow checker** 가 있어서 view (`&str`) 와 owned (`String`) 의 구분이 필요합니다. Python 은 문자열 불변 정책을 따르기 때문에 단일 타입으로 충분합니다. 반면 rok 은 GC 와 가변 컬렉션 일관성을 추구하므로, `List<T>` 나 `Dict<K,V>` 가 가변 in-place 인 것과 동형으로 `str` 도 가변 in-place 로 동작합니다.

## 2.15.2. `str` — 가변 UTF-8 문자열

```rok
#[lang("str_type")]
struct str {
    ptr: ConstPtr<u8>;
    len: usize;            // byte length (NOT char count)
    capacity: usize;       // 할당된 buffer 크기
}
```

`str` 는 언어 기본값이라 컴파일러가 직접 인식하므로 `#[keyword]` 가 따로 필요하지 않습니다. lowercase struct 이름이 사용자 코드의 타입 표기와 동일합니다.

이 타입은 GC heap 에 할당된 가변 buffer 위에 UTF-8 invariant 를 얹은 형태이며, List<u8> 와 같은 모델에 UTF-8 안전성을 추가로 보장한다고 이해하면 됩니다.

### 핵심 메서드

```rok
impl str {
    fn new() -> str;                        // 빈 문자열
    fn with_capacity(cap: usize) -> str;
    fn from_utf8(bytes: List<u8>) -> Result<str, Utf8Error>;

    fn len(self) -> usize;                  // byte 길이
    fn is_empty(self) -> bool;
    fn capacity(self) -> usize;
    fn as_bytes(self) -> Slice<u8>;
    fn as_ptr(self) -> ConstPtr<u8>;        // FFI

    // 가변 메서드 — in-place
    fn push(ref self, c: char);             // char 1 개 추가 (UTF-8 인코딩)
    fn push_str(ref self, s: str);
    fn pop(ref self) -> char | none;        // 마지막 char 제거 (UTF-8 디코딩 후)
    fn clear(ref self);
    fn insert(ref self, byte_idx: usize, c: char);
    fn remove(ref self, byte_idx: usize) -> char;
    fn reserve(ref self, additional: usize);
    fn shrink_to_fit(ref self);

    // 읽기 메서드 — 새 str 또는 view
    fn chars(self) -> CharsIter;            // Unicode scalar iterator
    fn bytes(self) -> BytesIter;            // u8 iterator
    fn char_indices(self) -> CharIndicesIter;
    fn lines(self) -> LinesIter;
    fn split(self, sep: str) -> SplitIter;
    fn split_whitespace(self) -> SplitWhitespaceIter;

    fn starts_with(self, pat: str) -> bool;
    fn ends_with(self, pat: str) -> bool;
    fn contains(self, pat: str) -> bool;
    fn find(self, pat: str) -> usize | none;
    fn rfind(self, pat: str) -> usize | none;

    fn trim(self) -> str;                   // 새 str 반환 (필요 시)
    fn trim_start(self) -> str;
    fn trim_end(self) -> str;

    fn to_lowercase(self) -> str;           // 새 str (Unicode 변환)
    fn to_uppercase(self) -> str;
    fn replace(self, from: str, to: str) -> str;

    fn parse<T: FromStr>(self) -> Result<T, T::Err>;

    // UTF-8 안전 slicing — byte boundary 검증
    // (s[a:b] desugar 가 본 메서드 dispatch)
    fn slice(self, range: Range<usize>) -> str;        // view 또는 새 str
}
```

### 사용 예 — 가변 in-place

다음 예시는 가변 in-place 동작을 보여줍니다.

```rok
let s: str = "hello";
s.push(' ');
s.push_str("world");
s;                       //→ "hello world"

let t: str = "";
for word in ["a", "b", "c"] {
    t.push_str(word);
}
t;                       //→ "abc"

// 빈 str 누적
let buf: str = str::new();
for x in xs {
    buf.push_str(f"{x},");
}
```

### 문자열 리터럴 — `"hello"`

```rok
let s: str = "hello";        // GC heap 의 str 객체
```

`"hello"` 리터럴은 GC heap 의 str 객체로 lower 되며, 타입은 `str` 입니다. 처음에는 immortal 영역의 const buffer 를 가리키고, 사용자가 mutate 하는 시점에 GC heap 으로 **copy-on-write** 됩니다.

### 트레잇 구현

| 트레잇                      | impl                           | 동작                                       |
|-----------------------------|--------------------------------|--------------------------------------------|
| `Display`                   | ✓                              | self 그대로                                |
| `Debug`                     | ✓                              | `"..."` 이스케이프                         |
| `Clone`                     | ✓                              | 깊은 복사 (새 buffer)                      |
| `Copy`                      | ✗                              | 가변 컨테이너라 **Copy 안 함**             |
| `PartialEq`                 | ✓                              | byte-wise                                  |
| `Eq`                        | ✓                              |                                            |
| `PartialOrd`                | ✓                              | lexicographic byte                         |
| `Ord`                       | ✓                              |                                            |
| `Hash`                      | ✓                              |                                            |
| `IntoIterator`              | **없음**                       | `.chars()` 명시 — byte vs char 모호성 회피 |
| `Add<str>` (`s + t`)        | ✓ → str (concat — 새 str)      |                                            |
| `AddAssign<str>` (`s += t`) | ✓ → in-place (`push_str` 위임) |                                            |
| `FromIterator<char>`        | ✓                              | `(chars).collect::<str>()`                 |
| `FromIterator<str>`         | ✓                              | `(strs).collect::<str>()`                  |

> [!NOTE]
> `str` 는 **가변 컨테이너** 라 List/Dict 와 마찬가지로 `Copy` **하지 않음** . `let t = s;`
> 는 **참조 공유** — 두 변수가 같은 buffer 가리킴. 독립 사본 원하면 `s.clone()` 명시.

## 2.15.3. `s[a:b]` 슬라이싱 — UTF-8 안전

```rok
let s: str = "hello, 세계!";
s.len();                       //→ 16  (byte 길이 — 한국어 3바이트 × 2)
s.chars().count();             //→ 9   (char 개수)

let s2 = s[7:13];              //→ "세계"  (byte boundary OK)
let s3 = s[8:9];               // panic — UTF-8 sequence 중간 byte
                               //         IndexError("not a char boundary")
```

`Slice` 트레잇 impl 이 byte boundary 를 검증한 후 새 str 을 반환합니다. 잘못된 경계에서는 panic 이 발생합니다.

### 직접 indexing 거부

```rok
let s: str = "hello";
let c = s[0];                  // ERROR — `Index<usize>` 정의 안 함
                               //         이유: byte vs char 모호 + UTF-8 안전성
```

이 경우 `s.chars().next()` 또는 `s.as_bytes()[0]` 처럼 의도를 명시해야 합니다.

## 2.15.4. `char` — Unicode scalar value (표준 라이브러리 추가)

```rok
#[keyword("char")]
#[lang("char_type")]
struct Char {
    value: u32;        // Unicode code point — surrogate 제외
}
```

`char` 는 **표준 라이브러리 추가** 타입으로, `#[keyword("char")]` 로 등록되어 사용자 코드는 lowercase `char` 로 표기합니다. struct 본문은 PascalCase `Char` 입니다.

Unicode scalar value 의 범위는 0..=0xD7FF 와 0xE000..=0x10FFFF 의 합집합입니다. Surrogate 범위인 0xD800..=0xDFFF 는 **자동 niche** 로 활용되어, `char | none` 이 4B 로 인코딩될 수 있습니다.

### 리터럴 / 메서드

```rok
let c: char = 'A';                   //→ U+0041
let k: char = '한';                  //→ U+D55C
let e: char = '😀';                  //→ U+1F600

c.to_lowercase();                    //→ 'a'  (Unicode 변환)
c.is_alphabetic();                   //→ true
c.is_ascii();                        //→ true
'1'.to_digit(10);                    //→ 1 (Result<u32, _>)
c.len_utf8();                        //→ 1 (UTF-8 인코딩 시 byte 수)
e.len_utf8();                        //→ 4
```

### 변환

```rok
char::from_u32(0x41);                //→ 'A' (Result, surrogate 거부)
char::from_digit(5, 10);             //→ '5'

let n: u32 = c as u32;               //→ 65 (lossless)
let c2: char = n as char;            //→ ERROR — `as` 거부 (try_into 권장)
let c3 = char::try_from(n)?;         //→ Result<char, _>
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

`str` 는 항상 UTF-8 valid 임이 보장되며, 그 근거는 다음과 같습니다. 우선 생성자가 검증을 수행하므로 `str::from_utf8(bytes)` 는 `Result` 를 반환합니다. 다음으로 슬라이싱이 byte boundary 를 검증하여 멀티바이트 시퀀스 중간을 거부합니다. 또한 `push` 나 `insert` 같은 in-place 수정 역시 UTF-8 무결성을 보장합니다. 마지막으로 `rok::bare::str_from_bytes_unchecked` 만이 검증을 우회할 수 있으며 이는 unsafe tier 에 속합니다.

### 검증 우회의 이유

UTF-8 invariant 가 **깨지면** 어떤 문제가 생기는지 살펴볼 필요가 있습니다. 우선 `chars()` 가 `IndexError` 를 던지거나 무한 루프에 빠질 수 있고, 멀티바이트 sequence 가 잘려서 표시가 깨질 수 있으며, 더 나아가 overlong encoding 같은 security 이슈가 발생할 수도 있습니다.

따라서 표준 라이브러리가 빌드 시점에 검증을 보장함으로써, 사용자 코드에서 `str.chars()` 등이 항상 안전하게 동작합니다. unsafe 진입 없이는 invariant 가 깨질 수 없습니다.

### byte ↔ str 변환

```rok
let bytes: List<u8> = [0x68, 0x65, 0x6C, 0x6C, 0x6F];
let s: Result<str, Utf8Error> = str::from_utf8(bytes);
                                          //→ Ok("hello")

let invalid: List<u8> = [0xFF, 0xFE];
let s: Result<str, Utf8Error> = str::from_utf8(invalid);
                                          //→ Err(Utf8Error { ... })
```

## 2.15.7. `FromStr` 트레잇 — 파싱

```rok
trait FromStr {
    type Err;
    fn from_str(s: str) -> Result<Self, Err>;
}
```

`str.parse::<T>()` 호출은 이 트레잇을 통해 dispatch 됩니다.

```rok
let n: i32 = "42".parse()?;            // FromStr for i32
let f: f64 = "3.14".parse()?;          // FromStr for f64
let b: bool = "true".parse()?;         // FromStr for bool

// 사용자 정의
impl FromStr for Color {
    type Err = ParseColorError;
    fn from_str(s: str) -> Result<Color, ParseColorError> { ... }
}
let c: Color = "red".parse()?;
```

`FromStr` 은 [2.11 conversion](211-conversion.md) 의 `TryFrom<str>` 와 **별개** 입니다. `str → T` 변환만 다루며 fallible 함을 명시적으로 드러낸다는 차이가 있습니다.

## 2.15.8. 모듈 매핑

세 타입의 등록 방식과 구현 위치는 다음 표에 정리되어 있습니다.

| 타입     | struct 이름          | 등록                 | 구현            |
|----------|----------------------|----------------------|-----------------|
| `str`    | `str` (lowercase)    | 언어 기본값          | 표준 라이브러리 |
| `c_char` | `c_char` (lowercase) | 언어 기본값          | 표준 라이브러리 |
| `char`   | `Char` (PascalCase)  | `#[keyword("char")]` | 표준 라이브러리 |

intrinsic 부착에 관해서는, `str` 메서드 본문 일부와 `char` 메서드 일부에 `#[intrinsic("rok.str.*")]` 또는 `#[intrinsic("rok.char.*")]` 가 부착됩니다.

## 2.15.9. 다른 언어와의 비교

| 측면         | Rust              | Python 3                | Java                    | rok                                |
|--------------|-------------------|-------------------------|-------------------------|------------------------------------|
| 타입 분리    | `&str` + `String` | `str` (단일)            | `String`                | `str` (단일, **가변**)             |
| mutability   | `String` 만 가변  | **불변**                | **불변**                | **`str` 가변**                     |
| 인코딩       | UTF-8             | unspecified (구현 의존) | UTF-16                  | UTF-8                              |
| char         | `char` (4B)       | `str` (single-char)     | `char` (2B = code unit) | `char` (4B Unicode scalar)         |
| FFI byte     | `c_char` / `u8`   | `bytes`                 | `byte`                  | `c_char` (1B target dep)           |
| len 의미     | byte len          | char count              | UTF-16 code unit        | byte len (`.chars().count()` 명시) |
| index `s[i]` | **없음**          | char (Python str)       | UTF-16 code unit        | **없음** (chars/bytes 명시)        |

이 비교를 토대로 rok 의 **가장 큰 결정**들을 정리해 보겠습니다. 먼저 str 을 가변 단일 타입으로 두어 Rust 의 view/owned 분리를 하지 않으며, 이는 GC 와 가변 컬렉션 모델의 일관성을 위한 선택입니다. 다음으로 `len()` 이 **byte length** 를 의미하도록 하여 Rust 와 동형으로 맞추었고, char 개수는 `chars().count()` 로 따로 얻습니다. 또한 `s[i]` 를 **제공하지 않음**으로써 Rust 와 동형을 유지하고 byte vs char 모호성을 회피합니다. `char` 를 4B Unicode scalar 로 정의함으로써 Java 의 2B UTF-16 code unit 이 야기하는 surrogate 함정을 피할 수 있게 되었습니다. 마지막으로 `c_char` 를 별도 타입으로 두어 `char` 와 ABI 를 명확히 구분합니다.
