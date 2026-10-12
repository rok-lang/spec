# 6.2. 내장 속성

이 절에서는 컴파일러가 직접 처리하는 내장 속성을 정의합니다. `#[cfg]` 는 [조건부 컴파일](603-cfg.md) 에서 따로 다루며, 매직 속성과 메타 블록에 관한 내용은 [`#[lang(...)]` 매직](604-lang-magic.md) 이후의 절에서 이어 설명합니다.

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

```rok
#[derive(Clone, PartialEq, Eq, Hash, Debug)]
struct Point { x: i64; y: i64; }
```

`#[derive(...)]` 의 인자에는 트레잇 이름들을 콤마로 구분하여 나열합니다. 그러면 나열된 각 트레잇에 대해 해당 타입에 어울리는 자동 구현이 생성됩니다.

### 내장 derive 가능 트레잇

`Clone`, `Copy`, `Debug`, `Default`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash` 만 derive 할 수 있습니다. 각 트레잇의 생성 규칙과 요구 조건은 [stdlib 2.13.11](../../refs-standard-libraries/200-core/213-marker.md#21311-derive-가능한-트레잇) 에서 정의합니다.

- `Display` 는 derive 할 수 없습니다. 표시 형식은 타입 작성자가 정합니다 ([stdlib 2.10](../../refs-standard-libraries/200-core/210-fmt.md)).
- `Eq` / `Ord` 를 상위 트레잇 (`PartialEq` / `PartialOrd`, `Eq`) 없이 derive 하면 컴파일 에러입니다. `Copy` 는 `Clone` 이 필요하고 `Drop` 과 함께 쓸 수 없습니다.
- `union` 에 `#[derive]` 를 붙이면 컴파일 에러입니다.

### 사용자 정의 derive 와의 결합

`#[derive(MyTrait)]` 처럼 사용자 정의 트레잇도 derive 할 수 있습니다. 다만 이렇게 사용하려면 `MyTrait` 가 `#[impl_derive(MyTrait)]` 메타 블록 ([6.8](608-meta-blocks.md)) 으로 정의되어 있어야 합니다. 위의 내장 derive 이름은 메타 블록이 다시 정의할 수 없습니다.

내장 derive 와 사용자 정의 derive 는 하나의 `#[derive(...)]` 안에 자유롭게 섞어 사용할 수 있습니다.

```rok
#[derive(PartialEq, Eq, Clone, MyCustomTrait)]
struct Foo { ... }
```

## 6.2.2. `#[repr(...)]` 메모리 표현 명시

```rok
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
|-----------------------------|------------------------------------------------------------------------------------------------------|
| `C`                         | C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 호환 레이아웃                  |
| `transparent`               | 단일 필드 newtype. 내부 타입과 완전히 같은 표현                                                      |
| `packed`                    | 정렬을 무시하고 빽빽이 묶음 ([FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 한정) |
| `align(N)`                  | 최소 정렬을 N 바이트로 강제                                                                          |
| `u8` ~ `u64` / `i8` ~ `i64` | `enum` 의 discriminant 타입 강제                                                                     |

`#[repr]` 가 명시되지 않은 경우에는 [`struct`](../002-types/203-struct.md) / [`enum`](../002-types/204-enum.md) / [`union`](../002-types/205-union.md) 에 대해 컴파일러가 자유롭게 결정한 레이아웃이 적용됩니다.

## 6.2.3. `#[link(...)]` 외부 라이브러리 명시

```rok
#[link("m")]
extern "C" {
    fn sin(x: f64) -> f64;
    fn cos(x: f64) -> f64;
}

#[link(name = "openssl", kind = "static")]
extern "C" { ... }
```

`#[link(...)]` 는 [`extern` 블록](../005-modules/509-extern-block.md) 이 연결될 외부 라이브러리를 명시하는 속성으로, `extern` 블록에 부착해 사용합니다.

| 인자 형식                   | 의미                        |
|-----------------------------|-----------------------------|
| `"libname"`                 | 단순 라이브러리 이름        |
| `name = "..."`              | 라이브러리 이름 (key-value) |
| `kind = "static" / "dylib"` | 링크 종류                   |

## 6.2.4. `#[inline]` 인라이닝 힌트

```rok
#[inline]
fn small_helper(x: i32) -> i32 { x + 1 }

#[inline(always)]
fn very_hot() { ... }

#[inline(never)]
fn never_inline() { ... }
```

| 인자     | 의미                            |
|----------|---------------------------------|
| (없음)   | 인라이닝 권장 (컴파일러가 결정) |
| `always` | 인라이닝 강제                   |
| `never`  | 인라이닝 금지                   |

