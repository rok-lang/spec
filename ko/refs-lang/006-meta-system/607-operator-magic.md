# 6.7. 연산자 매직

`#[infix]`, `#[prefix]`, `#[postfix]` 세 속성은 연산자 토큰을 트레잇 메서드에 연결하는 일만 합니다. 우선순위와 결합성은 [3.2.3](../003-expressions/302-precedence-and-associativity.md#323-우선순위와-결합성-표)의 표가 언어 수준에서 고정합니다. 이 속성으로 정하거나 바꿀 수 없습니다. [연산자 디스패치](../003-expressions/301-operator-dispatch.md)에서 연산자를 메서드 호출로 푸는 메커니즘은 이 세 매직 없이는 동작하지 않습니다.

```
infix_attr      ::= "#" "[" "infix" "(" string_lit ("," like_arg)? ("," "swap")? ")" "]"
prefix_attr     ::= "#" "[" "prefix" "(" string_lit ("," like_arg)? ")" "]"
postfix_attr    ::= "#" "[" "postfix" "(" string_lit ("," like_arg)? ")" "]"
like_arg        ::= "like" "=" string_lit
```

## 6.7.1. 세 속성

세 속성은 다음과 같이 씁니다.

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
    #[postfix("°", like = "?")]    // 확장 토큰, 후위 ? 와 같은 단계
    fn degrees(self) -> f64;
}
```

각 속성이 붙는 위치와 나타내는 연산 형태는 다음과 같습니다.

| 속성                  | 위치          | 형태                |
|-----------------------|---------------|---------------------|
| `#[infix(sym)]`       | 트레잇 메서드 | 이항 (`a SYM b`)    |
| `#[prefix(sym)]`      | 트레잇 메서드 | 전위 단항 (`SYM a`) |
| `#[postfix(sym)]`     | 트레잇 메서드 | 후위 단항 (`a SYM`) |

## 6.7.2. 인자

각 속성이 받는 인자는 다음과 같습니다.

| 인자   | 의미                                            | 필수 여부                      |
|--------|-------------------------------------------------|--------------------------------|
| `sym`  | 연산자 토큰 (문자열 리터럴)                     | 필수                           |
| `like` | 새 토큰이 따를 기존 토큰 (그 단계와 결합성)     | 3.2 표에 없는 토큰에만, 필수   |
| `swap` | 피연산자 순서를 바꿔 연결 (infix만)             | 선택                           |

`swap`은 이항 연산의 두 피연산자 순서를 바꾸어 연결합니다. `#[infix("∈", swap)]`가 붙은 `contains`가 있으면 `a ∈ b`는 `b.contains(a)`로 풀립니다. 컨테이너가 왼쪽에 오지 않는 연산자(`∈ ∉`)를 위한 인자입니다.

3.2 표에 있는 토큰에는 `like`를 쓸 수 없습니다. 우선순위나 결합성을 지정하는 인자(`prec`, `left` / `right` / `none` 등)는 없습니다. 쓰면 컴파일 에러입니다.

```rok
#[infix("+")]
#[prefix("-")]
#[infix("⊕", like = "+")]      // 새 토큰, + 와 같은 단계, 같은 결합성
```

## 6.7.3. 연결 메커니즘

`#[infix("+")]`를 트레잇 메서드 `add`에 붙이면 컴파일러는 다음과 같이 동작합니다.

먼저 `+` 토큰을 그 트레잇에 연결합니다. 그다음 식 `a + b`를 `<typeof(a) as 그_트레잇<typeof(b)>>::add(a, b)` 형태로 풉니다. 우선순위와 결합성은 [우선순위와 결합성](../003-expressions/302-precedence-and-associativity.md)의 표에서 `+`에 고정된 값(7, 좌결합)을 그대로 씁니다.

세 속성은 모두 같은 패턴을 따릅니다. 연결되는 것은 토큰, 위치, 트레잇 메서드뿐입니다.

## 6.7.4. Unicode와 다중 alias

한 메서드에 `#[infix]`를 여러 개 붙이면 여러 토큰을 같은 메서드에 매핑할 수 있습니다.

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

ASCII 토큰과 Unicode 토큰을 같은 의미로 묶는 방법입니다. 3.2 표는 `≤`, `≥`, `≠`, `≡`의 단계를 대응하는 ASCII 토큰과 같은 단계로 정합니다.

## 6.7.5. 충돌 검사

같은 (토큰, 위치) 조합에 서로 다른 트레잇이 붙으면 등록하는 시점에 에러입니다.

```rok
trait MyAdd { #[infix("+")] fn add(...) ... }
// 다른 라이브러리 / 모듈.
trait Add { #[infix("+")] fn add(...) ... }
// → '+' 이항 위치에 트레잇이 둘. ERROR.
```

(토큰, 위치) 조합마다 등록은 하나만 허용됩니다.

## 6.7.6. 같은 토큰의 여러 위치

`-`처럼 전위와 이항 양쪽에 쓸 수 있는 토큰은 위치마다 따로 등록한 것으로 다룹니다.

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

예를 들어 식 `-a + b`에서는 전위 `-`가 이항 `+`보다 강하게 묶이므로 `(-a) + b`로 해석됩니다.

이항 `-`와 전위 `-`는 별개의 매핑이므로 서로 다른 트레잇에 붙어도 문제가 없습니다.

