# 0.4. `#[lang]` 슬롯 매핑 표

`#[lang(slot_name)]` 속성([refs-lang 6.4](../../refs-lang/006-meta-system/604-lang-magic.md))은 표준 라이브러리 항목을 컴파일러가 찾을 수 있는 **슬롯**에 등록합니다. 이 장은 모든 슬롯의 이름과 의미, 붙는 항목을 표 하나에 모읍니다.

이 표가 슬롯에 관한 사양의 권위 있는 출처입니다. 컴파일러, 표준 라이브러리, 사용자 모두 이 표를 기준으로 맞춥니다. 슬롯 이름이 흔들리면 언어와 라이브러리 사이의 약속이 깨집니다. 그래서 슬롯을 바꿀 때는 항상 이 표부터 고칩니다.

## 0.4.1. 슬롯 분류

슬롯 48개는 맡은 책임에 따라 네 그룹으로 나뉩니다. 아래 표는 그룹마다 역할과 슬롯 수를 보여 줍니다. 이어지는 절의 세부 표가 어느 그룹에 속하는지도 이 표로 알 수 있습니다.

| 그룹   | 책임                                                                 | slot 수 |
|--------|----------------------------------------------------------------------|---------|
| 타입   | Rok이 **구체 타입**으로 인식하는 표준 라이브러리 타입               | 14      |
| 트레잇 | Rok이 **연산자 / 제어 흐름 풀어 쓰기**, derive, 스레드 경계 검사에 쓰는 트레잇 | 30      |
| 메서드 | Rok이 **자동으로 호출**하는 메서드 (drop)                           | 1       |
| 함수   | Rok이 **intrinsic처럼 호출**하는 표준 라이브러리 함수               | 3       |
| 합계   |                                                                      | **48**  |

## 0.4.2. 타입 슬롯(14)

타입 슬롯은 컴파일러가 리터럴이나 키워드를 만났을 때 낮출 구체 타입을 가리킵니다. 이 그룹은 다시 두 부류로 나뉩니다. 첫째는 소문자 식별자로 바로 드러나는 언어 기본 타입입니다. 이 타입은 키워드를 따로 등록하지 않아도 되고, 사용자가 매일 쓰는 원시 타입입니다.

다음은 언어 기본 타입(내장 원시 타입, `#[keyword]` 불필요)입니다.

| slot 이름    | 의미                         | 표준 라이브러리 항목    | 참조 |
|--------------|------------------------------|-------------------------|------|
| `i32_type`   | `i32` primitive              | 내장 (메서드: `rok::core::num`) | 2.1  |
| `i64_type`   | `i64` primitive              | 내장 (메서드: `rok::core::num`) | 2.1  |
| `f32_type`   | `f32` primitive              | 내장 (메서드: `rok::core::num`) | 2.1  |
| `f64_type`   | `f64` primitive              | 내장 (메서드: `rok::core::num`) | 2.1  |
| `usize_type` | `usize` primitive            | 내장 (메서드: `rok::core::num`) | 2.1  |
| `isize_type` | `isize` primitive            | 내장 (메서드: `rok::core::num`) | 2.1  |
| `bool_type`  | `bool` primitive             | 내장 (메서드: `rok::core::num`) | 2.1  |
| `cchar_type` | `c_char` primitive (FFI)     | 내장 (impl: `rok::core::num`) | 2.15 |
| `str_type`   | `str` primitive (UTF-8 view) | 내장 (메서드: `rok::core::str`) | 2.15 |
| `char_type`  | `char` primitive (Unicode scalar) | 내장 (메서드: `rok::core::str`) | 2.15 |

둘째는 언어 문법과 직접 얽힌 슬롯입니다. 자동으로 승격(auto-promote)하는 `int`와 `float`, `none` sentinel, `Range` 리터럴이 여기에 속합니다. `int`와 `float`은 `#[keyword(...)]`로 소문자 이름을 드러내는 PascalCase 라이브러리 struct입니다. `none`은 내장입니다. 라이브러리는 `pub type NoneSentinel = none`으로 그 타입에 `NoneSentinel`이라는 이름을 붙입니다.

다음은 문법과 얽힌 슬롯입니다.

| slot 이름    | 의미                      | 표준 라이브러리 항목                                      | 참조 |
|--------------|---------------------------|-----------------------------------------------------------|------|
| `none_type`  | `none` 리터럴의 타입      | 내장 (별칭: `rok::core::marker::NoneSentinel`) | 2.13 |
| `int_type`   | `int` auto-promote 타입   | `rok::core::num::bigint::Int` + `#[keyword("int")]` | 2.1  |
| `float_type` | `float` auto-promote 타입 | `rok::core::num::bigfloat::Float` + `#[keyword("float")]` | 2.1  |
| `range_type` | `Range<T>` (`a..b`)       | `rok::core::range::Range` + `#[literal( expr ".." expr )]`  | 2.7  |