`#[inline]` 은 어디까지나 최적화에 대한 힌트일 뿐이며, 프로그램의 의미 자체에는 영향을 주지 않습니다.

## 6.2.5. `#[test]` 테스트 함수 마커

```rok
#[test]
fn test_addition() {
    assert_eq(1i32 + 1i32, 2i32);
}
```

`#[test]` 를 함수에 부착하면 그 함수가 테스트 함수임을 표시하게 됩니다. 그러면 테스트 러너 (`rokc test` 등) 가 이러한 함수들을 자동으로 수집하여 실행해 줍니다. 일반 빌드에서는 마치 `#[cfg(test)]` 가 함께 붙어 있는 것처럼 제외되니 참고합니다.

## 6.2.6. `#[cold]` 드물게 호출되는 함수

```rok
#[cold]
fn handle_error() { ... }
```

`#[cold]` 는 해당 함수가 드물게 호출된다는 사실을 컴파일러에게 알려 주는 속성입니다. 이 정보를 받은 코드 생성기는 해당 함수를 핵심 코드 경로 바깥으로 배치하여 캐시 친화성을 한층 높여 줍니다.

## 6.2.7. `#[deprecated]` 사용 자제 표시

```rok
#[deprecated]
fn old_api() { ... }

#[deprecated = "use new_api instead"]
fn old_api2() { ... }
```

`#[deprecated]` 가 부착된 항목이 사용되면 경고가 발생하지만, 에러로 이어지지는 않습니다. `= "..."` 형태로 메시지를 함께 적어 두면 그 내용이 경고에 포함되어 표시됩니다.

## 6.2.8. `#[doc = "..."]` 문서 주석 형태

```rok
#[doc = "이 함수는 두 정수의 합을 반환한다."]
fn add(x: i32, y: i32) -> i32 { x + y }
```

[어휘 구조](../001-lexical/102-lexical-structure.md) 에서 다루는 `///` 문서 주석은 어휘 단계에서 이러한 형태의 속성으로 변환됩니다.

```rok
/// 이 함수는 ...
fn f() { ... }

// 같은 의미
#[doc = "이 함수는 ..."]
fn f() { ... }
```

내부 형태인 `#![doc]` 또한 `//!` 와 동일한 효과를 냅니다.

## 6.2.9. `#[allow(...)]` / `#[deny(...)]` / `#[warn(...)]` 린트 제어

```rok
#[allow(unused)]
fn _placeholder() { }

#[deny(deprecated)]
fn strict_zone() { ... }
```

이 속성들은 부착된 항목의 범위 안에서 린터의 진단 수준을 변경해 줍니다. `allow` 는 진단을 무시하고, `warn` 은 진단을 경고로 발생시키며, `deny` 는 진단을 에러로 발생시킵니다.

자세한 린트 카테고리는 컴파일러 및 린터 사양에서 정의하며, 본 사양의 범위에는 포함되지 않습니다.

## 6.2.10. `#[non_exhaustive]` 외부에서의 완전성 검사 회피

```rok
#[non_exhaustive]
pub enum ApiError {
    NotFound,
    Unauthorized,
}
```

`#[non_exhaustive]` 는 `enum` 또는 `struct` 에 부착할 수 있습니다. 이 속성이 붙어 있으면 외부 패키지에서 해당 타입을 매칭할 때, `match` 가 모든 변형을 다 처리하더라도 catch-all (`_ => ...`) 을 추가로 요구하게 됩니다. 이러한 제약 덕분에 라이브러리 작성자는 SemVer 호환을 깨뜨리지 않으면서도 향후 새로운 변형을 자유롭게 추가할 수 있습니다.

## 6.2.11. `#[must_use]` 결과 무시 경고

```rok
#[must_use]
fn calculate() -> i32 { ... }
```

`#[must_use]` 가 붙은 함수의 반환값이 사용되지 않은 채 버려지면 경고가 발생합니다. 결과를 의도적으로 무시하고자 할 때에는 [매개변수](../004-functions/402-parameters.md) 에서 설명한 `_ = ...` 를 명시하면 됩니다.

## 6.2.12. 정리

지금까지 살펴본 속성들은 컴파일러와 도구가 직접 인식하는 내장 속성이며, 메타 블록을 통한 사용자 정의는 따로 필요하지 않습니다.

`#[lang]`, `#[literal]` 같은 매직 속성은 언어의 의미 자체와 결합되어 있으므로, 별도의 문서에서 정의해 두었습니다.
