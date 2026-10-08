# 6. 메타 시스템 개요

본 블록은 nok 의 메타 시스템 (속성 attribute, 매직 magic, 메타 블록 meta [block](../000-meta/007-glossary.md#block-블록)) 을
정의한다. 이 시스템은 [언어와 표준 라이브러리의 경계](../000-meta/004-language-vs-stdlib.md) 모델 (언어가 작은 코어를 제공하고
표준 라이브러리가 의미를 채우는 형태) 을 가능하게 하는 핵심 메커니즘이다.

## 목차

- [6.1. 속성 문법](601-attribute-syntax.md)
- [6.2. 내장 속성](602-built-in-attributes.md)
- [6.3. `#[cfg]` 조건부 컴파일](603-cfg.md)
- [6.4. `#[lang(...)]` 매직](604-lang-magic.md)
- [6.5. `#[literal(...)]` 매직](605-literal-magic.md)
- [6.6. `#[keyword(...)]` 매직](606-keyword-magic.md)
- [6.7. 연산자 매직](607-operator-magic.md)
- [6.8. 사용자 정의 메타 블록](608-meta-blocks.md)
- [6.9. 템플릿 토큰과 확장](609-template-tokens.md)

## 세 단계의 구조

```
attribute (속성)
  ├── 일반 속성 — 컴파일러 / 도구에 힌트 / 메타데이터 부착
  └── 매직 (magic) — 라이브러리가 언어 의미와 결합되는 통로

meta block (메타 블록) — 사용자 정의 derive / attribute 의 정의
  └── template token (@Self, @field, ...)
```

각 단계의 역할은 다음과 같다.

### 속성 (attribute)

```nok
#[derive(Display, Clone)]
struct Point { x: f64; y: f64; }

#[inline(always)]
fn fast() { ... }

#[cfg(target_os = "linux")]
fn linux_only() { ... }
```

- `#[...]` 또는 `#![...]` 형태로 항목에 부착한다.
- 컴파일러가 인식하는 내장 속성 (예: `#[derive]`, `#[repr]`, `#[cfg]`) 과 사용자 정의 속성이 같은 형태를 공유한다.
- 자세한 형태는 [속성 문법](601-attribute-syntax.md) 과 [내장 속성](602-built-in-attributes.md) 에서 정의한다.

### 매직 (magic)

언어와 라이브러리를 연결하는 특별한 속성.

```nok
#[lang(slot_name)]              // stdlib 항목 ↔ 컴파일러 슬롯 연결
#[literal("...")]                // 리터럴 문법 등록
#[keyword("name")]               // 새 키워드 등록
```

추가로 연산자 매직 패밀리.

```nok
#[infix("+", prec, assoc)]
#[prefix("-", prec)]
#[postfix("?", prec)]
```

이 매직들이 `1 + 2` 같은 언어 토큰을 라이브러리 트레잇 메서드 호출로 연결한다.
[`#[literal]` 매직](605-literal-magic.md) 부터 [연산자 매직](607-operator-magic.md) 까지에서 정의한다.

### 메타 블록 (meta block)

사용자가 자신만의 derive 또는 attribute 를 정의하는 코드 생성기다. Rust 의 proc macro 자리에 해당한다.

```nok
#[impl_derive(MyTrait)] pub @derive {
    impl MyTrait for @Self {
        @for_fields {
            // 필드 별 코드 생성
        }
    }
}
```

- `#[impl_derive]` / `#[impl_attribute]` / `#[impl_derive_attribute]` 세 종류의 메타 블록이 있다.
- 본체에 템플릿 토큰 (`@Self`, `@field`, `@for_fields` 등) 을 써서 코드 생성 규칙을 작성한다.
- proc macro 스타일의 토큰 스트림 조작은 도입하지 않는다. 템플릿 토큰만으로 충분한 표현력을 제공한다.

자세한 정의는 [메타 블록](608-meta-blocks.md) 에서 다룬다.

## 핵심 원칙

### 1. 토큰 스트림 조작 없음

Rust 의 `syn` / `quote` 같은 임의 토큰 조작 매크로는 nok 에 없다. 사용자 코드 생성은
템플릿 블록과 메타 토큰으로만 가능하다.

이 제약은 사용자 정의 derive 의 코드를 컴파일러가 이해할 수 있게 유지하기 위함이다. 임의
토큰 변환은 예측 불가성, 디버깅 어려움, 도구 표면 폭발이라는 트레이드오프를 강제한다.

### 2. 매직은 stdlib 가 사용

`#[lang]` / `#[literal]` / `#[keyword]` / 연산자 매직은 주로 표준 라이브러리가 사용한다.
사용자 코드에서 쓸 일은 거의 없다.

이 매직들이 표준 라이브러리가 언어와 결합되어 의미를 만드는 메커니즘이다. [언어와 표준 라이브러리의 경계](../000-meta/004-language-vs-stdlib.md) 가
메타 시스템을 통해 실현된다.

### 3. 매크로 위생 (hygiene)

메타 블록의 코드 생성은 위생 (hygienic) 이다. 생성된 코드 안의 식별자는 생성 컨텍스트의
이름 공간만 보고, 사용처의 이름 공간을 오염시키지 않는다.

이 원칙은 사용자 코드와 메타 블록 사이의 예측 불가능한 충돌을 차단한다.

### 4. 컴파일 타임만

모든 메타 처리는 컴파일 타임에 끝난다. 런타임에 새 derive 를 등록하거나 새 매직을
주입하는 형태는 없다.
