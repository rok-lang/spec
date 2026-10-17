# 3.11. 패턴

패턴은 값을 모양으로 검사하면서 그 부분을 변수에 바인딩하는 구조입니다. [`match` 표현식](312-match-expression.md) 의 가지, [`if` 표현식](310-if-expression.md) 의 `if let`, [분해 대입](315-destructuring.md) 의 `let` 좌변 등 여러 자리에서 등장합니다.

```
pattern         ::= or_pattern
or_pattern      ::= unary_pattern ("|" unary_pattern)*
unary_pattern   ::= literal_pattern
                  | identifier_pattern
                  | "_"                                  // wildcard
                  | type_check_pattern
                  | tuple_pattern
                  | struct_pattern
                  | enum_variant_pattern
                  | range_pattern
                  | binding_pattern
                  | "(" pattern ")"                      // grouping
binding_pattern ::= identifier "@" unary_pattern
type_check_pattern  ::= identifier ":" type_expr
tuple_pattern   ::= "(" (pattern ("," pattern)* ","?)? ")"
struct_pattern  ::= path "{" (field_pattern ("," field_pattern)* ","?)? "}"
field_pattern   ::= identifier (":" pattern)?
enum_variant_pattern ::= path ("(" (pattern | "..") ("," (pattern | ".."))* ","? ")")?   // ".." 는 한 번만
range_pattern   ::= literal_pattern (".." | "..=") literal_pattern
```

## 3.11.1. 패턴 종류

### 리터럴 패턴

리터럴 패턴은 값을 상수와 비교합니다.

```rok
0
3.14f64
"hello"
true
none
'A'
```

[숫자 리터럴](../001-lexical/106-numeric-literals.md) 부터 [진리값과 센티넬 리터럴](../001-lexical/109-other-literals.md) 까지의 모든 리터럴 형식이 패턴 자리에 올 수 있고, 매칭은 동등성 비교 (`==`) 로 이루어집니다. 피검사 식이 합 타입이면 리터럴 / 범위 패턴은 그것이 들어가는 항 타입에 맞춥니다 (`1` 은 `i32 | str` 의 `i32` 항).

### 식별자 패턴 (바인딩)

식별자 패턴은 새 변수를 도입하면서 매칭합니다.

```rok
x
count
name
```

