# 6. 메타 시스템 개요

이 장은 Rok의 메타 시스템인 속성(attribute), 매직(magic), 메타 [블록](../000-meta/007-glossary.md#block-블록)(meta block)을 정의합니다. 메타 시스템은 [언어와 표준 라이브러리의 경계](../000-meta/004-language-vs-standard-libraries.md) 모델을 떠받치는 중심 메커니즘입니다. 언어는 작은 코어만 두고, 그 의미는 표준 라이브러리가 채웁니다.

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

메타 시스템은 세 단계로 이루어집니다.

```
attribute
  ├── plain attribute: hints / metadata for compiler / tools
  └── magic: channel where the library binds to language meaning

meta definition: user-defined code generator, one template engine (6.8)
  ├── derive / attribute / per-field  (triggered by #[..] on a type or item)
  └── macro                           (triggered by a call, name!(..))
      └── template token (@Self, @field, ..) + matcher (typed fragments)
```

### 속성(attribute)

속성은 다음과 같이 씁니다.

```rok
#[derive(Debug, Clone)]
struct Point { x: f64; y: f64; }

#[inline(always)]
fn fast() { ... }

#[cfg(target_os = "linux")]
fn linux_only() { ... }
```

속성은 `#[...]` 또는 `#![...]` 형태로 항목에 붙습니다. 컴파일러가 인식하는 내장 속성(`#[derive]`, `#[repr]`, `#[cfg]` 등)과 사용자 정의 속성은 형태가 같습니다. 자세한 형태는 [속성 문법](601-attribute-syntax.md)과 [내장 속성](602-built-in-attributes.md)이 정의합니다.

### 매직(magic)

매직은 언어와 라이브러리를 잇는 특별한 속성입니다.

```rok
#[lang(slot_name)]              // 표준 라이브러리 항목 ↔ 컴파일러 슬롯 연결
#[literal("...")]                // 리터럴 문법 등록
#[keyword("name")]               // 새 키워드 등록
```

여기에 연산자 매직 패밀리가 더해집니다.

```rok
#[infix("+")]
#[prefix("-")]
#[infix("⊕", like = "+")]
```

이 매직은 `1 + 2` 같은 언어 토큰을 라이브러리 트레잇 메서드 호출로 연결합니다. 자세한 정의는 [`#[literal]` 매직](605-literal-magic.md)부터 [연산자 매직](607-operator-magic.md)까지에 있습니다.

### 메타 정의(meta definition)

메타 정의는 코드 생성기입니다. 세 형태로는 사용자가 derive나 속성을 직접 정의합니다. Rust의 derive 매크로와 attribute 매크로(proc macro)를 대신하는 형태입니다. 넷째 형태인 [매크로](608-meta-blocks.md#6812-macro-형태)는 호출 `name!(..)`로 트리거되고, 호출하는 쪽이 쓴 내용으로 코드를 생성합니다. 함수형 매크로에 해당합니다. 네 형태는 템플릿 엔진 하나를 함께 씁니다.

```rok
#[impl_derive(MyTrait)] pub @derive {
    impl MyTrait for @Self {
        @for_fields {
            // 필드 별 코드 생성
        }
    }
}
```

네 형태는 모두 같은 모양(`#[impl_<kind>(name)]`, 가시성, `@` 시작자, 블록)으로 씁니다. 속성으로 트리거되는 형태는 `#[impl_derive]`, `#[impl_attribute]`, `#[impl_derive_attribute]`입니다. 호출로 트리거되는 형태는 `#[impl_macro(name)] @macro { .. }`이고 `name!(..)`로 호출합니다. 본체에는 `@Self`, `@field`, `@for_fields` 같은 템플릿 토큰으로 코드 생성 규칙을 씁니다. 매크로는 여기에 타입이 있는 조각의 매처(`@x: expr`)를 더합니다. proc macro 방식의 토큰 스트림 조작은 없습니다. 템플릿 토큰만으로 표현력이 충분합니다. 자세한 정의는 [메타 정의](608-meta-blocks.md)에 있습니다.

## 핵심 원칙

메타 시스템은 다음 네 원칙을 따릅니다.

### 1. 토큰 스트림 조작 없음

Rust의 `syn`이나 `quote`처럼 토큰을 조작하는 절차적 매크로는 Rok에 없습니다. 사용자 코드 생성은 패턴과 템플릿으로만 합니다. 속성으로 트리거되는 메타 블록도, 호출로 트리거되는 매크로([6.8.12](608-meta-blocks.md#6812-macro-형태))도 마찬가지입니다. 매처가 입력의 일부를 바인딩하고 템플릿이 그것을 옮겨 쓸 뿐입니다. 어느 단계도 임의 코드를 실행하거나 원시 토큰 스트림을 고쳐 쓰지 않습니다.

이렇게 제한하는 이유는 컴파일러, 포매터, 언어 서버가 생성된 코드를 따라갈 수 있어야 하기 때문입니다. 임의의 토큰 변환을 허용하면 대가가 따릅니다. 결과를 예측하기 어렵고, 디버깅이 힘들고, 도구가 다룰 범위가 폭발적으로 늘어납니다. 매크로는 이 규칙을 지키면서 호출 지점에서 코드를 생성합니다. 다른 형태로는 할 수 없는 일입니다.

### 2. 매직은 표준 라이브러리가 사용

`#[lang]`, `#[literal]`, `#[keyword]`, 연산자 매직은 주로 표준 라이브러리가 씁니다. 사용자 코드에서 직접 쓸 일은 드뭅니다.

표준 라이브러리는 이 매직으로 언어와 결합해 의미를 만듭니다. [언어와 표준 라이브러리의 경계](../000-meta/004-language-vs-standard-libraries.md)는 이 메타 시스템으로 실현됩니다.

### 3. 매크로 위생(hygiene)

메타 블록의 코드 생성은 위생적(hygienic)입니다. 생성된 코드 안의 식별자는 생성 컨텍스트의 이름 공간만 봅니다. 사용처의 이름 공간은 더럽히지 않습니다.

이 원칙 덕분에 사용자 코드와 메타 블록 사이에 예측할 수 없는 충돌이 생기지 않습니다.

### 4. 컴파일 타임만

모든 메타 처리는 컴파일 타임에 끝납니다. 런타임에 새 derive를 등록하거나 새 매직을 주입할 수는 없습니다.
