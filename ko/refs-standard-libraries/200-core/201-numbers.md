# 2.1. 숫자 타입

`rok::core::num` 모듈은 rok 의 모든 수치 타입을 정의합니다. rok 은 두 세계의 숫자를
함께 두며, 사용자가 자유롭게 어느 쪽을 쓸지 선택할 수 있습니다.

```
int / float    (auto-promote, 무한 정밀도, 사용자 친화적 default)
fixed primitive (i8..i64, u8..u64, isize, usize, f32, f64; 하드웨어 친화적)
```

이 두 세계는 서로 **공존** 합니다. bare 리터럴은 `int` / `float` 로 풀리고,
suffix 가 붙은 리터럴 (`42i32`, `3.14f64`) 은 fixed primitive 로 해제됩니다.
즉 어느 세계를 쓸지는 전적으로 사용자의 선택입니다.

## 2.1.1. 두 세계의 숫자 — 분리하는 이유

### `int` / `float` — auto-promote 세계

이 세계는 무한 정밀도를 지향합니다. overflow 자체가 발생하지 않으며, 메모리
한계에 도달했을 때만 OOM 으로 끝납니다. 작은 값은 hardware fixed-width 로
저장되는 small-int optimization 을 거치고, 값이 커지면 heap 의 BigInt 로 자동
promote 됩니다. 그 결과 사용자는 산술 정확성을 보장받게 되어, `I32_MAX + 1` 가
silent wrap 되어 황당한 결과를 내는 일이 없습니다.

### fixed primitive — 하드웨어 친화 세계

반면 fixed primitive 는 하드웨어에 가까운 세계입니다. `i32` / `f64` 등 14
종으로 구성되며, syntax 는 [refs-lang 2.1 타입 표현식](../../refs-lang/002-types/201-type-expressions.md) 이
정의합니다. 메모리와 성능을 정밀하게 제어해야 하는 상황 (FFI 인터페이스, SIMD,
큰 array, 핫 루프) 에 적합하고, overflow 정책이 명시되어 있습니다 ([2.3 ops
산술](203-ops.md) 의 fixed-width 산술 절을 참고합니다).

### 어느 쪽이 default 인가

bare 리터럴 (`42`, `3.14`) 은 **`int` / `float`** 가 됩니다. 즉 사용자가 별도의
typing 을 하지 않으면 자동으로 auto-promote 세계에서 코드가 작성됩니다.

```rok
let total = 0;                    // int (auto-promote)
for x in xs { total = total + x; }
// total 이 i32 범위 넘어가도 자동 promote — silent overflow 없음

let perf_critical: i64 = 0;       // 명시 i64 (fixed-width)
for x in big_xs { perf_critical = perf_critical + x as i64; }
// 사용자가 i64 범위 안에서 안전성 책임 — wrap 가능
```

## 2.1.2. `int` — 무한 정밀도 정수

```rok
#[keyword("int")]
#[lang("int_type")]
struct Int { ... }  // 내부 표현: small-int 또는 BigInt

impl Int {
    fn new() -> int;              // 0 으로 초기화
    fn from_i64(n: i64) -> int;
    fn from_u64(n: u64) -> int;
    fn from_str(s: str, radix: u32) -> int | none;  // 파싱

    fn to_i32(self) -> i32 | none;  // 범위 안이면 값, 밖이면 none
    fn to_i64(self) -> i64 | none;
    fn to_f64(self) -> f64;        // 항상 가능 (정밀도 손실 가능)
    ...
}
```

`#[keyword("int")]` 가 식별자 `int` 를 키워드로 등록하기 때문에, 사용자
코드에서는 PascalCase `Int` 가 아닌 lowercase `int` 로 타입을 표기하게 됩니다
([refs-lang 6.6](../../refs-lang/006-meta-system/606-keyword-magic.md) 가
권위입니다). 그리고 `#[lang("int_type")]` 가 컴파일러의 bare integer literal
lookup 슬롯과 연결됩니다.

