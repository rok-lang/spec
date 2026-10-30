# 2.11. 변환 트레잇

`rok::core::convert` 모듈은 값을 한 타입에서 다른 타입으로 옮기는 트레잇 패밀리를 정의합니다. Rok 변환 정책의 중심 원칙은 silent 정보 손실을 거부하는 것입니다. 그래서 `.into()`는 lossless 변환만 허용합니다. 정보를 잃을 수 있는 변환은 사용자가 정책을 명시적으로 선택해야 합니다.

[refs-lang 2.14 형 변환](../../refs-lang/002-types/214-cast-and-conversion.md)의 문법은 이 트레잇들로 dispatch됩니다.

## 2.11.1. 변환 정책 - 4 갈래

Rok은 변환을 **의도**에 따라 네 갈래로 나눕니다. 사용자는 매번 어느 갈래인지 명시해야 합니다.

| 갈래         | 메서드                              | 의미                                  | 실패 시                     |
|--------------|-------------------------------------|---------------------------------------|-----------------------------|
| **lossless** | `.into::<T>()`                      | 정보 손실 0, 항상 성공                | 컴파일 / 런타임 에러        |
| **fallible** | `.try_into::<T>()`                  | 손실 가능, 호출자가 처리              | 대상의 `TryFrom::Error`를 throw |
| **saturate** | `.saturating_into::<T>()`           | 범위 밖이면 i32::MAX / MIN으로 clamp | 명시 saturate (사용자 선택) |
| **bit**      | `.wrapping_into::<T>()` 또는 `as T` | bit 재해석: 모듈러 산술 / 비트 패턴   | 명시 wrap (사용자 선택)     |

각 갈래의 메서드 이름이 곧 의도입니다. `.into()`를 적으면 **lossless** 의도이고, `.saturating_into()`를 적으면 **saturate** 의도입니다. 컴파일러는 의미를 추측하지 않습니다.

### 디자인 근거

silent saturate나 silent wrap은 사용자 모르게 값을 바꿔서 디버그하기 어려운 버그를 만듭니다. Rok은 JS가 Number → BigInt를 도입하며 얻은 교훈과 NumPy가 wrap에서 OverflowError로 옮겨 가며 얻은 교훈을 함께 받아들였습니다.

## 2.11.2. `From<U>` 트레잇 - lossless 변환

```rok
trait From<U> {
    fn from(value: U) -> Self;
}
```

`From::from(u)`는 항상 성공하며 정보를 전혀 잃지 않습니다.

### 사용 예

```rok
let n: i32 = 42;
let big: i64 = i64::from(n);     // i32 → i64 (lossless 항상 가능)

let x: int = int::from(42i32);   // i32 → int (lossless)
```

### import

`From`은 [묵시 `use rok::core::**;`](../000-meta/002-import.md) 덕분에 바로 쓸 수 있으므로 따로 `use`를 선언할 필요가 없습니다.

## 2.11.3. `Into` 트레잇 - `From` 의 mirror

```rok
trait Into {
    fn into<T: From<Self>>(self) -> T;
}

impl<S> Into for S {
    fn into<T: From<S>>(self) -> T { T::from(self) }
}
```

`Into`는 타입 파라미터가 없는 트레잇이고, 제네릭 메서드 `into` 하나를 가집니다. blanket impl 하나가 모든 타입에 `into`를 주므로, `x.into()`는 `T::from(x)`입니다. 대상 타입 `T`는 기대 타입 (`let` 주석, 인자 자리 등)이나 `x.into::<T>()`에서 정해집니다. 사용자는 `Into`를 구현하지 않고 `From`만 작성합니다.

변환 가능성을 바운드로 적을 때는 `From` 쪽으로 씁니다. `T: Into<U>`라는 바운드는 없습니다.

```rok
fn widen<T, U: From<T>>(x: T) -> U { x.into() }      // OK (U: From<T>)
fn widen2<T: Into<U>, U>(x: T) -> U { x.into() }     // ERROR: Into 는 타입 인자를 받지 않음
```

