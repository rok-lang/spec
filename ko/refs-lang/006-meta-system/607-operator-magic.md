# 6.7. 연산자 매직

`#[infix]` / `#[prefix]` / `#[postfix]` 세 속성은 연산자 토큰을 트레잇 메서드에 연결한다.
[연산자 디스패치](../003-expressions/301-operator-dispatch.md) 의 desugar 메커니즘이 동작하는 데 필요한 핵심 매직이다.

```
infix_attr      ::= "#" "[" "infix" "(" string_lit "," "prec" "=" int_lit "," assoc ")" "]"
prefix_attr     ::= "#" "[" "prefix" "(" string_lit "," "prec" "=" int_lit ")" "]"
postfix_attr    ::= "#" "[" "postfix" "(" string_lit "," "prec" "=" int_lit ")" "]"
assoc           ::= "left" | "right" | "none"
```

## 6.7.1. 세 속성

```nok
trait Add<Rhs = Self> {
    type Output;
    #[infix("+", prec = 9, left)]
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

| 속성                         | 위치          | 형태                |
| ---------------------------- | ------------- | ------------------- |
| `#[infix(sym, prec, assoc)]` | 트레잇 메서드 | 이항 (`a SYM b`)    |
| `#[prefix(sym, prec)]`       | 트레잇 메서드 | 전위 단항 (`SYM a`) |
| `#[postfix(sym, prec)]`      | 트레잇 메서드 | 후위 단항 (`a SYM`) |

## 6.7.2. 인자

| 인자    | 의미                               | 필수?                       |
| ------- | ---------------------------------- | --------------------------- |
| `sym`   | 연산자 토큰 (문자열 리터럴)        | 필수                        |
| `prec`  | 우선순위 등급 (정수)               | infix, prefix, postfix 모두 |
| `assoc` | 결합성 (`left` / `right` / `none`) | infix 만                    |

prefix 와 postfix 는 단항이므로 결합성이 없다.

```nok
#[infix("+", prec = 9, left)]
#[prefix("-", prec = 11)]
#[postfix("!", prec = 13)]
```

## 6.7.3. 연결 메커니즘

`#[infix("+", prec = 9, left)]` 가 트레잇 메서드 `add` 에 부착되면 다음과 같이 동작한다.

1. `+` 토큰이 그 트레잇의 연결 자리가 된다.
2. 컴파일러는 `a + b` 를 `<typeof(a) as 그_트레잇<typeof(b)>>::add(a, b)` 로 desugar 한다.
3. 우선순위 9, 좌결합으로 [우선순위와 결합성](../003-expressions/302-precedence-and-associativity.md) 의 표에 등록된다.

세 속성 모두 같은 패턴을 따른다. 토큰, 위치, 우선순위가 트레잇 메서드와 결합한다.

## 6.7.4. Unicode 및 다중 alias

같은 메서드에 여러 `#[infix]` 를 부착하면 여러 토큰이 같은 메서드에 매핑된다.

```nok
trait Cmp<Rhs = Self> {
    #[infix("<=", prec = 8, none)]
    #[infix("≤", prec = 8, none)]
    fn le(self, rhs: Rhs) -> bool;

    #[infix(">=", prec = 8, none)]
    #[infix("≥", prec = 8, none)]
    fn ge(self, rhs: Rhs) -> bool;

    #[infix("==", prec = 4, none)]
    #[infix("≡", prec = 4, none)]
    fn eq(self, rhs: Rhs) -> bool;
}
```

- ASCII 토큰과 Unicode 토큰을 같은 의미로 묶기 위함이다.
- 모든 alias 의 우선순위와 결합성이 같아야 한다. 다르면 등록 시점에 충돌한다.

## 6.7.5. 충돌 검사

같은 (토큰, 위치) 조합에 서로 다른 트레잇이 부착되면 등록 시점에 ERROR 가 발생한다.

```nok
trait MyAdd { #[infix("+", prec = 5)] fn add(...) ... }
// 다른 라이브러리 / 모듈:
trait Add { #[infix("+", prec = 9)] fn add(...) ... }
// → '+' 의 등록이 두 곳에서 다른 우선순위로 충돌. ERROR.
```

각 (토큰, 위치) 조합에는 유일한 등록만 허용된다.

## 6.7.6. 같은 토큰의 여러 위치

`-` 같이 전위와 이항 모두 가능한 토큰은 별개의 등록이다.

```nok
trait Sub<Rhs = Self> {
    type Output;
    #[infix("-", prec = 9, left)]
    fn sub(self, rhs: Rhs) -> Output;
}

trait Neg {
    type Output;
    #[prefix("-", prec = 11)]
    fn neg(self) -> Output;
}
```

`-a + b` → 전위 `-` 가 이항 `+` 보다 강하게 묶여서 `(-a) + b`.

이항 `-` 와 전위 `-` 는 별개 매핑이며, 다른 트레잇에 부착되어도 OK 이다.

