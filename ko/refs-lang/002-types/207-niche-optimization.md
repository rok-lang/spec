# 2.7. niche 최적화

niche는 어떤 타입의 유효한 값에서 절대 나오지 않는 비트 패턴입니다. 합 타입의 태그를 이런 비트 패턴에 겹쳐 저장하면 태그 자리를 따로 두지 않아도 됩니다. 그러면 메모리 오버헤드가 없는 합 타입이 됩니다.

이 최적화는 [합 타입](206-sum-types.md)의 메모리 표현을 컴파일러에 맡기므로 할 수 있습니다.

## 2.7.1. 동기

```rok
let p: Ptr<u8> | none      // 64-bit 포인터 + none
```

위 타입은 16바이트(포인터 8 + 태그 1 + 패딩 7)가 아니라 8바이트로 표현할 수 있어야 합니다. 포인터의 `null` 비트 패턴(0x0000_0000_0000_0000)은 유효한 포인터에서 나오지 않으므로, 그 자리를 `none`에 쓸 수 있기 때문입니다.

과학 코드와 데이터 코드에서는 이 절약이 메모리 효율의 중심입니다. 이 설계는 Julia의 isbits `union`에서 착안했습니다.

## 2.7.2. 자동 niche

`none`의 비트는 구현이 정합니다. 항 하나와 `none`만으로 된 합 `T | none`은 `T`에 niche가 있으면 크기가 `T`와 같고, `none`은 그 niche 값으로 표현합니다. niche 값은 모든 비트가 0인 형태로 저장합니다. niche 값이 0이 아니면 항은 niche 스칼라를 niche 값과 XOR한 형태로 저장합니다. 그 결과는 0이 되지 않습니다. 따라서 모든 niche 합에서 0으로 채운 저장소는 `none`으로 읽힙니다.

| 타입                                       | niche 값                              |
|--------------------------------------------|---------------------------------------|
| `bool`                                     | `2`                                   |
| `char` (컴파일러 원시 타입)                | surrogate `0xD800`                    |
| 필드 없는 `enum`, 페이로드에 GC 참조가 없는 `enum` | 쓰지 않는 첫 태그               |
| `extern "C" fn(..) -> R`                   | null                                  |
| `Ptr<T>` / `ConstPtr<T>` / `ManyPtr<T>`    | null (`#[niche(0)]`)                  |
| 필드 하나의 `struct` / 튜플                | 그 필드의 niche                       |
| 여러 필드의 `struct` / 튜플                | niche가 있는 첫 필드의 niche (`bool`과 부동소수 niche는 건너뜀) |

- 페이로드에 GC 참조를 담는 `enum`에는 niche가 없습니다.
- `bool`이나 부동소수 스칼라의 0이 아닌 niche(`bool`, 또는 `#[niche(NAN)]` 부동소수 래퍼)는 필드가 하나인 래퍼를 거칠 때만 전달됩니다. 필드가 여럿인 집합 타입은 그런 필드를 건너뜁니다. 그래서 `(i64, bool) | none`은 태그를 두고, `(i64, char) | none`은 `char`의 niche를 씁니다.
- niche 값이 0이면(null 포인터, `extern "C" fn`, `NonZero`) 항의 비트가 바뀌지 않습니다. 그래서 이런 `T | none`은 C ABI를 거쳐 nullable 값으로 그대로 전달됩니다([5.9.4](../005-modules/509-extern-block.md#null이-될-수-있는-c-함수-포인터-extern-c-fn---r--none)).
- `none` 외에 항이 둘 이상인 합은 태그를 둡니다.

```rok
Ptr<u8> | none           // 8 바이트, null 이 none
bool | none              // 1 바이트
```

## 2.7.3. 자동 niche 없음

정수(`i8` .. `i64`, `u8` .. `u64`, `usize`, `isize`)와 부동소수(`f32`, `f64`)는 모든 비트 패턴이 유효하므로 niche가 없습니다. niche가 필요하면 래퍼(2.7.6)나 `#[niche]`(2.7.4)를 씁니다.

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

- 인자로는 정수, `MIN` / `MAX`(`T::MIN`), `NAN` / `INFINITY` / `NEG_INFINITY`(`F::NAN`), 범위 `a..=b` / `..b` / `a..`, 그리고 이들 여러 개를 쓸 수 있습니다. 인자는 인스턴스화된 필드 타입을 기준으로 해석합니다.
- 적은 값은 그 타입의 유효한 값이 아니라는 약속입니다. 컴파일러는 이 약속을 검사하지 않습니다. 그 값이 실제로 나타나면 합 타입이 항을 잘못 판별합니다.

## 2.7.5. 부착 가능한 위치

`#[niche(..)]`는 필드가 하나인 `struct`(tuple newtype 또는 이름 있는 필드 하나)에 붙입니다. `enum`, `union`, 필드가 여럿인 `struct`에 붙이면 에러입니다.

## 2.7.6. 표준 라이브러리 래퍼

| 이름          | 제외하는 값                           | 정의                          |
|---------------|---------------------------------------|-------------------------------|
| `NonZero<T>`  | 0 (정수 T만)                          | [stdlib 2.2.2](../../refs-standard-libraries/200-core/202-non-zero.md#222-nonzerot-0-금지) |
| `NonNan<F>`   | NaN                                   | [stdlib 2.2.3](../../refs-standard-libraries/200-core/202-non-zero.md#223-nonnanf-nan-금지) |
| `Finite<F>`   | NaN, ±∞                               | [stdlib 2.2.4](../../refs-standard-libraries/200-core/202-non-zero.md#224-finitef-nan--금지) |
| `Positive<T>` | 음수 (0은 포함). unsigned `T`는 niche 없음 (`Positive<u32> \| none`은 8바이트) | [stdlib 2.2.5](../../refs-standard-libraries/200-core/202-non-zero.md#225-positivet-음수-금지) |
