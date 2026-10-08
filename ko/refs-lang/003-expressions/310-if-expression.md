# 3.10. `if` 표현식

`if` 와 `else` 는 진리값에 따라 분기하는 표현식입니다.

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

여기서 `cond` 는 `bool` 타입의 표현식이고, 본체와 `else` 본체는 모두 [블록 표현식](309-block-expressions.md)입니다. `else if` 는 `else { if ... }` 의 줄임으로, 별도의 토큰이 아니라 자연스러운 결합으로 보면 됩니다.

## 3.10.2. 표현식으로서의 값

`if` 가 값으로 사용되려면 `else` 절이 있어야 합니다.

```rok
let a = if cond { 1i32 } else { 2i32 };          // OK, i32
let b = if cond { 1i32 };                          // OK, 단 b 의 타입이 unit (else 없는 경우)
```

`else` 가 없는 `if` 는 한쪽 분기만 있는 셈이라 값을 반환하지 않는 형태로 처리되며, 그 결과는 unit 이 됩니다. 반면에 `else` 가 있는 `if` 는 양쪽 분기의 값이 합쳐져 결과를 만들어 냅니다.

### 분기 결과 타입

[표현식 개요](README.md) 의 분기 결과 규칙에 따라, `if` 식의 결과 타입은 분기들의 합 타입이 됩니다. 받는 슬롯의 타입은 별개로 다루어지며, [`any` 슬롯](../002-types/202-any-slot.md) 의 변수 선언 규칙을 따르게 됩니다.

```rok
// 슬롯에 타입 명시 없음 → any
let x = if cond { 1i32 } else { 2i32 };          // 식 결과 i32, 슬롯 x: any
let y = if cond { 1i32 } else { 2.0f64 };        // 식 결과 i32|f64, 슬롯 y: any

// 슬롯에 합 타입 명시 → 항 일치 검사
let w: i32 | f64 = if cond { 1i32 } else { 2.0f64 };   // OK

// 슬롯에 단일 타입 명시 → 식 결과가 그 타입이어야
let z: i32 = if cond { 1i32 } else { 2.0f64 };   // ERROR: 식 결과 i32|f64
```

`else if` 체인의 모든 분기, 즉 모든 `else if` 와 마지막 `else` 가 식의 결과 합 타입의 항이 됩니다.

## 3.10.3. 조건 표현식의 위치

```rok
if cond { ... }
```

조건 `cond` 자리에는 임의의 `bool` 표현식이 올 수 있습니다. 호출이나 메서드 호출, 비교, 논리 결합 등이 모두 가능합니다.

```rok
if user.is_admin() && !user.is_locked { ... }
if items.len() > 0 { ... }
```

`bool` 이 아닌 타입은 컴파일 에러로 처리됩니다. 0 이나 빈 문자열 등을 `false` 로 보는 식의 암묵적 truthy 변환은 도입하지 않습니다. 진리값이 필요하면 명시적 변환이나 비교를 적어야 합니다.

```
if x { ... }              // ERROR if x: i32 (i32 는 bool 이 아님)
if x != 0i32 { ... }      // OK
```

## 3.10.4. divergent 분기

[블록 표현식](309-block-expressions.md) 의 divergent 블록은 분기 결과 타입 검사에서 임의의 타입과 호환됩니다.

```rok
let x: i32 = if cond {
    42i32
} else {
    return -1i32;          // divergent, i32 자리에서도 OK
};
```

이 규칙 덕분에 조기 종료를 자연스럽게 표현할 수 있습니다.

## 3.10.5. `if let`: 패턴 매칭과의 결합

`if` 의 조건 자리에 패턴 매칭을 결합한 형태도 있습니다.

```rok
if let v: User = compute() {       // compute(): User | none
    use(v);
} else {
    fallback();
}
```

형식은 `if let 패턴 = 식 { ... } else { ... }` 의 모양입니다. 식의 결과가 패턴에 매치되면 본체가 실행되고, 패턴이 만들어 낸 변수 바인딩은 본체 안에서 가시 범위에 들어옵니다. 매치되지 않은 경우에는 `else` 본체가 실행됩니다.

`if let` 의 패턴 문법은 [패턴](311-patterns.md) 에서 정의하고 있습니다.

## 3.10.6. chained `if let` (없음)

여러 `if let` 을 한 줄에 묶는 체이닝, 즉 Rust RFC 2497 식의 형태는 도입하지 않습니다.

```
if let A(a) = x && let B(b) = y { ... }     // 채택 안 함
```

대안으로는 [패턴](311-patterns.md) 의 튜플 패턴이나 중첩 `if let` 을 사용하면 됩니다.

## 3.10.7. `else` 의 본체 형태 제약

`else` 뒤에는 블록 (`{ ... }`) 또는 다른 `if` 식만 올 수 있습니다. 일반 식을 곧바로 두는 형태는 허용하지 않습니다.

```
if a { 1 } else 2          // ERROR: else 뒤에 블록 필요
if a { 1 } else { 2 }      // OK
if a { 1 } else if b { 2 } else { 3 }    // OK, else 뒤 if
```

이 제약은 모호한 dangling `else` 를 차단하기 위한 것입니다.

## 3.10.8. 평가 순서

`if` 의 평가는 다음과 같이 진행됩니다.

1. `cond` 평가.
2. `cond` 가 `true` 면 `then` 본체 평가, `false` 면 `else` 본체 평가.

평가는 단축 평가입니다. `cond` 가 결정된 즉시 한쪽만 평가되고, 다른 쪽의 부작용은 일어나지 않습니다. `else if` 체인은 위에서 아래로 검사하다가 처음 `true` 가 되는 분기를 실행하게 됩니다.
