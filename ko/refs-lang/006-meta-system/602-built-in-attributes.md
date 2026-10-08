# 6.2. 내장 속성

컴파일러가 직접 처리하는 내장 속성을 정의한다. `#[cfg]` 는 [조건부 컴파일](603-cfg.md) 에서 다루고,
매직과 메타 블록은 [`#[lang(...)]` 매직](604-lang-magic.md) 이후 절에서 다룬다.

```
derive_attr     ::= "#" "[" "derive" "(" trait_name ("," trait_name)* ","? ")" "]"
trait_name      ::= identifier
repr_attr       ::= "#" "[" "repr" "(" repr_arg ("," repr_arg)* ")" "]"
repr_arg        ::= "C" | "transparent" | "packed" | "align" "(" int_lit ")"
                  | identifier                          // primitive int repr
inline_attr     ::= "#" "[" "inline" ("(" ("always" | "never") ")")? "]"
deprecated_attr ::= "#" "[" "deprecated" ("(" string_lit ")")? "]"
```

## 6.2.1. `#[derive(...)]` 자동 트레잇 구현

```nok
#[derive(Display, Clone, Eq, Ord, Hash, Debug)]
struct Point { x: f64; y: f64; }
```

- 인자에 트레잇 이름들을 콤마로 나열한다.
- 각 트레잇에 대해 그 타입의 적절한 자동 구현이 생성된다.

### 내장 derive 가능 트레잇

| 트레잇    | 의미                                   |
| --------- | -------------------------------------- |
| `Eq`      | 동등성 (==, !=)                        |
| `Ord`     | 순서 (<, >, <=, >=). `Eq` 도 함께 필요 |
| `Hash`    | 해시 가능                              |
| `Display` | 사용자 표시 (`f"{x}"` 의 기본 형식)    |
| `Debug`   | 디버그 표시 (`f"{x:?}"`)               |
| `Clone`   | 복사 가능 (`.clone()`)                 |

자동 구현의 동작은 표준 라이브러리에서 정의한다. 예를 들어 `Eq` 는 모든 필드의 동등성으로 비교하고,
`Hash` 는 모든 필드의 해시를 결합한다.

### 사용자 정의 derive 와의 결합

`#[derive(MyTrait)]` 처럼 사용자 정의 트레잇도 derive 할 수 있다. 단, `MyTrait` 가 메타 블록 (후속
사양) 에서 `#[impl_derive(MyTrait)]` 로 정의되어 있어야 한다.

내장 derive 와 사용자 derive 는 하나의 `#[derive(...)]` 안에 자유롭게 섞을 수 있다.

```nok
#[derive(Eq, Clone, MyCustomTrait)]
struct Foo { ... }
```

## 6.2.2. `#[repr(...)]` 메모리 표현 명시

```nok
#[repr(C)]
struct CCompatible { x: i32; y: i32; }

#[repr(transparent)]
struct Wrapped(InnerType);

#[repr(packed)]
struct Tight { a: u8; b: u32; }

#[repr(align(16))]
struct Aligned { ... }

#[repr(u32)]
enum Status { ... }
```

