# 2.2. niche 래퍼: `NonZero` / `NonNan` / `Finite` / `Positive`

`rok::core::num::niche`에는 값을 부분집합으로 제한하는 래퍼(wrapper)가 있습니다. 각 래퍼는 `#[niche(..)]`([refs-lang 2.7.4](../../refs-lang/002-types/207-niche-optimization.md#274-opt-in-niche-niche))를 붙인 newtype입니다. 래퍼가 제외한 값은 `T | none`의 niche가 됩니다.

## 2.2.1. 크기

niche가 있는 `T | none`은 크기가 `T`와 같습니다([refs-lang 2.7.2](../../refs-lang/002-types/207-niche-optimization.md#272-자동-niche)). `none`은 그 niche 값으로 표현합니다. 실제 비트는 구현이 정합니다.

| 타입                     | `T \| none` 크기 | `none`의 niche 값 |
|--------------------------|-----------------|--------------------|
| `i32 \| none`             | 8 B (태그)      | 없음               |
| `NonZero<i32> \| none`    | 4 B             | 0                  |
| `f64 \| none`             | 16 B (태그)     | 없음               |
| `NonNan<f64> \| none`     | 8 B             | NaN                |
| `Finite<f64> \| none`     | 8 B             | NaN, ±∞            |
| `Positive<i32> \| none`   | 4 B             | 음수               |
| `Positive<u32> \| none`   | 8 B (태그)      | 없음               |

`NonZero`처럼 niche 값이 0이면 페이로드 비트가 바뀌지 않습니다. 그래서 `NonZero<T> | none`은 0이 `none`인 정수로 C ABI를 건넙니다.

## 2.2.2. `NonZero<T>`: 0 금지

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

정수 타입만 받습니다. 부동소수 타입에는 `NonNan` / `Finite`를 씁니다.

| 연산                        | 결과                                              |
|-----------------------------|---------------------------------------------------|
| `NonZero<T> * NonZero<T>`   | `NonZero<T>`. 곱이 넘쳐 0으로 돌아오면 panic이 일어남 |
| `NonZero<T> + - / NonZero<T>` | `T`                                             |
| `T / NonZero<T>`            | `T`. 0으로 나누기가 없음                         |

`TryFrom<T>`는 0이면 `ZeroError`를 던집니다. `From<NonZero<T>> for T`는 손실 없이 변환합니다.

```rok
let n: NonZero<i32> = NonZero::new(42) ?? panic("zero");
let nz: NonZero<i32> = try 7.try_into();
let v: i32 = nz.into();
```

## 2.2.3. `NonNan<F>`: NaN 금지

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

- `Eq`, `Ord`, `Hash`를 구현합니다. `-0.0 == 0.0`이고 해시도 같습니다.
- `+ - * /`의 결과는 `F`입니다. ∞ − ∞ 같은 결과는 NaN일 수 있습니다.
- `TryFrom<F>`는 NaN이면 `ConversionError`를 던집니다.

## 2.2.4. `Finite<F>`: NaN, ±∞ 금지

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
- `+ - *`의 결과는 `Result<Finite<F>, OverflowError>`입니다. 결과가 ±∞이면 `Err`입니다.
- `TryFrom<F>`는 NaN / ±∞이면 `ConversionError`를 던집니다.

## 2.2.5. `Positive<T>`: 음수 금지

```rok
#[niche(T::MIN..=-1)]
struct Positive<T>(T);

impl<T: NicheInt> Positive<T> {
    fn new(value: T) -> Positive<T> | none;            // 음수면 none. 0 은 포함
    unsafe fn new_unchecked(value: T) -> Positive<T>;
    fn get(self) -> T;
}
```

값은 0 이상입니다. 부호 없는 `T`에는 음수가 없으므로 niche가 없습니다.

## 2.2.6. `new_unchecked`

`new_unchecked`는 `unsafe fn`입니다. 불변식을 어기면 미정의 동작(UB)입니다. 이때 `T | none`이 그 값을 `none`으로 읽을 수 있습니다.

```rok
let n: i32 = compute();                                // 0 이 아님을 호출자가 보장
let nz: NonZero<i32> = unsafe { NonZero::new_unchecked(n) };
```

## 2.2.7. 검증

- const 자리에서 `NonZero::new(0)` 같은 식은 컴파일 타임에 `none`으로 평가됩니다. 여기에 `?? panic(..)`이 붙으면 컴파일 에러입니다([refs-lang 5.2.4](../../refs-lang/005-modules/502-const.md#524-상수-식과-평가기)).
- 실행 중에는 `new`(`T | none`)나 `try_from`(던짐)으로 검증합니다.

## 2.2.8. 정의 요약

| 타입          | 바운드       | `#[niche]`                                 |
|---------------|--------------|--------------------------------------------|
| `NonZero<T>`  | `NicheInt`   | `0`                                        |
| `Positive<T>` | `NicheInt`   | `T::MIN..=-1`                              |
| `NonNan<F>`   | `NicheFloat` | `F::NAN`                                   |
| `Finite<F>`   | `NicheFloat` | `F::NAN, F::INFINITY, F::NEG_INFINITY`     |

## 2.2.9. 다른 언어와 비교

| 측면         | Rust                          | Rok                                   |
|--------------|-------------------------------|---------------------------------------|
| `NonZero<T>` | `NonZero<T>` (정수)           | `NonZero<T: NicheInt>`                |
| `NonNan` / `Finite` | 없음 (외부 크레이트)   | 내장                                  |
| 자동 niche   | `Option<Box<T>>` = `Box<T>`   | `T \| none`                            |
| 사용자 niche | 불안정 속성                    | `#[niche(..)]`                         |
