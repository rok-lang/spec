# 2.1. 숫자 타입

`rok::core::num` 모듈은 Rok의 모든 수치 타입을 정의합니다. Rok에는 두 세계의 숫자가 함께 있고, 어느 쪽을 쓸지는 사용자가 고릅니다.

```
int / float    (auto-promote, 무한 정밀도, opt-in)
fixed primitive (i8..i64, u8..u64, isize, usize, f32, f64; 하드웨어 친화적, 기본)
```

두 세계는 **공존**합니다. 접미사 없는 리터럴은 기대 타입을 따르고, 기대 타입이 없으면 `i32` / `f64`가 됩니다([refs-lang 1.6.3](../../refs-lang/001-lexical/106-numeric-literals.md#163-접미사-없는-리터럴)). `int` / `float`는 타입을 적은 자리에서만 쓰입니다.

## 2.1.1. 두 세계의 숫자: 분리하는 이유

### `int` / `float`: auto-promote 세계

이 세계는 무한 정밀도를 지향합니다. 오버플로는 일어나지 않고, 메모리 한계에 닿을 때만 OOM으로 끝납니다. 작은 값은 small-int 최적화로 하드웨어 고정 폭 형태로 저장합니다. 값이 커지면 힙의 BigInt로 자동 승격(promote)합니다. `int` / `float` 값은 불변입니다. 힙의 BigInt도 제자리에서 바뀌지 않고 공유만 되므로 `Copy`입니다. 덕분에 산술 결과는 항상 정확합니다. `I32_MAX + 1`이 조용히 한 바퀴 돌아 엉뚱한 값이 되는 일은 없습니다.

### 고정 폭 원시 타입: 하드웨어 친화 세계

반면 fixed primitive는 하드웨어에 가까운 세계입니다. `i32` / `f64` 등 15종이고, 문법은 [refs-lang 2.1 타입 표현식](../../refs-lang/002-types/201-type-expressions.md)이 정의합니다. FFI 인터페이스, SIMD, 큰 배열, 핫 루프처럼 메모리와 성능을 정밀하게 제어해야 하는 상황에 알맞습니다. 오버플로 정책도 명시되어 있습니다([2.3 ops
산술](203-ops.md)의 fixed-width 산술 절 참고).

### 기본으로 쓰는 쪽

접미사 없는 리터럴(`42`, `3.14`)은 기대 타입이 없으면 **`i32` / `f64`**가 됩니다. auto-promote 세계는 `int` / `float`를 적은 자리에서만 쓰입니다.

```rok
let total: int = 0;               // int (auto-promote), 명시
for x in xs { total = total + x; }
// total 이 i32 범위 넘어가도 자동 promote, silent overflow 없음

let n = 0;                        // i32 (기대 타입 없음)
let perf_critical: i64 = 0;       // i64: 기대 타입을 따름
```

## 2.1.2. `int`: 무한 정밀도 정수

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

    // 비트: 부호 비트가 끝없이 반복되는 2 의 보수
    fn count_ones(self) -> u64;                 // |self| 의 1 비트 수
    fn trailing_zeros(self) -> u64 | none;      // 0 이면 none
    fn bit(self, n: u64) -> bool;               // 음수는 위쪽 비트가 모두 1
    fn set_bit(ref self, n: u64, value: bool);
    fn is_power_of_two(self) -> bool;           // 0 과 음수는 false
    fn next_power_of_two(self) -> int;          // self 이상인 가장 작은 2 의 거듭제곱; self <= 1 이면 1
    fn ilog2(self) -> u64;                      // floor(log2(self)); self > 0 이 아니면 panic
    fn checked_ilog2(self) -> u64 | none;

    // 바이트: signed 면 가장 짧은 2 의 보수, 아니면 절댓값
    fn to_le_bytes(self, signed: bool = true) -> List<u8>;   // !signed 이고 self < 0 이면 panic
    fn to_be_bytes(self, signed: bool = true) -> List<u8>;
    fn from_le_bytes(bytes: List<u8> | Slice<u8>, signed: bool = true) -> int;   // 바이트가 없으면 0
    fn from_be_bytes(bytes: List<u8> | Slice<u8>, signed: bool = true) -> int;

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

이 메서드는 고정 폭 정수의 이름과 panic 규칙을 따릅니다. 비트 수와 비트 위치는 `bit_length`처럼 `u64`입니다. 폭에 묶인 비트 메서드(`leading_zeros`, `rotate_left`, `swap_bytes` 등)는 폭이 없으면 뜻이 없으므로 뺍니다. 음수에는 1인 비트가 끝없이 많으므로 `count_ones`는 `|self|`를 셉니다(Python의 `bit_count`). 바이트는 값이 허락하는 만큼 짧습니다. `128`은 부호 있는 형태로 `[128, 0]`, 부호 없는 형태로 `[128]`이고, `-1`은 `[255]`, 0은 `[0]`입니다. `pow`는 `u32` 지수를 받습니다. 그보다 큰 지수는 0과 ±1을 뺀 모든 밑에서 결과가 적어도 2^32비트가 됩니다. `modpow`는 결과가 `|m|`보다 작게 머무르므로 `int` 지수를 받습니다. 결과의 부호는 [`%%`](203-ops.md#modulo-내림-modulo)처럼 `m`의 부호를 따릅니다. 음수 지수는 에러입니다(모듈러 역원 없음).

`Int`는 `rok::core::num::bigint::Int`에, `Float`는 `rok::core::num::bigfloat::Float`에 정의합니다. `#[keyword("int")]`가 식별자 `int`를 타입 키워드로 등록하고, 이 등록은 모든 패키지에 적용됩니다. 그래서 사용자 코드에서는 PascalCase `Int`가 아니라 소문자 `int`로 타입을 적습니다. 권위 있는 정의는 [refs-lang 6.6](../../refs-lang/006-meta-system/606-keyword-magic.md)에 있습니다. 컴파일러는 `#[lang("int_type")]`로 `int` 타입을 찾습니다.

`int`의 주요 성질은 다음과 같습니다. 메모리에서 작은 값은 small-int 최적화로 64비트 워드 안에 직접 저장하고, 큰 값은 힙의 BigInt 표현으로 자동 승격합니다. 이 과정은 사용자에게 드러나지 않습니다. 표현은 항상 정규형입니다. 값이 i64 범위에 들어가면 small 형태이고 벗어나면 BigInt입니다. 산술 결과가 다시 i64 범위로 돌아오면 small 형태로 돌아갑니다. 같은 값의 표현은 언제나 같습니다. **lang slot**은 `int_type`입니다([0.4 lang slot
표](../000-meta/004-lang-slot-table.md)). 접미사 없는 정수 리터럴은 기대 타입이 `int`일 때만 `int`가 됩니다.

### 산술

`int`끼리 계산하면 결코 한 바퀴 돌거나(wrap) 포화(saturate)하지 않습니다.

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

산술 트레잇([2.3 ops](203-ops.md)) `Add`, `Sub`, `Mul`, `Div`, `Rem`을 모두 구현합니다. `Div`는 소수점 아래를 버리는 정수 나눗셈이므로 `7 / 2 = 3`, `-7 / 2 = -3`입니다. 비교 트레잇([2.5
cmp](205-cmp.md)) `Eq`, `Ord`, `Hash`도 모두 구현합니다.

### 주요 사용처

`int`는 여러 곳에 쓰입니다. REPL이나 Jupyter에서 산술 기본 타입으로 쓰면 매번 타입을 적지 않아도 결과가 정확합니다. 수치 시뮬레이션에서 합이나 곱을 누적할 때도 오버플로를 걱정하지 않아도 됩니다. 암호 연산이나 큰 정수 처리도 BigInt 라이브러리를 따로 쓰지 않고 표준 라이브러리만으로 할 수 있습니다.

### 한계

`int`에도 한계가 있습니다. 메모리 면에서는 BigInt로 승격할 때 힙 할당이 필요합니다. 그래서 핫 루프나 큰 배열에는 fixed primitive를 권장합니다. 성능은 small-int 범위 안에서는 fixed primitive와 비슷하지만, BigInt로 승격한 뒤에는 느릴 수밖에 없습니다. FFI에서는 `extern "C"` 함수가 fixed primitive만 받습니다. 그래서 `int`는 `i64` 등으로 명시적으로 변환한 뒤 넘겨야 합니다.

## 2.1.3. `float`: 지수 무제한 부동소수

```rok
#[keyword("float")]
#[lang("float_type")]
struct Float { ... }  // 내부: f64 또는 BigFloat (지수가 f64 밖일 때)

impl Float {
    fn from_f32(x: f32) -> float;
    fn from_f64(x: f64) -> float;
    fn from_int(n: int) -> float;                  // 가장 가까운 float, 동률은 짝수 쪽
    fn parse_or_none(s: str) -> float | none;      // FromStr 은 대신 던짐
    fn try_parse(s: str) -> float | none;          // 0.1.6부터 deprecated: parse_or_none

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

`#[keyword("float")]`는 `float` 키워드를 등록하고 struct `Float`로 연결합니다. 그래서 사용자 코드에는 소문자 `float`만 보입니다.

`float` 값은 53비트 유효숫자(f64와 같은 정밀도)와 무제한(i64) 지수로 이루어집니다. 보통은 f64 표현을 씁니다. 유한한 결과가 f64 범위를 넘거나(오버플로) 0 쪽으로 너무 작아지면(언더플로) `inf` / `0`이 되는 대신 BigFloat 표현으로 승격합니다.

- `inf` / `NaN`은 0으로 나누기나 `inf` / `NaN` 피연산자에서만 생깁니다.
- `%`는 정확한 fmod입니다.
- `Display`는 f64의 `%g`와 같은 모양입니다. 승격한 값은 `1.5e+400`처럼 지수로 씁니다.

### 한계 명시

`float`의 한계도 분명히 밝혀 둡니다. 정밀도는 무한이 아니라 53비트입니다. 그래서 IEEE 754 f64와 같은 반올림 오차가 생깁니다. 예를 들어 `0.1 + 0.2`가 `0.3`이 되지 않는 현상은 Rok에서도 똑같이 나타납니다. `Eq`도 구현하지 않습니다. `float`는 NaN을 다루려고 `Eq` 마커 트레잇 없이 `PartialEq`만 구현합니다. 전체 위계는 [2.5 cmp](205-cmp.md)에 있습니다.

### lang slot

컴파일러는 `float_type`([0.4 lang slot 표](../000-meta/004-lang-slot-table.md))으로 `float` 타입을 찾습니다. 접미사 없는 실수 리터럴은 기대 타입이 `float`일 때만 `float`가 됩니다.

## 2.1.4. 고정 폭 원시 타입

[refs-lang 2.1 타입 표현식](../../refs-lang/002-types/201-type-expressions.md)이 원시 타입 15개의 **이름**을 정의합니다. 이 절은 그 위에 표준 라이브러리의 메서드와 동작을 정의합니다.

### 정수(10종)

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

### 부동소수(2종)

| 타입  | 비트 폭 | 표준                                 |
|-------|---------|--------------------------------------|
| `f32` | 32      | IEEE 754 binary32 (single precision) |
| `f64` | 64      | IEEE 754 binary64 (double precision) |

### 부울 / 문자(3종)

| 타입     | 의미                                                                                  |
|----------|---------------------------------------------------------------------------------------|
| `bool`   | `true` / `false`                                                                      |
| `char`   | 유니코드 스칼라 값(4바이트), 컴파일러 원시 타입. 메서드는 [2.15](215-string.md) |
| `c_char` | C ABI 호환 char(1바이트, 부호 있음), FFI 전용                                         |

### 두 부류: 언어 내장과 표준 라이브러리 추가

이 장의 수치 / 부울 / 문자 타입은 **두 부류**로 나뉩니다.

| 부류                       | 구성원                                                                                                | 등록 방식                              |
|----------------------------|-------------------------------------------------------------------------------------------------------|----------------------------------------|
| **16 언어 기본값**         | `i8`/`i16`/`i32`/`i64`, `u8`/`u16`/`u32`/`u64`, `isize`/`usize`, `f32`/`f64`, `bool`, `char`, `c_char`, `str` | 언어가 직접 인식, `#[keyword]` 불필요  |
| **2 표준 라이브러리 추가** | `int`, `float`                                                                                | `#[keyword(...)]` 등록                 |

언어 기본값은 컴파일러(렉서/파서)가 직접 인식하는 토큰입니다. 그래서 표준 라이브러리가 키워드로 따로 등록하지 않습니다. 덕분에 사용자는 이 이름을 다시 정의할 수 없습니다.

이에 비해 표준 라이브러리가 추가하는 `int` / `float`는 `#[keyword(...)]`로 등록합니다. 이 모듈의 `Int` / `Float` 정의에 `#[keyword]` 속성이 붙습니다(권위 있는 정의는 [refs-lang
6.6](../../refs-lang/006-meta-system/606-keyword-magic.md)).

### 내장 원시 타입의 표준 라이브러리 형태

```rok
impl i32 { ... }              // 고유 상수와 메서드 (아래)
impl Add for i32 { ... }      // 트레잇 impl: Add, PartialOrd, Display, Clone, ...
impl PartialEq for c_char { ... }
```

표준 라이브러리는 언어 기본 원시 타입의 struct를 선언하지 않습니다. 고유 메서드와 트레잇(`Add`, `Display`, `Clone` 등)의 `impl` 블록만 붙입니다. 타입을 알아보는 일은 컴파일러가 맡습니다.

### 산술 / 비교 트레잇 impl

원시 타입은 각각 [2.3 ops](203-ops.md), [2.4 ops-assign](204-ops-assign.md), [2.5 cmp](205-cmp.md)의 트레잇을 구현합니다. 모든 연산자 메서드와 `PartialEq` / `PartialOrd` 메서드에는 `#[intrinsic("rok.OP.TYPE")]`가 붙습니다. 그래서 이 메서드는 컴파일러 백엔드 PrimOp로 바로 이어집니다.

### 메서드(원시 타입별)

정수 / 부동소수 타입에는 각각 다음 메서드가 있습니다. 예시는 `i32`입니다.

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

`i8`, `i16`, `i64`, `isize`에도 각자의 폭으로 같은 메서드가 있습니다. 부호 없는 타입(`u8`..`u64`, `usize`)에는 부호 관련 메서드(`abs`, `unsigned_abs`, `signum`, `is_positive`,
`is_negative`, `checked_isqrt`, `checked_abs`, `saturating_neg`, `saturating_abs`,
`wrapping_abs`)를 뺀 나머지가 있습니다. 이때 `abs_diff`는 같은 타입을 반환하고, `checked_neg`는 0이 아닌 모든 값에서 `none`입니다. 대신 2의 거듭제곱 메서드가 더 있습니다.

```rok
impl u32 {
    fn is_power_of_two(self) -> bool;
    fn next_power_of_two(self) -> u32;                 // self 이상인 가장 작은 2 의 거듭제곱; overflow 면 panic
    fn checked_next_power_of_two(self) -> u32 | none;
}
```

정수의 최솟값 / 최댓값은 함수가 아니라 연관 상수 `i32::MIN` / `i32::MAX`입니다. 인자 없는 고유 메서드 `min()` / `max()`를 두면 같은 이름의 `Ord::min` / `Ord::max`(`a.min(b)`)를 가리기 때문입니다.

부동소수 타입(`f32`, `f64`)에는 다음 상수와 메서드가 있습니다. `f32`의 집합은 `f64` 자리에 `f32`를, 비트에 `u32`를 쓴 것과 같습니다.

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

    fn nan() -> f64;             // 0.1.6부터 deprecated: f64::NAN
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
    fn lerp(self, other: f64, t: f64) -> f64;  // t = 0 이면 self, t = 1 이면 other; 양 끝에서 정확, t 에 단조
    fn next_after(self, toward: f64) -> f64;   // `toward` 쪽으로 한 칸 (C nextafter); 같으면 toward
    fn next_up(self) -> f64;                   // self 보다 큰 가장 작은 값; inf 와 NaN 은 그대로
    fn next_down(self) -> f64;                 // self 보다 작은 가장 큰 값

    fn sqrt(self) -> f64;        // Sqrt 트레잇 (2.3)
    fn cbrt(self) -> f64;
    fn hypot(self, other: f64) -> f64;   // overflow 없는 sqrt(self² + other²)
    fn pow(self, exp: f64) -> f64;
    fn powf(self, exp: f64) -> f64;      // 0.1.6부터 deprecated: = pow
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

## 2.1.5. 리터럴의 타입 결정

[refs-lang 1.6 숫자 리터럴](../../refs-lang/001-lexical/106-numeric-literals.md)의 규칙은 다음과 같습니다.

| 형태      | 결정 타입                                         |
|-----------|---------------------------------------------------|
| `42`      | 기대 타입(정수 / 실수 타입), 없으면 `i32`          |
| `3.14`    | 기대 타입(실수 타입), 없으면 `f64`                 |
| `42i32`   | `i32`                                             |
| `42u64`   | `u64`                                             |
| `3.14f32` | `f32`                                             |
| `3.14f64` | `f64`                                             |
| `'A'`     | `char`                                            |
| `c'A'`    | `c_char` (원시 타입)                              |
| `0xFF`    | 10진과 같음. 기대 타입, 없으면 `i32`               |
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

전체 변환 매트릭스는 [2.11 변환](211-conversion.md)에 있습니다.

### `Zero` / `One`

```rok
trait Zero { fn zero() -> Self; }       // 덧셈 항등원
trait One  { fn one() -> Self; }        // 곱셈 항등원
```

모든 숫자 원시 타입과 `int` / `float`가 두 트레잇을 구현합니다. `Iterator::sum` / `product`가 이 트레잇을 씁니다([2.7.1](207-iter.md#271-iterator-단계-진행)).

## 2.1.6. 산술 결과 타입

### `int + int = int`

auto-promote가 적용되므로 결과는 항상 `int`입니다. 고정 폭 타입으로 추론하지 않습니다.

```rok
let a: int = 1000;
let b: int = 2000;
let c = a + b;       // int (3000)
```

### `i32 + i32 = i32`

고정 폭은 그대로 유지됩니다. 더 넓은 타입이 필요하면 명시적으로 변환해야 합니다.

```rok
let a: i32 = 5;
let b: i32 = 3;
let c: i32 = a + b;   // i32

let wider: i64 = (a as i64) + (b as i64);   // i64
```

### `int + i32`(혼합: 거부)

```rok
let a: int = 5;       // int
let b: i32 = 3;       // i32
let c = a + b;        // 컴파일 에러: type mismatch
```

이때는 명시적으로 변환해야 합니다.

```rok
let c1 = a + int::from_i32(b);    // int (a + int(3))
let c2 = (a.to_i32()? + b);       // i32 (a 가 i32 범위 안이면 OK)
```

이 정책으로 타입이 조용히 섞이는 일을 막습니다. Rok은 Julia와 달리 자동으로 승격하지 않습니다. 어느 세계를 쓸지는 사용자가 명시합니다.

섞인 숫자에 컴파일러가 붙이는 help는 좁은 쪽을 넓은 쪽 타입으로 바꾸므로, 제안하는 변환에서 값이 사라지지 않습니다. 정수는 부동소수점으로, 원시 타입은 `int`나 `float`로 갑니다. 둘 다 부호 있는 정수이거나 둘 다 부호 없는 정수이면 좁은 쪽이 넓은 쪽으로 갑니다. 부호 있는 타입과 부호 없는 타입이 만나면 부호 있는 쪽으로 가고, 둘의 폭이 같고 64비트보다 좁으면 둘 다 한 단계 넓은 부호 있는 타입으로 갑니다.

```rok
let a: i32 = 1;
let b: i64 = 5000000000;
let c = a + b;   // help: convert one side: `(a as i64) + b`
let u: u32 = 3;
let d = a + u;   // help: convert both sides: `(a as i64) + (u as i64)`
```

## 2.1.7. 고정 폭 원시 타입 ↔ int 변환

fixed primitive와 `int` 사이는 명시적으로만 변환할 수 있습니다.

| 변환                     | 메서드                                | 실패 시                      |
|--------------------------|---------------------------------------|------------------------------|
| `i32` → `int`            | `.into::<int>()` 또는 `int::from_i32` | 항상 성공 (손실 없음)        |
| `int` → `i32`            | `.try_into::<i32>()`                  | `OverflowError`를 던짐      |
| `int` → `i32` (saturate) | `.saturating_into::<i32>()`           | i32::MIN 또는 MAX            |
| `int` → `i32` (wrap)     | `.wrapping_into::<i32>()`             | 모듈러 산술                  |
| `int` → `f64`            | `.into::<f64>()` 또는 `int::to_f64`   | 가장 가까운 값으로 반올림 (`From`의 문서화된 예외, [2.11.8](211-conversion.md)). `float` → `f64`도 같은 예외 |

## 2.1.8. niche 최적화 개요

[refs-lang 2.7
niche](../../refs-lang/002-types/207-niche-optimization.md)가 메커니즘을 정의합니다. 자동 niche가 적용되는 원시 타입은 다음과 같습니다.

`bool | none`은 1바이트(niche 값 `2`)이고, `Ptr<T> | none` / `ConstPtr<T> | none`은 포인터 크기(null이 none)입니다([refs-lang 2.7.2](../../refs-lang/002-types/207-niche-optimization.md#272-자동-niche)). 그러나 `i8`..`i64`,
`u8`..`u64`, `f32`, `f64` 자체에는 자동 niche가 적용되지 않습니다. 모든 비트 패턴이 유효한 값이기 때문입니다.

opt-in niche가 필요하면 [2.2 non-zero](202-non-zero.md)의 `NonZero<T>` / `NonNan<F>` / `Finite<F>` / `Positive<T>` 래퍼를 씁니다.

## 2.1.9. lang slot 매핑

언어 기본값 16종(`i8`..`i64`, `u8`..`u64`, `isize`, `usize`, `f32`, `f64`, `bool`,
`c_char`, `char`, `str`)은 컴파일러가 이름으로 인식합니다. 이 타입에 lang slot을 묶는 표준 라이브러리 항목은 없고 struct도 선언하지 않습니다. 라이브러리는 `impl` 블록만 추가하므로(2.1.4) `#[keyword]`나 `#[lang]`이 필요하지 않습니다.

표준 라이브러리가 추가하는 2종은 PascalCase struct입니다. `#[keyword(...)]`로 등록하고 `#[lang(...)]`으로 lang slot에 묶습니다.

| primitive | lang slot    | struct 이름 | 등록                  |
|-----------|--------------|-------------|-----------------------|
| `int`     | `int_type`   | `Int`       | `#[keyword("int")]`   |
| `float`   | `float_type` | `Float`     | `#[keyword("float")]` |

참고로 `Range<T>`(`a..b`)는 [2.7 iter](207-iter.md)가 정의합니다.

## 2.1.10. 다른 언어와 비교: 설계 근거

Rok의 두 세계 모델(auto-promote `int`와 fixed primitive)은 여러 언어에서 영향을 받았습니다. Smalltalk의 SmallInteger ↔ LargeInteger는 auto-promote의 원형입니다. Lisp와 Scheme의 수 체계(numeric tower: fixnum / bignum /
rational / real)도 중요한 참고 대상입니다. **Python int**는 무한 정밀도를 기본으로 두지만 fixed primitive가 없습니다. **Julia**는 fixed primitive를 기본으로 둡니다. Rok도 리터럴의 기본은 fixed(`i32` / `f64`)이고, `int`는 opt-in입니다.

언어별 차이는 다음과 같습니다. **Python**에는 int만 있고 고정 폭 타입이 없습니다. 그래서 FFI나 성능이 중요한 작업에서는 NumPy에 의존합니다. **Julia**는 Rok처럼 fixed가 기본이고, 무한 정밀도가 필요하면 `BigInt`를 명시합니다. **Rust**에는 fixed만 있어서 사용자가 매번 타입을 적어야 합니다. **Rok**에는 두 세계가 함께 있습니다. 리터럴은 기대 타입을 따르고(없으면 `i32` / `f64`), 무한 정밀도가 필요한 자리에는 `int` / `float`를 적습니다.
