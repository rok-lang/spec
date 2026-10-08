# 6.7. 연산자 매직

`#[infix]`, `#[prefix]`, `#[postfix]` 이 세 속성은 연산자 토큰을 트레잇 메서드에 연결해 주는 역할을 합니다. [연산자 디스패치](../003-expressions/301-operator-dispatch.md) 의 desugar 메커니즘이 동작하기 위해서 반드시 필요한 핵심 매직이라고 할 수 있습니다.

```
infix_attr      ::= "#" "[" "infix" "(" string_lit "," "prec" "=" int_lit "," assoc ")" "]"
prefix_attr     ::= "#" "[" "prefix" "(" string_lit "," "prec" "=" int_lit ")" "]"
postfix_attr    ::= "#" "[" "postfix" "(" string_lit "," "prec" "=" int_lit ")" "]"
assoc           ::= "left" | "right" | "none"
```

## 6.7.1. 세 속성

세 속성이 어떻게 쓰이는지 먼저 예시로 살펴봅니다.

```rok
trait Add<Rhs = Self> {
    type Output;
    #[infix("+", prec = 7, left)]
    fn add(self, rhs: Rhs) -> Output;
}

trait Neg {
    type Output;
    #[prefix("-", prec = 11)]
    fn neg(self) -> Output;
}

trait IsValid {
    #[postfix("?", prec = 12)]
    fn is_valid(self) -> bool;
}
```

각 속성이 어디에 부착되며 어떤 형태의 연산을 표현하는지를 정리하면 다음 표와 같습니다.

| 속성                         | 위치          | 형태                |
|------------------------------|---------------|---------------------|
| `#[infix(sym, prec, assoc)]` | 트레잇 메서드 | 이항 (`a SYM b`)    |
| `#[prefix(sym, prec)]`       | 트레잇 메서드 | 전위 단항 (`SYM a`) |
| `#[postfix(sym, prec)]`      | 트레잇 메서드 | 후위 단항 (`a SYM`) |

## 6.7.2. 인자

각 속성이 받는 인자의 의미와 필수 여부는 다음과 같이 구성됩니다.

| 인자    | 의미                               | 필수?                       |
|---------|------------------------------------|-----------------------------|
| `sym`   | 연산자 토큰 (문자열 리터럴)        | 필수                        |
| `prec`  | 우선순위 등급 (정수)               | infix, prefix, postfix 모두 |
| `assoc` | 결합성 (`left` / `right` / `none`) | infix 만                    |

prefix 와 postfix 는 단항 연산자이기 때문에 결합성이라는 개념이 적용되지 않습니다.

```rok
#[infix("+", prec = 7, left)]
#[prefix("-", prec = 11)]
#[postfix("!", prec = 13)]
```

## 6.7.3. 연결 메커니즘

`#[infix("+", prec = 7, left)]` 가 트레잇 메서드 `add` 에 부착되었을 때 컴파일러가 어떻게 동작하는지 살펴봅니다.

먼저 `+` 토큰이 그 트레잇의 연결 자리로 잡힙니다. 이어서 컴파일러는 `a + b` 라는 식을 `<typeof(a) as 그_트레잇<typeof(b)>>::add(a, b)` 형태로 desugar 하게 됩니다. 그리고 우선순위 7, 좌결합이라는 속성과 함께 [우선순위와 결합성](../003-expressions/302-precedence-and-associativity.md) 의 표에 등록됩니다.

세 속성 모두 동일한 패턴을 따르고 있어서, 토큰과 위치, 그리고 우선순위가 트레잇 메서드와 결합되는 방식으로 정리됩니다.

## 6.7.4. Unicode 및 다중 alias

같은 메서드에 여러 개의 `#[infix]` 를 부착하면 여러 토큰이 같은 메서드에 매핑되도록 만들 수 있습니다.

```rok
trait Cmp<Rhs = Self> {
    #[infix("<=", prec = 4, none)]
    #[infix("≤", prec = 4, none)]
    fn le(self, rhs: Rhs) -> bool;

    #[infix(">=", prec = 4, none)]
    #[infix("≥", prec = 4, none)]
    fn ge(self, rhs: Rhs) -> bool;

    #[infix("==", prec = 4, none)]
    #[infix("≡", prec = 4, none)]
    fn eq(self, rhs: Rhs) -> bool;
}
```

