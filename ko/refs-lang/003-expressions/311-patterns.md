# 3.11. 패턴

패턴은 값을 모양으로 검사하고 부분을 변수에 바인딩하는 구조다. [`match` 표현식](312-match-expression.md) 의 가지, [`if` 표현식](310-if-expression.md) 의 `if let`, [분해 대입](315-destructuring.md) 의 `let` 좌변 등에서 등장한다.

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
type_check_pattern  ::= identifier ":" type_expr
tuple_pattern   ::= "(" (pattern ("," pattern)* ","?)? ")"
struct_pattern  ::= path "{" (field_pattern ("," field_pattern)* ","?)? "}"
field_pattern   ::= identifier (":" pattern)?
enum_variant_pattern ::= path ("(" pattern ("," pattern)* ","? ")")?
range_pattern   ::= literal_pattern (".." | "..=") literal_pattern
```

## 3.11.1. 패턴 종류

### 리터럴 패턴

값을 상수와 비교한다.

```nok
0
3.14f64
"hello"
true
none
'A'
```

[숫자 리터럴](../001-lexical/106-numeric-literals.md) 부터 [진리값과 센티넬 리터럴](../001-lexical/109-other-literals.md) 까지의 모든 리터럴 형식이 패턴 자리에 올 수 있다. 매칭은 동등성 비교 (`==`).

### 식별자 패턴 (바인딩)

새 변수를 도입하면서 매칭한다.

```nok
x
count
name
```

- 매칭 대상이 무엇이든 항상 매칭되며, 그 값을 식별자에 바인딩한다.
- [키워드](../001-lexical/105-keywords.md) 의 예약어와는 충돌하면 안 된다.

### 와일드카드 패턴: `_`

```nok
_
```

- 매칭 대상이 무엇이든 항상 매칭되지만, 바인딩하지 않는다 (값을 폐기).
- 사용하지 않을 부분을 명시적으로 무시할 때 쓴다.

### 타입 검사 패턴: `이름: 타입`

```nok
n: i32
v: User
```

- 좌변의 동적 슬롯 또는 합 타입의 값이 지정된 타입일 때만 매칭.
- 매칭에 성공하면 그 타입으로 좁혀진 값이 식별자에 바인딩된다.

이 패턴은 [`any` 슬롯](../002-types/202-any-slot.md) 과 [합 타입](../002-types/206-sum-types.md) 의 좁히기 메커니즘을 패턴 형태로 표현한다.

```nok
match v {
    n: i32 => use_int(n),       // v 가 i32 면 매칭
    s: str => use_str(s),       // v 가 str 면 매칭
}
```

### 튜플 패턴

```nok
(a, b)
(_, second)
(first, ..rest)
(..init, last)
(head, ..mid, tail)
(..)
```

- 튜플의 각 항을 부분 패턴으로 매칭.
- `..` 는 나머지 묶기. 튜플의 일부를 모두 무시하거나 하나의 변수에 묶는다.
- `..` 는 한 패턴에서 한 번만 등장할 수 있다.
- 길이는 매칭 대상과 일치해야 한다 (`..` 가 있으면 자유 매칭).

### `enum` 패턴

```nok
Circle(r)
Rectangle(w, h)
Pending
Click { x, y }
Click { x, .. }
```

- `enum` 변형 이름으로 시작.
- tuple 변형은 위치 부분 패턴 (`Circle(r)`).
- named 변형은 필드 이름 + 부분 패턴 (`Click { x, y }`).
- unit 변형은 이름만 (`Pending`).

> [!NOTE]
> 다른 언어의 `Option<T>` 자리는 nok 에서 [합 타입](../002-types/206-sum-types.md) 의 `T | none` 으로 표현된다. `Some(v)` 나 `None` 같은 `enum` 변형 패턴이 없다. `T | none` 의 매칭은 타입 검사 패턴 (`v: T`) 과 리터럴 패턴 (`none`) 으로 한다.

### `struct` 패턴

```nok
Point { x, y }
Point { x, .. }
Point { x: px, y: py }
Point { x, y: 0.0 }      // y 는 리터럴 매칭, x 는 바인딩
```

- `struct` 이름 + `{ ... }` + 필드 부분 패턴.
- 필드 이름과 바인딩 변수 이름이 같으면 단축 (`Point { x }` = `Point { x: x }`).
- `..` 로 나머지 필드를 무시.

### 타입 패턴 (단순)

```nok
i32
f64
bool
```

- 값이 해당 타입인지만 검사 (바인딩 안 함).
- 합 타입이나 `any` 슬롯의 분기에 자주 쓴다.

### 범위 패턴

```nok
1..5
1..=5
'a'..='z'
```

[범위 표현식](307-range-expressions.md) 의 토큰을 패턴 자리에서 사용한다. 값이 그 범위 안에 들면 매칭.

### or 패턴

```nok
1 | 2 | 3
"yes" | "y" | "Y"
Pending | Circle(0.0f64)
```

- `|` 로 여러 부분 패턴을 결합.
- 부분 패턴 중 하나라도 매칭되면 전체 매칭.
- 부분 패턴들이 바인딩하는 변수의 이름과 타입이 같아야 한다.

```nok
match v {
    Ok(x) | OkAlt(x) => use(x),     // 두 변형 모두 같은 이름 x 의 같은 타입에 바인딩
}
```

## 3.11.2. 가드

```nok
패턴 if 조건
```

- 패턴이 매칭한 후 추가 `bool` 조건을 검사.
- 조건이 `false` 면 그 가지는 매칭 실패로 처리.

```nok
match (x, y) {
    (a, b) if a == b => "equal",
    (a, b) if a > b  => "first bigger",
    _                => "second bigger",
}
```

가드는 [`if` 표현식](310-if-expression.md) 의 조건과 동일한 의미의 `bool` 타입 표현식이다.

## 3.11.3. 완전성 검사

패턴 집합이 가능한 모든 입력을 덮는지를 검사한다. 정확한 규칙은 [`match` 표현식](312-match-expression.md) 에서 정의한다.

요약: 닫힌 합 타입이나 `enum` 의 매칭은 모든 변형을 덮거나, 와일드카드, 식별자 패턴, 타입 패턴으로 나머지를 덮어야 한다.

## 3.11.4. or 패턴의 변수 바인딩 일관성

```nok
v: i32 | _: i32 => v        // ERROR: 한쪽은 바인딩, 한쪽은 무시
n: i32 | n: i64 => ...      // ERROR: n 의 타입이 i32 인지 i64 인지 모호
A(x) | B(y) => ...          // ERROR: x / y 가 비대칭
```

or 패턴 안의 모든 부분 패턴은 같은 이름의 변수를 같은 타입으로 바인딩해야 한다.

## 3.11.5. 패턴 사용 위치 요약

| 위치            | 형태                                    |
| --------------- | --------------------------------------- |
| `match` 가지    | `match v { 패턴 => 식, ... }`           |
| `if let`        | `if let 패턴 = 식 { ... } else { ... }` |
| `let-else`      | `let 패턴 = 식 else { ... };`           |
| `let` 분해 대입 | `let 패턴 = 식;`                        |
| `for` 루프      | `for 패턴 in 반복 { ... }`              |

각 위치에서 어떤 패턴이 허용되는지의 차이는 해당 위치 사양에서 정의한다. 일부 위치는 반증 가능 (refutable) 패턴을 허용하고, 일부는 반증 불가능 (irrefutable) 만 허용한다.

## 3.11.6. 반증 가능과 반증 불가능

- 반증 가능 (refutable). 매칭이 실패할 수 있는 패턴 (리터럴, `enum` 변형, 타입 검사, 범위 등).
- 반증 불가능 (irrefutable). 항상 매칭하는 패턴 (식별자, 와일드카드, 분해되는 모양이 타입으로 보장된 튜플이나 `struct`).

`let` 분해 대입은 반증 불가능 패턴만 허용한다. `match`, `if let`, `let-else` 는 양쪽 모두 허용한다.
