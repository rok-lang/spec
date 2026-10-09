# 6.1. 속성 문법

이 절에서는 속성 (attribute) 의 어휘 및 구문 형태와 부착 규칙을 정의합니다. 속성은 다음과 같은 문법을 따릅니다.

```
attribute       ::= outer_attr | inner_attr
outer_attr      ::= "#" "[" attr_body "]"
inner_attr      ::= "#" "!" "[" attr_body "]"
attr_body       ::= path attr_args?
attr_args       ::= "(" attr_arg ("," attr_arg)* ","? ")"
                  | "=" literal
attr_arg        ::= literal | identifier | attr_body | identifier "=" literal
```

## 6.1.1. 도입자

속성을 시작하는 도입자는 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md) 에서 정의된 두 토큰을 사용합니다.

### `#[...]` 외부 속성

외부 속성은 바로 뒤에 오는 항목에 부착됩니다. 예를 들어 다음과 같이 작성할 수 있습니다.

```rok
#[derive(Display)]
struct Point { x: f64; y: f64; }

#[inline(always)]
fn fast() { ... }
```

이처럼 `#[...]` 는 그 다음에 이어지는 항목에 부착되는 형태입니다.

### `#![...]` 내부 속성

내부 속성은 자신을 둘러싸는 항목 또는 모듈에 부착됩니다. 다음 예시에서 그 차이를 확인할 수 있습니다.

```rok
#![cfg(target_os = "linux")]      // 이 모듈 / 파일 전체에 적용

mod m {
    #![allow(unused)]              // 모듈 m 자체에 적용
    fn helper() { ... }
}
```

내부 속성은 모듈, 파일, 함수 본체의 첫 부분에 등장합니다. 외부 속성과 내부 속성의 이러한 구분은 [어휘 구조](../001-lexical/102-lexical-structure.md) 의 `///` (외부 문서 주석) 과 `//!` (내부 문서 주석) 의 차이와 동일한 패턴을 따릅니다.

## 6.1.2. 본체 형태

속성의 본체는 `#[name]`, `#[name(args)]`, `#[name = value]` 의 세 가지 형태를 가집니다.

### 단순 이름

가장 단순한 형태는 다음과 같이 이름만 사용하는 형태입니다.

```rok
#[test]
#[inline]
#[derive_attr(default)]
```

이름만 있는 형태는 부울 플래그 또는 마커로 사용됩니다.

### 인자 동반

다음과 같이 인자를 동반하는 형태도 자주 쓰입니다.

```rok
#[derive(Display, Clone, Eq)]
#[repr(C)]
#[link("m")]
#[inline(always)]
#[cfg(target_os = "linux")]
#[infix("+")]
```

이는 ` 이름(인자들)` 형태이며, 인자들은 콤마로 구분합니다. 각 인자는 식별자, 리터럴, 키-값 (`like = "+"`), 또는 다시 중첩된 속성형 (`derive(...)` 안의 `Display`) 가 될 수 있습니다.

### 키-값

마지막으로 키-값 형태가 있습니다.

```rok
#[doc = "이 함수는 ..."]
#[link_name = "external_symbol"]
```

이는 ` 이름 = 값` 형태이며, 값은 리터럴입니다. 인자 동반 형태 안의 키-값 표기와 동일한 모습이지만, 본체 자체가 단일 키-값으로 이루어진다는 점이 다릅니다.

## 6.1.3. 부착 가능한 항목

속성은 다음과 같은 항목들에 부착할 수 있습니다.

| 항목                        | 예                                      |
|-----------------------------|-----------------------------------------|
| 함수 / 메서드               | `#[inline] fn f() { ... }`              |
| `struct` / `enum` / `union` | `#[derive(Eq)] struct S { ... }`        |
| `trait` / `impl`            | `#[derive(Display)] trait T { ... }`    |
| `const` / `static`          | `#[deprecated] const X: i32 = 0;`       |
| 모듈                        | `#[cfg(test)] mod tests { ... }`        |
| 항목 안의 필드 / 변형       | `struct S { #[deprecated] x: i32; }`    |
| 트레잇 메서드 / 연관 항목   | `#[infix("+")] fn add(...);`            |
| `use` / `extern`            | `#[link("m")] extern "C" { ... }`       |
| 식 (특정 자리)              | `#[cfg(test)] my_test_call();` (제한적) |

