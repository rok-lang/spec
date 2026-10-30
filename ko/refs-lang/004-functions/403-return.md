# 4.3. 반환

이 절은 함수 본체가 결과 값을 만드는 규칙을 정의합니다.

## 4.3.1. 핵심 규칙: `return` 은 필수가 아님

함수 본체는 [블록 표현식](../003-expressions/309-block-expressions.md)에서 정의하는 블록이며, 본체 마지막 식의 값이 그대로 함수의 반환값이 됩니다.

```rok
fn add(x: i32, y: i32) -> i32 {
    x + y                         // 자동 반환
}
```

`return` 키워드는 조기 종료가 필요할 때만 씁니다. 정상 흐름에서 값 하나를 반환할 때는 마지막 식으로 적는 것이 관례입니다.

## 4.3.2. 명시적 `return`

명시적 반환의 문법은 다음과 같습니다.

```
return_expr     ::= "return" expr?
```

```rok
return expr;
return;
```

`return expr;`은 `expr`의 값을 반환하며 함수를 끝내고, `return;`은 값 없이(unit을 반환하며) 함수를 끝냅니다. `return` 식 자체는 divergent이므로, 그 자리에서 평가 흐름이 함수 밖으로 빠져나갑니다.

```rok
fn early(x: i32) -> i32 {
    if x < 0 { return 0; }        // 조기 종료
    x * 2
}

fn validate(s: str) -> bool {
    if s.len() == 0 { return false; }
    if s.starts_with("#") { return false; }
    true
}
```

## 4.3.3. 반환 타입과 본체의 일치

[함수 선언](401-function-declaration.md)에서 정의한 반환 타입은 본체의 모든 반환 경로와 맞아야 합니다. 반환 경로란 본체의 마지막 식 타입과 모든 `return expr;`의 `expr` 타입입니다.

이 타입들이 서로 다르면 [표현식 개요](../003-expressions/README.md)의 분기 결과 규칙에 따라 합 타입으로 합쳐지며, 그 합 타입이 명시한 반환 타입과 호환되어야 합니다.

```rok
fn maybe_int(cond: bool) -> i32 | str {
    if cond { return 42i32; }
    "no"                          // 두 경로의 합 = i32 | str, 명시와 일치
}
```

## 4.3.4. 반환 타입 생략

[함수 선언](401-function-declaration.md)에서 본 대로 반환 타입은 생략할 수 있으며, 생략하면 본체에서 추론합니다.

```rok
fn double(x: i32) {
    x * 2                          // 추론: i32
}
```

추론한 타입은 모든 값 경로(`return expr`와 마지막 식)의 타입이 같으면 그 타입이고, 다르면 그 합입니다. 값 경로와 값이 없는 경로가 섞이거나 반환 타입이 자기 자신에 의존하면 컴파일 에러입니다. 그래서 재귀 함수에는 `-> T`가 필요합니다. 본체의 끝이나 `return`에 닿는 경로가 하나도 없으면 추론한 타입은 diverging 타입입니다(4.3.7).

## 4.3.5. 다중 반환: 튜플로 묶기

여러 값을 반환하려면 튜플을 씁니다.

```rok
fn divmod(n: i32, d: i32) -> (i32, i32) {
    let q = n / d;
    let r = n % d;
    (q, r)
}

let (quotient, remainder) = divmod(10i32, 3i32);
```

튜플 반환 타입은 [타입 표현식](../002-types/201-type-expressions.md)의 튜플 타입을 따르고, 받는 쪽에서는 [분해 대입](../003-expressions/315-destructuring.md)의 튜플 분해를 쓰면 됩니다.

별도의 다중 반환 키워드나 구문은 두지 않습니다. 튜플로 충분합니다.

## 4.3.6. 결과를 폐기하는 호출

함수 결과를 받지 않을 때는 다음과 같이 씁니다.

```rok
compute();                        // 결과 폐기 (식 + ;)
_ = compute();                    // 의도적 폐기 (명시적)
```

두 형태 모두 결과를 버리지만, `_ = ...`는 결과를 일부러 무시한다는 표시입니다. 도구나 린터가 결과를 무시하면 경고하는 경우, `_ = ...` 형태는 확인했다는 뜻이 되어 그 경고를 끕니다.

## 4.3.7. divergent 함수

본체가 절대 정상 종료하지 않는 함수도 있습니다. 항상 `return`, `throw`, 무한 루프 등으로 흐름이 빠져나가는 함수입니다.

```rok
fn fail_always() throws {
    throw ValueError { msg: "fatal" };
}

fn loop_forever() {
    loop { }
}
```

이런 함수의 반환 타입은 어떤 값도 가질 수 없다는 뜻입니다. 이 타입은 `fn fail() -> ! { ... }`처럼 `!`(never)로 적습니다.

divergent 함수의 호출은 [블록 표현식](../003-expressions/309-block-expressions.md)의 divergent 블록처럼 어떤 타입과도 호환됩니다.

```rok
let x: i32 = if cond {
    42i32
} else {
    fail_always()                 // divergent: i32 자리에서 OK
};
```

## 4.3.8. `return` 의 위치

`return` 식은 함수 본체 안 어디에나 올 수 있습니다. 블록 안, 루프 안, `match` 가지 안 등 어디서든 쓸 수 있습니다.

```rok
fn find(xs: List<i32>, target: i32) -> i32 | none {
    for x in xs {
        if x == target { return x; }   // 루프 안에서 함수 자체 반환
    }
    none
}
```

루프나 `match` 등의 안쪽에 있는 `return`은 둘러싼 함수 자체에서 빠져나갑니다. 루프만 빠져나가려면 [루프](../003-expressions/313-loop-expressions.md)의 `break`를 씁니다.

## 4.3.9. 반환값의 평가 시점

`return expr;`의 `expr`은 `return` 식을 평가할 때 완전히 평가되고, 그다음 함수에서 빠져나갑니다.

```rok
fn f() -> i32 {
    return compute_a() + compute_b();   // 둘 다 평가 후 합 반환
}
```

그래서 `return` 안에 부작용이 있는 식이 있으면 반환 직전에 그 부작용이 일어납니다.

## 4.3.10. `return` 과 try/catch

`try` 본체 안의 `return`도 둘러싼 함수에서 빠져나갑니다. 다만 그 과정에서 [에러 제어 흐름](../003-expressions/314-error-control-flow.md)의 `finally` 본체가 있으면 반드시 평가합니다.

```rok
fn cleanup_aware() -> i32 {
    try {
        return 42i32;             // finally 가 먼저 실행됨
    } finally {
        cleanup();
    }
}
```

위 함수의 경우 `cleanup()`을 호출한 뒤에 `42i32`를 반환합니다.

## 4.3.11. `return` 과 클로저 / 중첩 함수

클로저나 중첩 함수의 `return`은 그 클로저나 함수 자체에서 빠져나갈 뿐, 둘러싼 함수에서 빠져나가지 않습니다.

```rok
fn outer() -> i32 {
    let f = |x: i32| -> i32 {
        if x < 0 { return 0; }   // 이 클로저에서만 빠져나감
        x * 2
    };
    f(-5i32);                     // 이 호출에서 f 가 0 을 반환
    100i32                         // outer 자체는 100 반환
}
```

중첩 함수와 클로저 정의의 자세한 내용은 [클로저](405-closure.md)에서 다룹니다.