`int` 의 핵심 특성을 정리해 보면 다음과 같습니다. 메모리 표현 측면에서 작은
값은 64-bit word 안에 직접 저장되는 small-int optimization 을 거치고, 큰 값은
heap 의 BigInt 표현으로 자동 promote 되며, 이 모든 과정은 사용자에게
투명하게 처리됩니다. **promote 시점** 은 산술 결과가 small-int 범위 (`-2^62 ~
2^62-1` 정도) 를 벗어날 때이며, 그 이후로는 BigInt 표현이 유지됩니다. 다시
값이 작아지더라도 small-int 로 demote 되지는 않는데, 이는 성능 안정성을 위한
선택입니다. **lang slot** 으로는 `int_type` ([0.4 lang slot
표](../000-meta/004-lang-slot-table.md)) 이 사용되며, 이는 bare integer literal
(`42`) 의 타입을 결정합니다.

### 산술

`int` 끼리의 모든 산술은 절대 wrap 되거나 saturate 되지 않습니다.

```rok
let big = 1;
big = big * 1000;
big = big * 1000;
big = big * 1000;
big = big * 1000;
big = big * 1000;
big = big * 1000;
// big = 10^18 — i64 범위 거의 끝
big = big * 1000;
// big = 10^21 — i64 범위 밖, BigInt promote
big = big * big;
// big = 10^42 — 더 큰 BigInt, 정확
```

산술 트레잇 ([2.3 ops](203-ops.md)) 의 `Add`, `Sub`, `Mul`, `Div`, `Rem` 이
모두 구현되어 있습니다. 여기서 `Div` 는 정수 나눗셈 (절삭) 으로 동작하므로,
`7 / 2 = 3`, `-7 / 2 = -3` 과 같이 계산됩니다. 비교 트레잇 ([2.5
cmp](205-cmp.md)) 도 `Eq`, `Ord`, `Hash` 가 모두 구현되어 있습니다.

### 주요 사용처

`int` 는 다양한 상황에서 활용됩니다. REPL 이나 Jupyter 의 산술 default 로
사용되어 매번 typing 하지 않아도 정확성이 보장되며, 수치 시뮬레이션의 누적
합이나 곱을 계산할 때 overflow 걱정이 없습니다. 또한 암호 연산이나 큰 정수
처리에서 별도의 BigInt 라이브러리 의존성 없이 표준 라이브러리만으로 작업이
가능합니다.

### 한계

물론 `int` 에도 한계는 존재합니다. 메모리 측면에서 BigInt promote 시 heap
할당이 필요하므로, 핫 루프나 큰 array 에서는 fixed primitive 가 권장됩니다.
성능 측면에서도 small-int 범위 안에서는 fixed primitive 와 비슷하지만, BigInt
promote 이후에는 느려질 수밖에 없습니다. 그리고 FFI 의 경우 `extern "C"`
함수가 fixed primitive 만 받기 때문에, `int` 는 `i64` 등으로 명시 변환한 뒤
전달해야 합니다.

## 2.1.3. `float` — 무한 정밀도 부동소수

```rok
#[keyword("float")]
#[lang("float_type")]
struct Float { ... }  // 내부: f64 또는 BigFloat (드문 경우)

impl Float {
    fn from_f64(x: f64) -> float;
    fn from_str(s: str) -> float | none;

    fn to_f32(self) -> f32 | none;     // 정밀도 손실 가능 시 none?
    fn to_f64(self) -> f64;
    ...
}
```

`#[keyword("float")]` 가 `float` 키워드를 등록하면서 struct `Float` 로
라우팅합니다. 따라서 사용자 코드에서는 lowercase `float` 만 보이게 됩니다.

`float` 는 기본적으로 f64 표현을 사용합니다. 산술 결과가 f64 범위를 넘어가는
경우 (지수 영역 초과) BigFloat (multi-precision) 으로 promote 됩니다.

### 한계 명시

`float` 의 한계도 분명히 짚어 두어야 합니다. **정밀도** 측면에서 f64 표현
안에서는 IEEE 754 와 동일한 round-off 가 발생합니다. 예를 들어 `0.1 + 0.2` 가
`0.3` 이 되지 않는 현상은 rok 에서도 동일하게 나타납니다. 또한 `Eq` 가
구현되어 있지 않은데, `float` 는 `Eq` 마커 트레잇을 구현하지 않고 NaN 처리를
위해 `PartialEq` 만 구현합니다. 자세한 위계는 [2.5 cmp](205-cmp.md) 를
참고합니다. overflow 와 NaN, Inf 처리는 IEEE 754 를 그대로 따릅니다.