## 6.7.7. 점-접두 (브로드캐스트) 매핑

[브로드캐스트 연산자](../003-expressions/303-broadcast-operators.md) 의 점-접두 토큰도 같은 매직으로 등록한다.

```nok
trait DotAdd<Rhs = Self> {
    type Output;
    #[infix(".+", prec = 9, left)]
    fn dot_add(self, rhs: Rhs) -> Output;
}

trait DotMul<Rhs = Self> {
    type Output;
    #[infix(".*", prec = 10, left)]
    fn dot_mul(self, rhs: Rhs) -> Output;
}
```

토큰만 다를 뿐 등록 메커니즘은 같다.

## 6.7.8. 매핑의 가시성

`#[infix]` 등의 등록은 그 트레잇이 보이는 곳에서만 효력이 있다. 해당 트레잇이 보이지 않으면 연산자도 작동하지 않는다.

이 규칙은 라이브러리 격리에 자연스럽다. 어떤 패키지의 사용자 정의 연산자 트레잇은 그
패키지를 import 한 코드에서만 작동한다.

표준 라이브러리의 연산자는 `core::prelude` 의 자동 가시 항목 ([표준 라이브러리 계층](../000-meta/005-stdlib-tiers.md)) 으로 항상 활성화되어 있다.

## 6.7.9. 사용자 정의 연산자

사용자도 자기 트레잇에 `#[infix]` 등을 부착해 연산자 매핑을 할 수 있다.

```nok
trait CustomCombine<Rhs = Self> {
    type Output;
    #[infix("⊕", prec = 7, left)]
    fn combine(self, rhs: Rhs) -> Output;
}

impl CustomCombine for MyType { ... }

let r = a ⊕ b;            // 사용자 토큰
```

다만 `⊕` 같은 토큰은 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md) 의 확장 토큰표에 등록되어 있어야 한다.
사용자 정의 연산자가 새 유니코드 토큰을 도입하는 경우, lexer 가 그 토큰을 인식해야 한다.

이 등록 메커니즘 (어휘 토큰 등록) 의 정확한 형태는 표준 라이브러리와 컴파일러 사양에서
추가로 정의한다.

## 6.7.10. 우선순위와 결합성 등급

[우선순위와 결합성](../003-expressions/302-precedence-and-associativity.md) 의 등급표는 표준 라이브러리의 등록으로 만들어진다. 사용자
정의 등록도 같은 표에 들어간다.

```
12   member access (.), method call, indexing
11   prefix unary (-, !, ~)
10   * / %
 9   + -
 8   << >>
 7   &
 6   ^
 5   |
 4   == != < > <= >=
 3   &&
 2   ||
 1   .. ..=
 0   ?? (null 합체)
-1   = += -= *= ... (대입)
```

사용자가 등급을 새로 정할 때는 표준 라이브러리의 등급에 충돌하지 않게 끼워 넣는다.
일반적으로 `7` (비트 AND) 과 `8` (시프트) 사이의 빈 자리에 새 등록을 두는 식이다.

## 6.7.11. 평가 시점

`#[infix]` 등의 등록은 parse 직후의 메타 확장 단계에서 처리된다. 이후 모든 식 파싱은
등록된 표를 참조한다.

따라서 같은 컴파일 단위 안에서 등록과 사용의 순서는 무관하다 (scanner pre-pass 가 미리 모은다).

## 6.7.12. 우선순위가 다른 alias 금지

같은 트레잇 메서드의 여러 alias 는 우선순위와 결합성이 모두 같아야 한다.

```nok
trait Cmp {
    #[infix("<", prec = 8, none)]
    #[infix("less_than", prec = 5, left)]    // ERROR — 다른 우선순위
    fn lt(self, rhs: Self) -> bool;
}
```

이는 alias 의 의미가 같다는 가정에서 비롯한 제약이다.

## 6.7.13. 트레잇 메서드 외 부착 금지

`#[infix]` / `#[prefix]` / `#[postfix]` 는 트레잇의 메서드 시그니처에만 부착할 수 있다. 자유 함수,
`impl` 메서드, 일반 트레잇 메서드 (트레잇 외부) 에는 부착할 수 없다.

```nok
fn my_add(a: i32, b: i32) -> i32 { a + b }
#[infix("+++")]                              // ERROR — 자유 함수에 부착 불가
fn my_concat(a: str, b: str) -> str { ... }
```

이 제약은 연산자 매핑을 트레잇 시스템과 결합시켜 일관된 디스패치를 보장하기 위함이다.

## 6.7.14. 정리

연산자 매직은 `#[lang]` 과 함께 [언어와 표준 라이브러리의 경계](../000-meta/004-language-vs-stdlib.md) 에서 언어 토큰과
라이브러리 트레잇의 결합을 구현하는 핵심 메커니즘이다.

표준 라이브러리의 연산자 사양은 본 매직을 통해 모든 토큰의 의미를 결정한다.
