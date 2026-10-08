# 3.1. 연산자 디스패치

[구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md) 의 연산자 토큰이 트레잇 메서드 호출로 변환되는 일반 원칙을 정의한다. 각 연산자가 어느 트레잇의 어느 메서드에 매핑되는지의 구체 매트릭스는 표준 라이브러리에서 정의한다.

## 3.1.1. 모든 연산자는 트레잇 메서드

[언어와 표준 라이브러리의 경계](../000-meta/004-language-vs-stdlib.md) 에서 정한 바와 같이, 연산자 구문은 언어가 제공하고 동작은 표준 라이브러리의 트레잇이 결정한다.

```
a + b         →   <typeof(a) as Add<typeof(b)>>::add(a, b)
a - b         →   <typeof(a) as Sub<typeof(b)>>::sub(a, b)
a * b         →   <typeof(a) as Mul<typeof(b)>>::mul(a, b)
a < b         →   <typeof(a) as Cmp<typeof(b)>>::lt(a, b)
-a            →   <typeof(a) as Neg>::neg(a)
!a            →   <typeof(a) as Not>::not(a)
```

모든 연산자가 같은 패턴을 따른다. 언어가 토큰을 인식하고, 트레잇 메서드 호출로 변환한다.

## 3.1.2. 연산자 → 트레잇 매핑은 표준 라이브러리에서

각 연산자 토큰이 어떤 트레잇 메서드로 매핑되는지의 정확한 표는 표준 라이브러리에서 정의된다. 본 문서는 매핑이 있다는 사실과 디스패치 메커니즘만 책임진다.

표준 라이브러리는 메타 시스템의 `#[infix]` / `#[prefix]` / `#[postfix]` 속성을 통해 매핑을 트레잇 메서드에 부착한다. 정확한 메커니즘은 [`#[infix]` / `#[prefix]` / `#[postfix]` 매직](../006-meta-system/607-operator-magic.md) 에서 정의한다.

```
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
```

위는 트레잇과 메서드 시그니처의 모양이며, 2장의 트레잇 사양과 6장의 매직 사양이 합쳐서 의미를 만든다. 본 3장은 언어가 토큰을 보고 위 호출로 변환한다는 사실까지 정의한다.

## 3.1.3. 디스패치는 vtable 없이

연산자 호출의 디스패치는 [트레잇-as-타입 금지](../002-types/212-trait-as-type-banned.md) 의 원칙에 따라 vtable 간접 호출 없이 해소된다.

- 정적 슬롯. 컴파일 타임에 트레잇 메서드가 결정되어 직접 함수 호출로 컴파일된다.
- `any` 슬롯. [`any` 슬롯](../002-types/202-any-slot.md) 의 런타임 태그 검사로 트레잇 메서드를 결정한 뒤 직접 호출한다. 비용은 태그 검사 한 번 + 직접 호출.

두 경우 모두 vtable lookup 으로 인한 추가 간접이 없다. 과학·수치 코드의 핵심 루프가 이 비용을 치르지 않도록 하는 것이 [트레잇-as-타입 금지](../002-types/212-trait-as-type-banned.md) 의 동기이며, 그 귀결이 본 디스패치 규칙이다.

## 3.1.4. 매핑이 없는 연산

특정 타입 쌍에 대해 트레잇 구현이 없으면 그 연산자 호출은 컴파일 에러다.

```
let a: i32 = 1i32;
let b: str = "hi";
let c = a + b;          // ERROR: Add 가 (i32, str) 에 구현되지 않음
```

진단은 어떤 트레잇이 빠졌는지 명시한다.

```
ERROR: cannot apply '+' to 'i32' and 'str'
       (no impl Add<str> for i32)
```

## 3.1.5. 단항, 이항, 후위

연산자는 위치에 따라 다음 셋 중 하나로 분류된다.

| 위치                | 형태     | 매핑 메커니즘      |
| ------------------- | -------- | ------------------ |
| 이항 (infix)        | `a OP b` | `#[infix("OP")]`   |
| 전위 단항 (prefix)  | `OP a`   | `#[prefix("OP")]`  |
| 후위 단항 (postfix) | `a OP`   | `#[postfix("OP")]` |

같은 토큰이 여러 위치에 등장할 수 있다. 예를 들어 `-` 은 이항 (`a - b`) 과 전위 단항 (`-a`) 양쪽에 모두 쓰인다. 위치는 토큰이 등장한 문맥으로 결정된다 (피연산자가 앞에 있으면 후위, 양쪽에 있으면 이항 등).

## 3.1.6. 우선순위와 결합

위치별 디스패치 외에, 연산자는 우선순위 (precedence) 와 결합성 (associativity) 을 갖는다. 이들은 [우선순위와 결합성](302-precedence-and-associativity.md) 에서 정의한다.

본 문서에서 알아둘 것은, 우선순위·결합성도 토큰 → 트레잇 매핑의 일부로 표준 라이브러리에 등록된다는 사실이다. `#[infix("+", prec, assoc)]` 형태로 트레잇 메서드에 부착된다.

## 3.1.7. 사용자 정의 연산자

표준 라이브러리는 `#[infix]` / `#[prefix]` / `#[postfix]` 를 통해 기본 연산자에 의미를 부여한다. 사용자 코드도 같은 메커니즘으로 추가 트레잇에 새 매핑을 부착할 수 있다.

```
#[infix("+", prec = 9, left)]
fn my_add(self, rhs: Rhs) -> Output { ... }
```

같은 (토큰, 위치, 타입 쌍) 에 대한 매핑은 유일해야 한다. 충돌 시 컴파일 에러.

[구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md) 의 고정 토큰표에 없는 글자열을 새 연산자로 등록하려면 같은 메커니즘으로 토큰 자체를 등록해야 한다. 정확한 절차는 [`#[infix]` / `#[prefix]` / `#[postfix]` 매직](../006-meta-system/607-operator-magic.md) 에서 정의한다.

## 3.1.8. Unicode 연산자

`≤`, `≥`, `≠`, `⊕`, `∈` 등 ASCII 영역 밖의 연산자는 표준 라이브러리가 같은 트레잇의 다중 alias 로 등록할 수 있다.

```
trait Cmp<Rhs = Self> {
    #[infix("<=")]
    #[infix("≤")]
    fn le(self, rhs: Rhs) -> bool;
}
```

같은 트레잇 메서드에 여러 토큰이 매핑되어 동일한 동작을 한다. 사용자는 둘 중 어느 표기를 써도 같은 결과를 얻는다.