### lang slot

`float_type` ([0.4 lang slot 표](../000-meta/004-lang-slot-table.md)) 이 bare
float literal (`3.14`) 의 타입을 담당합니다.

## 2.1.4. fixed-width primitive

[refs-lang 2.1 타입 표현식](../../refs-lang/002-types/201-type-expressions.md) 이 14 primitive 의 **이름** 을
정의합니다. 이 절은 그 위에서 표준 라이브러리의 메서드와 동작을 정의합니다.

### 정수 (10 종)

| 타입    | 비트 폭                         | 범위           |
|---------|---------------------------------|----------------|
| `i8`    | 8                               | -128 ~ 127     |
| `i16`   | 16                              | -32768 ~ 32767 |
| `i32`   | 32                              | -2³¹ ~ 2³¹-1   |
| `i64`   | 64                              | -2⁶³ ~ 2⁶³-1   |
| `isize` | target pointer width (32 or 64) | target-dep     |
| `u8`    | 8                               | 0 ~ 255        |
| `u16`   | 16                              | 0 ~ 65535      |
| `u32`   | 32                              | 0 ~ 2³²-1      |
| `u64`   | 64                              | 0 ~ 2⁶⁴-1      |
| `usize` | target pointer width            | target-dep     |

### 부동소수 (2 종)

| 타입  | 비트 폭 | 표준                                 |
|-------|---------|--------------------------------------|
| `f32` | 32      | IEEE 754 binary32 (single precision) |
| `f64` | 64      | IEEE 754 binary64 (double precision) |

### 부울 / 문자 (3 종)

| 타입     | 의미                                                                                  |
|----------|---------------------------------------------------------------------------------------|
| `bool`   | `true` / `false`                                                                      |
| `char`   | UTF-8 codepoint (4-byte storage); 표준 라이브러리 타입 ([2.15 string](215-string.md)) |
| `c_char` | C ABI 호환 char (1-byte signed); FFI 전용                                             |

> [!NOTE]
> [refs-lang 1.8](../../refs-lang/001-lexical/108-character-literals.md) 의 결정에
> 따라 `char` 는 표준 라이브러리 타입으로 정의되며, `c_char` 만 primitive 로 남습니다. 이
> 절의 14 primitive 중에서 `char` 는 표준 라이브러리 정의로 다뤄집니다 ([2.15
> string](215-string.md)).

### 두 부류 — 언어 기본값 vs 표준 라이브러리 추가

이 장의 수치 / 부울 / 문자 타입은 **두 부류** 로 나뉘어 있습니다.

| 부류                       | 구성원                                                                                                | 등록 방식                              |
|----------------------------|-------------------------------------------------------------------------------------------------------|----------------------------------------|
| **15 언어 기본값**         | `i8`/`i16`/`i32`/`i64`, `u8`/`u16`/`u32`/`u64`, `isize`/`usize`, `f32`/`f64`, `bool`, `c_char`, `str` | 언어가 직접 인식 — `#[keyword]` 불필요 |
| **3 표준 라이브러리 추가** | `int`, `float`, `char`                                                                                | `#[keyword(...)]` 등록                 |

언어 기본값은 컴파일러 (lexer/parser) 가 직접 인식하는 토큰이라서, 표준 라이브러리가
별도로 키워드 등록을 하지 않습니다. 그 결과 사용자가 이 이름들을 재정의할 수
없다는 사실이 자동으로 보장됩니다.

이에 비해 표준 라이브러리 추가 (`int` / `float` / `char`) 는 `#[keyword(...)]` 로
등록됩니다. 이 모듈의 `Int` / `Float` / `Char` 정의에서 `#[keyword]` attribute
가 부착됩니다 ([refs-lang
6.6](../../refs-lang/006-meta-system/606-keyword-magic.md) 가 권위입니다).

### 언어 기본값 14 primitive 의 표준 라이브러리 형태

```rok
struct i32 { ... }        // 사용자 직접 사용 (소문자 식별자가 곧 타입)
struct f64 { ... }
struct bool { ... }
struct c_char { ... }
struct str { ... }
// ... 나머지 11 종 동형
```