기본 타입을 내장한 컴파일러는 `i32_type` ... `str_type`, `none_type`, `char_type`에 아무것도 붙이지 않아도 됩니다. 참조 컴파일러가 그렇습니다. 참조 컴파일러는 이 슬롯 이름을 받아들이지만 라이브러리는 여기에 어떤 항목도 붙이지 않습니다. `str`과 `c_char`는 라이브러리 struct가 아니고, `NoneSentinel`은 `#[lang]` 항목이 아니라 별칭입니다. 0.4.6에서 이 가운데 하나를 찾는 자리에서는 컴파일러가 내장 타입을 씁니다.

## 0.4.3. 트레잇 슬롯(30)

트레잇 슬롯은 컴파일러가 사용자가 쓴 문법을 표준 라이브러리 트레잇 호출로 풀어 쓸 때 찾아갈 트레잇을 가리킵니다. `for` 루프, 인덱싱, 슬라이싱, 포맷, `await` 같은 핵심 문법이 모두 여기 등록된 트레잇으로 풀립니다. 그래서 이 표는 언어 문법과 라이브러리를 잇는 가장 굵은 다리입니다.

| slot 이름             | 의미                       | 표준 라이브러리 트레잇         | 참조 |
|-----------------------|----------------------------|--------------------------------|------|
| `iterator_trait`      | `for x in xs`의 단계 진행 | `rok::core::iter::Iterator`     | 2.7  |
| `into_iterator_trait` | `for x in xs`의 시작       | `rok::core::iter::IntoIterator` | 2.7  |
| `index_trait`         | `arr[i]` 읽기              | `rok::core::index::Index` | 2.3  |
| `index_set_trait`     | `arr[i] = v` 쓰기          | `rok::core::index::IndexSet` | 2.3  |
| `slice_trait`         | `s[1:3]` 슬라이싱          | `rok::core::index::Sliceable` | 2.3  |
| `display_trait`       | `f"{x}"` 기본 포맷         | `rok::core::fmt::Display`       | 2.10 |
| `debug_trait`         | `f"{x:?}"` 디버그 포맷     | `rok::core::fmt::Debug`         | 2.10 |
| `future_trait`        | `await expr`의 단계 진행   | `rok::core::future::Future`     | 2.20 |

다음은 연산자 트레잇 슬롯입니다. 연산자 토큰을 어느 메서드로 디스패치할지는 각 트레잇 메서드에 붙은 `#[infix]` / `#[prefix]`([refs-lang 6.7](../../refs-lang/006-meta-system/607-operator-magic.md))가 정합니다. 우선순위는 [refs-lang 3.2](../../refs-lang/003-expressions/302-precedence-and-associativity.md) 표가 고정합니다. 아래 슬롯은 고정된 자리입니다. 컴파일러는 원시 타입 연산을 낮추거나 진단을 낼 때 이 자리로 "덧셈 트레잇", "비교 트레잇" 등을 이름으로 찾습니다. 복합 대입(`AddAssign` 등, 2.4), broadcast(`DotAdd` 등, 2.6), floored modulo(`Modulo`, `%%`) 트레잇은 `#[infix]` 등록만으로 충분하므로 슬롯이 없습니다.

