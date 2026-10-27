# 2.1. 숫자 타입

`rok::core::num` 모듈은 Rok 의 모든 수치 타입을 정의합니다. Rok 은 두 세계의 숫자를
함께 두며, 사용자가 자유롭게 어느 쪽을 쓸지 선택할 수 있습니다.

```
int / float    (auto-promote, 무한 정밀도, opt-in)
fixed primitive (i8..i64, u8..u64, isize, usize, f32, f64; 하드웨어 친화적, 기본)
```

이 두 세계는 서로 **공존** 합니다. 접미사 없는 리터럴은 기대 타입을 따르며, 기대 타입이 없으면 `i32` / `f64` 가 됩니다 ([refs-lang 1.6.3](../../refs-lang/001-lexical/106-numeric-literals.md#163-접미사-없는-리터럴)). `int` / `float` 는 타입을 적은 자리에서만 쓰입니다.

## 2.1.1. 두 세계의 숫자 - 분리하는 이유

### `int` / `float` - auto-promote 세계

이 세계는 무한 정밀도를 지향합니다. overflow 자체가 발생하지 않으며, 메모리
한계에 도달했을 때만 OOM 으로 끝납니다. 작은 값은 hardware fixed-width 로
저장되는 small-int optimization 을 거치고, 값이 커지면 heap 의 BigInt 로 자동
promote 됩니다. `int` / `float` 값은 불변이며, heap 의 BigInt 도 제자리에서 바뀌지 않고 공유만 됩니다 (그래서 `Copy` 입니다). 그 결과 사용자는 산술 정확성을 보장받게 되어, `I32_MAX + 1` 가
silent wrap 되어 황당한 결과를 내는 일이 없습니다.

### fixed primitive - 하드웨어 친화 세계

반면 fixed primitive 는 하드웨어에 가까운 세계입니다. `i32` / `f64` 등 15
종으로 구성되며, syntax 는 [refs-lang 2.1 타입 표현식](../../refs-lang/002-types/201-type-expressions.md) 이
정의합니다. 메모리와 성능을 정밀하게 제어해야 하는 상황 (FFI 인터페이스, SIMD,
큰 array, 핫 루프) 에 적합하고, overflow 정책이 명시됩니다 ([2.3 ops
산술](203-ops.md) 의 fixed-width 산술 절을 참고합니다).

### 어느 쪽이 default 인가

접미사 없는 리터럴 (`42`, `3.14`) 은 기대 타입이 없으면 **`i32` / `f64`** 가 됩니다. auto-promote 세계는 `int` / `float` 를 적은 자리에서만 쓰입니다.

```rok
let total: int = 0;               // int (auto-promote), 명시
for x in xs { total = total + x; }
// total 이 i32 범위 넘어가도 자동 promote, silent overflow 없음

let n = 0;                        // i32 (기대 타입 없음)
let perf_critical: i64 = 0;       // i64: 기대 타입을 따름
```

## 2.1.2. `int` - 무한 정밀도 정수

```rok
#[keyword("int")]
#[lang("int_type")]
struct Int { ... }  // 내부 표현: small-int 또는 BigInt

impl Int {
    fn new() -> int;              // 0 으로 초기화
    fn from_i32(n: i32) -> int;
    fn from_i64(n: i64) -> int;
    fn from_u64(n: u64) -> int;
    fn parse_radix(s: str, radix: u32) -> int | none;  // 진법 파싱 (10 진은 FromStr)
    fn to_str_radix(self, radix: u32) -> str;   // 진법 2..=36, 소문자 숫자, 음수면 `-` 가 앞; 그 밖의 진법은 panic

    fn to_i32(self) -> i32 | none;  // 범위 안이면 값, 밖이면 none
    fn to_i64(self) -> i64 | none;
    fn to_u64(self) -> u64 | none;  // 음수이거나 u64::MAX 를 넘으면 none
    fn to_f64(self) -> f64;        // 가장 가까운 f64, 동률은 짝수 쪽; f64::MAX 를 넘으면 ±inf

    fn is_zero(self) -> bool;
    fn is_negative(self) -> bool;
    fn is_positive(self) -> bool;               // 0 이면 false
    fn signum(self) -> int;                     // -1, 0 또는 1
    fn abs(self) -> int;
    fn bit_length(self) -> u64;                 // 앞쪽 0 을 뺀 |self| 의 비트 수; 0 이면 0

    fn div_rem(self, rhs: int) -> (int, int);   // (self / rhs, self % rhs), 버림; rhs == 0 이면 panic
    fn pow(self, exp: u32) -> int;              // exp: 고정 폭 정수처럼 u32
    fn isqrt(self) -> int;                      // floor(sqrt(self)); self < 0 이면 panic
    fn checked_isqrt(self) -> int | none;       // self < 0 이면 none
    fn div_euclid(self, rhs: int) -> int;       // rem_euclid >= 0 이 되는 몫; rhs == 0 이면 panic
    fn rem_euclid(self, rhs: int) -> int;       // 0..rhs.abs() 안; rhs == 0 이면 panic
    fn modpow(self, exp: int, m: int) -> int;   // self^exp %% m; m == 0 이거나 exp < 0 이면 panic
    fn gcd(self, other: int) -> int;            // 음수가 아님; gcd(0, 0) == 0
    fn lcm(self, other: int) -> int;            // 음수가 아님; 피연산자 중 하나가 0 이면 0
    ...
}
```

이것들은 고정 폭 정수의 이름과 panic 을 따릅니다. `pow` 는 `u32` 지수를 받습니다: 그보다 큰 지수는 0 과 ±1 을 뺀 모든 밑에서 적어도 2^32 bit 를 냅니다. `modpow` 는 결과가 `|m|` 보다 작게 머무르므로 `int` 지수를 받으며, 결과는 [`%%`](203-ops.md#modulo---floored-modulo-) 처럼 `m` 의 부호를 가집니다. 음수 지수는 에러입니다 (modular inverse 없음).

`Int` 는 `rok::core::num::bigint::Int` 에 정의됩니다 (`Float` 는
`rok::core::num::bigfloat::Float`). `#[keyword("int")]` 가 식별자 `int` 를 타입 키워드로
등록하며, 이 등록은 모든 패키지에 적용됩니다. 그래서 사용자 코드에서는 PascalCase
`Int` 가 아닌 lowercase `int` 로 타입을 표기합니다
([refs-lang 6.6](../../refs-lang/006-meta-system/606-keyword-magic.md) 가
권위입니다). 그리고 `#[lang("int_type")]` 로 컴파일러가 `int` 타입을 찾습니다.

`int` 의 핵심 특성은 다음과 같습니다. 메모리 표현 측면에서 작은
값은 64-bit word 안에 직접 저장되는 small-int optimization 을 거치고, 큰 값은
heap 의 BigInt 표현으로 자동 promote 되며, 이 모든 과정은 사용자에게
투명하게 처리됩니다. 표현은 항상 정규형입니다. 값이 i64 범위에 들어가면 small 형태, 벗어나면 BigInt 이며, 산술 결과가 다시 i64 범위로 돌아오면 small 형태로 돌아갑니다. 같은 값은 같은 표현을 가집니다. **lang slot** 으로는 `int_type` ([0.4 lang slot
표](../000-meta/004-lang-slot-table.md)) 이 사용됩니다. 접미사 없는 정수 리터럴은 기대 타입이 `int` 일 때만 `int` 가 됩니다.

### 산술

`int` 끼리의 모든 산술은 절대 wrap 되거나 saturate 되지 않습니다.

```rok
let big: int = 1;
big = big * 1000;
big = big * 1000;
big = big * 1000;
big = big * 1000;
big = big * 1000;
big = big * 1000;
// big = 10^18, i64 범위 거의 끝
big = big * 1000;
// big = 10^21, i64 범위 밖, BigInt promote
big = big * big;
// big = 10^42, 더 큰 BigInt, 정확
```

산술 트레잇 ([2.3 ops](203-ops.md)) 의 `Add`, `Sub`, `Mul`, `Div`, `Rem` 이
모두 구현됩니다. 여기서 `Div` 는 정수 나눗셈 (절삭) 으로 동작하므로,
`7 / 2 = 3`, `-7 / 2 = -3` 과 같이 계산됩니다. 비교 트레잇 ([2.5
cmp](205-cmp.md)) 도 `Eq`, `Ord`, `Hash` 가 모두 구현됩니다.

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

## 2.1.3. `float` - 지수 무제한 부동소수

```rok
#[keyword("float")]
#[lang("float_type")]
struct Float { ... }  // 내부: f64 또는 BigFloat (지수가 f64 밖일 때)

impl Float {
    fn from_f32(x: f32) -> float;
    fn from_f64(x: f64) -> float;
    fn from_int(n: int) -> float;                  // 가장 가까운 float, 동률은 짝수 쪽
    fn try_parse(s: str) -> float | none;          // FromStr 도 구현

    fn to_f32(self) -> f32 | none;     // 유한 값이 f32 범위 밖일 때만 none
    fn to_f64(self) -> f64;            // 가장 가까운 f64; f64::MAX 를 넘으면 ±inf, subnormal 아래면 ±0
    fn to_int(self) -> int | none;     // 0 쪽으로 버림; NaN / ±inf 면 none

    fn is_nan(self) -> bool;
    fn is_infinite(self) -> bool;
    fn is_finite(self) -> bool;
    fn is_zero(self) -> bool;
    fn is_sign_negative(self) -> bool;
    fn abs(self) -> float;
    ...
}
```

`#[keyword("float")]` 가 `float` 키워드를 등록하면서 struct `Float` 로
라우팅합니다. 따라서 사용자 코드에서는 lowercase `float` 만 보이게 됩니다.

`float` 의 값은 53-bit 유효숫자 (f64 와 같은 정밀도) 와 무제한 (i64) 지수를 가집니다.
보통은 f64 표현을 쓰고, 유한한 결과가 f64 범위를 넘거나 (overflow) 0 쪽으로
너무 작아지면 (underflow) `inf` / `0` 이 되는 대신 BigFloat 표현으로 promote 됩니다.

- `inf` / `NaN` 은 0 으로 나누기나 `inf` / `NaN` 피연산자에서만 생깁니다.
- `%` 는 정확한 fmod 입니다.
- `Display` 는 f64 의 `%g` 와 같은 모양이며, promote 된 값은 `1.5e+400` 처럼 지수로 씁니다.

### 한계 명시

`float` 의 한계도 분명히 짚어 두어야 합니다. 정밀도는 무한이 아니라 53 bit 입니다.
그래서 IEEE 754 f64 와 동일한 round-off 가 발생합니다. 예를 들어 `0.1 + 0.2` 가
`0.3` 이 되지 않는 현상은 Rok 에서도 동일하게 나타납니다. 또한 `Eq` 가
구현되어 있지 않은데, `float` 는 `Eq` 마커 트레잇을 구현하지 않고 NaN 처리를
위해 `PartialEq` 만 구현합니다. 자세한 위계는 [2.5 cmp](205-cmp.md) 를
참고합니다.

### lang slot

`float_type` ([0.4 lang slot 표](../000-meta/004-lang-slot-table.md)) 으로 컴파일러가 `float` 타입을 찾습니다. 접미사 없는 실수 리터럴은 기대 타입이 `float` 일 때만 `float` 가 됩니다.

## 2.1.4. fixed-width primitive

[refs-lang 2.1 타입 표현식](../../refs-lang/002-types/201-type-expressions.md) 이 15 primitive 의 **이름** 을
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
| `char`   | Unicode scalar value (4 바이트), 컴파일러 primitive. 메서드는 [2.15](215-string.md) |
| `c_char` | C ABI 호환 char (1-byte signed); FFI 전용                                             |

### 두 부류 - 언어 기본값 vs 표준 라이브러리 추가

이 장의 수치 / 부울 / 문자 타입은 **두 부류** 로 나뉘어 있습니다.

| 부류                       | 구성원                                                                                                | 등록 방식                              |
|----------------------------|-------------------------------------------------------------------------------------------------------|----------------------------------------|
| **16 언어 기본값**         | `i8`/`i16`/`i32`/`i64`, `u8`/`u16`/`u32`/`u64`, `isize`/`usize`, `f32`/`f64`, `bool`, `char`, `c_char`, `str` | 언어가 직접 인식, `#[keyword]` 불필요  |
| **2 표준 라이브러리 추가** | `int`, `float`                                                                                | `#[keyword(...)]` 등록                 |

언어 기본값은 컴파일러 (lexer/parser) 가 직접 인식하는 토큰이라서, 표준 라이브러리가
별도로 키워드 등록을 하지 않습니다. 그 결과 사용자가 이 이름들을 재정의할 수
없다는 사실이 자동으로 보장됩니다.

이에 비해 표준 라이브러리 추가 (`int` / `float`) 는 `#[keyword(...)]` 로
등록됩니다. 이 모듈의 `Int` / `Float` 정의에서 `#[keyword]` attribute
가 부착됩니다 ([refs-lang
6.6](../../refs-lang/006-meta-system/606-keyword-magic.md) 가 권위입니다).

### 언어 기본값 primitive 의 표준 라이브러리 형태

```rok
impl i32 { ... }              // 고유 상수와 메서드 (아래)
impl Add for i32 { ... }      // 트레잇 impl: Add, PartialOrd, Display, Clone, ...
impl PartialEq for c_char { ... }
```

표준 라이브러리는 언어 기본 primitive 에 대해 struct 를 선언하지 않습니다. 고유 메서드와
트레잇 (`Add`, `Display`, `Clone`, ...) 을 위한 `impl` 블록만 부착합니다. 타입 인식 자체는
컴파일러가 책임집니다.

### 산술 / 비교 트레잇 impl

각 primitive 가 [2.3 ops](203-ops.md), [2.4 ops-assign](204-ops-assign.md),
[2.5 cmp](205-cmp.md) 의 트레잇을 구현합니다. 모든 연산자 메서드와 `PartialEq` / `PartialOrd` 메서드에는
`#[intrinsic("rok.OP.TYPE")]` 가 부착되어 컴파일러 백엔드 PrimOp 와 직결
됩니다.

### 메서드 (각 primitive 별)

각 정수 / 부동소수 타입은 다음 메서드를 노출합니다 (예시는 `i32` 입니다).

```rok
impl i32 {
    const MIN: i32 = -2147483648;
    const MAX: i32 = 2147483647;
    const BITS: u32 = 32;

    fn pow(self, exp: u32) -> i32;                // overflow 는 산술을 따름 (debug panic, release wrap)
    fn abs(self) -> i32;
    fn unsigned_abs(self) -> u32;                 // overflow 없는 |self| (i32::MIN 이면 2^31)
    fn abs_diff(self, rhs: i32) -> u32;
    fn signum(self) -> i32;                       // -1, 0 또는 1
    fn is_positive(self) -> bool;
    fn is_negative(self) -> bool;
    fn div_euclid(self, rhs: i32) -> i32;         // rem_euclid >= 0 이 되는 몫
    fn rem_euclid(self, rhs: i32) -> i32;         // 0..rhs.abs() 안
    fn isqrt(self) -> i32;                        // floor(sqrt(self)); self < 0 이면 panic
    fn checked_isqrt(self) -> i32 | none;         // self < 0 이면 none
    fn ilog2(self) -> u32;                        // floor(log2(self)); self > 0 이 아니면 panic
    fn ilog10(self) -> u32;
    fn ilog(self, base: i32) -> u32;              // self > 0 이고 base >= 2 가 아니면 panic
    fn checked_ilog2(self) -> u32 | none;
    fn checked_ilog10(self) -> u32 | none;
    fn checked_ilog(self, base: i32) -> u32 | none;

    // 비트
    fn count_ones(self) -> u32;
    fn count_zeros(self) -> u32;
    fn leading_zeros(self) -> u32;
    fn trailing_zeros(self) -> u32;
    fn leading_ones(self) -> u32;
    fn trailing_ones(self) -> u32;
    fn rotate_left(self, n: u32) -> i32;
    fn rotate_right(self, n: u32) -> i32;

    // 바이트 순서
    fn swap_bytes(self) -> i32;
    fn to_be(self) -> i32;                        // 네이티브 순서에서 big-endian 으로
    fn to_le(self) -> i32;
    fn from_be(x: i32) -> i32;
    fn from_le(x: i32) -> i32;
    fn to_le_bytes(self) -> [u8; 4];
    fn to_be_bytes(self) -> [u8; 4];
    fn to_ne_bytes(self) -> [u8; 4];              // 네이티브 순서
    fn from_le_bytes(bytes: [u8; 4]) -> i32;
    fn from_be_bytes(bytes: [u8; 4]) -> i32;
    fn from_ne_bytes(bytes: [u8; 4]) -> i32;

    // overflow 명시 변형
    fn checked_add(self, rhs: i32) -> i32 | none;
    fn checked_sub(self, rhs: i32) -> i32 | none;
    fn checked_mul(self, rhs: i32) -> i32 | none;
    fn checked_div(self, rhs: i32) -> i32 | none; // rhs == 0 과 MIN / -1 이면 none
    fn checked_rem(self, rhs: i32) -> i32 | none;
    fn checked_neg(self) -> i32 | none;
    fn checked_shl(self, n: u32) -> i32 | none;   // n >= BITS 이면 none
    fn checked_shr(self, n: u32) -> i32 | none;
    fn checked_pow(self, exp: u32) -> i32 | none;
    fn checked_abs(self) -> i32 | none;

    fn saturating_add(self, rhs: i32) -> i32;     // i32::MAX 또는 MIN 으로 clamp
    fn saturating_sub(self, rhs: i32) -> i32;
    fn saturating_mul(self, rhs: i32) -> i32;
    fn saturating_neg(self) -> i32;
    fn saturating_pow(self, exp: u32) -> i32;
    fn saturating_abs(self) -> i32;

    fn wrapping_add(self, rhs: i32) -> i32;       // 모듈러 산술
    fn wrapping_sub(self, rhs: i32) -> i32;
    fn wrapping_mul(self, rhs: i32) -> i32;
    fn wrapping_div(self, rhs: i32) -> i32;       // MIN / -1 은 MIN; rhs == 0 은 여전히 panic
    fn wrapping_rem(self, rhs: i32) -> i32;       // MIN % -1 은 0
    fn wrapping_neg(self) -> i32;
    fn wrapping_shl(self, n: u32) -> i32;         // n % BITS 만큼 shift
    fn wrapping_shr(self, n: u32) -> i32;
    fn wrapping_pow(self, exp: u32) -> i32;
    fn wrapping_abs(self) -> i32;

    // wrap 된 결과와 overflow 여부
    fn overflowing_add(self, rhs: i32) -> (i32, bool);
    fn overflowing_sub(self, rhs: i32) -> (i32, bool);
    fn overflowing_mul(self, rhs: i32) -> (i32, bool);
    fn overflowing_div(self, rhs: i32) -> (i32, bool);
    fn overflowing_rem(self, rhs: i32) -> (i32, bool);
    fn overflowing_neg(self) -> (i32, bool);
    fn overflowing_shl(self, n: u32) -> (i32, bool);  // n >= BITS 이면 true
    fn overflowing_shr(self, n: u32) -> (i32, bool);
    fn overflowing_pow(self, exp: u32) -> (i32, bool);
}
```

`i8`, `i16`, `i64`, `isize` 는 각자의 폭으로 같은 집합을 가집니다. unsigned 타입
(`u8`..`u64`, `usize`) 은 부호 관련 메서드 (`abs`, `unsigned_abs`, `signum`, `is_positive`,
`is_negative`, `checked_isqrt`, `checked_abs`, `saturating_neg`, `saturating_abs`,
`wrapping_abs`) 를 뺀 같은 집합을 가지며, `abs_diff` 는 같은 타입을 반환하고 `checked_neg` 는
0 이 아닌 모든 값에서 `none` 입니다. 대신 2 의 거듭제곱 메서드가 추가됩니다.

```rok
impl u32 {
    fn is_power_of_two(self) -> bool;
    fn next_power_of_two(self) -> u32;                 // self 이상인 가장 작은 2 의 거듭제곱; overflow 면 panic
    fn checked_next_power_of_two(self) -> u32 | none;
}
```

정수의 최솟값 / 최댓값은 함수가 아니라 연관 상수 `i32::MIN` / `i32::MAX` 입니다. 인자 없는 고유 메서드 `min()` / `max()` 를 두면 같은 이름의 `Ord::min` / `Ord::max` (`a.min(b)`) 를 가리기 때문입니다.

부동소수 (`f32`, `f64`) 는 다음 상수와 메서드를 제공합니다 (`f32` 는 `f64` 자리에 `f32`,
비트에 `u32` 를 쓴 같은 집합입니다).

```rok
impl f64 {
    const EPSILON: f64;          // 2^-52, 1.0 과 그다음 f64 사이의 간격
    const MAX: f64;              // 가장 큰 유한 값
    const MIN: f64;              // -MAX
    const MIN_POSITIVE: f64;     // 가장 작은 양의 normal 값
    const NAN: f64;
    const INFINITY: f64;
    const NEG_INFINITY: f64;
    const PI: f64;
    const TAU: f64;              // 2π
    const E: f64;
    const FRAC_PI_2: f64;        // π/2
    const FRAC_PI_4: f64;        // π/4
    const FRAC_1_PI: f64;        // 1/π
    const SQRT_2: f64;
    const FRAC_1_SQRT_2: f64;
    const LN_2: f64;
    const LN_10: f64;
    const LOG2_E: f64;
    const LOG10_E: f64;

    fn nan() -> f64;             // f64::NAN
    fn infinity() -> f64;        // f64::INFINITY
    fn neg_infinity() -> f64;
    fn epsilon() -> f64;         // f64::EPSILON

    fn is_nan(self) -> bool;
    fn is_infinite(self) -> bool;
    fn is_finite(self) -> bool;
    fn is_normal(self) -> bool;  // 0, subnormal, 무한대, NaN 이 모두 아님
    fn is_sign_positive(self) -> bool;
    fn is_sign_negative(self) -> bool;

    fn floor(self) -> f64;
    fn ceil(self) -> f64;
    fn round(self) -> f64;       // 절반은 0 에서 먼 쪽으로
    fn trunc(self) -> f64;
    fn fract(self) -> f64;       // self - self.trunc()

    fn abs(self) -> f64;
    fn signum(self) -> f64;      // 양수면 1.0, 음수면 -1.0, ±0.0 이면 그 0, NaN 이면 NaN
    fn copysign(self, sign: f64) -> f64;
    fn recip(self) -> f64;       // 1.0 / self
    fn min(self, other: f64) -> f64;     // NaN 피연산자는 숫자에 짐
    fn max(self, other: f64) -> f64;
    fn clamp(self, lo: f64, hi: f64) -> f64;   // NaN 은 NaN 그대로
    fn div_euclid(self, rhs: f64) -> f64;
    fn rem_euclid(self, rhs: f64) -> f64;
    fn mul_add(self, a: f64, b: f64) -> f64;   // 반올림 한 번으로 self * a + b

    fn sqrt(self) -> f64;        // Sqrt 트레잇 (2.3)
    fn cbrt(self) -> f64;
    fn hypot(self, other: f64) -> f64;   // overflow 없는 sqrt(self² + other²)
    fn pow(self, exp: f64) -> f64;
    fn powf(self, exp: f64) -> f64;      // = pow
    fn powi(self, n: i32) -> f64;
    fn exp(self) -> f64;
    fn exp2(self) -> f64;
    fn exp_m1(self) -> f64;      // e^self - 1, 0 근처에서 정확
    fn ln(self) -> f64;
    fn ln_1p(self) -> f64;       // ln(1 + self), 0 근처에서 정확
    fn log(self, base: f64) -> f64;
    fn log2(self) -> f64;
    fn log10(self) -> f64;
    fn sin(self) -> f64;
    fn cos(self) -> f64;
    fn tan(self) -> f64;
    fn asin(self) -> f64;
    fn acos(self) -> f64;
    fn atan(self) -> f64;
    fn atan2(self, other: f64) -> f64;   // 점 (other, self) 의 각
    fn sinh(self) -> f64;
    fn cosh(self) -> f64;
    fn tanh(self) -> f64;
    fn asinh(self) -> f64;
    fn acosh(self) -> f64;
    fn atanh(self) -> f64;
    fn to_degrees(self) -> f64;
    fn to_radians(self) -> f64;

    fn to_bits(self) -> u64;     // 비트 패턴
    fn from_bits(bits: u64) -> f64;
}
```

## 2.1.5. literal → 어느 타입?

[refs-lang 1.6 숫자 리터럴](../../refs-lang/001-lexical/106-numeric-literals.md) 의 규칙은 다음과 같습니다.

| 형태      | 결정 타입                                         |
|-----------|---------------------------------------------------|
| `42`      | 기대 타입 (정수 / 실수 타입), 없으면 `i32`         |
| `3.14`    | 기대 타입 (실수 타입), 없으면 `f64`                |
| `42i32`   | `i32`                                             |
| `42u64`   | `u64`                                             |
| `3.14f32` | `f32`                                             |
| `3.14f64` | `f64`                                             |
| `'A'`     | `char`                                            |
| `c'A'`    | `c_char` (primitive)                              |
| `0xFF`    | 10 진과 같음: 기대 타입, 없으면 `i32`              |
| `0xFFi32` | `i32`                                             |

### 기대 타입과 범위 검사

범위는 컴파일 타임에 검사합니다.

```rok
let x: i32 = 42;            // i32
let y: i32 = 9999999999;    // 컴파일 에러 (i32 범위 밖)
let z: f32 = 3.14;          // f32
let f: f64 = 1;             // f64: 정수 리터럴은 실수 슬롯에 들어감
let w: u8 = 256;            // 컴파일 에러 (256 ∉ u8 range 0..=255)
let i: i32 = 1.5;           // 컴파일 에러: 실수 리터럴은 정수 슬롯 불가
let big: int = 9999999999;  // int
```

자세한 변환 매트릭스는 [2.11 변환](211-conversion.md) 에 정리됩니다.

### `Zero` / `One`

```rok
trait Zero { fn zero() -> Self; }       // 덧셈 항등원
trait One  { fn one() -> Self; }        // 곱셈 항등원
```

모든 숫자 primitive 와 `int` / `float` 가 두 트레잇을 구현합니다. `Iterator::sum` / `product` 가 이를 씁니다 ([2.7.1](207-iter.md#271-iterator---단계-진행)).

## 2.1.6. 산술 결과 타입

### `int + int = int`

auto-promote 가 적용되므로 결과는 항상 `int` 자체가 됩니다. fixed-width 추론은
이루어지지 않습니다.

```rok
let a: int = 1000;
let b: int = 2000;
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

### `int + i32` (혼합 - 거부)

```rok
let a: int = 5;       // int
let b: i32 = 3;       // i32
let c = a + b;        // 컴파일 에러: type mismatch
```

이 경우 명시 변환이 강제됩니다.

```rok
let c1 = a + int::from_i32(b);    // int (a + int(3))
let c2 = (a.to_i32()? + b);       // i32 (a 가 i32 범위 안이면 OK)
```

이러한 정책 덕분에 silent type mixing 이 차단됩니다. Rok 은 Julia 처럼 자동
promote 를 하지 않으며, 어느 세계를 쓸지는 사용자가 명시하도록 되어 있습니다.

## 2.1.7. fixed primitive ↔ int 변환

fixed primitive 와 `int` 사이의 변환은 명시적으로만 가능합니다.

| 변환                     | 메서드                                | 실패 시                      |
|--------------------------|---------------------------------------|------------------------------|
| `i32` → `int`            | `.into::<int>()` 또는 `int::from_i32` | 항상 성공 (lossless)         |
| `int` → `i32`            | `.try_into::<i32>()`                  | `OverflowError` 를 throw     |
| `int` → `i32` (saturate) | `.saturating_into::<i32>()`           | i32::MIN 또는 MAX            |
| `int` → `i32` (wrap)     | `.wrapping_into::<i32>()`             | 모듈러 산술                  |
| `int` → `f64`            | `.into::<f64>()` 또는 `int::to_f64`   | 가장 가까운 값으로 반올림 (`From` 의 문서화된 예외, [2.11.8](211-conversion.md)). `float` → `f64` 도 같은 예외 |

## 2.1.8. niche 최적화 (간단)

[refs-lang 2.7
niche](../../refs-lang/002-types/207-niche-optimization.md) 가 메커니즘을
정의합니다. 자동 niche 가 적용되는 primitive 는 다음과 같습니다.

`bool | none` 은 1 byte (niche 값 `2`), `Ptr<T> | none` / `ConstPtr<T> | none`
은 pointer 크기 (null 이 none) 입니다 ([refs-lang 2.7.2](../../refs-lang/002-types/207-niche-optimization.md#272-자동-niche)). 그러나 `i8`..`i64`,
`u8`..`u64`, `f32`, `f64` 자체에는 자동 niche 가 적용되지 않습니다. 모든 bit
pattern 이 유효한 값이기 때문입니다.

opt-in niche 가 필요한 경우에는 [2.2 non-zero](202-non-zero.md) 의
`NonZero<T>` / `NonNan<F>` / `Finite<F>` / `Positive<T>` wrapper 를 사용합니다.

## 2.1.9. lang slot 매핑

언어 기본값 16 종 (`i8`..`i64`, `u8`..`u64`, `isize`, `usize`, `f32`, `f64`, `bool`,
`c_char`, `char`, `str`) 은 컴파일러가 이름으로 인식합니다. 이들에 lang slot 을 묶는 표준
라이브러리 항목은 없고 struct 도 선언되지 않습니다. 라이브러리는 `impl` 블록만 추가하므로
(2.1.4) `#[keyword]` 나 `#[lang]` 이 필요하지 않습니다.

표준 라이브러리가 추가하는 2 종은 `#[keyword(...)]` 로 등록되고 `#[lang(...)]` 으로
lang slot 에 묶이는 PascalCase struct 입니다.

| primitive | lang slot    | struct 이름 | 등록                  |
|-----------|--------------|-------------|-----------------------|
| `int`     | `int_type`   | `Int`       | `#[keyword("int")]`   |
| `float`   | `float_type` | `Float`     | `#[keyword("float")]` |

참고로 `Range<T>` (`a..b`) 는 [2.7 iter](207-iter.md) 가 정의합니다.

## 2.1.10. 다른 언어와의 비교 - 디자인 결정 근거

Rok 의 두-세계 모델 (`int` auto-promote 와 fixed primitive) 은 여러 언어에서
영향을 받았습니다. Smalltalk 의 SmallInteger ↔ LargeInteger 는 auto-promote
의 원형이라고 할 수 있고, Lisp 와 Scheme 의 numeric tower (fixnum / bignum /
rational / real) 도 중요한 참고가 됩니다. 또한 **Python int** 는 무한 정밀도를
default 로 두지만 fixed primitive 가 없다는 차이가 있고, **Julia** 는 fixed
primitive 를 default 로 둡니다. Rok 도 리터럴의 기본은 fixed (`i32` / `f64`) 이고, `int` 는 opt-in 입니다.

각 언어와의 차이는 다음과 같습니다. **Python** 은 int 만 제공하며,
fixed-width 가 없어서 FFI 나 성능 critical 한 상황에서는 NumPy 에 의존하게
됩니다. **Julia** 는 Rok 과 같이 fixed 가 default 이며, 무한 정밀도가
필요하면 `BigInt` 를 명시합니다. **Rust** 는 fixed only 라서 매번
사용자가 type 을 적어야만 합니다. **Rok** 는 두 세계가 공존하도록
하여, 리터럴은 기대 타입을 따르고 (없으면 `i32` / `f64`), 무한 정밀도가 필요한 자리에서 `int` / `float` 를 적도록 설계됩니다.
