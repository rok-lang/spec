# 6.4. `#[lang(...)]` 매직

`#[lang(slot_name)]` 속성은 표준 라이브러리의 항목을 컴파일러가 직접 인식하는 슬롯에 연결하는 장치입니다. 컴파일러와 런타임이 어떤 항목이 어떤 의미를 갖는지 알 수 있도록 하는 통로 역할을 합니다.

## 6.4.1. 동기

[언어와 표준 라이브러리의 경계](../000-meta/004-language-vs-standard-libraries.md) 에서 살펴본 것처럼, 언어는 `+`, `-` 같은 구문만 제공하고 그 의미는 표준 라이브러리가 결정하도록 정해 두었습니다. 이 모델이 제대로 동작하려면 컴파일러는 어떤 라이브러리 트레잇이나 타입이 어떤 언어 의미와 연결되는지를 반드시 알고 있어야 합니다.

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

#[lang("none_type")]
struct NoneSentinel;               // 'none' 리터럴이 평가되는 타입
```

`#[lang("slot_name")]` 의 형태로 작성하며, 인자로 슬롯 이름을 문자열 리터럴로 받습니다. 트레잇, 타입, 함수, `const`, `static` 같은 항목에 부착할 수 있습니다.

## 6.4.3. 슬롯 이름의 분류

슬롯 이름은 컴파일러가 미리 알고 있는 정해진 집합이기 때문에, 사용자가 임의의 슬롯을 새로 만들어 낼 수는 없습니다.

대표적인 슬롯들을 예시로 정리하면 다음과 같습니다.

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

전체 슬롯 목록과 각 슬롯의 부착 항목은 [표준 라이브러리 0.4 `#[lang]` 슬롯 매핑 표](../../refs-standard-libraries/000-meta/004-lang-slot-table.md) 가 권위를 가지며, 위 표는 그 일부를 발췌한 것입니다. 슬롯 집합을 바꿀 때에는 0.4 표에서부터 갱신합니다.

## 6.4.4. 매핑의 효과

`#[lang("add_trait")]` 가 부착된 트레잇이 있으면, 컴파일러는 그 트레잇을 덧셈 의미와 연결해 둡니다. 그래서 이후에 `+` 연산자가 등장하면 다음과 같이 desugar 됩니다.

```
a + b    →    <typeof(a) as 그_트레잇<typeof(b)>>::add(a, b)
```

여기서 `그_트레잇` 이 바로 `#[lang("add_trait")]` 가 부착된 트레잇을 가리킵니다.

## 6.4.5. 사용자 코드의 사용 (거의 없음)

`#[lang]` 은 거의 표준 라이브러리에서만 사용하는 속성입니다. 사용자 코드가 이 속성을 직접 부착하는 경우는 _no_std_ 환경에서 자체 라이브러리를 구성하거나, 임베디드나 운영체제처럼 언어 핵심을 다시 정의해야 하는 환경 정도로 한정됩니다. 일반적인 사용자가 `#[lang]` 을 직접 부착할 일은 없다고 보아도 무방합니다.

## 6.4.6. 한 슬롯에 한 항목

같은 슬롯 이름에 여러 항목을 부착하면 컴파일 에러가 발생합니다.

```rok
#[lang("add_trait")] trait MyAdd { ... }
#[lang("add_trait")] trait OtherAdd { ... }    // ERROR - 슬롯 중복
```

각 슬롯은 오직 하나의 항목에만 유일하게 매핑되도록 설계됩니다. 이 유일성은 프로그램 전체 (링크되는 모든 패키지) 기준입니다. 따라서 `rok::core` 가 정의한 슬롯 (`panic_fn`, `oom_fn`, `unreachable_fn` 등) 을 사용자 패키지가 다시 부착해 바꿀 수 없으며, 부착하면 컴파일 에러입니다.

## 6.4.7. 슬롯의 시그니처 검사

각 슬롯은 어떤 시그니처의 항목을 받을 수 있는지가 미리 정해져 있습니다.

```rok
#[lang("add_trait")]
trait Add<Rhs = Self> {              // 슬롯이 요구하는 형태
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}
```

만약 시그니처가 슬롯이 요구하는 형태와 맞지 않으면 컴파일 에러로 처리됩니다.

```rok
#[lang("add_trait")]
trait BadAdd { fn run() }            // ERROR - 'add_trait' 슬롯의 형태와 다름
```

## 6.4.8. 알려지지 않은 슬롯 이름

```rok
#[lang("nonexistent_slot")]
trait T { ... }
```

알려지지 않은 슬롯 이름을 사용하면 마찬가지로 컴파일 에러가 됩니다.

## 6.4.9. 슬롯의 안정성

슬롯 이름의 집합 자체가 언어 사양의 일부이기 때문에, 슬롯이 추가되거나 변경되거나 제거되면 언어 버전이 함께 변경됩니다 (SemVer).

## 6.4.10. 매직 속성과 일반 속성의 차이

| 측면         | `#[derive]`, `#[inline]` 등  | `#[lang]`              |
|--------------|------------------------------|------------------------|
| 누가 부착    | 사용자                       | 표준 라이브러리 (주로) |
| 효과         | 코드 생성 또는 컴파일러 힌트 | 언어 의미와 결합       |
| 슬롯 또는 키 | 자유 (속성 이름이 식별)      | 고정된 슬롯 집합       |

`#[lang]` 은 메타 시스템 안에서 가장 특권적인 속성에 해당합니다. 다른 매직, 즉 `#[literal]` 이나 `#[keyword]`, 그리고 연산자 매직은 모두 `#[lang]` 의 세분화된 형태로 볼 수 있습니다.

## 6.4.11. `#[lang]` 과 다른 매직의 관계

연산자 매직 (`#[infix]` 등) 은 보통 `#[lang(...)]` 가 부착된 트레잇의 메서드에 함께 부착되어 사용됩니다.

```rok
#[lang("add_trait")]
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}
```

여기서 `#[lang("add_trait")]` 는 이 트레잇이 덧셈 의미가 자리 잡는 곳임을 알려 주고, `#[infix("+", ...)]` 는 이 메서드가 `+` 토큰의 디스패치 대상임을 표시합니다. 이렇게 두 매직이 함께 작동하면서 언어와 라이브러리를 양방향으로 연결해 주게 됩니다.

`#[literal]` 과 `#[keyword]` 또한 각각 리터럴 자리와 키워드 등록 슬롯에 연결되는 도구로 이해할 수 있습니다.
