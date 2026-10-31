# 2.11. 변환 트레잇

`rok::core::convert` 모듈은 값을 한 타입에서 다른 타입으로 옮기는 트레잇 묶음을 정의합니다. Rok 변환 정책의 중심 원칙은 소리 없이 정보를 잃는 변환을 거부하는 것입니다. 그래서 `.into()`는 무손실(lossless) 변환만 허용합니다. 정보를 잃을 수 있는 변환에서는 사용자가 정책을 직접 골라야 합니다.

[refs-lang 2.14 형 변환](../../refs-lang/002-types/214-cast-and-conversion.md)의 문법은 이 트레잇으로 디스패치됩니다.

## 2.11.1. 변환 정책: 네 갈래

Rok은 변환을 **의도**에 따라 네 갈래로 나눕니다. 사용자는 매번 어느 갈래인지 밝혀야 합니다.

| 갈래         | 메서드                              | 의미                                  | 실패 시                     |
|--------------|-------------------------------------|---------------------------------------|-----------------------------|
| **lossless** | `.into::<T>()`                      | 정보 손실 없음, 항상 성공             | 컴파일 / 런타임 에러        |
| **fallible** | `.try_into::<T>()`                  | 정보를 잃을 수 있음, 호출자가 처리    | 대상의 `TryFrom::Error`를 던짐 |
| **saturate** | `.saturating_into::<T>()`           | 범위 밖이면 i32::MAX / MIN으로 고정   | 명시한 포화(사용자 선택)    |
| **bit**      | `.wrapping_into::<T>()` 또는 `as T` | 비트 재해석: 모듈러 산술 / 비트 패턴  | 명시한 랩어라운드(사용자 선택) |

각 갈래의 메서드 이름이 곧 의도입니다. `.into()`를 적으면 **lossless** 의도이고, `.saturating_into()`를 적으면 **saturate** 의도입니다. 컴파일러는 의미를 짐작하지 않습니다.

### 설계 근거

소리 없는 포화나 랩어라운드는 사용자 모르게 값을 바꿉니다. 그 결과 디버그하기 어려운 버그가 생깁니다. Rok은 두 교훈을 함께 받아들였습니다. 하나는 JS가 Number → BigInt를 도입하며 얻은 교훈이고, 다른 하나는 NumPy가 랩어라운드에서 OverflowError로 옮겨 가며 얻은 교훈입니다.

## 2.11.2. `From<U>` 트레잇: 무손실 변환

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

`From`은 [묵시 `use rok::core::**;`](../000-meta/002-import.md) 덕분에 바로 쓸 수 있습니다. 따로 `use`를 선언하지 않아도 됩니다.

## 2.11.3. `Into` 트레잇: `From`의 반대 방향

```rok
trait Into {
    fn into<T: From<Self>>(self) -> T;
}

impl<S> Into for S {
    fn into<T: From<S>>(self) -> T { T::from(self) }
}
```

`Into`는 타입 매개변수가 없는 트레잇이고, 제네릭 메서드 `into` 하나만 있습니다. 포괄 구현(blanket impl) 하나가 모든 타입에 `into`를 줍니다. 그래서 `x.into()`는 `T::from(x)`입니다. 대상 타입 `T`는 기대 타입(`let` 주석, 인자 자리 등)이나 `x.into::<T>()`로 정해집니다. 사용자는 `Into`를 구현하지 않고 `From`만 작성합니다.

변환할 수 있다는 조건을 바운드로 적을 때는 `From` 쪽으로 씁니다. `T: Into<U>`라는 바운드는 없습니다.

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

### `From` 우선: 설계 패턴

명시 변환은 다음 두 형태로 쓸 수 있습니다.

```rok
// 형태 1: From 직접
let big = i64::from(42i32);

// 형태 2: into 메서드
let big: i64 = (42i32).into();
```