매칭 대상이 무엇이든 항상 매칭에 성공하며, 그 값을 식별자에 바인딩합니다. 다만 [키워드](../001-lexical/105-keywords.md) 의 예약어와 충돌해서는 안 됩니다. 맨 이름이 `use` 로 가져온 `enum` 변형으로 해소되면 바인딩이 아니라 그 변형의 `enum` 패턴이 됩니다 ([5.5.7](../005-modules/505-use.md#enum-변형-가져오기)).

### 와일드카드 패턴: `_`

```rok
_
```

와일드카드 패턴 역시 매칭 대상이 무엇이든 항상 매칭되지만, 값을 바인딩하지 않고 폐기합니다. 사용하지 않을 부분을 명시적으로 무시할 때 사용합니다.

타입을 붙인 와일드카드 `_: T` 는 타입 검사 패턴처럼 값이 `T` 일 때만 매칭되며, 바인딩하지 않습니다.

```rok
match v {
    _: i32 => "int",          // i32 이면 매칭, 바인딩 없음
    _ => "other",
}
```

### 타입 검사 패턴: ` 이름: 타입`

```rok
n: i32
v: User
```

타입 검사 패턴은 좌변의 동적 슬롯이나 합 타입의 값이 지정된 타입일 때만 매칭에 성공합니다. 매칭에 성공하면 그 타입으로 좁혀진 값이 식별자에 바인딩됩니다.

이 패턴은 [`any` 슬롯](../002-types/202-any-slot.md) 과 [합 타입](../002-types/206-sum-types.md) 의 좁히기 메커니즘을 패턴 형태로 표현한 것입니다.

`:` 뒤의 타입은 `type_expr` 전체이므로 `|` 를 포함합니다. `n: i32 | i64` 는 값을 합 `i32 | i64` 로 좁히는 타입 검사 패턴 하나입니다. 타입 검사 패턴들의 or 패턴은 괄호로 적습니다: `(n: i32) | (m: i64)`.

```rok
match v {
    n: i32 => use_int(n),       // v 가 i32 면 매칭
    s: str => use_str(s),       // v 가 str 면 매칭
}
```

### 튜플 패턴

```rok
(a, b)
(_, second)
(first, ..rest)
(..init, last)
(head, ..mid, tail)
(..)
```

`(p)` 는 그룹 패턴으로 `p` 와 같으며 튜플이 아닙니다. 항이 하나인 튜플은 후행 콤마로 `(p,)` 라고 씁니다. 튜플 타입 (`(T,)`) 과 튜플 식 (`(e,)`) 도 같은 규칙입니다.

튜플 패턴은 튜플의 각 항을 부분 패턴으로 매칭합니다. `..` 는 나머지 묶기로, 튜플의 일부를 모두 무시하거나 하나의 변수에 묶을 때 씁니다. `..` 는 한 패턴에서 한 번만 등장할 수 있고, 길이는 매칭 대상과 일치해야 합니다 (`..` 가 있으면 자유롭게 매칭됩니다).

### `enum` 패턴

```rok
Circle(r)
Rectangle(w, h)
Pending
Click { x, y }
Click { x, .. }
Rectangle(..)
Rectangle(w, ..)
```

`enum` 패턴은 변형 이름으로 시작합니다. tuple 변형은 위치 부분 패턴 (`Circle(r)`) 으로, named 변형은 필드 이름과 부분 패턴 (`Click { x, y }`) 으로, unit 변형은 이름만 (`Pending`) 으로 표현합니다. tuple 변형 패턴에는 나머지 `..` 를 한 번 쓸 수 있습니다 (`V(..)`, `V(a, ..)`, `V(.., z)`).

> [!NOTE]
> 다른 언어의 `Option<T>` 자리는 rok 에서 [합 타입](../002-types/206-sum-types.md) 의 `T | none` 으로 표현됩니다. 따라서 `Some(v)` 나 `None` 같은 `enum` 변형 패턴은 없습니다. `T | none` 의 매칭은 타입 검사 패턴 (`v: T`) 과 리터럴 패턴 (`none`) 으로 이루어집니다.

### `struct` 패턴

```rok
Point { x, y }
Point { x, .. }
Point { x: px, y: py }
Point { x, y: 0.0 }      // y 는 리터럴 매칭, x 는 바인딩
```

`struct` 패턴은 `struct` 이름과 `{ ... }` 안의 필드 부분 패턴으로 구성됩니다. 필드 이름과 바인딩 변수 이름이 같으면 단축할 수 있어서 `Point { x }` 는 `Point { x: x }` 와 같습니다. `..` 로 나머지 필드를 무시할 수도 있습니다.

### 타입 패턴 (단순)

```rok
i32
f64
bool
```

피검사 식이 합 타입이나 `any` 이면, 맨 타입 이름 (`f64 =>`, `i32 | str =>`) 은 그 타입인지만 검사하고 아무것도 바인딩하지 않는 타입 패턴입니다. 그 밖의 피검사 식에서는 같은 이름이 식별자 패턴 (바인딩) 입니다.

### 범위 패턴

```rok
1..5
1..=5
'a'..='z'
```

[범위 표현식](307-range-expressions.md) 의 토큰을 패턴 자리에서 그대로 사용하며, 값이 그 범위 안에 들면 매칭에 성공합니다. 정수와 `char` 에 쓸 수 있습니다 (`'a'..='z'`).

### or 패턴

```rok
1 | 2 | 3
"yes" | "y" | "Y"
Pending | Circle(0.0f64)
```

or 패턴은 `|` 로 여러 부분 패턴을 결합한 것으로, 부분 패턴 중 하나라도 매칭되면 전체가 매칭됩니다. 다만 부분 패턴들이 바인딩하는 변수의 이름과 타입은 모두 같아야 합니다.

```rok
match v {
    Ok(x) | OkAlt(x) => use(x),     // 두 변형 모두 같은 이름 x 의 같은 타입에 바인딩
}
```

### 바인딩 패턴: `name @ pat`

```rok
n @ 1..=9
p @ Point { x: 0, .. }
```

`name @ pat` 은 부분 패턴 `pat` 이 매칭되면 값 전체를 `name` 에 바인딩합니다. `pat` 안의 바인딩도 함께 만들어집니다. or 패턴은 괄호로 묶어 `n @ (1 | 2)` 처럼 씁니다. 바인딩 패턴은 `match` 뿐 아니라 `let` 과 `for` 의 패턴에도 쓸 수 있습니다.

```rok
match age {
    n @ 0..=17 => f"minor {n}",
    n          => f"adult {n}",     // 범위는 열린 집합이라 catch-all 필요
}
```

## 3.11.2. 가드

```rok
패턴 if 조건
```

가드는 패턴이 매칭한 후에 추가로 검사하는 `bool` 조건이며, 조건이 `false` 면 그 가지는 매칭 실패로 처리됩니다.

```rok
match (x, y) {
    (a, b) if a == b => "equal",
    (a, b) if a > b  => "first bigger",
    _                => "second bigger",
}
```

가드는 [`if` 표현식](310-if-expression.md) 의 조건과 동일한 의미의 `bool` 타입 표현식입니다.

## 3.11.3. 완전성 검사

완전성 검사는 패턴 집합이 가능한 모든 입력을 덮는지를 확인하는 절차이며, 정확한 규칙은 [`match` 표현식](312-match-expression.md) 에서 정의합니다.

요약하면, 닫힌 합 타입이나 `enum` 의 매칭은 모든 변형을 덮거나, 와일드카드, 식별자 패턴, 타입 패턴으로 나머지를 덮어야 합니다.

## 3.11.4. or 패턴의 변수 바인딩 일관성

```rok
(v: i32) | (_: i32) => v    // ERROR: 한쪽은 바인딩, 한쪽은 무시
(n: i32) | (n: i64) => ...  // ERROR: n 의 타입이 다름. 진단은 n: i32 | i64 를 제안
n: i32 | i64 => ...         // OK: n: i32 | i64
A(x) | B(y) => ...          // ERROR: x / y 가 비대칭
```

or 패턴 안의 모든 부분 패턴은 같은 이름의 변수를 같은 타입으로 바인딩해야 합니다. 한 이름은 모든 대안에서 타입 하나입니다.

## 3.11.5. 패턴 사용 위치 요약

| 위치            | 형태                                    |
|-----------------|-----------------------------------------|
| `match` 가지    | `match v { 패턴 => 식, ... }`           |
| `if let`        | `if let 패턴 = 식 { ... } else { ... }` |
| `let-else`      | `let 패턴 = 식 else { ... };`           |
| `let` 분해 대입 | `let 패턴 = 식;`                        |
| `for` 루프      | `for 패턴 in 반복 { ... }`              |

각 위치에서 어떤 패턴이 허용되는지의 차이는 해당 위치의 사양에서 정의합니다. 일부 위치는 반증 가능 (refutable) 패턴을 허용하고, 일부는 반증 불가능 (irrefutable) 패턴만 허용합니다.

## 3.11.6. 반증 가능과 반증 불가능

반증 가능 (refutable) 패턴은 매칭이 실패할 수 있는 패턴으로 리터럴, `enum` 변형, 타입 검사, 범위 등이 여기에 해당합니다. 반면 반증 불가능 (irrefutable) 패턴은 항상 매칭에 성공하는 패턴이며, 식별자, 와일드카드, 그리고 분해되는 모양이 타입으로 보장된 튜플이나 `struct` 가 여기에 속합니다.

`let` 분해 대입은 반증 불가능 패턴만 허용하지만, `match`, `if let`, `let-else` 는 양쪽 모두 허용합니다.
