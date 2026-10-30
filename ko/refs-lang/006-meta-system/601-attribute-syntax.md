# 6.1. 속성 문법

이 절은 속성(attribute)의 어휘 형태, 구문 형태, 부착 규칙을 정의합니다. 속성의 문법은 다음과 같습니다.

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

속성의 도입자로는 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)에서 정의한 두 토큰을 씁니다.

### `#[...]` 외부 속성

외부 속성은 바로 뒤에 오는 항목에 붙습니다.

```rok
#[derive(Debug)]
struct Point { x: f64; y: f64; }

#[inline(always)]
fn fast() { ... }
```

`#[...]`는 그다음에 오는 항목에 붙습니다.

### `#![...]` 내부 속성

내부 속성은 자신을 둘러싼 항목이나 모듈에 붙습니다.

```rok
#![cfg(target_os = "linux")]      // 이 모듈 / 파일 전체에 적용

mod m {
    #![allow(unused)]              // 모듈 m 자체에 적용
    fn helper() { ... }
}
```

내부 속성은 파일이나 `mod { .. }` 본체의 맨 앞에만 올 수 있습니다. 함수 본체 안에는 올 수 없으므로 함수에 외부 속성을 붙입니다. 외부 속성과 내부 속성의 구분은 [어휘 구조](../001-lexical/102-lexical-structure.md)에서 `///`(외부 문서 주석)와 `//!`(내부 문서 주석)를 구분하는 방식과 같습니다.

## 6.1.2. 본체 형태

속성 본체는 `#[name]`, `#[name(args)]`, `#[name = value]` 세 가지 형태가 있습니다.

### 단순 이름

가장 단순한 형태는 이름만 쓰는 것입니다.

```rok
#[test]
#[inline]
#[derive_attr(default)]
```

이름만 있는 형태는 부울 플래그나 마커로 씁니다.

### 인자 동반

인자를 함께 쓰는 형태도 자주 씁니다.

```rok
#[derive(Debug, Clone, PartialEq)]
#[repr(C)]
#[link("m")]
#[inline(always)]
#[cfg(target_os = "linux")]
#[infix("+")]
```

형태는 ` 이름(인자들)`이고 인자는 쉼표로 구분합니다. 각 인자는 식별자, 리터럴, 키-값(`like = "+"`), 중첩된 속성 형태(`derive(...)` 안의 `Display`) 중 하나일 수 있습니다.

### 키-값

마지막은 키-값 형태입니다.

```rok
#[doc = "이 함수는 ..."]
#[link_name = "external_symbol"]
```

형태는 ` 이름 = 값`이고 값은 리터럴입니다. 인자 동반 형태 안의 키-값 표기와 모양은 같습니다. 다른 점은 본체 전체가 키-값 하나라는 것입니다.

## 6.1.3. 부착 가능한 항목

속성은 다음 항목에 붙일 수 있습니다.

| 항목                        | 예                                      |
|-----------------------------|-----------------------------------------|
| 함수 / 메서드               | `#[inline] fn f() { ... }`              |
| `struct` / `enum` / `union` | `#[derive(PartialEq, Eq)] struct S { ... }` (`union`에 derive를 붙이면 에러) |
| `trait` / `impl`            | `#[deprecated] trait T { ... }`         |
| `const` / `static`          | `#[deprecated] const X: i32 = 0;`       |
| 모듈                        | `#[cfg(test)] mod tests { ... }`        |
| 항목 안의 필드 / 변형 (튜플 변형의 필드 포함) | `struct S { #[deprecated] x: i32; }`, `enum E { V(#[cfg(x)] i32) }` |
| 트레잇 메서드 / 연관 항목   | `#[infix("+")] fn add(...);`            |
| `use` / `extern`            | `#[link("m")] extern "C" { ... }`       |
| 문 (`#[cfg]`만)            | `#[cfg(test)] my_test_call();`          |
| match 가지 (`#[cfg]`만)     | `#[cfg(debug_assertions)] Op::Trace => trace(),` |
| struct 리터럴 필드 (`#[cfg]`만) | `S { #[cfg(x)] extra: 1, .. }`      |
| 함수 매개변수 (`#[cfg]`만) | `fn f(#[cfg(x)] extra: i32) { ... }`    |

각 위치에서 어떤 속성이 의미가 있는지는 그 속성의 정의가 정합니다. 예를 들어 `#[derive]`는 타입에만, `#[inline]`은 함수에만 적용됩니다.

## 6.1.4. 다중 부착

한 항목에 속성을 여러 개 붙일 수 있습니다.

```rok
#[derive(Debug, Clone)]
#[repr(C)]
#[deprecated]
struct OldPoint { x: f64; y: f64; }
```

각 `#[...]`를 줄마다 따로 써도 되고, 읽기 편하다면 한 줄에 여러 개를 써도 됩니다.

```rok
#[inline] #[cold] fn rarely_called() { ... }
```

## 6.1.5. 속성의 효력 시점

속성이 효과를 내는 시점은 속성마다 다릅니다.

- 컴파일러가 인식하는 속성(예: `#[inline]`, `#[derive]`)은 컴파일 타임에 컴파일러가 직접 처리합니다.
- 매직 속성(예: `#[lang]`, `#[literal]`, `#[infix]`)은 컴파일러가 언어 의미와 결합해 처리합니다.
- 사용자 정의 속성(메타 블록에서 정의한 것)은 템플릿 확장 패스에서 코드를 생성합니다.

이 처리는 모두 파싱 직후의 메타 확장 단계에서 끝납니다. 자세한 흐름은 [템플릿 토큰과 확장](609-template-tokens.md)에서 정의합니다.

## 6.1.6. 알려지지 않은 속성

컴파일러가 인식하지 못하는 속성을 만날 수 있습니다.

```rok
#[mystery_attribute]
fn f() { ... }
```

내장 속성이 아니고, 어떤 메타 블록도 정의하거나 검사하지 않는 속성은 컴파일 에러입니다. 검사에는 `@field.has_attr(x)`도 포함됩니다. 따라서 오타가 조용히 무시되는 일은 없습니다. 내장 속성은 컴파일러가 붙은 자리와 인자 형태를 검사합니다.

## 6.1.7. 속성과 위생(hygiene)

항목에 붙은 속성이 코드를 생성하면 생성된 코드 안의 식별자는 [6. 메타 시스템](README.md)의 위생 규칙을 따릅니다. 그래서 사용자 코드의 이름 공간과 충돌하지 않습니다. 자세한 위생 규칙은 [템플릿 토큰과 확장](609-template-tokens.md)에서 정의합니다.

## 6.1.8. 같은 토큰의 여러 매핑

한 메서드에 속성을 여러 개 붙일 수도 있습니다.

```rok
#[infix("≤")]
#[infix("<=")]
fn le(self, rhs: Self) -> bool;
```

이 경우 두 토큰(`≤`, `<=`)이 모두 같은 메서드를 부릅니다. 의미가 같은 유니코드 토큰과 ASCII 토큰을 묶을 때 자주 쓰는 패턴입니다.

## 6.1.9. 속성과 가시성/일반 키워드의 상호작용

속성은 가시성 키워드(`pub`, `pub(pkg)` 등), 함수 수식어(`async`, `yield` 등)와 순서에 상관없이 함께 쓸 수 있습니다.

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

관례상 속성, 가시성, 수식어 순서(`#[inline] pub async fn ...`)로 씁니다. 사양이 순서를 강제하지는 않습니다.
