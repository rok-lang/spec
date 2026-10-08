# 2.11. 변환 트레잇

`rok::core::convert` 모듈은 값을 한 타입에서 다른 타입으로 옮기는 트레잇 패밀리를 정의합니다. rok 의 변환 정책에서 가장 핵심이 되는 원칙은 silent 정보 손실을 거부하는 것입니다. 따라서 `.into()` 는 lossless 한 변환만을 허용하며, 손실이 발생할 수 있는 변환의 경우에는 사용자가 명시적으로 그 정책을 선택하도록 설계되어 있습니다.

[refs-lang 2.14 형 변환](../../refs-lang/002-types/214-cast-and-conversion.md) 의 syntax 는 본 트레잇으로 dispatch 됩니다.

## 2.11.1. 변환 정책 — 4 갈래

rok 은 변환의 **의도** 에 따라 네 갈래로 분리해 다룹니다. 사용자는 매번 어느 갈래에 해당하는지를 명시해야 합니다.

| 갈래         | 메서드                              | 의미                                  | 실패 시                     |
|--------------|-------------------------------------|---------------------------------------|-----------------------------|
| **lossless** | `.into::<T>()`                      | 정보 손실 0 — 항상 성공               | 컴파일 / 런타임 에러        |
| **fallible** | `.try_into::<T>()`                  | 손실 가능 — 호출자가 처리             | `Result<T, OverflowError>`  |
| **saturate** | `.saturating_into::<T>()`           | 범위 밖이면 i32::MAX / MIN 으로 clamp | 명시 saturate (사용자 선택) |
| **bit**      | `.wrapping_into::<T>()` 또는 `as T` | bit 재해석 — 모듈러 산술 / 비트 패턴  | 명시 wrap (사용자 선택)     |

각 갈래의 메서드 이름이 그 자체로 의도를 명시해 줍니다. 예를 들어 사용자가 `.into()` 를 적으면 **lossless** 의도이고, `.saturating_into()` 를 적으면 **saturate** 의도가 됩니다. 컴파일러는 어떤 의미인지를 추측하지 않습니다.

### 디자인 근거

silent saturate 나 silent wrap 은 사용자가 모르는 사이에 값이 변형되어 디버그하기 어려운 버그를 만들어 냅니다. rok 은 JS 가 Number → BigInt 를 도입하면서 얻은 교훈, 그리고 NumPy 가 wrap 에서 OverflowError 로 이동하면서 얻은 교훈을 함께 받아들였습니다.

## 2.11.2. `From<U>` 트레잇 — lossless 변환

```rok
trait From<U> {
    fn from(value: U) -> Self;
}
```

`From::from(u)` 는 항상 성공하는 변환이며, 정보 손실이 전혀 발생하지 않습니다.

### 사용 예

```rok
let n: i32 = 42;
let big: i64 = i64::from(n);     // i32 → i64 (lossless 항상 가능)

let x: int = int::from(42i32);   // i32 → int (lossless)
```

### import

`From` 은 [묵시 `use rok::core::**;`](../000-meta/002-import.md) 에 의해 자동으로 사용 가능하므로, 별도의 `use` 선언이 필요하지 않습니다.

## 2.11.3. `Into<T>` 트레잇 — `From` 의 mirror

```rok
trait Into<T> {
    fn into(self) -> T;
}
```

`From<U> for T` 가 구현되면 그에 대응하는 `Into<T> for U` 가 자동으로 구현됩니다. 따라서 사용자는 `Into` 를 직접 구현하지 않고 `From` 만 작성하면 됩니다.

### 사용 예

```rok
let n: i32 = 42;
let big: i64 = n.into();         // i32::into::<i64>() 자동 — From<i32> for i64 사용
let big_typed: i64 = (42i32).into();
```

### `From` 우선 — 디자인 패턴

스택 명시 변환은 다음 두 형태로 표현할 수 있습니다.

```rok
// 형태 1: From 직접
let big = i64::from(42i32);

// 형태 2: into 메서드
let big: i64 = (42i32).into();
```

두 형태는 의미상 동등합니다. `from` 은 대상 타입 명시가 호출 자리에서 그대로 보이는 반면, `into` 는 **annotation** 이 대상 타입을 결정하게 합니다. 따라서 가독성이나 패턴에 따라 적절한 쪽을 선택하면 됩니다.

### lossless 의 컴파일 검증

만약 `From<U> for T` 가 impl 되어 있지 않은 조합을 시도한다면 컴파일 에러가 발생합니다.

```rok
let big: i64 = 1000;
let small: i32 = big.into();   // 컴파일 에러 — From<i64> for i32 없음
                                //   (lossless 불가)
```