| slot 이름             | 의미                                | 표준 라이브러리 트레잇                | 참조   |
|---------------------|-----------------------------------|-----------------------------|------|
| `add_trait`         | `a + b`                           | `rok::core::ops::Add`        | 2.3  |
| `sub_trait`         | `a - b`                           | `rok::core::ops::Sub`        | 2.3  |
| `mul_trait`         | `a * b`                           | `rok::core::ops::Mul`        | 2.3  |
| `div_trait`         | `a / b`                           | `rok::core::ops::Div`        | 2.3  |
| `rem_trait`         | `a % b`                           | `rok::core::ops::Rem`        | 2.3  |
| `neg_trait`         | `-a` (전위)                         | `rok::core::ops::Neg`        | 2.3  |
| `not_trait`         | `!a`                              | `rok::core::ops::Not`        | 2.3  |
| `bitnot_trait`      | `~a`                              | `rok::core::ops::BitNot`     | 2.3  |
| `bitand_trait`      | `a & b`                           | `rok::core::ops::BitAnd`     | 2.3  |
| `bitor_trait`       | `a \| b`                          | `rok::core::ops::BitOr`      | 2.3  |
| `bitxor_trait`      | `a ^ b`                           | `rok::core::ops::BitXor`     | 2.3  |
| `shl_trait`         | `a << b`                          | `rok::core::ops::Shl`        | 2.3  |
| `shr_trait`         | `a >> b`                          | `rok::core::ops::Shr`        | 2.3  |
| `logical_and_trait` | `a && b`                          | `rok::core::ops::LogicalAnd` | 2.3  |
| `logical_or_trait`  | `a \|\| b`                        | `rok::core::ops::LogicalOr`  | 2.3  |
| `eq_trait`          | `==` / `!=`, literal 패턴           | `rok::core::ops::PartialEq` | 2.5  |
| `cmp_trait`         | `<` / `>` / `<=` / `>=`, range 패턴 | `rok::core::ops::PartialOrd` | 2.5  |
| `ord_trait`         | 완전 순서 (`#[derive(Ord)]`)          | `rok::core::cmp::Ord`        | 2.5  |
| `hash_trait`        | 해시 (`#[derive(Hash)]`)            | `rok::core::cmp::Hash`       | 2.5  |
| `clone_trait`       | 복제 (`#[derive(Clone)]`)           | `rok::core::marker::Clone`   | 2.13 |
| `send_trait`        | 자동 marker: 스레드 이동 안전       | `rok::core::marker::Send`    | 2.13 |
| `sync_trait`        | 자동 marker: 스레드 공유 안전       | `rok::core::marker::Sync`    | 2.13 |

> [!NOTE]
> 비교 슬롯 이름과 트레잇 이름은 일대일로 대응하지 않습니다. `eq_trait`는 동등 연산자를 정의하는 `PartialEq`에, `cmp_trait`는 순서 비교 연산자를 정의하는 `PartialOrd`에 붙습니다. marker 트레잇 `Eq`, `Copy`, `Default`에는 슬롯이 없습니다. 합 타입이 이 트레잇을 구조적으로 얻기 때문입니다. `T`가 `Eq`이면 `T | none`도 `Eq`이고, 그 `Default`는 `none`입니다. 컴파일러는 이 트레잇을 `rok::core`의 고정 경로로 찾습니다.

## 0.4.4. 함수 슬롯(3)

함수 슬롯은 컴파일러가 intrinsic처럼 직접 호출하는 표준 라이브러리 함수를 가리킵니다. 주로 비정상 종료 경로(`throw`, OOM, 도달 불가)가 여기에 모입니다. 정상 흐름 밖에서 진입점을 하나로 모아야 할 때 씁니다.

| slot 이름        | 의미                             | 표준 라이브러리 함수            | 참조 |
|------------------|----------------------------------|---------------------------------|------|
| `panic_fn`       | panic이 일어나면 호출하는 abort 함수 | `rok::core::panic::panic`       | 2.14 |
| `oom_fn`         | OOM abort (런타임 할당 실패는 `rok_oom` 직접) | `rok::core::panic::oom`         | 2.14 |
| `unreachable_fn` | 컴파일러가 unreachable 자리 호출 | `rok::core::panic::unreachable` | 2.14 |

## 0.4.5. 메서드 슬롯(1)