표준 라이브러리는 impl 블록만 부착하여 트레잇 (`Add`, `Display`, `Clone`, ...) 을
구현합니다. 타입 인식 자체는 컴파일러가 책임집니다.

### 산술 / 비교 트레잇 impl

각 primitive 가 [2.3 ops](203-ops.md), [2.4 ops-assign](204-ops-assign.md),
[2.5 cmp](205-cmp.md) 의 트레잇을 구현합니다. 모든 impl 에는
`#[intrinsic("rok.OP.TYPE")]` 가 부착되어 컴파일러 백엔드 PrimOp 와 직결
됩니다.

### 메서드 (각 primitive 별)

각 정수 / 부동소수 타입은 다음 메서드를 노출합니다 (예시는 `i32` 입니다).

```rok
impl i32 {
    fn min() -> i32;               // -2147483648
    fn max() -> i32;               // 2147483647
    fn abs(self) -> i32;
    fn pow(self, exp: u32) -> i32;
    fn count_ones(self) -> u32;
    fn leading_zeros(self) -> u32;
    fn trailing_zeros(self) -> u32;
    fn rotate_left(self, n: u32) -> i32;
    fn rotate_right(self, n: u32) -> i32;
    fn swap_bytes(self) -> i32;
    fn to_le_bytes(self) -> [u8; 4];
    fn to_be_bytes(self) -> [u8; 4];
    fn from_le_bytes(bytes: [u8; 4]) -> i32;
    fn from_be_bytes(bytes: [u8; 4]) -> i32;

    // overflow 명시 변형
    fn checked_add(self, rhs: i32) -> i32 | none;
    fn checked_sub(self, rhs: i32) -> i32 | none;
    fn checked_mul(self, rhs: i32) -> i32 | none;
    fn checked_div(self, rhs: i32) -> i32 | none;
    fn checked_rem(self, rhs: i32) -> i32 | none;
    fn checked_neg(self) -> i32 | none;
    fn checked_shl(self, n: u32) -> i32 | none;
    fn checked_shr(self, n: u32) -> i32 | none;

    fn saturating_add(self, rhs: i32) -> i32;     // i32::MAX 또는 MIN 으로 clamp
    fn saturating_sub(self, rhs: i32) -> i32;
    fn saturating_mul(self, rhs: i32) -> i32;

    fn wrapping_add(self, rhs: i32) -> i32;       // 모듈러 산술
    fn wrapping_sub(self, rhs: i32) -> i32;
    fn wrapping_mul(self, rhs: i32) -> i32;
    fn wrapping_neg(self) -> i32;
    fn wrapping_shl(self, n: u32) -> i32;
    fn wrapping_shr(self, n: u32) -> i32;
}
```

unsigned, 부동소수, isize 도 비슷한 패턴으로 구성되며, 각 타입에 적합한
메서드만 노출됩니다.

부동소수 (`f32`, `f64`) 의 경우 추가 메서드가 다음과 같이 제공됩니다.

```rok
impl f64 {
    fn nan() -> f64;             // f64::NAN
    fn infinity() -> f64;        // f64::INFINITY
    fn neg_infinity() -> f64;
    fn epsilon() -> f64;         // 2^-52

    fn is_nan(self) -> bool;
    fn is_infinite(self) -> bool;
    fn is_finite(self) -> bool;
    fn is_sign_positive(self) -> bool;
    fn is_sign_negative(self) -> bool;

    fn floor(self) -> f64;
    fn ceil(self) -> f64;
    fn round(self) -> f64;
    fn trunc(self) -> f64;
    fn fract(self) -> f64;       // self - self.trunc()

    fn abs(self) -> f64;
    fn signum(self) -> f64;      // -1.0, 0.0, or 1.0

    fn sqrt(self) -> f64;
    fn cbrt(self) -> f64;
    fn pow(self, exp: f64) -> f64;
    fn exp(self) -> f64;
    fn ln(self) -> f64;
    fn log(self, base: f64) -> f64;
    fn log2(self) -> f64;
    fn log10(self) -> f64;
    fn sin(self) -> f64;
    fn cos(self) -> f64;
    fn tan(self) -> f64;
    fn asin(self) -> f64;
    fn acos(self) -> f64;
    fn atan(self) -> f64;
    fn atan2(self, other: f64) -> f64;
    fn sinh(self) -> f64;
    fn cosh(self) -> f64;
    fn tanh(self) -> f64;

    fn to_bits(self) -> u64;     // bit pattern
    fn from_bits(bits: u64) -> f64;
}
```

