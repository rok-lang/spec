# 6.1. 속성 문법

속성 (attribute) 의 어휘 및 구문 형태와 부착 규칙을 정의한다.

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

[구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md) 에서 정의된 두 토큰을 사용한다.

### `#[...]` 외부 속성

바로 뒤의 항목에 부착한다.

```nok
#[derive(Display)]
struct Point { x: f64; y: f64; }

#[inline(always)]
fn fast() { ... }
```

`#[...]` 는 그 다음 항목에 부착된다.

### `#![...]` 내부 속성

자신을 둘러싸는 항목 또는 모듈에 부착한다.

```nok
#![cfg(target_os = "linux")]      // 이 모듈 / 파일 전체에 적용

mod m {
    #![allow(unused)]              // 모듈 m 자체에 적용
    fn helper() { ... }
}
```

내부 속성은 모듈, 파일, 함수 본체의 첫 부분에 등장한다.

이 외부와 내부의 차이는 [어휘 구조](../001-lexical/102-lexical-structure.md) 의 `///` (외부 문서 주석) 과 `//!` (내부 문서 주석) 의 차이와 같은 패턴이다.

## 6.1.2. 본체 형태

`#[name]`, `#[name(args)]`, `#[name = value]` 의 세 가지 형태가 있다.

### 단순 이름

```nok
#[test]
#[inline]
#[derive_attr(default)]
```

이름만 있는 형태로, 부울 플래그 또는 마커로 쓴다.

### 인자 동반

```nok
#[derive(Display, Clone, Eq)]
#[repr(C)]
#[link("m")]
#[inline(always)]
#[cfg(target_os = "linux")]
#[infix("+", prec = 9, left)]
```

`이름(인자들)` 형태로, 인자는 콤마로 구분한다.

각 인자는 식별자, 리터럴, 키-값 (`prec = 9`), 또는 다시 중첩 속성형 (`derive(...)` 안의
`Display`) 이다.

### 키-값

```nok
#[doc = "이 함수는 ..."]
#[link_name = "external_symbol"]
```

`이름 = 값` 형태로, 값은 리터럴이다. 인자 동반의 키-값과 같은 표기지만, 본체 자체가 단일 키-값이다.

## 6.1.3. 부착 가능한 항목

속성은 다음 항목에 부착할 수 있다.

| 항목                        | 예                                      |
| --------------------------- | --------------------------------------- |
| 함수 / 메서드               | `#[inline] fn f() { ... }`              |
| `struct` / `enum` / `union` | `#[derive(Eq)] struct S { ... }`        |
| `trait` / `impl`            | `#[derive(Display)] trait T { ... }`    |
| `const` / `static`          | `#[deprecated] const X: i32 = 0;`       |
| 모듈                        | `#[cfg(test)] mod tests { ... }`        |
| 항목 안의 필드 / 변형       | `struct S { #[deprecated] x: i32; }`    |
| 트레잇 메서드 / 연관 항목   | `#[infix("+")] fn add(...);`            |
| `use` / `extern`            | `#[link("m")] extern "C" { ... }`       |
| 식 (특정 자리)              | `#[cfg(test)] my_test_call();` (제한적) |

각 위치에서 어떤 속성이 의미를 갖는지는 해당 속성의 정의에 달려 있다 (`#[derive]` 는 타입에만,
`#[inline]` 은 함수에만 적용되는 식).

## 6.1.4. 다중 부착

같은 항목에 여러 속성을 부착할 수 있다.

```nok
#[derive(Display, Clone)]
#[repr(C)]
#[deprecated]
struct OldPoint { x: f64; y: f64; }
```

각 `#[...]` 를 별도 줄에 적을 수도 있고, 한 줄에 여러 개를 적을 수도 있다 (가독성에 따라).

```nok
#[inline] #[cold] fn rarely_called() { ... }
```

## 6.1.5. 평가/효과 시점

속성의 효과 시점은 속성마다 다르다.

- 컴파일러 인식 속성 (예: `#[inline]`, `#[derive]`): 컴파일 타임에 컴파일러가 직접 처리한다.
- 매직 속성 (예: `#[lang]`, `#[literal]`, `#[infix]`): 컴파일러가 언어 의미와 결합한다.
- 사용자 정의 속성 (메타 블록 출처): 템플릿 확장 패스에서 코드를 생성한다.

이 모든 처리는 parse 직후의 메타 확장 단계에서 끝난다. 자세한 흐름은 [템플릿 토큰과 확장](609-template-tokens.md) 에서 정의한다.

## 6.1.6. 알려지지 않은 속성

```nok
#[mystery_attribute]
fn f() { ... }
```

컴파일러가 인식하지 못하는 속성은 메타 블록의 정의를 찾는다. 그래도 없으면 컴파일 에러를 낸다 (또는 경고. 정확한 정책은 컴파일러 구현이 결정한다).

이 정책은 오타 (typo) 가 조용히 무시되는 함정을 막는다.

## 6.1.7. 속성과 위생 (hygiene)

속성이 항목에 부착되어 코드 생성으로 이어질 때, 생성되는 코드 안의 식별자는 [6. 메타 시스템](README.md) 의 위생 규칙을 따른다. 사용자 코드의 이름 공간과 충돌하지 않는다.

자세한 위생 규칙은 [템플릿 토큰과 확장](609-template-tokens.md) 에서 정의한다.

## 6.1.8. 같은 토큰의 여러 매핑

```nok
#[infix("≤", prec = 8, none)]
#[infix("<=", prec = 8, none)]
fn le(self, rhs: Self) -> bool;
```

같은 메서드에 여러 속성을 부착하면 두 토큰 (`≤`, `<=`) 이 모두 같은 메서드로 디스패치된다.

이 사용 패턴은 같은 의미의 유니코드와 ASCII 토큰을 묶는 데 자주 쓰인다.

## 6.1.9. 속성과 가시성/일반 키워드의 상호작용

속성은 가시성 키워드 (`pub`, `pub(pkg)` 등), 함수 수식어 (`async`, `yield`) 등과 자유로운 순서로 결합한다.

```nok
#[inline]
pub fn f() { ... }

pub
#[inline]
fn g() { ... }                     // 같은 의미

pub async fn h() { ... }
#[inline] pub async fn h() { ... }
pub #[inline] async fn h() { ... }
```

관용적으로는 속성 → 가시성 → 수식어 순서 (`#[inline] pub async fn ...`) 를 쓰지만, 사양은 순서를 강제하지 않는다.