### 사용 예

```rok
let n: i32 = 42;
let big: i64 = n.into();         // i32::into::<i64>() 자동, From<i32> for i64 사용
let big_typed: i64 = (42i32).into();
```

### `From` 우선 - 디자인 패턴

명시 변환은 다음 두 형태로 쓸 수 있습니다.

```rok
// 형태 1: From 직접
let big = i64::from(42i32);

// 형태 2: into 메서드
let big: i64 = (42i32).into();
```

두 형태는 의미가 같습니다. `from`은 대상 타입이 호출 자리에 그대로 보이고, `into`는 **annotation**이 대상 타입을 정합니다. 가독성이나 패턴에 맞는 쪽을 고르면 됩니다.

### lossless 의 컴파일 검증

`From<U> for T`가 impl되어 있지 않은 조합을 쓰면 컴파일 에러입니다.

```rok
let big: i64 = 1000;
let small: i32 = big.into();   // 컴파일 에러: From<i64> for i32 없음
                                //   (lossless 불가)
```

이 경우에는 `try_into`, `saturating_into`, `as` 중 하나를 명시적으로 선택해야 합니다.

```rok
let small: i32 = try big.try_into();       // throws, 호출자 처리
let small: i32 = big.saturating_into();    // saturate 명시
let small: i32 = big as i32;               // wrap 명시
```

## 2.11.4. `TryFrom<U>` 트레잇 - fallible 변환

```rok
trait TryFrom<U> {
    type Error = ConversionError;
    fn try_from(value: U) throws (Self::Error) -> Self;
}
```

`try_from`은 변환이 실패할 수 있다는 사실을 시그니처에서 직접 표현합니다. 에러 타입은 연관 타입 `Error`이며, 기본값은 [`ConversionError`](214-error.md)입니다 (impl이 생략하면 기본값, [refs-lang 2.8.3](../../refs-lang/002-types/208-trait.md)). 숫자 변환 impl (정수 ↔ 정수, 부동소수 → 정수, `int` / `float` 변환, `TryFrom<u32> for char`)은 `type Error = OverflowError;`를 선언합니다. `throws` 시그니처를 가지므로 사용자는 호출을 `try`로 표시해 전파하거나, `catch`로 처리하거나, `try?`로 `T | none`을 받을 수 있습니다.

```rok
let big: i64 = 9999999999;
let small_result: i32 = try i32::try_from(big);  // try 로 전파
let small_catch: i32 = try i32::try_from(big) catch e: OverflowError { default };
let small_opt: i32 | none = try? i32::try_from(big);
```

### `TryInto` 자동 mirror

`From` ↔ `Into`와 같은 패턴입니다.

```rok
trait TryInto {
    fn try_into<T: TryFrom<Self>>(self) throws (T::Error) -> T;
}
impl<S> TryInto for S { ... }            // = try T::try_from(self)
```

숫자 변환의 `try_from`이 던지는 `OverflowError { op, ty }` ([2.14](214-error.md))의 `op`은 `"try_from"`, `ty`는 대상 타입 이름 (`"i32"` 등)입니다.

```rok
let small: i32 = try big.try_into();     // = try i32::try_from(big)
```

## 2.11.5. `saturating_into` / `wrapping_into` - 명시 정책

정보를 잃을 수 있는 변환은 사용자가 값을 바꾸는 정책을 직접 명시합니다. 이를 위한 메서드가 다음 두 개입니다.

```rok
trait SaturatingFrom<U> {
    fn saturating_from(value: U) -> Self;    // 범위 밖이면 가까운 끝
}
trait SaturatingInto {
    fn saturating_into<T: SaturatingFrom<Self>>(self) -> T;
}

trait WrappingFrom<U> {
    fn wrapping_from(value: U) -> Self;      // 하위 비트 유지 (= as)
}
trait WrappingInto {
    fn wrapping_into<T: WrappingFrom<Self>>(self) -> T;
}
```

