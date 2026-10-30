# 6.4. `#[lang(...)]` 매직

`#[lang(slot_name)]` 속성은 표준 라이브러리의 항목을 컴파일러가 직접 인식하는 슬롯에 연결합니다. 컴파일러와 런타임은 이 속성으로 어떤 항목이 어떤 의미를 갖는지 압니다.

## 6.4.1. 동기

[언어와 표준 라이브러리의 경계](../000-meta/004-language-vs-standard-libraries.md)에서 설명했듯이, 언어는 `+`, `-` 같은 구문만 두고 그 의미는 표준 라이브러리가 정합니다. 이 모델이 동작하려면 컴파일러는 어떤 라이브러리 트레잇이나 타입이 어떤 언어 의미와 연결되는지 알아야 합니다.

```
compiler slot           ← language-known anchor
                          (e.g. "add trait", "option slot", "iterator trait")

library item            ← trait / type / fn defined by standard libraries

  ←── #[lang(slot)] connects the two sides ──→
```

## 6.4.2. 형태

```
lang_attr       ::= "#" "[" "lang" "(" string_lit ")" "]"
```

```rok
#[lang("add_trait")]
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}

#[lang("iterator_trait")]
trait Iterator { ... }

#[lang("display_trait")]
trait Display { ... }              // 기본 f"{x}" 포맷
```

`#[lang("slot_name")]` 형태로 쓰며, 인자로 슬롯 이름을 문자열 리터럴로 받습니다. 트레잇, 타입, 함수, `const`, `static` 같은 항목에 붙일 수 있습니다.

## 6.4.3. 슬롯 이름의 분류

슬롯 이름은 컴파일러가 미리 알고 있는 고정된 집합입니다. 사용자가 새 슬롯을 만들 수는 없습니다.

대표적인 슬롯은 다음과 같습니다.

| 슬롯                    | 연결되는 항목 종류 | 컴파일러 사용                 |
|-------------------------|--------------------|-------------------------------|
| `"add_trait"`           | 트레잇             | `+` 연산자 desugar            |
| `"sub_trait"`           | 트레잇             | `-` 연산자 desugar            |
| `"mul_trait"`           | 트레잇             | `*` 연산자 desugar            |
| ...                     | ...                | ...                           |
| `"none_type"`           | 타입               | `none` 리터럴의 타입          |
| `"iterator_trait"`      | 트레잇             | `for` 루프 desugar            |
| `"into_iterator_trait"` | 트레잇             | `for` 루프 desugar            |
| `"future_trait"`        | 트레잇             | `await` desugar               |
| `"index_trait"`         | 트레잇             | `[i]` 인덱싱 desugar          |
| `"slice_trait"`         | 트레잇             | `[1:3]` 슬라이싱 desugar      |
| `"display_trait"`       | 트레잇             | `f"{x}"` 형식화               |
| `"debug_trait"`         | 트레잇             | `f"{x:?}"` 디버그 형식        |
| `"clone_trait"`         | 트레잇             | `#[derive(Clone)]`            |
| `"eq_trait"`            | 트레잇             | `==` / `!=` 연산자 (`PartialEq`) |
| ...                     | ...                | ...                           |

전체 슬롯 목록과 각 슬롯에 붙는 항목은 [표준 라이브러리 0.4 `#[lang]` 슬롯 매핑 표](../../refs-standard-libraries/000-meta/004-lang-slot-table.md)가 기준이며, 위 표는 그 일부입니다. 슬롯 집합을 바꿀 때는 0.4 표부터 고칩니다.

## 6.4.4. 매핑의 효과

`#[lang("add_trait")]`가 붙은 트레잇이 있으면 컴파일러는 그 트레잇을 덧셈 의미와 연결합니다. 그래서 `+` 연산자는 다음과 같이 desugar됩니다.

```
a + b    →    <typeof(a) as 그_트레잇<typeof(b)>>::add(a, b)
```

여기서 `그_트레잇`은 `#[lang("add_trait")]`가 붙은 트레잇입니다.

## 6.4.5. 사용자 코드의 사용 (거의 없음)

`#[lang]`은 거의 표준 라이브러리에서만 씁니다. 사용자 코드가 이 속성을 직접 붙이는 경우는 _no_std_ 환경에서 자체 라이브러리를 만들거나, 임베디드나 운영체제처럼 언어 코어를 다시 정의해야 하는 환경뿐입니다. 일반 사용자가 `#[lang]`을 직접 붙일 일은 없습니다.

## 6.4.6. 한 슬롯에 한 항목

같은 슬롯 이름을 여러 항목에 붙이면 컴파일 에러입니다.

```rok
#[lang("add_trait")] trait MyAdd { ... }
#[lang("add_trait")] trait OtherAdd { ... }    // ERROR: 슬롯 중복
```

각 슬롯은 정확히 한 항목에 매핑됩니다. 이 유일성은 프로그램 전체(링크되는 모든 패키지)에 걸쳐 성립합니다. 따라서 `rok::core`가 정의한 슬롯(`panic_fn`, `oom_fn`, `unreachable_fn` 등)을 사용자 패키지가 다시 붙여 바꿀 수 없으며, 붙이면 컴파일 에러입니다.

## 6.4.7. 슬롯의 시그니처 검사

각 슬롯이 받는 항목의 시그니처는 미리 정해져 있습니다.

```rok
#[lang("add_trait")]
trait Add<Rhs = Self> {              // 슬롯이 요구하는 형태
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}
```

시그니처가 슬롯이 요구하는 형태와 맞지 않으면 컴파일 에러입니다.

```rok
#[lang("add_trait")]
trait BadAdd { fn run() }            // ERROR: 'add_trait' 슬롯의 형태와 다름
```

## 6.4.8. 알려지지 않은 슬롯 이름

```rok
#[lang("nonexistent_slot")]
trait T { ... }
```

알려지지 않은 슬롯 이름을 쓰는 것도 컴파일 에러입니다.

## 6.4.9. 슬롯의 안정성

슬롯 이름의 집합은 언어 사양의 일부입니다. 그래서 슬롯을 추가, 변경, 제거하면 언어 버전도 바뀝니다(SemVer).

## 6.4.10. 매직 속성과 일반 속성의 차이

| 측면         | `#[derive]`, `#[inline]` 등  | `#[lang]`              |
|--------------|------------------------------|------------------------|
| 누가 부착    | 사용자                       | 표준 라이브러리 (주로) |
| 효과         | 코드 생성 또는 컴파일러 힌트 | 언어 의미와 결합       |
| 슬롯 또는 키 | 자유 (속성 이름이 식별)      | 고정된 슬롯 집합       |

`#[lang]`은 메타 시스템에서 가장 특권이 큰 속성입니다. 다른 매직(`#[literal]`, `#[keyword]`, 연산자 매직)은 모두 `#[lang]`을 세분한 형태로 볼 수 있습니다.

## 6.4.11. `#[lang]` 과 다른 매직의 관계

연산자 매직(`#[infix]` 등)은 보통 `#[lang(...)]`가 붙은 트레잇의 메서드에 붙여 씁니다.

```rok
#[lang("add_trait")]
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}
```

여기서 `#[lang("add_trait")]`는 덧셈의 의미가 이 트레잇에 있음을 선언하고, `#[infix("+", ...)]`는 이 메서드가 `+` 토큰의 디스패치 대상임을 표시합니다. 두 매직이 함께 언어와 라이브러리를 양방향으로 연결합니다.

`#[literal]`과 `#[keyword]`도 각각 리터럴 자리와 키워드 등록 슬롯에 연결하는 도구로 볼 수 있습니다.
