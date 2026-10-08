# 2.2. niche wrapper — `NonZero` / `NonNan` / `Finite`

`rok::core::num::niche` 모듈은 값의 부분집합을 강제하는 wrapper 타입을 제공합니다. 이 wrapper 들은 컴파일 타임에 invariant 를 보장할 뿐만 아니라, sum type `T | none` 의 메모리 최적화에서도 핵심적인 역할을 담당합니다.

## 2.2.1. 동기 — wrapper 가 필요한 이유

[refs-lang 2.6 sum type](../../refs-lang/002-types/206-sum-types.md) 의 `T | none` 은 niche 가 존재할 경우 **tag 없이** 인코딩이 가능하며, 이를 통해 메모리를 절약하고 branch 도 제거할 수 있습니다.

| 타입                  | `T \| none` 크기                | tag                         |
|-----------------------|--------------------------------|-----------------------------|
| `i32 \| none`          | 8B (tag 1 + payload 4 + align) | 필요                        |
| `NonZero<i32> \| none` | **4B**                         | 불필요 (0 이 none)          |
| `f64 \| none`          | 16B (tag + payload + align)    | 필요                        |
| `NonNan<f64> \| none`  | **8B**                         | 불필요 (NaN 이 none)        |
| `Ptr<T> \| none`       | **8B**                         | 불필요 (null 이 none, 자동) |
| `bool \| none`         | **1B**                         | 불필요 (2 가 none, 자동)    |

자동 niche 가 부여되지 않는 **수치 primitive** (i32/i64/u*/f32/f64) 의 경우, 오직 wrapper 를 통해서만 niche 를 획득할 수 있습니다.

## 2.2.2. `NonZero<T>` — 0 금지

```rok
#[niche(0)]
struct NonZero<T: Primitive>(T);
```

이 타입은 T 의 0 값을 **제외**합니다. 따라서 `NonZero<i32> | none` 의 크기는 4B 이며, 이때 0 이 none 의 표현으로 사용됩니다.

### 핵심 메서드

```rok
impl<T: Primitive> NonZero<T> {
    fn new(value: T) -> NonZero<T> | none {
        if value == 0 { none } else { NonZero(value) }
    }

    fn new_unchecked(value: T) -> NonZero<T> {
        // SAFETY: 호출자가 value != 0 보장
        NonZero(value)
    }

    fn get(self) -> T;
}
```

### 사용 예

```rok
let n: NonZero<i32> = NonZero::new(42).unwrap();
let z: NonZero<i32> | none = NonZero::new(0);     //→ none

n.get();                  //→ 42

// 메모리 최적화 자동 적용
let m: NonZero<i32> | none = NonZero::new(7);     // 4B (tag 없음, 0 이 none)
```

### 산술

```rok
impl<T: Primitive> Mul<NonZero<T>> for NonZero<T> {
    type Output = NonZero<T>;
    fn mul(self, rhs: NonZero<T>) -> NonZero<T> {
        // SAFETY: NonZero × NonZero ≠ 0 (overflow 가 0 으로 wrap 되는 경우는 별도 처리)
        NonZero::new_unchecked(self.0 * rhs.0)
    }
}
```

`NonZero × NonZero` 은 다시 `NonZero` 가 되어 수학적 보존이 성립합니다. 반면 `NonZero + NonZero` 는 결과가 **0 이 될 수 있기** 때문에 `NonZero` 를 반환할 수 없으며, 따라서 `T` 를 반환하게 됩니다.

| 연산                      | 결과 타입            |
|---------------------------|----------------------|
| `NonZero<T> × NonZero<T>` | `NonZero<T>`         |
| `NonZero<T> + NonZero<T>` | `T` (0 가능)         |
| `NonZero<T> - NonZero<T>` | `T` (0 가능)         |
| `NonZero<T> / NonZero<T>` | `T` (0 / 0 = ?)      |
| `T / NonZero<T>`          | `T` (panic-free div) |

위 표의 마지막 행에서 보이듯이, `i32 / NonZero<i32>` 는 0-나눗셈 panic 이 발생하지 않는 안전한 div 연산입니다. 이러한 동작을 위해 `Div` 트레잇이 `NonZero` 인자를 받는 별도의 impl 을 노출하는 방식이 권장됩니다.

## 2.2.3. `NonNan<F>` — NaN 금지

```rok
#[niche(f64::NAN)]
struct NonNan<F: float>(F);     // F = f32 또는 f64
```

이 타입은 부동소수점에서 **NaN 을 제외**합니다. 그 결과 `NonNan<f64> | none` 의 크기는 8B 가 되며, NaN 이 none 의 표현으로 활용됩니다.

### 핵심 메서드

```rok
impl<F: float> NonNan<F> {
    fn new(value: F) -> NonNan<F> | none {
        if value.is_nan() { none } else { NonNan(value) }
    }

    fn new_unchecked(value: F) -> NonNan<F>;
    fn get(self) -> F;
}
```