두 형태는 의미가 같습니다. `from`은 대상 타입이 호출 자리에 그대로 보입니다. `into`는 **타입 주석**이 대상 타입을 정합니다. 가독성이나 패턴에 맞는 쪽을 고르면 됩니다.

### 무손실의 컴파일 타임 검증

`From<U> for T`가 구현되지 않은 조합을 쓰면 컴파일 에러입니다.

```rok
let big: i64 = 1000;
let small: i32 = big.into();   // 컴파일 에러: From<i64> for i32 없음
                                //   (lossless 불가)
```

이때는 `try_into`, `saturating_into`, `as` 중 하나를 직접 골라 써야 합니다.

```rok
let small: i32 = try big.try_into();       // throws, 호출자 처리
let small: i32 = big.saturating_into();    // saturate 명시
let small: i32 = big as i32;               // wrap 명시
```

## 2.11.4. `TryFrom<U>` 트레잇: 실패할 수 있는 변환

```rok
trait TryFrom<U> {
    type Error = ConversionError;
    fn try_from(value: U) throws (Self::Error) -> Self;
}
```

`try_from`은 변환이 실패할 수 있다는 사실을 시그니처로 드러냅니다. 에러 타입은 연관 타입 `Error`이고, 기본값은 [`ConversionError`](214-error.md)입니다. 구현이 이 타입을 생략하면 기본값을 씁니다([refs-lang 2.8.3](../../refs-lang/002-types/208-trait.md)). 숫자 변환 구현(정수 ↔ 정수, 부동소수 → 정수, `int` / `float` 변환, `TryFrom<u32> for char`)은 `type Error = OverflowError;`를 선언합니다. 시그니처가 `throws`이므로 호출하는 쪽은 세 가지 중 하나를 고릅니다. 호출에 `try`를 붙여 전파하거나, `catch`로 처리하거나, `try?`로 `T | none`을 받습니다.

```rok
let big: i64 = 9999999999;
let small_result: i32 = try i32::try_from(big);  // try 로 전파
let small_catch: i32 = try i32::try_from(big) catch e: OverflowError { default };
let small_opt: i32 | none = try? i32::try_from(big);
```

### 자동 `TryInto`

`From` ↔ `Into`와 같은 패턴입니다.

```rok
trait TryInto {
    fn try_into<T: TryFrom<Self>>(self) throws (T::Error) -> T;
}
impl<S> TryInto for S { ... }            // = try T::try_from(self)
```

숫자 변환의 `try_from`은 `OverflowError { op, ty }`([2.14](214-error.md))를 던집니다. 이때 `op`은 `"try_from"`이고, `ty`는 대상 타입 이름(`"i32"` 등)입니다.

```rok
let small: i32 = try big.try_into();     // = try i32::try_from(big)
```

## 2.11.5. `saturating_into` / `wrapping_into`: 명시 정책

정보를 잃을 수 있는 변환에서는 사용자가 값을 어떻게 바꿀지 정책을 직접 밝힙니다. 이때 쓰는 메서드는 다음 두 개입니다.

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

`From` / `Into`와 같은 모양입니다. 변환 자체는 `SaturatingFrom` / `WrappingFrom` 구현이 맡습니다. `saturating_into` / `wrapping_into`는 포괄 구현이 주는 제네릭 메서드입니다.

### `saturating_into`: 범위 끝으로 clamp

```rok
let big: i64 = 9999999999;
let small: i32 = big.saturating_into();    //→ i32::MAX (= 2147483647)

let big_neg: i64 = -9999999999;
let small_neg: i32 = big_neg.saturating_into();  //→ i32::MIN
```

이 정책은 DSP, 신호 처리, UI 값 고정 같은 영역에 쓸모가 있습니다.

### `wrapping_into`: 모듈러 산술

```rok
let big: i64 = 0x100000001;
let small: i32 = big.wrapping_into();      //→ 1 (하위 32 비트)
```

`as i32`와 의미가 같지만, 의도를 메서드 호출로 드러냅니다. 이 정책은 해시 계산이나 비트 마스킹에 알맞습니다.