각 위치에서 어떤 속성이 의미를 갖는지는 해당 속성의 정의에 달려 있습니다. 예컨대 `#[derive]` 는 타입에만, `#[inline]` 은 함수에만 적용되는 식으로 동작합니다.

## 6.1.4. 다중 부착

같은 항목에 여러 속성을 함께 부착할 수도 있습니다.

```rok
#[derive(Display, Clone)]
#[repr(C)]
#[deprecated]
struct OldPoint { x: f64; y: f64; }
```

각 `#[...]` 를 별도의 줄에 적으셔도 되고, 가독성에 따라 한 줄에 여러 개를 적으셔도 됩니다.

```rok
#[inline] #[cold] fn rarely_called() { ... }
```

## 6.1.5. 평가/효과 시점

속성의 효과 시점은 속성마다 다릅니다.

- 컴파일러 인식 속성 (예: `#[inline]`, `#[derive]`) 은 컴파일 타임에 컴파일러가 직접 처리합니다.
- 매직 속성 (예: `#[lang]`, `#[literal]`, `#[infix]`) 은 컴파일러가 언어 의미와 결합하여 처리합니다.
- 사용자 정의 속성 (메타 블록 출처) 은 템플릿 확장 패스에서 코드를 생성합니다.

이러한 모든 처리는 parse 직후의 메타 확장 단계에서 마무리됩니다. 자세한 흐름은 [템플릿 토큰과 확장](609-template-tokens.md) 에서 정의합니다.

## 6.1.6. 알려지지 않은 속성

다음과 같이 컴파일러가 인식하지 못하는 속성을 만나는 경우가 있을 수 있습니다.

```rok
#[mystery_attribute]
fn f() { ... }
```

이 경우 컴파일러는 메타 블록의 정의를 찾아봅니다. 그래도 정의를 찾을 수 없으면 컴파일 에러를 발생시킵니다 (또는 경고를 내며, 정확한 정책은 컴파일러 구현이 결정합니다). 이러한 정책은 오타 (typo) 가 조용히 무시되는 함정을 막아 줍니다.

## 6.1.7. 속성과 위생 (hygiene)

속성이 항목에 부착되어 코드 생성으로 이어질 때, 생성되는 코드 안의 식별자는 [6. 메타 시스템](README.md) 의 위생 규칙을 따릅니다. 따라서 사용자 코드의 이름 공간과 충돌하지 않습니다. 자세한 위생 규칙은 [템플릿 토큰과 확장](609-template-tokens.md) 에서 정의합니다.

## 6.1.8. 같은 토큰의 여러 매핑

다음과 같이 같은 메서드에 여러 속성을 부착할 수도 있습니다.

```rok
#[infix("≤")]
#[infix("<=")]
fn le(self, rhs: Self) -> bool;
```

이 경우 두 토큰 (`≤`, `<=`) 이 모두 같은 메서드로 디스패치됩니다. 이러한 사용 패턴은 같은 의미를 가진 유니코드와 ASCII 토큰을 묶는 데 자주 활용됩니다.

## 6.1.9. 속성과 가시성/일반 키워드의 상호작용

속성은 가시성 키워드 (`pub`, `pub(pkg)` 등) 와 함수 수식어 (`async`, `yield`) 등과 자유로운 순서로 결합할 수 있습니다.

```rok
#[inline]
pub fn f() { ... }

pub
#[inline]
fn g() { ... }                     // 같은 의미

pub async fn h() { ... }
#[inline] pub async fn h() { ... }
pub #[inline] async fn h() { ... }
```

관용적으로는 속성 → 가시성 → 수식어 순서 (`#[inline] pub async fn ...`) 를 사용하지만, 사양 자체가 순서를 강제하지는 않습니다.
