# 0.4. `#[lang]` 슬롯 매핑 표

`#[lang(slot_name)]` 속성([refs-lang 6.4](../../refs-lang/006-meta-system/604-lang-magic.md)) 은 표준 라이브러리 항목을 컴파일러가 lookup 할 수 있는 **슬롯** 에 등록합니다. 이 장은 모든 슬롯 이름과 그 의미, 그리고 부착 대상 항목을 표 형태로 정리하여 한 자리에 모아 둡니다.

slot 표는 spec 의 권위 있는 자리이며, 컴파일러와 표준 라이브러리, 그리고 사용자가 모두 이 표를 참조하여 일관성을 유지합니다. 슬롯 이름이 흔들리면 언어와 라이브러리 사이의 약속이 무너지기 때문에, 변경은 본 표에서부터 시작합니다.

## 0.4.1. slot 분류

먼저 전체 48 개 슬롯이 어떤 책임을 지느냐에 따라 네 그룹으로 나뉩니다. 아래 표는 각 그룹의 역할과 슬롯 수를 한눈에 보여 주며, 이후 절들의 세부 표가 어떤 카테고리에 속하는지 가늠하게 합니다.

| 그룹   | 책임                                                                 | slot 수 |
|--------|----------------------------------------------------------------------|---------|
| 타입   | rok 이 **구체 타입** 으로 인식하는 표준 라이브러리 타입              | 14      |
| 트레잇 | rok 이 **operator / 제어 흐름 desugar**, derive, 스레드 경계 검사에 쓰는 트레잇 | 30      |
| 메서드 | rok 이 **자동 호출** 하는 메서드 (drop)                              | 1       |
| 함수   | rok 이 **intrinsic 호출** 로 부르는 표준 라이브러리 함수             | 3       |
| 합계   |                                                                      | **48**  |

## 0.4.2. 타입 슬롯 (14)

타입 슬롯은 컴파일러가 리터럴이나 키워드를 만났을 때 lower 시킬 구체 타입을 가리키는 자리입니다. 이 그룹은 다시 두 부류로 나뉘는데, 먼저 lowercase 식별자로 곧장 노출되는 언어 기본값 타입들이 있습니다. 이들은 키워드 등록이 따로 필요하지 않으며, 사용자가 매일 마주치는 primitive 들에 해당합니다.

언어 기본값 (lowercase struct, `#[keyword]` 불필요):

| slot 이름    | 의미                         | 표준 라이브러리 항목    | 참조 |
|--------------|------------------------------|-------------------------|------|
| `i32_type`   | `i32` primitive              | `rok::core::num::i32`    | 2.1  |
| `i64_type`   | `i64` primitive              | `rok::core::num::i64`    | 2.1  |
| `f32_type`   | `f32` primitive              | `rok::core::num::f32`    | 2.1  |
| `f64_type`   | `f64` primitive              | `rok::core::num::f64`    | 2.1  |
| `usize_type` | `usize` primitive            | `rok::core::num::usize`  | 2.1  |
| `isize_type` | `isize` primitive            | `rok::core::num::isize`  | 2.1  |
| `bool_type`  | `bool` primitive             | `rok::core::num::bool`   | 2.1  |
| `cchar_type` | `c_char` primitive (FFI)     | `rok::core::str::c_char` | 2.15 |
| `str_type`   | `str` primitive (UTF-8 view) | `rok::core::str::str`    | 2.15 |

다음으로 표준 라이브러리가 추가로 제공하는 PascalCase 타입들이 있습니다. 이쪽은 `#[keyword(...)]` 로 사용자에게 익숙한 소문자 별칭을 따로 노출하는데, `int` 와 `float` 의 auto-promote, `none` sentinel, `Range` 리터럴 같이 언어 문법과 직접 얽힌 항목들이 여기에 모입니다.

표준 라이브러리 추가 (PascalCase struct + `#[keyword(...)]`):