## 2.11.6. `as T`: 비트 재해석 / 숫자 캐스트

`as` 키워드([refs-lang 2.14](../../refs-lang/002-types/214-cast-and-conversion.md))는 수 변환과 비트 변환을 **명시**하는 수단입니다.

```rok
let n: i32 = 256;
let b: u8 = n as u8;             // 하위 8 비트 = 0
let b2: u8 = 257 as u8;          // 1
let b_neg: u8 = -1i32 as u8;     // 255

let f: f64 = 1.7;
let i: i32 = f as i32;            // 1 (절삭, IEEE 754 → integer)
let big: i32 = 1e10 as i32;       // i32::MAX (포화)
let nan: i32 = f64::NAN as i32;   // 0
let f2: f32 = 3.14f64 as f32;     // 3.14 → f32 (IEEE 754 narrowing)
```

`as`의 동작은 다음과 같습니다.

| 변환                                       | 의미                                               |
|--------------------------------------------|----------------------------------------------------|
| 정수 → 같은 폭 정수(부호 있음/없음 사이)   | 비트 패턴 그대로, 부호 해석만 바뀜                 |
| 좁은 정수 → 넓은 정수                      | 부호 있으면 부호 확장, 없으면 0 확장               |
| 넓은 정수 → 좁은 정수                      | 절삭(하위 비트만)                                  |
| 정수 → 부동소수                            | IEEE 754 표현으로(정밀도를 잃을 수 있음)           |
| 부동소수 → 정수                            | 0 쪽으로 절삭. 범위 밖은 MIN/MAX로 포화, NaN → 0(모든 최적화 수준) |
| 부동소수 → 같은/다른 폭 부동소수           | IEEE 754 반올림                                    |
| `u8` → `char`                              | 같은 코드 포인트(U+0000..=U+00FF)                  |
| `char` → 정수                              | 코드 포인트(좁은 정수면 절삭)                      |

`char`로 바꾸는 `as`는 `u8`에서만 됩니다. 다른 정수, 부동소수, `bool`, `c_char`에서 바꾸면 타입 에러입니다. 더 넓은 값은 `char::from_u32(n)`(`char | none`)으로 바꿉니다([2.15.4](215-string.md)).

### `as`의 한계

`as`는 수 변환과 비트 변환에만 씁니다. 임의 타입 사이의 변환은 컴파일러가 거부합니다.

```rok
struct Vec3 { x: f64; y: f64; z: f64; }
let v = Vec3 { x: 1.0, y: 2.0, z: 3.0 };
let s: str = v as str;     // 컴파일 에러: `as` 는 user struct 거부
```

사용자 타입 사이의 변환은 `From` / `Into` 트레잇으로 밝혀야 합니다.

## 2.11.7. 리터럴 문맥의 자동 From: 컴파일 타임 범위 검사

타입 주석이 붙은 리터럴에는 컴파일 타임 범위 검사와 무손실 변환이 함께 적용됩니다.

```rok
let x: i32 = 42;             // OK: 42 가 i32 범위
let y: i32 = 9999999999;     // 컴파일 에러: literal 이 i32 범위 밖
let z: f32 = 3.14;           // OK: round to nearest f32
let w: u8 = 256;             // 컴파일 에러: 256 ∉ u8 0..=255
```

