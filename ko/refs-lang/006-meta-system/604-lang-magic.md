# 6.4. `#[lang(...)]` 매직

`#[lang(slot_name)]` 속성은 표준 라이브러리의 항목을 컴파일러가 직접 인식하는 슬롯에 연결한다. 컴파일러와 런타임이 어떤 항목이 어떤 의미를 갖는지를 알게 하는 통로다.

## 6.4.1. 동기

[언어와 표준 라이브러리의 경계](../000-meta/004-language-vs-stdlib.md) 에서, 언어는 `+`, `-` 같은 구문만 제공하고 의미는
표준 라이브러리가 결정하기로 정했다.

이 모델이 동작하려면 컴파일러는 어떤 라이브러리 트레잇이나 타입이 어떤 언어 의미와 연결되는지 알아야 한다.

```
컴파일러 슬롯       ← 언어가 미리 알고 있는 자리
                       (예: "더하기 트레잇", "옵션 자리", "iterator 트레잇")

라이브러리 항목      ← stdlib 가 정의하는 trait / type / fn

  ←── #[lang(slot)] 가 두 쪽을 연결 ──→
```

## 6.4.2. 형태

```
lang_attr       ::= "#" "[" "lang" "(" string_lit ")" "]"
```

```nok
#[lang("add_trait")]
trait Add<Rhs = Self> {
    type Output;
    #[infix("+", prec = 9, left)]
    fn add(self, rhs: Rhs) -> Output;
}

#[lang("iterator_trait")]
trait Iterator { ... }

#[lang("none_type")]
struct NoneSentinel;               // 'none' 리터럴이 평가되는 타입
```

- `#[lang("slot_name")]` 형태이다. 인자로 슬롯 이름 (문자열 리터럴) 을 받는다.
- 항목 (트레잇, 타입, 함수, `const`, `static`) 에 부착한다.

## 6.4.3. 슬롯 이름의 분류

슬롯 이름은 컴파일러가 미리 알고 있는 정해진 집합이다. 사용자가 임의의 슬롯을 만들 수 없다.

대표 슬롯 (예시) 은 다음과 같다.

| 슬롯                    | 연결되는 항목 종류 | 컴파일러 사용                 |
| ----------------------- | ------------------ | ----------------------------- |
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
| `"eq_trait"`            | 트레잇             | `==` 연산자 + `#[derive(Eq)]` |
| ...                     | ...                | ...                           |

전체 슬롯 목록은 표준 라이브러리 사양과 컴파일러 사양 (7장과 8장) 에서 정의한다.

## 6.4.4. 매핑의 효과

`#[lang("add_trait")]` 가 부착된 트레잇에 대해 컴파일러는 그 트레잇을 덧셈 의미와 연결한다.

이후 `+` 연산자가 등장하면 다음과 같이 desugar 된다.

```
a + b    →    <typeof(a) as 그_트레잇<typeof(b)>>::add(a, b)
```

여기서 `그_트레잇` 은 `#[lang("add_trait")]` 가 부착된 트레잇이다.

## 6.4.5. 사용자 코드의 사용 (거의 없음)

`#[lang]` 은 거의 표준 라이브러리만 사용한다. 사용자 코드가 이 속성을 부착하는 경우는 다음과 같다.

- _no_std_ 환경에서 자체 라이브러리를 만들 때.
- 임베디드나 운영체제처럼 언어 핵심을 다시 정의해야 하는 환경.

일반 사용자가 `#[lang]` 을 직접 부착할 일은 없다.

## 6.4.6. 한 슬롯에 한 항목

같은 슬롯 이름에 여러 항목을 부착하면 컴파일 에러가 발생한다.

```nok
#[lang("add_trait")] trait MyAdd { ... }
#[lang("add_trait")] trait OtherAdd { ... }    // ERROR — 슬롯 중복
```

각 슬롯은 유일하게 한 항목에만 매핑된다.

## 6.4.7. 슬롯의 시그니처 검사

각 슬롯은 어떤 시그니처의 항목을 받는지 정해져 있다.

```nok
#[lang("add_trait")]
trait Add<Rhs = Self> {              // 슬롯이 요구하는 형태
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}
```

시그니처가 슬롯의 요구와 맞지 않으면 컴파일 에러다.

```nok
#[lang("add_trait")]
trait BadAdd { fn run() }            // ERROR — 'add_trait' 슬롯의 형태와 다름
```

## 6.4.8. 알려지지 않은 슬롯 이름

```nok
#[lang("nonexistent_slot")]
trait T { ... }
```

알려지지 않은 슬롯 이름은 컴파일 에러다.

## 6.4.9. 슬롯의 안정성

슬롯 이름의 집합은 언어 사양의 일부다. 슬롯이 추가, 변경, 제거되면 언어 버전이 바뀐다 (SemVer).

## 6.4.10. 매직 속성과 일반 속성의 차이

| 측면         | `#[derive]`, `#[inline]` 등  | `#[lang]`              |
| ------------ | ---------------------------- | ---------------------- |
| 누가 부착    | 사용자                       | 표준 라이브러리 (주로) |
| 효과         | 코드 생성 또는 컴파일러 힌트 | 언어 의미와 결합       |
| 슬롯 또는 키 | 자유 (속성 이름이 식별)      | 고정된 슬롯 집합       |

`#[lang]` 은 메타 시스템에서 가장 특권적인 속성이다. 다른 매직 (`#[literal]`, `#[keyword]`,
연산자 매직) 은 `#[lang]` 의 세분화된 형태로 볼 수 있다.

## 6.4.11. `#[lang]` 과 다른 매직의 관계

연산자 매직 (`#[infix]` 등) 은 보통 `#[lang(...)]` 가 부착된 트레잇의 메서드에 함께 부착된다.

```nok
#[lang("add_trait")]
trait Add<Rhs = Self> {
    type Output;
    #[infix("+", prec = 9, left)]
    fn add(self, rhs: Rhs) -> Output;
}
```

- `#[lang("add_trait")]`: 이 트레잇이 덧셈 의미의 자리이다.
- `#[infix("+", ...)]`: 이 메서드가 `+` 토큰의 디스패치 대상이다.

두 매직이 함께 언어와 라이브러리를 양방향으로 연결한다.

`#[literal]` 과 `#[keyword]` 도 각각 리터럴 자리와 키워드 등록 슬롯에 연결되는 도구로 볼 수 있다.