| slot 이름    | 의미                      | 표준 라이브러리 항목                                      | 참조 |
|--------------|---------------------------|-----------------------------------------------------------|------|
| `none_type`  | `none` 리터럴의 타입      | `rok::core::marker::NoneSentinel`                          | 2.13 |
| `int_type`   | `int` auto-promote 타입   | `rok::core::num::Int` + `#[keyword("int")]`                | 2.1  |
| `float_type` | `float` auto-promote 타입 | `rok::core::num::Float` + `#[keyword("float")]`            | 2.1  |
| `char_type`  | `char` Unicode scalar     | `rok::core::str::Char` + `#[keyword("char")]`              | 2.15 |
| `range_type` | `Range<T>` (`a..b`)       | `rok::core::iter::Range` + `#[literal( expr ".." expr )]`  | 2.7  |

## 0.4.3. 트레잇 슬롯 (30)

트레잇 슬롯은 사용자가 작성한 문법이 표준 라이브러리 트레잇 호출로 desugar 될 때 컴파일러가 어떤 트레잇을 찾아갈지 가리키는 자리입니다. `for` 루프나 인덱싱, 슬라이싱, 포맷, `await` 같은 핵심 문법이 모두 여기 등록된 트레잇을 통해 풀려나가므로, 이 표는 언어 문법과 라이브러리 사이의 가장 굵은 다리에 해당합니다.

| slot 이름             | 의미                       | 표준 라이브러리 트레잇         | 참조 |
|-----------------------|----------------------------|--------------------------------|------|
| `iterator_trait`      | `for x in xs` 의 단계 진행 | `rok::core::iter::Iterator`     | 2.7  |
| `into_iterator_trait` | `for x in xs` 의 시작      | `rok::core::iter::IntoIterator` | 2.7  |
| `index_trait`         | `arr[i]` 읽기              | `rok::core::ops::Index`         | 2.3  |
| `index_set_trait`     | `arr[i] = v` 쓰기          | `rok::core::ops::IndexSet`      | 2.3  |
| `slice_trait`         | `s[1:3]` 슬라이싱          | `rok::core::ops::Sliceable`     | 2.3  |
| `display_trait`       | `f"{x}"` 기본 포맷         | `rok::core::fmt::Display`       | 2.10 |
| `debug_trait`         | `f"{x:?}"` 디버그 포맷     | `rok::core::fmt::Debug`         | 2.10 |
| `future_trait`        | `await expr` 의 단계 진행  | `rok::core::future::Future`     | 2.20 |

다음으로 연산자 트레잇 슬롯이 있습니다. 연산자 토큰의 dispatch 대상은 각 트레잇 메서드에 부착된 `#[infix]` / `#[prefix]` ([refs-lang 6.7](../../refs-lang/006-meta-system/607-operator-magic.md)) 가 결정하고 우선순위는 [refs-lang 3.2](../../refs-lang/003-expressions/302-precedence-and-associativity.md) 표가 고정하며, 아래 슬롯은 컴파일러가 primitive lowering 과 진단에서 "덧셈 트레잇", "비교 트레잇" 을 이름으로 찾을 수 있게 하는 고정 자리입니다. 복합 대입 (`AddAssign` 등, 2.4), broadcast (`DotAdd` 등, 2.6), floored modulo (`Modulo`, `%%`) 트레잇은 `#[infix]` 등록만으로 충분하므로 슬롯을 두지 않습니다.

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
| `eq_trait`          | `==` / `!=`, literal 패턴           | `rok::core::cmp::PartialEq`  | 2.5  |
| `cmp_trait`         | `<` / `>` / `<=` / `>=`, range 패턴 | `rok::core::cmp::PartialOrd` | 2.5  |
| `ord_trait`         | 완전 순서 (`#[derive(Ord)]`)          | `rok::core::cmp::Ord`        | 2.5  |
| `hash_trait`        | 해시 (`#[derive(Hash)]`)            | `rok::core::cmp::Hash`       | 2.5  |
| `clone_trait`       | 복제 (`#[derive(Clone)]`)           | `rok::core::marker::Clone`   | 2.13 |
| `send_trait`        | 자동 marker — 스레드 이동 안전       | `rok::core::marker::Send`    | 2.13 |
| `sync_trait`        | 자동 marker — 스레드 공유 안전       | `rok::core::marker::Sync`    | 2.13 |