`T::from(1)` / `1.into()`처럼 리터럴로 구현을 고르는 규칙은 [refs-lang 3.1.3](../../refs-lang/003-expressions/301-operator-dispatch.md#impl-고르기)을 따릅니다. 리터럴은 컴파일 타임에 값을 알 수 있으므로 이렇게 할 수 있습니다. 런타임 값을 변환할 때는 `from`, `into`, `try_into`, `as` 중 하나를 적어야 합니다.

## 2.11.8. `From` impl 매트릭스: 원시 타입 사이

표준 라이브러리에 있는 `From<U> for T` 구현은 다음과 같습니다(무손실 변환만 표시).

### 정수 → 정수(signed)

| from \ to | i8 | i16 | i32 | i64           | isize         | int |
|-----------|----|-----|-----|---------------|---------------|-----|
| `i8`      |    | ✓   | ✓   | ✓             | ✓             | ✓   |
| `i16`     |    |     | ✓   | ✓             | ✓             | ✓   |
| `i32`     |    |     |     | ✓             | ✓ (target≥32) | ✓   |
| `i64`     |    |     |     |               | ✓ (target=64) | ✓   |
| `isize`   |    |     |     | ✓ (target≤64) |               | ✓   |

### 정수 → 정수(unsigned)

| from \ to | u8 | u16 | u32 | u64           | usize         | int (항상 무손실)     |
|-----------|----|-----|-----|---------------|---------------|-----------------------|
| `u8`      |    | ✓   | ✓   | ✓             | ✓             | ✓                     |
| `u16`     |    |     | ✓   | ✓             | ✓             | ✓                     |
| `u32`     |    |     |     | ✓             | ✓ (target≥32) | ✓                     |
| `u64`     |    |     |     |               | ✓ (target=64) | ✓                     |
| `usize`   |    |     |     | ✓ (target≤64) |               | ✓                     |

### unsigned → signed(무손실만)

| from \ to | i8 | i16 | i32 | i64 | int |
|-----------|----|-----|-----|-----|-----|
| `u8`      |    | ✓   | ✓   | ✓   | ✓   |
| `u16`     |    |     | ✓   | ✓   | ✓   |
| `u32`     |    |     |     | ✓   | ✓   |
| `u64`     |    |     |     |     | ✓   |

부호 없는 정수에서 폭이 같거나 더 좁은 부호 있는 정수로 가는 변환(예: u32 → i32)은 무손실을 보장하지 않습니다. 그래서 `try_into`나 `as`를 써야 합니다.

### 정수 → 부동소수

| from \ to | f32 | f64                     |
|-----------|-----|-------------------------|
| `i8`      | ✓   | ✓                       |
| `i16`     | ✓   | ✓                       |
| `i32`     |     | ✓                       |
| `i64`     |     | (`From` 없음: 정밀도 손실) |

f32의 가수는 24비트, f64의 가수는 53비트이므로 그 비트 폭 안에서만 무손실입니다. 부호 없는 정수에서 부동소수 타입으로 가는 `From` 구현은 없습니다. 이때는 `as`를 씁니다.

### 부동소수 사이

| from \ to | f32 | f64 | float |
|-----------|-----|-----|-------|
| `f32`     |     | ✓   | ✓     |
| `f64`     |     |     | ✓     |

f64 → f32는 좁히는 변환이므로 `try_into`, `saturating_into`, `as` 중 하나를 써야 합니다.

### int / float와 고정 폭 원시 타입

| from \ to | i32               | i64           | f64                 |
|-----------|-------------------|---------------|---------------------|
| `int`     | `try_into`만     | `try_into`만 | `into`(반올림, 예외) |
| `float`   | `try_into`(절삭)  | `try_into`    | `into`(예외)        |

`From<int> for f64`와 `From<float> for f64`는 2.11.2의 "From은 무손실" 규칙의 **문서화된 예외**입니다. `int`는 가장 가까운 f64로 반올림합니다. 지수가 f64 범위 밖으로 승격된 `float`는 f64로 나타낼 수 없어 정보를 잃습니다. 손실이 드러나야 하는 자리에서는 `to_f64()`나 `as f64`를 권장합니다.

`int → fixed`는 무손실을 보장하지 않습니다. int 값이 고정 폭 타입의 범위 밖일 수 있기 때문입니다. 그래서 `try_into`나 `saturating_into`를 써야 합니다.

반면 `fixed → int`는 항상 무손실입니다(`From<i32> for int` 등이 모두 구현되어 있습니다).

## 2.11.9. `From` impl: 사용자 타입

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

이때 포괄 구현 `Into`가 `c.into()`를 `Fahrenheit::from(c)`로 이어 줍니다. 그래서 사용자가 `Into`를 직접 작성하지 않아도 됩니다.

### `try_from`도 동일 패턴

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

모든 고정 폭 정수 타입은 자기 `FromStr` 위에 이런 식으로 `TryFrom<str>`을 구현합니다. 빈 텍스트, 잘못된 숫자, 범위 밖의 값은 `OverflowError { op: "try_from", ty }`를 던집니다. 부동소수 타입, `bool`, `char`에는 `FromStr`만 있습니다.

## 2.11.10. 산술 결과의 자동 변환: 없음

산술 결과의 auto-From([refs-lang 2.14](../../refs-lang/002-types/214-cast-and-conversion.md))은 **리터럴 문맥**에서만 적용됩니다. 값끼리 계산한 결과는 자동으로 변환되지 않습니다.

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

## 2.11.11. 다른 언어와 비교: 설계 근거

| 언어   | 변환 모델                                               | Rok과의 차이                                                                     |
|--------|---------------------------------------------------------|-----------------------------------------------------------------------------------|
| Rust   | `From`/`Into` 무손실, `as` 랩어라운드, `try_into` Result | Rok과 가장 비슷함. Rok은 `saturating_into`/`wrapping_into` 메서드를 더함 |
| Swift  | 모든 정수 변환이 기본으로 트랩, `.init(clamping:)`은 선택 | Rok은 트랩을 거부함: debug=panic + release=랩어라운드(고정 폭 산술), 변환은 명시 |
| Julia  | `Int32(big)`은 트랩, `convert` 함수 묶음                | Rok은 트레잇 메서드(`from`/`into`)로 통일                                        |
| Python | `int(x)`/`float(x)` 함수, 항상 무한 정밀도              | Rok의 `int` / `float`도 같은 생각, 고정 폭 원시 타입은 별개 세계               |
| C      | 암묵적 변환(`int` → `long` 자동)                        | Rok은 이를 거부해 소리 없는 타입 섞임을 막음                                     |

이 비교에서 나온 Rok의 주요 설계 결정은 다음과 같습니다.
- 소리 없이 정보를 잃는 변환을 거부합니다(Rust의 영향).
- 네 갈래(`into`, `try_into`, `saturating_into`, `as`) 중 하나를 적게 해서 의도를 드러냅니다.
- auto-From은 리터럴 문맥에서만 허용합니다. 런타임 값은 명시적으로 변환해야 합니다.
- `int` / `float`의 자동 승격과 고정 폭 원시 타입을 분리합니다(Python과 Smalltalk의 영향).

## 2.11.12. 모듈 매핑

| 트레잇              | lang slot            | 표준 라이브러리 위치 |
|---------------------|----------------------|----------------------|
| `From<U>`           | (없음, derive로 구현 가능) | 표준 라이브러리 |
| `Into`              | (포괄 구현)          | 표준 라이브러리      |
| `TryFrom<U>`        | (없음)               | 표준 라이브러리      |
| `TryInto`           | (포괄 구현)          | 표준 라이브러리      |
| `SaturatingFrom<U>` | (없음)               | 표준 라이브러리      |
| `SaturatingInto`    | (포괄 구현)          | 표준 라이브러리      |
| `WrappingFrom<U>`   | (없음)               | 표준 라이브러리      |
| `WrappingInto`      | (포괄 구현)          | 표준 라이브러리      |

원시 타입 사이의 무손실 변환 구현에는 모두 `#[intrinsic("rok.cast.FROM.TO")]`가 붙습니다.

```rok
impl From<i32> for i64 {
    #[intrinsic("rok.cast.i32.i64")]
    fn from(value: i32) -> i64 { ... }
}
```

`as T`의 수 캐스트도 같은 intrinsic을 씁니다. 무손실 분기와 랩어라운드 분기 중 어느 쪽인지는 컴파일러가 고릅니다.