`From` / `Into`와 같은 모양입니다. 변환 자체는 `SaturatingFrom` / `WrappingFrom` impl이고, `saturating_into` / `wrapping_into`는 blanket impl이 주는 제네릭 메서드입니다.

### `saturating_into` - 범위 끝으로 clamp

```rok
let big: i64 = 9999999999;
let small: i32 = big.saturating_into();    //→ i32::MAX (= 2147483647)

let big_neg: i64 = -9999999999;
let small_neg: i32 = big_neg.saturating_into();  //→ i32::MIN
```

이 정책은 DSP, 신호 처리, UI clamp 같은 영역에 쓸모가 있습니다.

### `wrapping_into` - 모듈러 산술

```rok
let big: i64 = 0x100000001;
let small: i32 = big.wrapping_into();      //→ 1 (하위 32 비트)
```

`as i32`와 의미가 같지만, 의도를 명시적인 메서드 호출로 드러냅니다. 이 정책은 hash 계산이나 비트마스킹에 알맞습니다.

## 2.11.6. `as T` - bit 재해석 / numeric cast

`as` 키워드 ([refs-lang 2.14](../../refs-lang/002-types/214-cast-and-conversion.md))는 numeric / bit 변환을 **명시**하는 수단입니다.

```rok
let n: i32 = 256;
let b: u8 = n as u8;             // 하위 8 비트 = 0
let b2: u8 = 257 as u8;          // 1
let b_neg: u8 = -1i32 as u8;     // 255

let f: f64 = 1.7;
let i: i32 = f as i32;            // 1 (절삭, IEEE 754 → integer)
let f2: f32 = 3.14f64 as f32;     // 3.14 → f32 (IEEE 754 narrowing)
```

`as`의 동작은 다음과 같습니다.

| 변환                                       | 의미                                               |
|--------------------------------------------|----------------------------------------------------|
| 정수 → 같은 폭 정수 (signed/unsigned 상호) | bit 패턴 그대로, 부호 해석만 바뀜                  |
| 좁은 정수 → 넓은 정수                      | sign-extend (signed) 또는 zero-extend (unsigned)   |
| 넓은 정수 → 좁은 정수                      | 절삭 (하위 비트만)                                 |
| 정수 → 부동소수                            | IEEE 754 표현으로 (정밀도 손실 가능)               |
| 부동소수 → 정수                            | 0 쪽으로 절삭. 범위 밖은 MIN/MAX로 saturate, NaN → 0 (모든 최적화 수준) |
| 부동소수 → 같은/다른 폭 부동소수           | IEEE 754 round                                     |
| `u8` → `char`                              | 그 code point (U+0000..=U+00FF)                    |
| `char` → 정수                              | code point (좁은 정수면 절삭)                      |

`char`로의 `as`는 `u8`에서만 됩니다. 다른 정수 / 부동소수 / `bool` / `c_char`는 타입 에러이며, 넓은 값은 `char::from_u32(n)` (`char | none`)으로 바꿉니다 ([2.15.4](215-string.md)).

### `as` 의 한계

`as`는 numeric 변환과 비트 변환에만 쓰입니다. 임의 타입 사이의 변환은 거부됩니다.

```rok
struct Vec3 { x: f64; y: f64; z: f64; }
let v = Vec3 { x: 1.0, y: 2.0, z: 3.0 };
let s: str = v as str;     // 컴파일 에러: `as` 는 user struct 거부
```

user 타입 사이의 변환은 `From` / `Into` 트레잇으로 명시해야 합니다.

## 2.11.7. literal context auto-From - 컴파일 타임 range check

annotation이 붙은 literal에는 컴파일 타임 range check와 lossless 변환이 함께 적용됩니다.

