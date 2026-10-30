# 2.2. niche wrapper - `NonZero` / `NonNan` / `Finite` / `Positive`

`rok::core::num::niche`는 값을 부분집합으로 제한하는 wrapper를 둡니다. 각 wrapper는 `#[niche(..)]` ([refs-lang 2.7.4](../../refs-lang/002-types/207-niche-optimization.md#274-opt-in-niche-niche))를 붙인 newtype이며, 제외된 값이 `T | none`의 niche가 됩니다.

## 2.2.1. 크기

niche가 있는 `T | none`은 `T`의 크기입니다 ([refs-lang 2.7.2](../../refs-lang/002-types/207-niche-optimization.md#272-자동-niche)). `none`은 그 niche 값으로 표현되며, 실제 비트는 구현이 정합니다.

| 타입                     | `T \| none` 크기 | `none`의 niche 값 |
|--------------------------|-----------------|--------------------|
| `i32 \| none`             | 8 B (태그)      | 없음               |
| `NonZero<i32> \| none`    | 4 B             | 0                  |
| `f64 \| none`             | 16 B (태그)     | 없음               |
| `NonNan<f64> \| none`     | 8 B             | NaN                |
| `Finite<f64> \| none`     | 8 B             | NaN, ±∞            |
| `Positive<i32> \| none`   | 4 B             | 음수               |
| `Positive<u32> \| none`   | 8 B (태그)      | 없음               |

niche 값이 0이면 (`NonZero`) 항의 비트가 그대로이므로, `NonZero<T> | none`은 0이 `none`인 정수로 C ABI를 건넙니다.

## 2.2.2. `NonZero<T>` - 0 금지

```rok
trait NicheInt { .. }            // 10 개의 정수 타입이 구현

#[niche(0)]
struct NonZero<T>(T);

impl<T: NicheInt> NonZero<T> {
    fn new(value: T) -> NonZero<T> | none;
    unsafe fn new_unchecked(value: T) -> NonZero<T>;   // value == 0 이면 UB
    fn get(self) -> T;
}
```

정수 타입만 받습니다. 부동소수는 `NonNan` / `Finite`를 씁니다.

| 연산                        | 결과                                              |
|-----------------------------|---------------------------------------------------|
| `NonZero<T> * NonZero<T>`   | `NonZero<T>`. 곱이 wrap되어 0이 되면 panic      |
| `NonZero<T> + - / NonZero<T>` | `T`                                             |
| `T / NonZero<T>`            | `T`. 0으로 나누기가 없음                         |

`TryFrom<T>`는 0이면 `ZeroError`를 던집니다. `ZeroError`는 이 모듈에 정의된 unit struct (`rok::core::num::niche::ZeroError`)이며 `Error`를 구현합니다 ([2.14.4](214-error.md#2144-내장-에러-타입)). `From<NonZero<T>> for T`는 lossless입니다.

```rok
let n: NonZero<i32> = NonZero::new(42) ?? panic("zero");
let nz: NonZero<i32> = try 7.try_into();
let v: i32 = nz.into();
```

## 2.2.3. `NonNan<F>` - NaN 금지

```rok
trait NicheFloat { .. }          // f32, f64 가 구현

#[niche(F::NAN)]
struct NonNan<F>(F);

impl<F: NicheFloat> NonNan<F> {
    fn new(value: F) -> NonNan<F> | none;
    unsafe fn new_unchecked(value: F) -> NonNan<F>;    // NaN 이면 UB
    fn get(self) -> F;
}
```

- `Eq`, `Ord`, `Hash`를 구현합니다. `-0.0 == 0.0`이며 해시도 같습니다.
- `+ - * /`의 결과는 `F`입니다 (∞ − ∞ 등이 NaN일 수 있음).
- `TryFrom<F>`는 NaN이면 `ConversionError`를 던집니다.

## 2.2.4. `Finite<F>` - NaN, ±∞ 금지

```rok
#[niche(F::NAN, F::INFINITY, F::NEG_INFINITY)]
struct Finite<F>(F);

impl<F: NicheFloat> Finite<F> {
    fn new(value: F) -> Finite<F> | none;
    unsafe fn new_unchecked(value: F) -> Finite<F>;    // NaN / ±∞ 이면 UB
    fn get(self) -> F;
}
```

- `Eq`, `Ord`, `Hash`를 구현합니다. `-0.0 == 0.0`입니다.
- `+ - *`는 `Result<Finite<F>, OverflowError>`입니다 (결과가 ±∞ 이면 `Err`).
- `TryFrom<F>`는 NaN / ±∞ 이면 `ConversionError`를 던집니다.

## 2.2.5. `Positive<T>` - 음수 금지

```rok
#[niche(T::MIN..=-1)]
struct Positive<T>(T);

impl<T: NicheInt> Positive<T> {
    fn new(value: T) -> Positive<T> | none;            // 음수면 none. 0 은 포함
    unsafe fn new_unchecked(value: T) -> Positive<T>;
    fn get(self) -> T;
}
```

0 이상의 값입니다. unsigned `T`에는 음수가 없으므로 niche가 없습니다.

## 2.2.6. `new_unchecked`

`new_unchecked`는 `unsafe fn`입니다. 불변을 어기면 정의되지 않은 동작이며, `T | none`이 그 값을 `none`으로 읽을 수 있습니다.

```rok
let n: i32 = compute();                                // 0 이 아님을 호출자가 보장
let nz: NonZero<i32> = unsafe { NonZero::new_unchecked(n) };
```

## 2.2.7. 검증

- const 자리에서 `NonZero::new(0)` 등은 컴파일 타임에 평가되어 `none`입니다. `?? panic(..)`이 붙으면 컴파일 에러입니다 ([refs-lang 5.2.4](../../refs-lang/005-modules/502-const.md#524-상수-식과-평가기)).
- 런타임에는 `new` (`T | none`)나 `try_from` (throw)으로 검증합니다.

## 2.2.8. 정의 요약

| 타입          | bound        | `#[niche]`                                 |
|---------------|--------------|--------------------------------------------|
| `NonZero<T>`  | `NicheInt`   | `0`                                        |
| `Positive<T>` | `NicheInt`   | `T::MIN..=-1`                              |
| `NonNan<F>`   | `NicheFloat` | `F::NAN`                                   |
| `Finite<F>`   | `NicheFloat` | `F::NAN, F::INFINITY, F::NEG_INFINITY`     |

## 2.2.9. 다른 언어와의 비교

| 측면         | Rust                          | Rok                                   |
|--------------|-------------------------------|---------------------------------------|
| `NonZero<T>` | `NonZero<T>` (정수)           | `NonZero<T: NicheInt>`                |
| `NonNan` / `Finite` | 없음 (외부 crate)      | 내장                                  |
| 자동 niche   | `Option<Box<T>>` = `Box<T>`   | `T \| none`                            |
| 사용자 niche | 불안정 attribute               | `#[niche(..)]`                         |