> [!NOTE]
> 비교 슬롯의 이름과 트레잇 이름은 일대일로 겹치지 않습니다. `eq_trait` 는 동등 연산자를 정의하는 `PartialEq` 에, `cmp_trait` 는 순서 비교 연산자를 정의하는 `PartialOrd` 에 부착됩니다. marker 트레잇인 `Eq` 는 메서드가 없고 컴파일러가 이름으로 찾을 일이 없으므로 슬롯을 두지 않습니다.

## 0.4.4. 함수 슬롯 (3)

함수 슬롯은 컴파일러가 intrinsic 처럼 직접 호출하는 표준 라이브러리 함수의 자리입니다. 주로 비정상 종료 경로 — `throw`, OOM, 도달 불가 — 가 여기 모이며, 정상 흐름이 아닌 곳에서 통일된 진입점이 필요할 때 이 슬롯들이 쓰입니다.

| slot 이름        | 의미                             | 표준 라이브러리 함수            | 참조 |
|------------------|----------------------------------|---------------------------------|------|
| `panic_fn`       | panic 시 호출하는 abort 함수     | `rok::core::panic::panic`       | 2.14 |
| `oom_fn`         | OOM abort (런타임 할당 실패는 `rok_oom` 직접) | `rok::core::panic::oom`         | 2.14 |
| `unreachable_fn` | 컴파일러가 unreachable 자리 호출 | `rok::core::panic::unreachable` | 2.14 |

## 0.4.5. 메서드 슬롯 (1)