이 경우에는 `try_into`, `saturating_into`, `as` 중 하나를 명시적으로 선택해야 합니다.

```rok
let small: i32 = big.try_into()?;          // Result, 호출자 처리
let small: i32 = big.saturating_into();    // saturate 명시
let small: i32 = big as i32;               // wrap 명시
```

## 2.11.4. `TryFrom<U>` 트레잇 — fallible 변환

```rok
trait TryFrom<U> {
    fn try_from(value: U) -> Self | OverflowError;
}
```

`try_from` 은 변환이 실패할 수 있다는 사실을 시그니처에서 직접 표현합니다. 합 타입 (`Self | OverflowError`) 을 반환하므로 사용자가 `match` 또는 `?` 로 처리할 수 있습니다.

```rok
let big: i64 = 9999999999;
let small_result: i32 = i32::try_from(big)?;     // ? 로 propagate
let small_match = match i32::try_from(big) {
    v: i32 => v,
    e: OverflowError => default,
};
```

### `TryInto<T>` 자동 mirror

`From` ↔ `Into` 와 동일한 패턴이 적용됩니다. 즉 `TryFrom<U> for T` 가 impl 되면 `TryInto<T> for U` 도 자동으로 구현됩니다.

```rok
let small: i32 = big.try_into()?;        // = i32::try_from(big)?
```

## 2.11.5. `saturating_into` / `wrapping_into` — 명시 정책

손실이 가능한 변환에 대해서는 사용자가 값 변형 정책을 직접 명시하게 됩니다. 그 수단으로 다음 두 메서드가 제공됩니다.

```rok
trait SaturatingInto<T> {
    fn saturating_into(self) -> T;
}

trait WrappingInto<T> {
    fn wrapping_into(self) -> T;
}
```

### `saturating_into` — 범위 끝으로 clamp

```rok
let big: i64 = 9999999999;
let small: i32 = big.saturating_into();    //→ i32::MAX (= 2147483647)

let big_neg: i64 = -9999999999;
let small_neg: i32 = big_neg.saturating_into();  //→ i32::MIN
```

이 정책은 DSP, 신호 처리, UI clamp 등의 영역에서 유용하게 쓰입니다.

### `wrapping_into` — 모듈러 산술

```rok
let big: i64 = 0x100000001;
let small: i32 = big.wrapping_into();      //→ 1 (하위 32 비트)
```

`as i32` 와 같은 의미이지만, 명시적인 메서드 호출 형태로 의도를 표현해 준다는 점이 다릅니다. 이 정책은 hash 계산이나 비트마스킹 도메인에 적합합니다.

## 2.11.6. `as T` — bit 재해석 / numeric cast

`as` 키워드 ([refs-lang 2.14](../../refs-lang/002-types/214-cast-and-conversion.md)) 는 numeric / bit 변환을 **명시** 하는 수단입니다.

```rok
let n: i32 = 256;
let b: u8 = n as u8;             // 하위 8 비트 = 0
let b2: u8 = 257 as u8;          // 1
let b_neg: u8 = -1i32 as u8;     // 255

let f: f64 = 1.7;
let i: i32 = f as i32;            // 1 (절삭, IEEE 754 → integer)
let f2: f32 = 3.14f64 as f32;     // 3.14 → f32 (IEEE 754 narrowing)
```

`as` 의 동작은 다음과 같이 정리할 수 있습니다.

| 변환                                       | 의미                                               |
|--------------------------------------------|----------------------------------------------------|
| 정수 → 같은 폭 정수 (signed/unsigned 상호) | bit 패턴 그대로 — 부호 해석만 바뀜                 |
| 좁은 정수 → 넓은 정수                      | sign-extend (signed) 또는 zero-extend (unsigned)   |
| 넓은 정수 → 좁은 정수                      | 절삭 (하위 비트만)                                 |
| 정수 → 부동소수                            | IEEE 754 표현으로 (정밀도 손실 가능)               |
| 부동소수 → 정수                            | 절삭 (truncate toward zero), NaN → 0, ±∞ → MIN/MAX |
| 부동소수 → 같은/다른 폭 부동소수           | IEEE 754 round                                     |

### `as` 의 한계

`as` 는 numeric 과 비트 변환에 한정됩니다. 따라서 임의 타입 사이의 변환은 거부됩니다.

```rok
struct Vec3 { x: f64; y: f64; z: f64; }
let v = Vec3 { x: 1.0, y: 2.0, z: 3.0 };
let s: str = v as str;     // 컴파일 에러 — `as` 는 user struct 거부
```