| 인자                        | 의미                                                                                                 |
| --------------------------- | ---------------------------------------------------------------------------------------------------- |
| `C`                         | C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 호환 레이아웃                  |
| `transparent`               | 단일 필드 newtype. 내부 타입과 완전히 같은 표현                                                      |
| `packed`                    | 정렬을 무시하고 빽빽이 묶음 ([FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 한정) |
| `align(N)`                  | 최소 정렬을 N 바이트로 강제                                                                          |
| `u8` ~ `u64` / `i8` ~ `i64` | `enum` 의 discriminant 타입 강제                                                                     |

`#[repr]` 가 없으면 [`struct`](../002-types/203-struct.md) / [`enum`](../002-types/204-enum.md) / [`union`](../002-types/205-union.md) 의 컴파일러 자유 레이아웃이 적용된다.

## 6.2.3. `#[link(...)]` 외부 라이브러리 명시

```nok
#[link("m")]
extern "C" {
    fn sin(x: f64) -> f64;
    fn cos(x: f64) -> f64;
}

#[link(name = "openssl", kind = "static")]
extern "C" { ... }
```

[`extern` 블록](../005-modules/509-extern-block.md) 의 외부 라이브러리 링크를 명시한다. `extern` 블록에 부착한다.

| 인자 형식                   | 의미                        |
| --------------------------- | --------------------------- |
| `"libname"`                 | 단순 라이브러리 이름        |
| `name = "..."`              | 라이브러리 이름 (key-value) |
| `kind = "static" / "dylib"` | 링크 종류                   |

## 6.2.4. `#[inline]` 인라이닝 힌트

```nok
#[inline]
fn small_helper(x: i32) -> i32 { x + 1 }

#[inline(always)]
fn very_hot() { ... }

#[inline(never)]
fn never_inline() { ... }
```

| 인자     | 의미                            |
| -------- | ------------------------------- |
| (없음)   | 인라이닝 권장 (컴파일러가 결정) |
| `always` | 인라이닝 강제                   |
| `never`  | 인라이닝 금지                   |

이 속성은 최적화 힌트이며, 의미에는 영향이 없다.

## 6.2.5. `#[test]` 테스트 함수 마커

```nok
#[test]
fn test_addition() {
    assert_eq(1i32 + 1i32, 2i32);
}
```

- 함수에 부착하면 그 함수가 테스트 함수임을 표시한다.
- 테스트 러너 (`nokc test` 등) 가 이 함수들을 자동으로 수집해 실행한다.
- 일반 빌드에서는 제외된다 (마치 `#[cfg(test)]` 가 같이 있는 것처럼).

## 6.2.6. `#[cold]` 드물게 호출되는 함수

```nok
#[cold]
fn handle_error() { ... }
```

- 컴파일러에게 드물게 호출됨을 알린다.
- 코드 생성기가 core 코드 경로 밖으로 배치하여 캐시 친화성을 높인다.

## 6.2.7. `#[deprecated]` 사용 자제 표시

```nok
#[deprecated]
fn old_api() { ... }

#[deprecated = "use new_api instead"]
fn old_api2() { ... }
```

- 부착된 항목이 사용되면 경고가 발생한다 (에러는 아님).
- 메시지 (`= "..."`) 는 경고에 포함된다.

## 6.2.8. `#[doc = "..."]` 문서 주석 형태

```nok
#[doc = "이 함수는 두 정수의 합을 반환한다."]
fn add(x: i32, y: i32) -> i32 { x + y }
```

[어휘 구조](../001-lexical/102-lexical-structure.md) 의 `///` 문서 주석은 어휘 단계에서 이 형태의 속성으로 변환된다.

```nok
/// 이 함수는 ...
fn f() { ... }

// 같은 의미
#[doc = "이 함수는 ..."]
fn f() { ... }
```

`#![doc]` (내부 형태) 도 `//!` 와 같은 효과를 낸다.

## 6.2.9. `#[allow(...)]` / `#[deny(...)]` / `#[warn(...)]` 린트 제어

```nok
#[allow(unused)]
fn _placeholder() { }

#[deny(deprecated)]
fn strict_zone() { ... }
```

- 린터의 진단 수준을 그 항목 범위 안에서 변경한다.
- `allow`: 진단을 무시한다.
- `warn`: 진단을 경고로 발생시킨다.
- `deny`: 진단을 에러로 발생시킨다.

자세한 린트 카테고리는 컴파일러 및 린터 사양에서 정의한다 (본 사양 범위 밖).

## 6.2.10. `#[non_exhaustive]` 외부에서의 완전성 검사 회피

```nok
#[non_exhaustive]
pub enum ApiError {
    NotFound,
    Unauthorized,
}
```

- `enum` 또는 `struct` 에 부착한다.
- 외부 패키지에서 이 타입을 매칭할 때, `match` 가 모든 변형을 다뤄도 catch-all (`_ => ...`) 을 추가로 요구한다.
- 라이브러리 작성자가 향후 변형을 SemVer 호환을 깨지 않고 추가할 수 있게 한다.

## 6.2.11. `#[must_use]` 결과 무시 경고

```nok
#[must_use]
fn calculate() -> i32 { ... }
```

- 함수의 반환값이 사용되지 않으면 경고가 발생한다.
- 결과를 의도적으로 무시하려면 [매개변수](../004-functions/402-parameters.md) 의 `_ = ...` 를 명시한다.

## 6.2.12. 정리

위 속성들은 컴파일러와 도구가 직접 인식하는 내장 속성이다. 메타 블록을 통한 사용자 정의는 필요 없다.

매직 속성 (`#[lang]`, `#[literal]` 등) 은 언어 의미 자체와 결합되며, 별도 문서에서 정의한다.