## 2.1.5. literal → 어느 타입?

[refs-lang 1.6 숫자 리터럴](../../refs-lang/001-lexical/106-numeric-literals.md)
과 [2.11 변환](211-conversion.md) 을 종합하면 다음과 같이 정리됩니다.

| 형태      | 결정 타입                        |
|-----------|----------------------------------|
| `42`      | `int` (bare integer)             |
| `3.14`    | `float` (bare float)             |
| `42i32`   | `i32`                            |
| `42u64`   | `u64`                            |
| `3.14f32` | `f32`                            |
| `3.14f64` | `f64`                            |
| `'A'`     | `char` (표준 라이브러리 타입)    |
| `c'A'`    | `c_char` (primitive)             |
| `0xFF`    | `int` — bare. hexadecimal 도 int |
| `0xFFi32` | `i32`                            |

### context auto-From

annotation 이 있는 경우 컴파일 타임 range check 와 lossless 변환이
이루어집니다.

```rok
let x: i32 = 42;            // int(42) → i32 (range OK, lossless)
let y: i32 = 9999999999;    // 컴파일 에러 (int 가 i32 범위 밖)
let z: f32 = 3.14;          // float → f32 (round to nearest)
let w: u8 = 256;            // 컴파일 에러 (256 ∉ u8 range 0..=255)
```

자세한 변환 매트릭스는 [2.11 변환](211-conversion.md) 에 정리되어 있습니다.

## 2.1.6. 산술 결과 타입

### `int + int = int`

auto-promote 가 적용되므로 결과는 항상 `int` 자체가 됩니다. fixed-width 추론은
이루어지지 않습니다.

```rok
let a = 1000;        // int
let b = 2000;        // int
let c = a + b;       // int (3000)
```

### `i32 + i32 = i32`

fixed-width 는 그대로 유지됩니다. 사용자가 wider type 을 원할 경우 명시 변환을
사용해야 합니다.

```rok
let a: i32 = 5;
let b: i32 = 3;
let c: i32 = a + b;   // i32

let wider: i64 = (a as i64) + (b as i64);   // i64
```

### `int + i32` (혼합 — 거부)

```rok
let a = 5;            // int
let b: i32 = 3;       // i32
let c = a + b;        // 컴파일 에러 — type mismatch
```

이 경우 명시 변환이 강제됩니다.

```rok
let c1 = a + int::from_i32(b);    // int (a + int(3))
let c2 = (a.to_i32()? + b);       // i32 (a 가 i32 범위 안이면 OK)
```

이러한 정책 덕분에 silent type mixing 이 차단됩니다. rok 은 Julia 처럼 자동
promote 를 하지 않으며, 어느 세계를 쓸지는 사용자가 명시하도록 되어 있습니다.

## 2.1.7. fixed primitive ↔ int 변환

fixed primitive 와 `int` 사이의 변환은 명시적으로만 가능합니다.

| 변환                     | 메서드                                | 실패 시                      |
|--------------------------|---------------------------------------|------------------------------|
| `i32` → `int`            | `.into::<int>()` 또는 `int::from_i32` | 항상 성공 (lossless)         |
| `int` → `i32`            | `.try_into::<i32>()`                  | `OverflowError` 를 throw     |
| `int` → `i32` (saturate) | `.saturating_into::<i32>()`           | i32::MIN 또는 MAX            |
| `int` → `i32` (wrap)     | `.wrapping_into::<i32>()`             | 모듈러 산술                  |
| `int` → `f64`            | `.into::<f64>()` 또는 `int::to_f64`   | 정밀도 손실 가능 (round)     |

## 2.1.8. niche 최적화 (간단)

[refs-lang 2.7
niche](../../refs-lang/002-types/207-niche-optimization.md) 가 메커니즘을
정의합니다. 자동 niche 가 적용되는 primitive 는 다음과 같습니다.