이는 ASCII 토큰과 Unicode 토큰을 같은 의미로 묶기 위한 장치입니다. 다만 모든 alias 의 우선순위와 결합성은 반드시 동일해야 하며, 만약 다르게 지정되면 등록 시점에 충돌이 발생하게 됩니다.

## 6.7.5. 충돌 검사

같은 (토큰, 위치) 조합에 서로 다른 트레잇이 부착되면 등록 시점에 ERROR 가 발생합니다.

```rok
trait MyAdd { #[infix("+", prec = 5)] fn add(...) ... }
// 다른 라이브러리 / 모듈.
trait Add { #[infix("+", prec = 7)] fn add(...) ... }
// → '+' 의 등록이 두 곳에서 다른 우선순위로 충돌. ERROR.
```

이렇듯 각 (토큰, 위치) 조합에는 오직 하나의 등록만 허용됩니다.

## 6.7.6. 같은 토큰의 여러 위치

`-` 처럼 전위와 이항 모두에서 사용 가능한 토큰은 별개의 등록으로 취급됩니다.

```rok
trait Sub<Rhs = Self> {
    type Output;
    #[infix("-", prec = 7, left)]
    fn sub(self, rhs: Rhs) -> Output;
}

trait Neg {
    type Output;
    #[prefix("-", prec = 11)]
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
    #[infix(".+", prec = 7, left)]
    fn dot_add(self, rhs: Rhs) -> Output;
}

trait DotMul<Rhs = Self> {
    type Output;
    #[infix(".*", prec = 8, left)]
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
    #[infix("⊕", prec = 7, left)]
    fn combine(self, rhs: Rhs) -> Output;
}

impl CustomCombine for MyType { ... }

let r = a ⊕ b;            // 사용자 토큰
```

다만 `⊕` 같은 토큰을 사용하려면 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md) 의 확장 토큰표에 미리 등록되어 있어야 합니다. 사용자 정의 연산자가 새로운 유니코드 토큰을 도입하는 경우에는 lexer 가 그 토큰을 인식할 수 있어야 한다는 점도 함께 고려해야 합니다.

이 등록 메커니즘, 즉 어휘 토큰 등록의 정확한 형태는 표준 라이브러리와 컴파일러 사양에서 추가로 정의됩니다.

## 6.7.10. 우선순위와 결합성 등급

연산자 등급은 표준 라이브러리에 부착된 `#[infix]`, `#[prefix]`, `#[postfix]` 속성에 의해 정의되며, 단일 진실 위치는 `rok/ops.rok` 입니다. 등급의 권위 있는 표는 [3.2.3 우선순위 표](../003-expressions/302-precedence-and-associativity.md) 에 정리되어 있고, 두 자리는 항상 함께 갱신됩니다.

사용자가 새로운 등급을 정할 때는 표준 라이브러리의 등급과 충돌하지 않도록 끼워 넣어야 합니다. 일반적으로 비트 AND (`6`) 과 비트 OR (`5`) 사이처럼 비어 있는 자리에 새 등록을 위치시키는 방식으로 처리합니다.

## 6.7.11. 평가 시점

`#[infix]` 등의 등록은 parse 직후의 메타 확장 단계에서 처리됩니다. 그 이후의 모든 식 파싱은 등록된 표를 참조하면서 진행됩니다.

따라서 같은 컴파일 단위 안에서 등록과 사용의 순서는 서로 무관합니다. scanner pre-pass 가 이를 미리 모아두기 때문입니다.

## 6.7.12. 우선순위가 다른 alias 금지

같은 트레잇 메서드에 부착되는 여러 alias 는 우선순위와 결합성이 모두 동일해야 합니다.

```rok
trait Cmp {
    #[infix("<", prec = 4, none)]
    #[infix("less_than", prec = 5, left)]    // ERROR — 다른 우선순위
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

연산자 매직은 `#[lang]` 과 더불어 [언어와 표준 라이브러리의 경계](../000-meta/004-language-vs-standard-libraries.md) 에서 언어 토큰과 라이브러리 트레잇의 결합을 구현해 주는 핵심 메커니즘입니다.

표준 라이브러리의 연산자 사양은 바로 이 매직을 통해서 모든 토큰의 의미를 결정하게 됩니다.