```rok
let x: i32 = 42;             // OK: 42 가 i32 범위
let y: i32 = 9999999999;     // 컴파일 에러: literal 이 i32 범위 밖
let z: f32 = 3.14;           // OK: round to nearest f32
let w: u8 = 256;             // 컴파일 에러: 256 ∉ u8 0..=255
```

`T::from(1)` / `1.into()`처럼 리터럴로 impl을 고르는 규칙은 [refs-lang 3.1.3](../../refs-lang/003-expressions/301-operator-dispatch.md#impl-고르기)을 따릅니다. literal은 컴파일 타임에 값이 알려져 있으므로 이렇게 할 수 있습니다. runtime 값을 변환할 때는 `from`, `into`, `try_into`, `as` 중 하나를 반드시 명시해야 합니다.

## 2.11.8. `From` impl 매트릭스 - primitive 사이

표준 라이브러리에 있는 `From<U> for T` impl은 다음과 같습니다 (lossless만 표시).

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

unsigned에서 폭이 같거나 더 좁은 signed로 가는 변환 (예: u32 → i32)은 lossless가 보장되지 않으므로 `try_into`나 `as`를 사용해야 합니다.

### 정수 → 부동소수

| from \ to | f32 | f64                     |
|-----------|-----|-------------------------|
| `i8`      | ✓   | ✓                       |
| `i16`     | ✓   | ✓                       |
| `i32`     |     | ✓                       |
| `i64`     |     | (`From` 없음: precision 손실) |

f32는 24-bit mantissa, f64는 53-bit mantissa이므로 그 비트 폭 안에서만 lossless가 성립합니다. unsigned 정수에서 부동소수 타입으로 가는 `From` impl은 없으므로 `as`를 씁니다.

### 부동소수 사이

| from \ to | f32 | f64 | float |
|-----------|-----|-----|-------|
| `f32`     |     | ✓   | ✓     |
| `f64`     |     |     | ✓     |

f64 → f32는 좁히는 변환이므로 `try_into`, `saturating_into`, `as` 중 하나를 사용해야 합니다.

### int / float 와 fixed primitive

| from \ to | i32               | i64           | f64                 |
|-----------|-------------------|---------------|---------------------|
| `int`     | `try_into`만     | `try_into`만 | `into` (반올림, 예외) |
| `float`   | `try_into` (절삭) | `try_into`    | `into` (예외)       |

`From<int> for f64`와 `From<float> for f64`는 2.11.2의 "From은 lossless"에 대한 **문서화된 예외**입니다. `int`는 가장 가까운 f64로 반올림하고, 지수가 f64 범위 밖으로 promote된 `float`는 f64로 표현할 수 없어 손실됩니다. 손실이 드러나야 하는 자리에서는 `to_f64()`나 `as f64`를 권장합니다.

`int → fixed`는 lossless가 보장되지 않으므로 (int 값이 fixed 범위 밖일 수 있음) `try_into`나 `saturating_into`를 사용해야 합니다.

반면 `fixed → int`는 항상 lossless입니다 (`From<i32> for int` 등이 모두 impl되어 있습니다).

## 2.11.9. `From` impl - 사용자 타입

사용자 타입 사이의 변환은 `impl From<U> for T`를 직접 작성해서 정의합니다.

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

이때 blanket `Into`가 `c.into()`를 `Fahrenheit::from(c)`로 이어 주므로, 사용자가 `Into`를 직접 작성할 필요는 없습니다.

### `try_from` 도 동일 패턴

```rok
impl TryFrom<str> for i32 {
    type Error = OverflowError;
    fn try_from(s: str) throws (Self::Error) -> i32 {
        // s 가 i32 range 안의 정수 string 이면 그 값, 아니면 OverflowError 를 throw
        ...
    }
}

let n: i32 = try "42".try_into();
```

모든 고정 폭 정수 타입은 자기 `FromStr` 위에 이런 식으로 `TryFrom<str>`을 구현합니다: 빈 텍스트, 잘못된 숫자, 범위 밖의 값은 `OverflowError { op: "try_from", ty }`를 던집니다. 부동소수 타입, `bool`, `char`는 `FromStr`만 제공합니다.

## 2.11.10. 산술 결과의 자동 변환 - 없음

산술 결과의 auto-From ([refs-lang 2.14](../../refs-lang/002-types/214-cast-and-conversion.md))은 **literal context**에서만 적용됩니다. 값끼리 계산한 결과는 자동으로 변환되지 않습니다.

```rok
let a: i32 = 5;
let b: i32 = 3;
let c: i32 = a + b;          // OK: i32 + i32 = i32

let big: i32 = a + b * 1000000;
// → a + b * 1000000 의 결과가 i32 범위면 OK, 밖이면 (debug=panic / release=wrap)
// silent saturate / silent promote 안 함

let auto: i64 = a + b;       // 컴파일 에러: i32 결과를 i64 에 silent promote 안 함
let auto: i64 = (a + b) as i64;  // 명시 변환 OK
let auto: i64 = (a + b).into();  // From<i32> for i64 (lossless) 명시
```

## 2.11.11. 다른 언어와의 비교 - 디자인 결정 근거

| 언어   | 변환 모델                                               | Rok과의 차이                                                                     |
|--------|---------------------------------------------------------|-----------------------------------------------------------------------------------|
| Rust   | `From`/`Into` lossless, `as` wrap, `try_into` Result    | Rok과 가장 유사. `saturating_into`/`wrapping_into` 메서드는 Rok이 명시 추가     |
| Swift  | 모든 정수 변환 trapping 기본, `.init(clamping:)` opt-in | Rok은 trapping 거부: debug=panic + release=wrap (fixed-width 산술), 변환은 명시  |
| Julia  | `Int32(big)` trapping, `convert` 함수 패밀리            | Rok은 trait 메서드로 통일 (`from`/`into`)                                        |
| Python | `int(x)`/`float(x)` 함수, 항상 무한 정밀도              | Rok의 `int` / `float`는 같은 정신, fixed primitive는 별개 세계                 |
| C      | implicit conversion (`int` → `long` 자동)               | Rok 명시 거부, silent type mixing 차단                                            |

이 비교에서 나온 Rok의 주요 디자인 결정은 다음과 같습니다. silent 정보 손실을 거부합니다 (Rust의 영향). 네 갈래 (`into`, `try_into`, `saturating_into`, `as`)를 명시하게 해서 의도를 드러내게 합니다. auto-From은 literal context에서만 허용하고, runtime 값은 명시적으로 변환해야 합니다. `int` / `float`의 auto-promote와 fixed primitive를 분리합니다 (Python과 Smalltalk의 영향).

## 2.11.12. 모듈 매핑

| 트레잇              | lang slot            | 표준 라이브러리 위치 |
|---------------------|----------------------|----------------------|
| `From<U>`           | (없음, derive 가능)  | 표준 라이브러리      |
| `Into`              | (blanket impl)       | 표준 라이브러리      |
| `TryFrom<U>`        | (없음)               | 표준 라이브러리      |
| `TryInto`           | (blanket impl)       | 표준 라이브러리      |
| `SaturatingFrom<U>` | (없음)               | 표준 라이브러리      |
| `SaturatingInto`    | (blanket impl)       | 표준 라이브러리      |
| `WrappingFrom<U>`   | (없음)               | 표준 라이브러리      |
| `WrappingInto`      | (blanket impl)       | 표준 라이브러리      |

primitive 사이의 모든 lossless 변환 impl에는 `#[intrinsic("rok.cast.FROM.TO")]`가 붙습니다.

```rok
impl From<i32> for i64 {
    #[intrinsic("rok.cast.i32.i64")]
    fn from(value: i32) -> i64 { ... }
}
```

`as T`의 numeric cast도 같은 intrinsic을 사용하며, lossless 분기와 wrapping 분기 중 어느 쪽인지는 컴파일러가 고릅니다.