`bool | none` 의 경우 1 byte 로 표현되며, none 은 bit pattern `2..=255` 중
하나로 인코딩됩니다. `Ptr<T> | none` 과 `ConstPtr<T> | none` 은 pointer 크기
그대로 표현되고, none 은 null pointer 로 처리됩니다. 그러나 `i8`..`i64`,
`u8`..`u64`, `f32`, `f64` 자체에는 자동 niche 가 적용되지 않습니다. 모든 bit
pattern 이 유효한 값이기 때문입니다.

opt-in niche 가 필요한 경우에는 [2.2 non-zero](202-non-zero.md) 의
`NonZero<T>` / `NonNan<F>` / `Finite<F>` wrapper 를 사용합니다.

## 2.1.9. lang slot 매핑

언어 기본값 14 종은 소문자 struct 이름을 사용합니다. 컴파일러가 직접
인식하므로 `#[keyword]` 가 필요하지 않습니다.

| primitive | lang slot    | struct 이름 | 등록      |
|-----------|--------------|-------------|-----------|
| `i8`      | `i8_type`    | `i8`        | 언어 기본 |
| `i16`     | `i16_type`   | `i16`       | 언어 기본 |
| `i32`     | `i32_type`   | `i32`       | 언어 기본 |
| `i64`     | `i64_type`   | `i64`       | 언어 기본 |
| `u8`      | `u8_type`    | `u8`        | 언어 기본 |
| `u16`     | `u16_type`   | `u16`       | 언어 기본 |
| `u32`     | `u32_type`   | `u32`       | 언어 기본 |
| `u64`     | `u64_type`   | `u64`       | 언어 기본 |
| `isize`   | `isize_type` | `isize`     | 언어 기본 |
| `usize`   | `usize_type` | `usize`     | 언어 기본 |
| `f32`     | `f32_type`   | `f32`       | 언어 기본 |
| `f64`     | `f64_type`   | `f64`       | 언어 기본 |
| `bool`    | `bool_type`  | `bool`      | 언어 기본 |
| `c_char`  | `cchar_type` | `c_char`    | 언어 기본 |
| `str`     | `str_type`   | `str`       | 언어 기본 |

표준 라이브러리가 추가하는 3 종은 PascalCase struct 와 `#[keyword(...)]` 로
등록됩니다.

| primitive | lang slot    | struct 이름 | 등록                  |
|-----------|--------------|-------------|-----------------------|
| `int`     | `int_type`   | `Int`       | `#[keyword("int")]`   |
| `float`   | `float_type` | `Float`     | `#[keyword("float")]` |
| `char`    | `char_type`  | `Char`      | `#[keyword("char")]`  |

참고로 `Range<T>` (`a..b`) 는 [2.7 iter](207-iter.md) 가 정의합니다.

## 2.1.10. 다른 언어와의 비교 — 디자인 결정 근거

rok 의 두-세계 모델 (`int` auto-promote 와 fixed primitive) 은 여러 언어에서
영향을 받았습니다. Smalltalk 의 SmallInteger ↔ LargeInteger 는 auto-promote
의 원형이라고 할 수 있고, Lisp 와 Scheme 의 numeric tower (fixnum / bignum /
rational / real) 도 중요한 참고가 됩니다. 또한 **Python int** 는 무한 정밀도를
default 로 두지만 fixed primitive 가 없다는 차이가 있고, **Julia** 는 fixed
primitive 를 default 로 두는 반면 rok 은 int default 와 fixed opt-in 으로
정반대 방향을 택합니다.

각 언어와의 차이를 정리하면 다음과 같습니다. **Python** 은 int 만 제공하며,
fixed-width 가 없어서 FFI 나 성능 critical 한 상황에서는 NumPy 에 의존하게
됩니다. **Julia** 는 fixed 가 default 라서 silent overflow 의 위험이 있고,
사용자가 이를 자주 잊는 경향이 있습니다. **Rust** 는 fixed only 라서 매번
사용자가 type 을 적어야만 합니다. 이에 반해 **rok** 는 두 세계가 공존하도록
하여, bare literal 의 default 가 안전한 `int` 이고 명시 typing 시에 fixed 로
들어가도록 설계되어 있습니다.
