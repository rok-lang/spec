# 4.3. 반환

함수 본체가 결과 값을 만들어내는 규칙을 정의한다.

## 4.3.1. 핵심 규칙: `return` 은 필수가 아님

함수 본체는 [블록 표현식](../003-expressions/309-block-expressions.md) 의 블록이다. 본체 마지막 식의 값이 함수의 반환값이 된다.

```nok
fn add(x: i32, y: i32) -> i32 {
    x + y                         // 자동 반환
}
```

`return` 키워드는 조기 종료가 필요할 때만 쓴다. 정상 흐름의 단일 반환은 마지막 식으로 표현하는 것이 관용.

## 4.3.2. 명시적 `return`

```
return_expr     ::= "return" expr?
```

```nok
return expr;
return;
```

- `return expr;`: `expr` 의 값을 반환하고 함수 종료.
- `return;`: 값 없이 (unit 반환) 함수 종료.
- `return` 식 자체는 divergent. 그 자리에서 평가 흐름이 빠져나간다.

```nok
fn early(x: i32) -> i32 {
    if x < 0 { return 0; }        // 조기 종료
    x * 2
}

fn validate(s: str) -> bool {
    if s.len() == 0 { return false; }
    if !s.is_ascii() { return false; }
    true
}
```

## 4.3.3. 반환 타입과 본체의 일치

[함수 선언](401-function-declaration.md) 에서 정의한 반환 타입은 본체의 모든 반환 경로에서 일치해야 한다.

- 본체의 마지막 식 타입.
- 모든 `return expr;` 의 `expr` 타입.

이들이 서로 다른 타입이면 [표현식 개요](../003-expressions/README.md) 의 분기 결과 규칙에 따라 합 타입으로 합쳐진다. 그 합이 명시 반환 타입과 호환되어야 한다.

```nok
fn maybe_int(cond: bool) -> i32 | str {
    if cond { return 42i32; }
    "no"                          // 두 경로의 합 = i32 | str, 명시와 일치
}
```

## 4.3.4. 반환 타입 생략

[함수 선언](401-function-declaration.md) 에서 본 대로, 반환 타입은 생략 가능. 생략 시 본체에서 추론한다.

```nok
fn double(x: i32) {
    x * 2                          // 추론: i32
}
```

- 단일 경로의 마지막 식 타입.
- 여러 경로 (`return`, 마지막 식) 가 있으면 그들의 합 타입.

추론 결과가 모호하면 명시 타입을 적도록 컴파일 에러.

## 4.3.5. 다중 반환: 튜플로 묶기

여러 값을 반환하려면 튜플을 사용.

```nok
fn divmod(n: i32, d: i32) -> (i32, i32) {
    let q = n / d;
    let r = n % d;
    (q, r)
}

let (quotient, remainder) = divmod(10i32, 3i32);
```

- 튜플 반환 타입은 [타입 표현식](../002-types/201-type-expressions.md) 의 튜플 타입.
- 받는 쪽은 [분해 대입](../003-expressions/315-destructuring.md) 의 튜플 분해.

별도의 다중 반환 키워드 / 구문은 도입하지 않는다. 튜플로 충분.

## 4.3.6. 결과를 폐기하는 호출

함수 결과를 받지 않을 때:

```nok
compute();                        // 결과 폐기 (식 + ;)
_ = compute();                    // 의도적 폐기 (명시적)
```

두 형태 모두 결과를 폐기하지만, `_ = ...` 는 사용자가 의도적으로 결과를 무시한다는 신호가 된다. 도구나 린터가 결과 무시 경고를 두는 경우 `_ = ...` 형태는 그 경고를 우회하는 acknowledgement 로 작동한다.

## 4.3.7. divergent 함수

본체가 절대 정상 종료하지 않는 함수. 항상 `return`, `throw`, 무한 루프로 빠져나간다.

```nok
fn fail_always() {
    throw Error("fatal");
}

fn loop_forever() {
    loop { }
}
```

이런 함수는 반환 타입이 어떤 값도 가질 수 없다는 의미다. 표준 라이브러리에서 이 자리를 위한 발산 타입 (Rust 의 `!` 나 `Never` 에 해당) 을 제공한다. 정확한 이름은 라이브러리 사양에서 정의.

divergent 함수의 호출은 [블록 표현식](../003-expressions/309-block-expressions.md) 의 divergent 블록과 같이 임의의 타입에 호환된다.

```nok
let x: i32 = if cond {
    42i32
} else {
    fail_always()                 // divergent — i32 자리에서 OK
};
```

## 4.3.8. `return` 의 위치

`return` 식은 함수 본체 안 어디에든 등장 가능. 블록 안, 루프 안, `match` 가지 안 등.

```nok
fn find(xs: List<i32>, target: i32) -> i32 | none {
    for x in xs {
        if x == target { return x; }   // 루프 안에서 함수 자체 반환
    }
    none
}
```

루프나 `match` 등 안쪽에 있는 `return` 은 둘러싸는 함수 자체에서 빠져나간다. 루프만 빠져나가려면 [루프](../003-expressions/313-loop-expressions.md) 의 `break`.

## 4.3.9. 반환값의 평가 시점

`return expr;` 의 `expr` 은 `return` 식이 평가될 때 완전히 평가된 뒤 함수에서 빠져나간다.

```nok
fn f() -> i32 {
    return compute_a() + compute_b();   // 둘 다 평가 후 합 반환
}
```

부작용이 있는 식이 `return` 안에 있으면 반환 직전에 그 부작용이 수행된다.

## 4.3.10. `return` 과 try/catch

`try` 본체 안의 `return` 도 둘러싸는 함수에서 빠져나간다. 그 과정에서 [에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 `finally` 본체가 있으면 반드시 평가된다.

```nok
fn cleanup_aware() -> i32 {
    try {
        return 42i32;             // finally 가 먼저 실행됨
    } finally {
        cleanup();
    }
}
```

위 함수는 `cleanup()` 을 호출한 뒤 `42i32` 를 반환한다.

## 4.3.11. `return` 과 클로저 / 중첩 함수

클로저나 중첩 함수의 `return` 은 그 클로저나 함수 자체에서 빠져나간다 (둘러싸는 함수가 아님).

```nok
fn outer() -> i32 {
    let f = |x: i32| -> i32 {
        if x < 0 { return 0; }   // 이 클로저에서만 빠져나감
        x * 2
    };
    f(-5i32);                     // 이 호출에서 f 가 0 을 반환
    100i32                         // outer 자체는 100 반환
}
```

중첩 함수와 클로저의 정의는 [클로저](405-closure.md) 에서 다룬다.
