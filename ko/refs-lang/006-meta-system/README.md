# 6. 메타 시스템 개요

본 블록에서는 rok 의 메타 시스템, 즉 속성 (attribute), 매직 (magic), 메타 [블록](../000-meta/007-glossary.md#block-블록) (meta block) 을 정의합니다. 이 시스템은 [언어와 표준 라이브러리의 경계](../000-meta/004-language-vs-standard-libraries.md) 모델을 가능하게 하는 핵심 메커니즘으로서, 언어가 작은 코어를 제공하고 표준 라이브러리가 그 의미를 채워 넣는 형태로 동작합니다.

## 목차

- [6.1. 속성 문법](601-attribute-syntax.md)
- [6.2. 내장 속성](602-built-in-attributes.md)
- [6.3. `#[cfg]` 조건부 컴파일](603-cfg.md)
- [6.4. `#[lang(...)]` 매직](604-lang-magic.md)
- [6.5. `#[literal(...)]` 매직](605-literal-magic.md)
- [6.6. `#[keyword(...)]` 매직](606-keyword-magic.md)
- [6.7. 연산자 매직](607-operator-magic.md)
- [6.8. 사용자 정의 메타 정의 (메타 블록과 매크로)](608-meta-blocks.md)
- [6.9. 템플릿 토큰과 확장](609-template-tokens.md)
- [6.10. 리플렉션](610-reflection.md)
- [6.11. 컴파일 시점 파일 포함](611-include.md)

## 세 단계의 구조

메타 시스템은 다음과 같이 세 단계로 구성됩니다.

```
attribute
  ├── plain attribute - hints / metadata for compiler / tools
  └── magic - channel where the library binds to language meaning

meta definition - user-defined code generator, one template engine (6.8)
  ├── derive / attribute / per-field  - triggered by #[..] on a type or item
  └── macro                           - triggered by a call, name!(..)
      └── template token (@Self, @field, ..) + matcher (typed fragments)
```

### 속성 (attribute)

속성은 다음과 같은 모습을 가집니다.

```rok
#[derive(Debug, Clone)]
struct Point { x: f64; y: f64; }

#[inline(always)]
fn fast() { ... }

#[cfg(target_os = "linux")]
fn linux_only() { ... }
```

속성은 `#[...]` 또는 `#![...]` 형태로 항목에 부착되며, 컴파일러가 인식하는 내장 속성 (예를 들어 `#[derive]`, `#[repr]`, `#[cfg]` 등) 과 사용자 정의 속성이 같은 형태를 공유합니다. 자세한 형태는 [속성 문법](601-attribute-syntax.md) 과 [내장 속성](602-built-in-attributes.md) 에서 정의합니다.

### 매직 (magic)

매직은 언어와 라이브러리를 연결하는 특별한 속성을 가리킵니다.

```rok
#[lang(slot_name)]              // 표준 라이브러리 항목 ↔ 컴파일러 슬롯 연결
#[literal("...")]                // 리터럴 문법 등록
#[keyword("name")]               // 새 키워드 등록
```

여기에 더해 연산자 매직 패밀리가 있습니다.

```rok
#[infix("+")]
#[prefix("-")]
#[infix("⊕", like = "+")]
```

이 매직들이 `1 + 2` 와 같은 언어 토큰을 라이브러리 트레잇 메서드 호출로 연결하는 역할을 담당합니다. 자세한 내용은 [`#[literal]` 매직](605-literal-magic.md) 부터 [연산자 매직](607-operator-magic.md) 까지에 걸쳐 정의됩니다.

### 메타 정의 (meta definition)

메타 정의는 코드 생성기입니다. 세 형태는 사용자가 자신만의 derive 또는 attribute 를 정의하게 하고 (Rust 의 derive·attribute proc macro 자리), 넷째인 [매크로](608-meta-blocks.md#6812-macro-형태) 는 호출 `name!(..)` 로 트리거되어 호출자가 쓴 것으로부터 코드를 생성합니다 (함수형 경우). 넷 다 하나의 템플릿 엔진을 공유합니다.

```rok
#[impl_derive(MyTrait)] pub @derive {
    impl MyTrait for @Self {
        @for_fields {
            // 필드 별 코드 생성
        }
    }
}
```

네 가지 형태가 모두 같은 방식 — `#[impl_<kind>(name)]`, 가시성, `@` 시작자, 블록 — 으로 쓰입니다. attribute 로 트리거되는 `#[impl_derive]`, `#[impl_attribute]`, `#[impl_derive_attribute]` 와, 호출로 트리거되는 `#[impl_macro(name)] @macro { .. }` (`name!(..)` 로 호출) 입니다. 본체에 `@Self`, `@field`, `@for_fields` 같은 템플릿 토큰으로 코드 생성 규칙을 작성하고, 매크로는 타입드 fragment 매처 (`@x: expr`) 를 더합니다. proc macro 스타일의 토큰 스트림 조작은 도입하지 않으며, 템플릿 토큰만으로도 충분한 표현력을 제공합니다. 자세한 정의는 [메타 정의](608-meta-blocks.md) 에서 다룹니다.

## 핵심 원칙

메타 시스템을 떠받치는 네 가지 핵심 원칙은 다음과 같습니다.

### 1. 토큰 스트림 조작 없음

Rust 의 `syn` 이나 `quote` 와 같은 절차적·토큰 조작 매크로는 rok 에 존재하지 않습니다. 모든 사용자 코드 생성 — attribute 로 트리거되는 메타 블록과 호출로 트리거되는 매크로 ([6.8.12](608-meta-blocks.md#6812-macro-형태)) 둘 다 — 은 오로지 패턴과 템플릿을 통해서만 이루어집니다. 매처가 입력의 일부를 바인딩하고 템플릿이 그것을 전사할 뿐, 어떤 단계도 임의 코드를 실행하거나 원시 토큰 스트림을 고쳐 쓰지 않습니다.

이러한 제약을 두는 이유는, 생성된 코드를 컴파일러·포매터·언어 서버가 따라갈 수 있는 상태로 유지하기 위함입니다. 임의의 토큰 변환을 허용하게 되면 예측 불가성, 디버깅의 어려움, 도구 표면의 폭발이라는 트레이드오프가 강제로 따라붙게 됩니다. 매크로는 이 규칙 안에 머물면서 호출부 코드 생성 (나머지 형태가 할 수 없는 것) 을 더합니다.

### 2. 매직은 표준 라이브러리가 사용

`#[lang]`, `#[literal]`, `#[keyword]`, 그리고 연산자 매직은 주로 표준 라이브러리에서 사용하게 되며, 사용자 코드에서 직접 쓸 일은 거의 없습니다.

이 매직들이야말로 표준 라이브러리가 언어와 결합되어 의미를 만들어 내는 메커니즘이며, [언어와 표준 라이브러리의 경계](../000-meta/004-language-vs-standard-libraries.md) 가 바로 이 메타 시스템을 통해 실현됩니다.

### 3. 매크로 위생 (hygiene)

메타 블록에서 이루어지는 코드 생성은 위생적 (hygienic) 입니다. 생성된 코드 안에 등장하는 식별자는 생성 컨텍스트의 이름 공간만을 바라보며, 사용처의 이름 공간을 오염시키지 않게 됩니다.

이 원칙을 통해 사용자 코드와 메타 블록 사이에서 발생할 수 있는 예측 불가능한 충돌을 차단할 수 있습니다.

### 4. 컴파일 타임만

모든 메타 처리는 컴파일 타임에 끝마치게 됩니다. 런타임에 새로운 derive 를 등록하거나 새로운 매직을 주입하는 형태는 지원하지 않습니다.