## 6.7.7. 점-접두(브로드캐스트) 매핑

[브로드캐스트 연산자](../003-expressions/303-broadcast-operators.md)의 점-접두 토큰도 같은 매직으로 등록합니다.

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

토큰만 다를 뿐 등록 메커니즘은 같습니다.

## 6.7.8. 매핑의 가시성

`#[infix]` 같은 속성으로 한 등록은 그 트레잇이 보이는 범위 안에서만 효력이 있습니다. 보인다는 것은 [5.6](../005-modules/506-visibility.md)에 따라 접근할 수 있다는 뜻입니다. 경로로 닿는 `pub` 트레잇이면 충분하고 `use`는 없어도 됩니다. 트레잇이 보이지 않는 곳에서는 연산자도 동작하지 않습니다.

이 규칙은 라이브러리 격리와 잘 맞습니다. 어떤 패키지가 정의한 사용자 정의 연산자 트레잇은 그 패키지를 가져온 코드 안에서만 동작합니다.

표준 라이브러리의 연산자 트레잇(`Add`, `Sub`, `Mul`, ...)은 `rok::core::ops`에 있습니다. 컴파일러가 모든 소스에 `use rok::core::**;`를 묵시적으로 적용합니다([표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md)). 그래서 따로 가져오지 않아도 산술 식과 비교 식이 동작합니다.

## 6.7.9. 사용자 정의 연산자

사용자도 자기 트레잇에 `#[infix]` 같은 속성을 붙여 연산자 매핑을 직접 정의할 수 있습니다.

```rok
trait CustomCombine<Rhs = Self> {
    type Output;
    #[infix("⊕")]
    fn combine(self, rhs: Rhs) -> Output;
}

impl CustomCombine for MyType { ... }

let r = a ⊕ b;            // 사용자 토큰
```

`⊕`처럼 3.2 표에 없는 토큰은 `like = "+"`처럼 기존 토큰 하나를 지정하고 그 토큰의 단계와 결합성을 따릅니다. 새 단계를 만들 수는 없습니다. 이 토큰은 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md#1104-확장-토큰)의 확장 토큰 조건을 만족해야 합니다. 모든 글자가 Unicode `Pattern_Syntax`여야 합니다. 이미 있는 고정 토큰과 같으면 에러입니다.

토큰이 고정 토큰처럼 시작해도 됩니다. 렉서는 가장 길게 일치하는 쪽을 고르므로 `#[infix("<-", like = "+")]`를 붙이면 `c <- 3`은 연산자 하나입니다. 이렇게 고정 토큰의 첫 글자로 시작하는 토큰은 그것을 등록한 패키지의 파일에서만 동작합니다. 다른 곳에서 `a<-1`은 그대로 `a < -1`입니다([1.10.4](../001-lexical/110-punctuation-and-operator-tokens.md#1104-확장-토큰)).

## 6.7.10. 우선순위와 결합성 등급

연산자 등급은 [3.2.3 우선순위 표](../003-expressions/302-precedence-and-associativity.md#323-우선순위와-결합성-표) 한 곳에서만 정합니다. 표준 라이브러리도 사용자 코드도 등급을 정하거나 바꿀 수 없습니다. 새 토큰은 `like`로 기존 단계 가운데 하나에 붙습니다.

## 6.7.11. 평가 시점

컴파일러는 `#[infix]` 등의 등록을 파싱 직후의 메타 확장 단계에서 처리합니다. 이후의 식 파싱은 모두 등록 표를 참조합니다.

스캐너 사전 패스(scanner pre-pass)가 등록을 미리 모으므로 같은 컴파일 단위 안에서는 등록과 사용의 순서가 상관없습니다. 3.2 표에 있는 토큰의 우선순위는 등록과 상관없이 처음부터 정해져 있습니다.

## 6.7.12. 다른 단계의 alias 금지

한 트레잇 메서드에 붙는 별칭(alias)은 모두 같은 단계의 토큰이어야 합니다.

```rok
trait Cmp {
    #[infix("<")]
    #[infix("⊲", like = "+")]    // ERROR: < 는 4 단계, + 는 7 단계
    fn lt(self, rhs: Self) -> bool;
}
```

별칭끼리는 의미가 같다고 가정하기 때문입니다.

## 6.7.13. 트레잇 메서드 외 부착 금지

`#[infix]`, `#[prefix]`, `#[postfix]`는 트레잇의 메서드 시그니처에만 붙일 수 있습니다. 자유 함수, `impl` 메서드, 트레잇 밖의 일반 메서드에는 붙일 수 없습니다.

```rok
fn my_add(a: i32, b: i32) -> i32 { a + b }
#[infix("+++")]                              // ERROR: 자유 함수에 부착 불가
fn my_concat(a: str, b: str) -> str { ... }
```

연산자 매핑을 트레잇 시스템에 묶어야 디스패치가 일관되기 때문입니다.

## 6.7.14. 정리

연산자 매직은 `#[lang]`과 함께 [언어와 표준 라이브러리의 경계](../000-meta/004-language-vs-standard-libraries.md)에서 언어 토큰을 라이브러리 트레잇에 연결하는 중심 메커니즘입니다.

표준 라이브러리의 연산자 사양은 이 매직으로 모든 토큰의 의미를 정합니다.