메서드 슬롯은 사용자가 호출을 적지 않아도 컴파일러가 자동으로 부르는 메서드를 가리킵니다. 여기에는 `using` 바인딩을 해제할 때 부르는 폐기 메서드 하나뿐입니다. `?` 연산자는 컴파일러가 `T | none`에 맞게 직접 풀어 쓰므로 메서드 슬롯이 없습니다([refs-lang 3.14.10](../../refs-lang/003-expressions/314-error-control-flow.md#31410--연산자-none-조기-반환)). 값을 복제하는 `Clone::clone`은 자동으로 호출되지 않습니다([2.13.2](../200-core/213-marker.md#2132-clone-깊은-복제)). 그래서 메서드 슬롯이 아니라 트레잇 슬롯 `clone_trait`로 등록합니다.

| slot 이름     | 의미                    | 표준 라이브러리 메서드          | 참조 |
|---------------|-------------------------|---------------------------------|------|
| `drop_method` | `using` 블록 해제 시 호출 | `rok::core::marker::Drop::drop` | 2.13 |

## 0.4.6. 슬롯 사용처: 컴파일러가 슬롯을 찾는 위치

앞 절은 "어떤 슬롯이 있는가"를 나열했습니다. 이 절은 거꾸로 "컴파일러가 언제 어떤 슬롯을 찾는가"를 정리합니다. 사용자가 쓴 문법 한 줄이 표준 라이브러리의 어떤 항목으로 풀리는지 추적하려면 다음 표에서 출발하세요.

| 컴파일러 동작                                | lookup 슬롯                              |
|----------------------------------------------|------------------------------------------|
| `none` 리터럴 낮추기                         | `none_type`                              |
| `42` (기대 타입 없음) 리터럴 타입             | `i32_type`                               |
| `3.14` (기대 타입 없음) 리터럴 타입           | `f64_type`                               |
| `"hello"` 리터럴 타입                        | `str_type`                               |
| `for x in xs` 풀어 쓰기                      | `into_iterator_trait`, `iterator_trait`  |
| `arr[i]` 풀어 쓰기                           | `index_trait`                            |
| `arr[i] = v` 풀어 쓰기                       | `index_set_trait`                        |
| `s[1:3]` 풀어 쓰기                           | `slice_trait`                            |
| `f"{x}"` 기본 자리 풀어 쓰기                 | `display_trait`                          |
| `f"{x:?}"` 풀어 쓰기                         | `debug_trait`                            |
| `await expr` 풀어 쓰기                       | `future_trait`                           |
| 산술 / 비트 / 논리 연산자의 원시 타입 낮추기 | `add_trait` ... `logical_or_trait`         |
| `==` / `!=`, literal 패턴 매칭               | `eq_trait`                               |
| `<` / `>` / `<=` / `>=`, range 패턴 매칭     | `cmp_trait`                              |
| `#[derive(Clone / Ord / Hash)]` 확장         | `clone_trait`, `ord_trait`, `hash_trait` |
| panic 낮추기 (`try!` 실패 포함)              | `panic_fn`                               |
| OOM 발생 시                                  | `oom_fn`                                 |
| `match`의 모든 가지를 검사한 뒤 도달 불가 자리 | `unreachable_fn`                       |
| `using` 블록 해제                            | `drop_method`                            |
| `Send` / `Sync` 자동 구현, `+ Send` 검사     | `send_trait`, `sync_trait`               |

## 0.4.7. 중복 / 누락 검출

같은 슬롯에 표준 라이브러리 항목이 두 개 붙으면 컴파일러가 거부합니다(refs-lang 6.4.6의 중복 검출 규칙). 반대로 `iterator_trait` 같은 필수 슬롯이 비어 있으면 컴파일러는 그 슬롯에 의존하는 기능을 컴파일하지 않습니다. 여기서는 `for` 루프가 그런 기능입니다. 두 규칙 모두 슬롯이라는 약속에 진실의 원천이 하나만 있도록 강제합니다.

실제로 붙는 항목은 [200-core/](../200-core/)의 각 장에 적혀 있습니다. 이 표의 모든 슬롯은 "참조" 열이 가리키는 장에 `#[lang(...)]`을 붙이는 위치가 있습니다. 이 표에 없는 슬롯 이름은 [refs-lang 6.4.8](../../refs-lang/006-meta-system/604-lang-magic.md)에 따라 컴파일 에러입니다. 예를 들어 `Stream`에는 슬롯이 없습니다([2.8.10](../200-core/208-stream.md)).

## 0.4.8. 슬롯 이름 정책

마지막으로 새 슬롯을 추가하거나 기존 이름을 바꿀 때 따를 명명 규칙을 정리합니다. 이름이 일관되면 표를 훑기만 해도 슬롯의 종류를 짐작할 수 있습니다. 그래서 이 정책을 가볍게 다루지 않습니다.

이름은 snake_case로 씁니다. 타입 슬롯에는 `_type`, 트레잇 슬롯에는 `_trait`, 함수 슬롯에는 `_fn`, 메서드 슬롯에는 `_method` 접미사를 붙입니다(`drop_method`). 트레잇 전체를 가리키는 슬롯은 `_trait` 하나로 통일합니다. 컴파일러가 그 트레잇의 여러 메서드나 derive를 다루더라도 마찬가지입니다. 메서드 슬롯은 컴파일러가 **특정 메서드 하나를 자동으로 호출**하는 경우로 한정합니다. 이 기준에 따라 `Clone`은 `clone_method`가 아니라 `clone_trait`로 등록합니다. 접미사는 붙는 위치도 정합니다. `_trait` 슬롯은 **트레잇 선언 자체**에, `_method` 슬롯은 **트레잇 안의 해당 메서드**에 붙습니다. `_type` 슬롯은 타입 선언에, `_fn` 슬롯은 함수 선언에 붙습니다. 다른 위치에 붙이면 [refs-lang 6.4.7](../../refs-lang/006-meta-system/604-lang-magic.md)의 시그니처 검사에서 컴파일러가 거부합니다.