메서드 슬롯은 사용자가 명시적으로 적지 않아도 컴파일러가 자동으로 부르는 메서드의 자리입니다. scope exit 시의 폐기 하나뿐입니다. `?` 연산자는 `T | none` 에 대해 컴파일러가 직접 풀어 쓰므로 ([refs-lang 3.14.10](../../refs-lang/003-expressions/314-error-control-flow.md#31410--연산자--none-조기-반환)) 메서드 슬롯을 두지 않습니다. 값의 복제 (`Clone::clone`) 는 자동으로 호출되지 않으므로 ([2.13.2](../200-core/213-marker.md#2132-clone--깊은-복제)) 메서드 슬롯이 아니라 트레잇 슬롯 `clone_trait` 로 등록됩니다.

| slot 이름     | 의미                    | 표준 라이브러리 메서드          | 참조 |
|---------------|-------------------------|---------------------------------|------|
| `drop_method` | `using` 블록 해제 시 호출 | `rok::core::marker::Drop::drop` | 2.13 |

## 0.4.6. 슬롯 사용 측, 컴파일러의 lookup 위치

앞 절들이 “어떤 슬롯이 존재하는가” 를 나열했다면, 이번 절은 거꾸로 “컴파일러가 언제 어떤 슬롯을 찾는가” 를 정리합니다. 사용자가 작성한 문법 한 줄이 표준 라이브러리의 어떤 항목으로 풀려나가는지 추적하고 싶을 때, 다음 표가 출발점이 됩니다.

| 컴파일러 동작                                | lookup 슬롯                              |
|----------------------------------------------|------------------------------------------|
| `none` 리터럴 lower                          | `none_type`                              |
| `42` (기대 타입 없음) literal type            | `i32_type`                               |
| `3.14` (기대 타입 없음) literal type          | `f64_type`                               |
| `"hello"` literal type                       | `str_type`                               |
| `for x in xs` desugar                        | `into_iterator_trait`, `iterator_trait`  |
| `arr[i]` desugar                             | `index_trait`                            |
| `arr[i] = v` desugar                         | `index_set_trait`                        |
| `s[1:3]` desugar                             | `slice_trait`                            |
| `f"{x}"` 기본 자리 desugar                   | `display_trait`                          |
| `f"{x:?}"` desugar                           | `debug_trait`                            |
| `await expr` desugar                         | `future_trait`                           |
| 산술 / 비트 / 논리 연산자 primitive lowering | `add_trait` … `logical_or_trait`         |
| `==` / `!=`, literal 패턴 매칭               | `eq_trait`                               |
| `<` / `>` / `<=` / `>=`, range 패턴 매칭     | `cmp_trait`                              |
| `#[derive(Clone / Ord / Hash)]` 확장         | `clone_trait`, `ord_trait`, `hash_trait` |
| panic (`try!` 실패 포함) lower               | `panic_fn`                               |
| OOM 발생 시                                  | `oom_fn`                                 |
| `match` 모든 arm exhaustive 체크 unreachable | `unreachable_fn`                         |
| `using` 블록 해제                            | `drop_method`                            |
| `Send` / `Sync` 자동 구현, `+ Send` 검사     | `send_trait`, `sync_trait`               |

## 0.4.7. 중복 / 누락 검출

같은 슬롯에 두 표준 라이브러리 항목이 부착되면 컴파일러가 이를 거부합니다(refs-lang 6.4.6 의 중복 검출 규칙). 반대로 필수 슬롯, 예컨대 `iterator_trait` 가 누락되면 그 슬롯에 의존하는 기능, 곧 `for` 루프의 컴파일이 거부됩니다. 양쪽 모두 슬롯이라는 약속이 단 하나의 진실을 갖도록 강제하는 장치입니다.

실제 부착 항목은 [200-core/](../200-core/) 의 각 장에 명시됩니다. 본 표의 모든 슬롯은 "참조" 열이 가리키는 장에 `#[lang(...)]` 부착 위치를 가집니다. 이 표에 없는 슬롯 이름은 [refs-lang 6.4.8](../../refs-lang/006-meta-system/604-lang-magic.md) 에 따라 컴파일 에러입니다 — 예컨대 `Stream` 은 슬롯을 두지 않습니다 ([2.8.10](../200-core/208-stream.md)).

## 0.4.8. 슬롯 이름 정책

마지막으로 새 슬롯을 추가하거나 기존 이름을 바꿀 때 따라야 할 명명 규칙을 정리해 둡니다. 일관된 이름은 표를 훑는 것만으로 슬롯의 종류를 짐작하게 해 주므로, 정책을 가볍게 보지 않습니다.

이름은 snake_case 로 적고, type 슬롯에는 `_type` 접미사를, trait 슬롯에는 `_trait` 접미사를 붙입니다. function 슬롯에는 `_fn` 접미사가 따라오며, method 슬롯에는 `_method` 접미사가 붙습니다 (`drop_method`). 트레잇 전체를 가리키는 슬롯은 컴파일러가 그 트레잇의 여러 메서드나 derive 를 다루더라도 `_trait` 하나로 통일하며, 메서드 슬롯은 컴파일러가 **특정 메서드 하나를 자동 호출** 하는 경우로 한정합니다. 이 기준에 따라 `Clone` 은 `clone_method` 가 아닌 `clone_trait` 로 등록됩니다. 접미사는 부착 위치도 결정합니다. `_trait` 슬롯은 **트레잇 선언 자체** 에, `_method` 슬롯은 **트레잇 안의 해당 메서드** 에, `_type` 슬롯은 타입 선언에, `_fn` 슬롯은 함수 선언에 부착하며, 다른 위치에 부착하면 [refs-lang 6.4.7](../../refs-lang/006-meta-system/604-lang-magic.md) 의 시그니처 검사에서 거부됩니다.
