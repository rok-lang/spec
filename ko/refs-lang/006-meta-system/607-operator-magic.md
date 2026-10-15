# 6.7. 연산자 매직

`#[infix]`, `#[prefix]`, `#[postfix]` 이 세 속성은 연산자 토큰을 트레잇 메서드에 연결하는 역할만 합니다. 우선순위와 결합성은 [3.2.3](../003-expressions/302-precedence-and-associativity.md#323-우선순위와-결합성-표) 의 표가 언어 차원에서 고정하며, 이 속성으로 정하거나 바꿀 수 없습니다. [연산자 디스패치](../003-expressions/301-operator-dispatch.md) 의 desugar 메커니즘이 동작하기 위해서 반드시 필요한 핵심 매직입니다.

```
infix_attr      ::= "#" "[" "infix" "(" string_lit ("," like_arg)? ("," "swap")? ")" "]"
prefix_attr     ::= "#" "[" "prefix" "(" string_lit ("," like_arg)? ")" "]"
postfix_attr    ::= "#" "[" "postfix" "(" string_lit ("," like_arg)? ")" "]"
like_arg        ::= "like" "=" string_lit
```

## 6.7.1. 세 속성

세 속성이 어떻게 쓰이는지 먼저 예시로 살펴봅니다.

```rok
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}

trait Neg {
    type Output;
    #[prefix("-")]
    fn neg(self) -> Output;
}

trait Degrees {
    #[postfix("°", like = "?")]    // 확장 토큰 — 후위 ? 와 같은 단계
    fn degrees(self) -> f64;
}
```

각 속성이 어디에 부착되며 어떤 형태의 연산을 표현하는지를 정리하면 다음 표와 같습니다.

| 속성                  | 위치          | 형태                |
|-----------------------|---------------|---------------------|
| `#[infix(sym)]`       | 트레잇 메서드 | 이항 (`a SYM b`)    |
| `#[prefix(sym)]`      | 트레잇 메서드 | 전위 단항 (`SYM a`) |
| `#[postfix(sym)]`     | 트레잇 메서드 | 후위 단항 (`a SYM`) |

## 6.7.2. 인자

각 속성이 받는 인자는 다음과 같습니다.

| 인자   | 의미                                            | 필수?                          |
|--------|-------------------------------------------------|--------------------------------|
| `sym`  | 연산자 토큰 (문자열 리터럴)                     | 필수                           |
| `like` | 새 토큰이 따를 기존 토큰 (그 단계와 결합성)     | 3.2 표에 없는 토큰에만, 필수   |
| `swap` | 피연산자 순서를 바꿔 연결 (infix 만)            | 선택                           |

`swap` 은 이항 연산의 두 피연산자를 바꾸어 연결합니다. `#[infix("∈", swap)]` 가 붙은 `contains` 는 `a ∈ b` 를 `b.contains(a)` 로 desugar 합니다. 컨테이너가 왼쪽이 아닌 연산자 (`∈ ∉`) 를 위한 것입니다.

3.2 표에 있는 토큰에는 `like` 를 적을 수 없습니다. 우선순위나 결합성을 지정하는 인자 (`prec`, `left` / `right` / `none` 등) 는 없으며, 적으면 컴파일 에러입니다.

```rok
#[infix("+")]
#[prefix("-")]
#[infix("⊕", like = "+")]      // 새 토큰 — + 와 같은 단계, 같은 결합성
```

## 6.7.3. 연결 메커니즘

`#[infix("+")]` 가 트레잇 메서드 `add` 에 부착되었을 때 컴파일러가 어떻게 동작하는지 살펴봅니다.

먼저 `+` 토큰이 그 트레잇의 연결 자리로 잡힙니다. 이어서 컴파일러는 `a + b` 라는 식을 `<typeof(a) as 그_트레잇<typeof(b)>>::add(a, b)` 형태로 desugar 하게 됩니다. 우선순위와 결합성은 [우선순위와 결합성](../003-expressions/302-precedence-and-associativity.md) 의 표에서 `+` 에 고정된 값 (7, 좌결합) 을 그대로 씁니다.

세 속성 모두 동일한 패턴을 따르며, 결합되는 것은 토큰과 위치, 트레잇 메서드뿐입니다.

## 6.7.4. Unicode 및 다중 alias

같은 메서드에 여러 개의 `#[infix]` 를 부착하면 여러 토큰이 같은 메서드에 매핑되도록 만들 수 있습니다.

```rok
trait Cmp<Rhs = Self> {
    #[infix("<=")]
    #[infix("≤")]
    fn le(self, rhs: Rhs) -> bool;

    #[infix(">=")]
    #[infix("≥")]
    fn ge(self, rhs: Rhs) -> bool;

    #[infix("==")]
    #[infix("≡")]
    fn eq(self, rhs: Rhs) -> bool;
}
```

이는 ASCII 토큰과 Unicode 토큰을 같은 의미로 묶기 위한 장치입니다. `≤`, `≥`, `≠`, `≡` 의 단계는 3.2 표가 대응하는 ASCII 토큰과 같은 단계로 정해 둡니다.

## 6.7.5. 충돌 검사

같은 (토큰, 위치) 조합에 서로 다른 트레잇이 부착되면 등록 시점에 ERROR 가 발생합니다.

```rok
trait MyAdd { #[infix("+")] fn add(...) ... }
// 다른 라이브러리 / 모듈.
trait Add { #[infix("+")] fn add(...) ... }
// → '+' 이항 위치에 트레잇이 둘. ERROR.
```

이렇듯 각 (토큰, 위치) 조합에는 오직 하나의 등록만 허용됩니다.

## 6.7.6. 같은 토큰의 여러 위치

`-` 처럼 전위와 이항 모두에서 사용 가능한 토큰은 별개의 등록으로 취급됩니다.

```rok
trait Sub<Rhs = Self> {
    type Output;
    #[infix("-")]
    fn sub(self, rhs: Rhs) -> Output;
}

trait Neg {
    type Output;
    #[prefix("-")]
    fn neg(self) -> Output;
}
```

예를 들어 `-a + b` 라는 식에서는 전위 `-` 가 이항 `+` 보다 더 강하게 묶이므로 `(-a) + b` 로 해석됩니다.

이항 `-` 와 전위 `-` 는 별개의 매핑으로 처리되기 때문에 서로 다른 트레잇에 부착되어도 아무런 문제가 없습니다.

## 6.7.7. 점-접두 (브로드캐스트) 매핑

[브로드캐스트 연산자](../003-expressions/303-broadcast-operators.md) 의 점-접두 토큰 역시 동일한 매직을 통해 등록됩니다.

```rok
trait DotAdd<Rhs = Self> {
    type Output;
    #[infix(".+")]
    fn dot_add(self, rhs: Rhs) -> Output;
}

trait DotMul<Rhs = Self> {
    type Output;
    #[infix(".*")]
    fn dot_mul(self, rhs: Rhs) -> Output;
}
```

토큰만 다를 뿐 등록 메커니즘은 완전히 같습니다.

## 6.7.8. 매핑의 가시성

`#[infix]` 같은 속성을 통한 등록은 그 트레잇이 보이는 범위 안에서만 효력을 발휘합니다. 즉 해당 트레잇이 보이지 않는 곳에서는 연산자도 작동하지 않게 됩니다.

이 규칙은 라이브러리 격리와 자연스럽게 어울립니다. 어떤 패키지에서 정의한 사용자 정의 연산자 트레잇은 그 패키지를 import 한 코드 안에서만 작동합니다.

표준 라이브러리의 연산자 트레잇 (`Add`, `Sub`, `Mul`, ...) 은 `rok::core::ops` 안에 자리 잡고 있습니다. 컴파일러가 모든 소스에 대해 묵시적으로 `use rok::core::**;` 를 적용해 주기 때문에 ([표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md)) 별도의 import 없이도 산술이나 비교 식이 자연스럽게 동작합니다.

## 6.7.9. 사용자 정의 연산자

사용자 역시 자신의 트레잇에 `#[infix]` 같은 속성을 부착해서 연산자 매핑을 직접 정의할 수 있습니다.

```rok
trait CustomCombine<Rhs = Self> {
    type Output;
    #[infix("⊕")]
    fn combine(self, rhs: Rhs) -> Output;
}

impl CustomCombine for MyType { ... }

let r = a ⊕ b;            // 사용자 토큰
```

`⊕` 처럼 3.2 표에 없는 토큰은 `like = "+"` 처럼 기존 토큰 하나를 지정해, 그 토큰의 단계와 결합성을 따릅니다. 새 단계를 만들 수는 없습니다. 이 토큰은 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md#1104-확장-토큰) 의 확장 토큰 조건을 만족해야 합니다.

## 6.7.10. 우선순위와 결합성 등급

연산자 등급의 단일 권위 위치는 [3.2.3 우선순위 표](../003-expressions/302-precedence-and-associativity.md#323-우선순위와-결합성-표) 입니다. 표준 라이브러리와 사용자 코드 모두 등급을 정하거나 바꿀 수 없으며, 새 토큰은 `like` 로 기존 단계 중 하나에 붙습니다.

## 6.7.11. 평가 시점

`#[infix]` 등의 등록은 parse 직후의 메타 확장 단계에서 처리됩니다. 그 이후의 모든 식 파싱은 등록된 표를 참조하면서 진행됩니다.

따라서 같은 컴파일 단위 안에서 등록과 사용의 순서는 서로 무관합니다. scanner pre-pass 가 이를 미리 모아두기 때문입니다. 3.2 표에 있는 토큰의 우선순위는 등록과 무관하게 처음부터 정해져 있습니다.

## 6.7.12. 다른 단계의 alias 금지

같은 트레잇 메서드에 부착되는 여러 alias 는 모두 같은 단계의 토큰이어야 합니다.

```rok
trait Cmp {
    #[infix("<")]
    #[infix("⊲", like = "+")]    // ERROR — < 는 4 단계, + 는 7 단계
    fn lt(self, rhs: Self) -> bool;
}
```

이는 alias 끼리는 의미가 같다는 가정에서 비롯된 제약입니다.

## 6.7.13. 트레잇 메서드 외 부착 금지

`#[infix]`, `#[prefix]`, `#[postfix]` 는 오직 트레잇의 메서드 시그니처에만 부착할 수 있습니다. 자유 함수나 `impl` 메서드, 일반 트레잇 메서드 (트레잇 외부) 에는 부착할 수 없습니다.

```rok
fn my_add(a: i32, b: i32) -> i32 { a + b }
#[infix("+++")]                              // ERROR — 자유 함수에 부착 불가
fn my_concat(a: str, b: str) -> str { ... }
```

이러한 제약을 두는 이유는 연산자 매핑을 트레잇 시스템과 결합시켜 일관된 디스패치를 보장하기 위함입니다.

## 6.7.14. 정리

연산자 매직은 `#[lang]` 과 더불어 [언어와 표준 라이브러리의 경계](../000-meta/004-language-vs-standard-libraries.md) 에서 언어 토큰과 라이브러리 트레잇의 결합을 구현하는 핵심 메커니즘입니다.

표준 라이브러리의 연산자 사양은 바로 이 매직을 통해서 모든 토큰의 의미를 결정합니다.
