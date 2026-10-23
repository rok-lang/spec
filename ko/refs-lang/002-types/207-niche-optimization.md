# 2.7. niche 최적화

niche 란 어떤 타입의 유효 비트 패턴 중 절대 등장하지 않는 비트 패턴을 말합니다. 합 타입의 태그를 이러한 비트 패턴에 겹쳐 저장하면 태그 자리를 따로 둘 필요가 없어지며, 그 결과 메모리 오버헤드가 0 인 합 타입이 만들어집니다.

이 최적화는 [합 타입](206-sum-types.md) 의 메모리 표현이 컴파일러의 자유라는 사실 위에서 작동합니다.

## 2.7.1. 동기

```rok
let p: Ptr<u8> | none      // 64-bit 포인터 + none
```

위 타입은 16 바이트 (포인터 8 + 태그 1 + 패딩 7) 가 아니라 8 바이트로 표현될 수 있어야 합니다. 그 이유는 포인터의 `null` 비트 패턴 (0x0000_0000_0000_0000) 이 유효한 포인터에서는 등장하지 않으므로, 그 자리를 `none` 의 자리로 활용할 수 있기 때문입니다.

이러한 절약은 과학·데이터 코드의 핵심 메모리 효율로 이어집니다 (Julia 의 isbits `union` 에서 영감을 얻었습니다).

## 2.7.2. 자동 niche

`none` 의 비트는 구현이 정합니다. 항 하나와 `none` 만으로 된 합 `T | none` 은 `T` 에 niche 가 있으면 `T` 의 크기이며, `none` 은 그 niche 값으로 표현됩니다. 0 으로 채운 저장소는 `none` 으로 읽힙니다.

| 타입                                       | niche 값                              |
|--------------------------------------------|---------------------------------------|
| `bool`                                     | `2`                                   |
| `char` (컴파일러 primitive)                | surrogate `0xD800`                    |
| 필드 없는 `enum`, payload 에 GC 참조가 없는 `enum` | 쓰지 않는 첫 tag                |
| `extern "C" fn(..) -> R`                   | null                                  |
| `Ptr<T>` / `ConstPtr<T>` / `ManyPtr<T>`    | null (`#[niche(0)]`)                  |
| 여러 필드의 `struct` / 튜플                | niche 가 있는 첫 필드의 niche          |

- payload 에 GC 참조를 담는 `enum` 은 niche 가 없습니다.
- niche 값이 0 이면 (null 포인터, `extern "C" fn`, `NonZero`) 항의 비트는 그대로이므로, 그런 `T | none` 은 C ABI 를 그대로 nullable 값으로 건넙니다 ([5.9.4](../005-modules/509-extern-block.md#nullable-c-함수-포인터-extern-c-fn---r--none)).
- `none` 말고 항이 둘 이상인 합은 태그를 둡니다.

```rok
Ptr<u8> | none           // 8 바이트, null 이 none
bool | none              // 1 바이트
```

## 2.7.3. 자동 없음

정수 (`i8` .. `i64`, `u8` .. `u64`, `usize`, `isize`) 와 부동소수 (`f32`, `f64`) 는 모든 비트 패턴이 유효하므로 niche 가 없습니다. niche 가 필요하면 wrapper (2.7.6) 나 `#[niche]` (2.7.4) 를 씁니다.

## 2.7.4. opt-in niche: `#[niche(..)]`

```
niche_attr ::= "#" "[" "niche" "(" niche_arg ("," niche_arg)* ","? ")" "]"
niche_arg  ::= int_lit | "-" int_lit | const_path | range
range      ::= niche_val "..=" niche_val | ".." niche_val | niche_val ".."
const_path ::= "MIN" | "MAX" | "NAN" | "INFINITY" | "NEG_INFINITY" | type "::" const_name
```

```rok
#[niche(0)]
struct UserId(u64);

#[niche(i32::MIN..=-1)]
struct NonNegative(i32);

#[niche(NAN, INFINITY, NEG_INFINITY)]
struct Real { value: f64; }
```

- 인자는 정수, `MIN` / `MAX` (`T::MIN`), `NAN` / `INFINITY` / `NEG_INFINITY` (`F::NAN`), 범위 `a..=b` / `..b` / `a..`, 그리고 그 여럿입니다. 인스턴스화된 필드 타입에 대해 읽습니다.
- 적힌 값은 그 타입의 유효한 값이 아니라는 약속입니다. 컴파일러는 이를 검사하지 않으며, 그 값이 나타나면 합 타입의 판별이 틀어집니다.

## 2.7.5. 부착 가능한 위치

`#[niche(..)]` 는 필드가 하나인 `struct` (tuple newtype 또는 이름 있는 필드 하나) 에 붙입니다. `enum`, `union`, 필드가 여럿인 `struct` 에 붙이면 에러입니다.

## 2.7.6. 표준 라이브러리 wrapper

| 이름          | 제외하는 값                           | 정의                          |
|---------------|---------------------------------------|-------------------------------|
| `NonZero<T>`  | 0 (정수 T 만)                         | [stdlib 2.2.2](../../refs-standard-libraries/200-core/202-non-zero.md#222-nonzerot---0-금지) |
| `NonNan<F>`   | NaN                                   | [stdlib 2.2.3](../../refs-standard-libraries/200-core/202-non-zero.md#223-nonnanf---nan-금지) |
| `Finite<F>`   | NaN, ±∞                               | [stdlib 2.2.4](../../refs-standard-libraries/200-core/202-non-zero.md#224-finitef---nan--금지) |
| `Positive<T>` | 음수 (0 은 포함). unsigned `T` 는 niche 없음 (`Positive<u32> \| none` 은 8 바이트) | [stdlib 2.2.5](../../refs-standard-libraries/200-core/202-non-zero.md#225-positivet---음수-금지) |