### `Eq` / `Ord` 구현

NonNan 의 가장 큰 가치는 `Eq` 와 `Ord` 를 구현할 수 있다는 점입니다. NaN 이 빠지면 IEEE 754 가 total order 를 이루기 때문입니다.

```rok
impl<F: float> Eq for NonNan<F> { }
impl<F: float> Ord for NonNan<F> {
    fn cmp(self, rhs: NonNan<F>) -> Ordering {
        // NaN 없으니 partial_cmp 결과가 항상 some
        self.0.partial_cmp(rhs.0).unwrap()
    }
}
```

이로써 `NonNan<f64>` 는 `BTreeMap` 의 키로 사용될 수 있고 `sort()` 도 가능해집니다. 일반 `f64` 로는 이러한 동작이 불가능합니다.

### 산술

| 연산                    | 결과 타입                 |
|-------------------------|---------------------------|
| `NonNan<F> + NonNan<F>` | `F` (∞ + (-∞) = NaN 가능) |
| `NonNan<F> × NonNan<F>` | `F` (∞ × 0 = NaN)         |
| `NonNan<F> / NonNan<F>` | `F` (0/0 = NaN)           |

NonNan 산술도 결과가 NaN 이 되지 않는다는 보장을 제공하지 못하기에, 결과 타입은 `F` 가 됩니다. 이 점이 `Finite` 와의 차이입니다.

## 2.2.4. `Finite<F>` — NaN + ±∞ 금지

```rok
#[niche(f64::NAN, f64::INFINITY, f64::NEG_INFINITY)]
struct Finite<F: float>(F);
```

이 타입은 NaN 과 양/음 무한대를 모두 제외합니다. 따라서 `Finite<f64> | none` 의 크기는 8B 입니다.

### 산술 보장

`Finite + Finite` 의 결과는 **대체로 Finite** 가 됩니다. 다만 overflow 가 발생하여 ∞ 가 될 가능성이 있을 때에는 panic 이나 `Result` 를 통해 처리됩니다.

```rok
impl<F: float> Add for Finite<F> {
    type Output = Result<Finite<F>, OverflowError>;
    fn add(self, rhs: Finite<F>) -> Result<Finite<F>, OverflowError> {
        let r = self.0 + rhs.0;
        if r.is_finite() { Ok(Finite(r)) } else { Err(OverflowError { ... }) }
    }
}
```

용도 측면에서 보면, 이 타입은 과학 계산에서 수치적 안정성 보장이 필요한 자리에 적합합니다. 일반 산술에 사용하기에는 부담이 크기 때문에, `f64` 로 작업한 뒤 검증 시점에만 `Finite::try_from` 을 적용하는 방식이 권장됩니다.

## 2.2.5. niche 자동 적용 매트릭스

| 타입                         | 자동 niche                  | `T \| none` 크기       |
|------------------------------|-----------------------------|-----------------------|
| `i*`, `u*`, `f*` (raw)       | **없음**                    | tag + payload + align |
| `bool`                       | 2..=255                     | 1B                    |
| `char`                       | surrogate (0xD800..=0xDFFF) | 4B                    |
| `Ptr<T>` / `ConstPtr<T>`     | null (0)                    | 8B                    |
| `enum E { A, B, C }` (3 var) | 미사용 discriminant         | sizeof(E)             |
| `NonZero<T>`                 | 0 (사용자 선언)             | sizeof(T)             |
| `NonNan<F>`                  | NaN                         | sizeof(F)             |
| `Finite<F>`                  | NaN + ∞ (3 niche)           | sizeof(F)             |
| 사용자 `#[niche(...)]` 적용  | 명시 EXPR                   | sizeof (필드 분석)    |

## 2.2.6. 사용자 opt-in — `#[niche(...)]`

```rok
#[niche(0)]
struct UserId(u64);

#[niche(I32_MIN, -1, 0)]
struct ValidScore(i32);          // 음수 / -1 / 0 거부

#[niche(I32_MIN..=-1)]
struct Unsigned(i32);            // 0 이상만

#[niche(f64::NAN, f64::INFINITY, f64::NEG_INFINITY)]
struct Real(f64);                // NonNan + NonInf — Finite 와 동일 의미
```

### 적용 가능 자리

이 attribute 는 newtype struct, 즉 단일 tuple field 를 가진 `struct X(T);` 형태에 직접 부착할 수 있습니다. 또한 복수 필드를 가진 struct 의 경우 niche 필드를 포함하면 자동으로 전파되지만, struct 자체에는 부착할 수 없습니다. 한편 primitive 나 enum, union 에는 직접 부착할 수 없으며 newtype 으로 wrap 하는 과정이 필요합니다.

```rok
#[niche(0)]
struct Inner(u32);
struct Outer { id: Inner; name: str; }   // Inner 의 niche 가 Outer 로 전파
                                            // Outer | none 은 sizeof(Outer)
```