user 타입 사이의 변환은 `From` / `Into` 트레잇으로 명시해 주셔야 합니다.

## 2.11.7. literal context auto-From — 컴파일 타임 range check

annotation 이 붙은 literal 의 경우에는 컴파일 타임 range check 와 lossless 변환이 함께 적용됩니다.

```rok
let x: i32 = 42;             // OK — 42 가 i32 범위
let y: i32 = 9999999999;     // 컴파일 에러 — literal 이 i32 범위 밖
let z: f32 = 3.14;           // OK — round to nearest f32
let w: u8 = 256;             // 컴파일 에러 — 256 ∉ u8 0..=255
```

이러한 메커니즘이 가능한 이유는 literal 자체가 컴파일 타임에 알려진 값이기 때문입니다. runtime 값을 변환하는 경우에는 `from`, `into`, `try_into`, `as` 중 하나를 반드시 명시해 주셔야 합니다.

## 2.11.8. `From` impl 매트릭스 — primitive 사이

표준 라이브러리가 제공하는 `From<U> for T` impl 표는 다음과 같습니다 (lossless 만 표시).

### 정수 → 정수 (signed)

| from \ to | i8 | i16 | i32 | i64           | isize         | int |
|-----------|----|-----|-----|---------------|---------------|-----|
| `i8`      |    | ✓   | ✓   | ✓             | ✓             | ✓   |
| `i16`     |    |     | ✓   | ✓             | ✓             | ✓   |
| `i32`     |    |     |     | ✓             | ✓ (target≥32) | ✓   |
| `i64`     |    |     |     |               | ✓ (target=64) | ✓   |
| `isize`   |    |     |     | ✓ (target≤64) |               | ✓   |

### 정수 → 정수 (unsigned)

| from \ to | u8 | u16 | u32 | u64           | usize         | int (always lossless) |
|-----------|----|-----|-----|---------------|---------------|-----------------------|
| `u8`      |    | ✓   | ✓   | ✓             | ✓             | ✓                     |
| `u16`     |    |     | ✓   | ✓             | ✓             | ✓                     |
| `u32`     |    |     |     | ✓             | ✓ (target≥32) | ✓                     |
| `u64`     |    |     |     |               | ✓ (target=64) | ✓                     |
| `usize`   |    |     |     | ✓ (target≤64) |               | ✓                     |

### unsigned → signed (lossless 만)

| from \ to | i8 | i16 | i32 | i64 | int |
|-----------|----|-----|-----|-----|-----|
| `u8`      |    | ✓   | ✓   | ✓   | ✓   |
| `u16`     |    |     | ✓   | ✓   | ✓   |
| `u32`     |    |     |     | ✓   | ✓   |
| `u64`     |    |     |     |     | ✓   |

unsigned 에서 같거나 더 좁은 signed 로의 변환 (예를 들어 u32 → i32) 은 lossless 가 보장되지 않으므로, 이 경우에는 `try_into` 또는 `as` 를 사용해 주셔야 합니다.

### 정수 → 부동소수

| from \ to | f32 | f64                     |
|-----------|-----|-------------------------|
| `i8`      | ✓   | ✓                       |
| `i16`     | ✓   | ✓                       |
| `i32`     |     | ✓ (precision 손실 가능) |
| `i64`     |     | (precision 손실)        |

f32 는 24-bit mantissa, f64 는 53-bit mantissa 이므로 그 비트 폭 안에서만 lossless 가 성립합니다.

### 부동소수 사이

| from \ to | f32 | f64 | float |
|-----------|-----|-----|-------|
| `f32`     |     | ✓   | ✓     |
| `f64`     |     |     | ✓     |

f64 → f32 는 narrowing 변환이므로 `try_into`, `saturating_into`, `as` 중 하나를 사용해 주셔야 합니다.

### int / float 와 fixed primitive

| from \ to | i32               | i64           | f64                 |
|-----------|-------------------|---------------|---------------------|
| `int`     | `try_into` 만     | `try_into` 만 | `into` (lossy 가능) |
| `float`   | `try_into` (절삭) | `try_into`    | `into`              |

`int → fixed` 의 경우 lossless 가 보장되지 않습니다 (int 가 fixed 범위 밖일 수 있기 때문입니다). 따라서 `try_into` 또는 `saturating_into` 를 사용해 주셔야 합니다.

반면 `fixed → int` 는 항상 lossless 입니다 (`From<i32> for int` 등이 모두 impl 되어 있습니다).

## 2.11.9. `From` impl — 사용자 타입

사용자 타입 사이의 변환은 `impl From<U> for T` 를 직접 작성해 정의합니다.

