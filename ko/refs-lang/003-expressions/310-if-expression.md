# 3.10. `if` 표현식

`if`와 `else`는 진리값에 따라 분기하는 표현식입니다.

## 3.10.1. 형태

```
if_expr     ::= "if" expr block ("else" (if_expr | block))?
              | "if" "let" pattern "=" expr block ("else" (if_expr | block))?
```

```rok
if cond { ... }
if cond { ... } else { ... }
if cond1 { ... } else if cond2 { ... } else { ... }
```

`cond`는 `bool` 타입의 표현식이고, 본체와 `else` 본체는 둘 다 [블록 표현식](309-block-expressions.md)입니다. `else if`는 `else { if ... }`의 줄임입니다. 별도의 토큰이 아니라 두 구문을 자연스럽게 이어 쓴 것입니다.

## 3.10.2. 표현식의 값

`if`가 unit이 아닌 값을 내려면 `else` 절이 있어야 합니다.

```rok
let a = if cond { 1i32 } else { 2i32 };          // OK, i32
let b = if cond { 1i32 };                          // OK, 단 b의 타입은 unit (else 없는 경우)
```

`else`가 없는 `if`는 분기가 한쪽뿐이므로 값을 반환하지 않고, 결과는 unit입니다. `else`가 있는 `if`는 양쪽 분기의 값을 합쳐 결과를 만듭니다.

### 분기 결과 타입

[표현식 개요](README.md)의 분기 결과 규칙에 따라, 모든 분기의 타입이 같으면 그 타입이 `if` 식의 결과 타입입니다. 분기 타입이 서로 다르면 받는 쪽에 명시한 타입이 있어야 합니다. 각 분기는 그 타입과 호환되어야 합니다. 합 타입은 자동으로 추론하지 않습니다. 받는 슬롯의 타입은 [변수 선언](../005-modules/501-variable-declaration.md)의 규칙을 따릅니다.

```rok
// 슬롯에 타입 명시 없음 → 식 결과 타입으로 추론
let x = if cond { 1i32 } else { 2i32 };          // 식 결과 i32, 슬롯 x: i32
let y = if cond { 1i32 } else { 2.0f64 };        // ERROR: 분기 타입 불일치, 타입 명시 필요

// 슬롯에 합 타입 명시 → 항 일치 검사
let w: i32 | f64 = if cond { 1i32 } else { 2.0f64 };   // OK

// 슬롯에 단일 타입 명시 → 식 결과가 그 타입이어야
let z: i32 = if cond { 1i32 } else { 2.0f64 };   // ERROR: f64 분기가 i32 에 안 맞음
```

`else if` 체인의 모든 분기(모든 `else if`와 마지막 `else`)가 이 검사 대상입니다.

## 3.10.3. 조건 표현식의 위치

```rok
if cond { ... }
```

조건 `cond` 자리에는 호출, 메서드 호출, 비교, 논리 결합 등 어떤 `bool` 표현식이든 올 수 있습니다.

```rok
if user.is_admin() && !user.is_locked { ... }
if items.len() > 0 { ... }
```

`bool`이 아닌 타입은 컴파일 에러입니다. 0이나 빈 문자열을 `false`로 보는 암묵적 truthy 변환은 없습니다. 진리값이 필요하면 변환이나 비교를 명시해야 합니다.

```
if x { ... }              // ERROR if x: i32 (i32 는 bool 이 아님)
if x != 0i32 { ... }      // OK
```

### 조건 자리의 struct 리터럴

다음 자리에서는 식의 최상위에 struct 리터럴을 둘 수 없습니다. 그 `{`가 본체의 시작으로 읽히기 때문입니다.

- `if` / `while` / `if let` / `while let`의 조건
- `match`의 피검사 식([3.12](312-match-expression.md))
- `for`의 반복 대상([3.13](313-loop-expressions.md))
- `using`의 자원 식([3.17](317-using-expression.md))

구분자 안(`( )`, `[ ]`, 호출 / 메서드 인자, 배열, 튜플, 인덱스, dict 리터럴, 블록 `{ }`)에서는 쓸 수 있습니다. 최상위에 쓰려면 괄호로 감쌉니다.

```rok
if S { a: 1 }.ok() { .. }       // ERROR: `S` 뒤의 `{` 가 본체
if (S { a: 1 }).ok() { .. }     // OK
if check(S { a: 1 }) { .. }     // OK: 인자 안
```

## 3.10.4. divergent 분기

[블록 표현식](309-block-expressions.md)의 divergent 블록은 분기 결과 타입 검사에서 어떤 타입과도 호환됩니다.

```rok
let x: i32 = if cond {
    42i32
} else {
    return -1i32;          // divergent, i32 자리에서도 OK
};
```

이 규칙 덕분에 조기 종료를 자연스럽게 쓸 수 있습니다.

## 3.10.5. `if let`: 패턴 매칭과 결합

`if`의 조건 자리에 패턴 매칭을 결합할 수 있습니다.

```rok
if let v: User = compute() {       // compute(): User | none
    consume(v);
} else {
    fallback();
}
```

```
if_let ::= "if" "let" pattern (":" type_expr)? "=" expr block ("else" (block | if_expr))?
```

- 타입이 없으면 식의 값을 `pattern`에 매칭합니다.
- `: T`가 있으면 값을 먼저 `T`로 좁힌 뒤 `pattern`으로 분해합니다. 이때 `pattern`은 `T`에 대해 반증 불가능해야 합니다(`if let (a, b): (i32, i32) = v { .. }`).
- 매칭되면 본체를 실행하고, 패턴의 바인딩은 본체 안에서 보입니다. 매칭되지 않으면 `else`를 실행합니다.

`if let`의 패턴 문법은 [패턴](311-patterns.md)에서 정의합니다.

## 3.10.6. chained `if let`(없음)

여러 `if let`을 한 줄로 잇는 Rust RFC 2497 식 체이닝은 채택하지 않습니다.

```
if let A(a) = x && let B(b) = y { ... }     // 채택 안 함
```

대신 [패턴](311-patterns.md)의 튜플 패턴이나 중첩 `if let`을 쓰세요.

## 3.10.7. `else`의 본체 형태 제약

`else` 뒤에는 블록(`{ ... }`)이나 다른 `if` 식만 올 수 있습니다. 일반 식을 바로 둘 수는 없습니다.

```
if a { 1 } else 2          // ERROR: else 뒤에 블록 필요
if a { 1 } else { 2 }      // OK
if a { 1 } else if b { 2 } else { 3 }    // OK, else 뒤 if
```

이 제약은 모호한 dangling `else`를 막습니다.

## 3.10.8. 평가 순서

`if`는 다음 순서로 평가합니다.

1. `cond` 평가.
2. `cond`가 `true`면 `then` 본체 평가, `false`면 `else` 본체 평가.

평가는 단축 평가입니다. `cond`가 정해지는 즉시 한쪽만 평가하고, 다른 쪽의 부작용은 일어나지 않습니다. `else if` 체인은 위에서 아래로 검사하다가 조건이 처음으로 `true`인 분기를 실행합니다.