### EXPR 형식

| 형식             | 예                          |
|------------------|-----------------------------|
| const value      | `0`, `-1`, `I32_MIN`        |
| range expression | `I32_MIN..=-1`, `0..`       |
| 복수 (콤마 구분) | `0, -1, I32_MIN`            |
| 특수 const       | `f64::NAN`, `f64::INFINITY` |

## 2.2.7. niche 검증 — 컴파일러 + 런타임

### 컴파일 타임 (literal)

```rok
let n: NonZero<i32> = NonZero::new(0).unwrap();   // 컴파일 시점 — unwrap 후 panic
```

const 자리에서는 const evaluation 을 통해 **`NonZero::new(0)` 자체** 가 컴파일 에러로 잡힙니다.

### 런타임 (`new` / `try_from`)

```rok
let n = read_input();
let nz: NonZero<i32> | none = NonZero::new(n);    // 런타임 검증
match nz {
    v: NonZero<i32> => process(v),
    none => log("zero input rejected"),
}
```

### `*::new_unchecked` — 검증 우회 entry point

`NonZero::new_unchecked`, `NonNan::new_unchecked` 등은 invariant 검증을 생략하고 wrapper 를 직접 구성합니다. 호출자가 invariant (예: `NonZero` 의 경우 `value != 0`) 를 보장해야 합니다.

```rok
let n: i32 = compute();      // 호출자가 0 아님 보장
let nz: NonZero<i32> = NonZero::new_unchecked(n);
```

이 호출은 `unsafe` 마커 트레잇이 부여된 함수가 아니라 **호출자 책임의 검증 우회 메서드** 입니다. 잘못된 입력이 주어진 경우에는 **undefined behavior** 가 발생하며, niche 가 none 으로 인식되거나 그 반대 상황이 일어날 수 있습니다. `new_unchecked` 는 같은 모듈 안에서만 호출되도록 사용 빈도를 최소화하는 것이 권장됩니다.

## 2.2.8. From / Into 변환

[2.11 conversion](211-conversion.md) 의 `From` / `TryFrom` 트레잇을 통해 변환이 이루어집니다.

```rok
// T → wrapper (fallible)
impl<T: Primitive> TryFrom<T> for NonZero<T> {
    type Error = ZeroError;
    fn try_from(v: T) -> Result<NonZero<T>, ZeroError> {
        NonZero::new(v).ok_or(ZeroError)
    }
}

// wrapper → T (lossless)
impl<T: Primitive> From<NonZero<T>> for T {
    fn from(nz: NonZero<T>) -> T { nz.0 }
}
```

### 사용 예

```rok
let n: i32 = 42;
let nz: NonZero<i32> = n.try_into()?;      // Result<NonZero<i32>, ZeroError>

let nz: NonZero<i32> = NonZero::new(7).unwrap();
let v: i32 = nz.into();                    // lossless
```

## 2.2.9. niche 메커니즘 매핑

| 타입         | niche 메커니즘                          |
|--------------|-----------------------------------------|
| `NonZero<T>` | `#[niche(0)]` (T = i*, u*, f*)          |
| `NonNan<F>`  | `#[niche(F::NAN)]`                      |
| `Finite<F>`  | `#[niche(NAN, INFINITY, NEG_INFINITY)]` |

## 2.2.10. 다른 언어와의 비교

| 측면         | Rust                                            | Swift                  | rok                              |
|--------------|-------------------------------------------------|------------------------|----------------------------------|
| `NonZero<T>` | `std::num::NonZero{I,U}*`                       | **없음**               | `NonZero<T: Primitive>` (제네릭) |
| `NonNan<F>`  | **없음** (3rd party)                            | **없음**               | `NonNan<F: float>` 내장          |
| `Finite<F>`  | **없음**                                        | **없음**               | `Finite<F: float>` 내장          |
| 자동 niche   | `Option<Box<T>>` = `Box<T>`                     | `nil` optional pointer | `T \| none` 이 자동 적용          |
| 사용자 niche | `NonZero` 같은 lib 만                           | **없음**               | `#[niche(...)]` attribute        |
| 산술 보존    | 일부 (`NonZeroI32::checked_mul` 결과 `NonZero`) | **없음**               | `NonZero × NonZero = NonZero`    |

rok 의 가장 큰 차이점은 다음과 같이 정리됩니다. 첫째, 제네릭 wrapper 측면에서 `NonZero<T>` 는 모든 primitive 에 대해 동일한 type 으로 사용됩니다. 이는 Rust 의 `NonZeroI32` 와 `NonZeroU64` 와 같이 분리된 형태와 다릅니다. 둘째, `NonNan` 과 `Finite` 가 내장되어 있어 과학 계산에서 자주 필요한 보장을 표준에서 직접 제공합니다. 셋째, 사용자 `#[niche(...)]` 를 통해 라이브러리 외에도 newtype struct 의 niche 를 직접 선언할 수 있습니다.