```rok
struct Celsius { degree: f64; }
struct Fahrenheit { degree: f64; }

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Fahrenheit {
        Fahrenheit { degree: c.degree * 9.0 / 5.0 + 32.0 }
    }
}

let c = Celsius { degree: 100.0 };
let f: Fahrenheit = c.into();   //→ Fahrenheit { degree: 212.0 }
```

이때 `Into<Fahrenheit>` 가 자동으로 mirror 되므로, 사용자가 `Into` 를 직접 작성할 필요는 없습니다.

### `try_from` 도 동일 패턴

```rok
impl TryFrom<str> for i32 {
    fn try_from(s: str) -> i32 | OverflowError {
        // s 가 i32 range 안의 정수 string 이면 some, 아니면 OverflowError
        ...
    }
}

let n: i32 = "42".try_into()?;
```

## 2.11.10. 산술 결과의 자동 변환 — 없음

arithmetic 결과의 auto-From ([refs-lang 2.14](../../refs-lang/002-types/214-cast-and-conversion.md)) 은 **literal context** 에서만 적용됩니다. value 산술 결과는 자동으로 변환되지 않습니다.

```rok
let a: i32 = 5;
let b: i32 = 3;
let c: i32 = a + b;          // OK — i32 + i32 = i32

let big: i32 = a + b * 1000000;
// → a + b * 1000000 의 결과가 i32 범위면 OK, 밖이면 (debug=panic / release=wrap)
// silent saturate / silent promote 안 함

let auto: i64 = a + b;       // 컴파일 에러 — i32 결과를 i64 에 silent promote 안 함
let auto: i64 = (a + b) as i64;  // 명시 변환 OK
let auto: i64 = (a + b).into();  // From<i32> for i64 (lossless) 명시
```

## 2.11.11. 다른 언어와의 비교 — 디자인 결정 근거

| 언어   | 변환 모델                                               | rok 과의 차이                                                                     |
|--------|---------------------------------------------------------|-----------------------------------------------------------------------------------|
| Rust   | `From`/`Into` lossless, `as` wrap, `try_into` Result    | rok 과 가장 유사 — `saturating_into`/`wrapping_into` 메서드는 rok 이 명시 추가    |
| Swift  | 모든 정수 변환 trapping 기본, `.init(clamping:)` opt-in | rok 은 trapping 거부 — debug=panic + release=wrap (fixed-width 산술), 변환은 명시 |
| Julia  | `Int32(big)` trapping, `convert` 함수 패밀리            | rok 은 trait 메서드로 통일 (`from`/`into`)                                        |
| Python | `int(x)`/`float(x)` 함수 — 항상 무한 정밀도             | rok 의 `int` / `float` 는 같은 정신, fixed primitive 는 별개 세계                 |
| C      | implicit conversion (`int` → `long` 자동)               | rok 명시 거부 — silent type mixing 차단                                           |

이러한 비교를 토대로 rok 이 채택한 핵심 디자인 결정은 다음과 같이 정리할 수 있습니다. 먼저 silent 정보 손실을 거부합니다 (Rust 의 영향을 받았습니다). 다음으로 4 갈래 (`into`, `try_into`, `saturating_into`, `as`) 를 명시함으로써 의도 표현을 강제합니다. 또한 literal context 에서만 auto-From 을 허용하며, runtime value 의 경우에는 명시를 강제합니다. 마지막으로 `int` / `float` 의 auto-promote 와 fixed primitive 를 분리하는데, 이는 Python 과 Smalltalk 의 영향에서 나온 결정입니다.

## 2.11.12. 모듈 매핑

| 트레잇              | lang slot            | 표준 라이브러리 위치 |
|---------------------|----------------------|----------------------|
| `From<U>`           | (없음 — derive 가능) | 표준 라이브러리      |
| `Into<T>`           | (자동 mirror)        | 표준 라이브러리      |
| `TryFrom<U>`        | (없음)               | 표준 라이브러리      |
| `TryInto<T>`        | (자동 mirror)        | 표준 라이브러리      |
| `SaturatingInto<T>` | (없음)               | 표준 라이브러리      |
| `WrappingInto<T>`   | (없음)               | 표준 라이브러리      |

primitive 사이의 모든 lossless 변환 impl 에는 `#[intrinsic("rok.cast.FROM.TO")]` 가 부착됩니다.

```rok
impl From<i32> for i64 {
    #[intrinsic("rok.cast.i32.i64")]
    fn from(value: i32) -> i64 { ... }
}
```

`as T` 의 numeric cast 도 같은 intrinsic 을 사용하며, lossless 와 wrapping 사이의 분기는 컴파일러가 선택하게 됩니다.
