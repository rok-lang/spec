# 4.3. 반환

이 절은 함수 본체가 결과 값을 만드는 규칙을 정의합니다.

## 4.3.1. 핵심 규칙: `return`은 필수가 아님

함수 본체는 [블록 표현식](../003-expressions/309-block-expressions.md)에서 정의하는 블록입니다. 본체 마지막 식의 값이 그대로 함수의 반환값이 됩니다.

```rok
fn add(x: i32, y: i32) -> i32 {
    x + y                         // 자동 반환
}
```

`return` 키워드는 조기 종료가 필요할 때만 씁니다. 정상 흐름에서 값 하나를 반환할 때는 관례대로 마지막 식으로 적습니다.

## 4.3.2. 명시적 `return`

명시적 반환의 문법은 다음과 같습니다.

```
return_expr     ::= "return" expr?
```

```rok
return expr;
return;
```

`return expr;`은 `expr`의 값을 반환하고 함수를 끝냅니다. `return;`은 값 없이 unit을 반환하고 함수를 끝냅니다. `return` 식은 divergent 식입니다. 평가 흐름은 그 자리에서 함수 밖으로 빠져나갑니다.

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

[표현식 개요](../003-expressions/README.md#분기-결과-타입)의 분기 결과 규칙처럼 명시한 반환 타입이 모든 경로의 받는 타입입니다. 각 경로는 그 타입과 호환되어야 하고, 명시한 타입이 각 경로의 타입을 받는 합 타입이면 경로마다 타입이 달라도 됩니다.

```rok
fn maybe_int(cond: bool) -> i32 | str {
    if cond { return 42i32; }
    "no"                          // 각 경로가 명시한 i32 | str에 맞음
}
```

## 4.3.4. 반환 타입 생략

[함수 선언](401-function-declaration.md)에서 본 대로 반환 타입은 생략할 수 있습니다. 생략하면 컴파일러가 본체에서 추론합니다. `pub`나 `pub(pkg)` 함수에서는 `->`를 생략하면 대신 `()`입니다([4.1.3](401-function-declaration.md#413-반환-타입-생략)).

```rok
fn double(x: i32) {
    x * 2                          // 추론: i32
}
```

추론한 타입은 모든 값 경로(`return expr`와 마지막 식)의 타입이며, 그 타입들은 같아야 합니다. 타입이 다른 경로에 합 타입을 지어내지 않으므로 컴파일 에러(E0041)이고, help는 반환 타입(`-> i32 | str`)을 쓰라고 합니다. 값 경로와 값이 없는 경로가 섞이거나 반환 타입이 자기 자신에 의존하면 컴파일 에러입니다. 그래서 재귀 함수에는 `-> T`가 필요합니다. 본체의 끝이나 `return`에 닿는 경로가 하나도 없으면 추론한 타입은 diverging 타입입니다(4.3.7).

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

튜플 반환 타입은 [타입 표현식](../002-types/201-type-expressions.md)의 튜플 타입을 따릅니다. 받는 쪽에서는 [분해 대입](../003-expressions/315-destructuring.md)의 튜플 분해를 씁니다.

별도의 다중 반환 키워드나 구문은 두지 않습니다. 튜플로 충분합니다.

## 4.3.6. 결과를 폐기하는 호출

함수 결과를 받지 않을 때는 다음과 같이 씁니다.

```rok
compute();                        // 결과 폐기 (식 + ;)
_ = compute();                    // 의도적 폐기 (명시적)
```

두 형태 모두 결과를 버립니다. 다만 `_ = ...`는 결과를 일부러 무시한다는 표시입니다. 도구나 린터가 무시한 결과를 경고할 때 `_ = ...` 형태는 확인했다는 뜻이 되어 그 경고를 끕니다.

## 4.3.7. divergent 함수

본체가 절대 정상 종료하지 않는 함수도 있습니다. 항상 `return`, `throw`, 무한 루프 등으로 흐름이 빠져나가는 함수입니다.

```rok
fn fail_always() throws {
    throw ValueError::new("fatal");
}

fn loop_forever() {
    loop { }
}
```

이런 함수의 반환 타입에는 값이 하나도 없습니다. 이 타입은 `fn fail() -> ! { ... }`처럼 `!`(never)로 적습니다.

divergent 함수의 호출은 [블록 표현식](../003-expressions/309-block-expressions.md)의 divergent 블록처럼 어떤 타입과도 호환됩니다.

```rok
let x: i32 = if cond {
    42i32
} else {
    fail_always()                 // divergent: i32 자리에서 OK
};
```

## 4.3.8. `return`의 위치

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

## 4.3.10. `return`과 try/catch

`try` 본체 안의 `return`도 둘러싼 함수에서 빠져나갑니다. 다만 그 과정에서 [에러 제어 흐름](../003-expressions/314-error-control-flow.md)의 `finally` 본체가 있으면 빠져나가는 길에 그 본체를 항상 평가합니다.

```rok
fn cleanup_aware() -> i32 {
    try {
        return 42i32;             // finally 가 먼저 실행됨
    } finally {
        cleanup();
    }
}
```

위 함수는 `cleanup()`을 호출한 뒤 `42i32`를 반환합니다.

## 4.3.11. `return`과 클로저 / 중첩 함수

클로저나 중첩 함수의 `return`은 그 클로저나 함수 자체에서 빠져나갑니다. 둘러싼 함수에서 빠져나가지는 않습니다.

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

문장인 호출의 [트레일링 클로저](408-trailing-closure.md) 안에 값 없이 쓴 `return;`은 함수에서 빠져나가는 것처럼 읽히지만, 실제로는 그 클로저 호출 한 번만 끝내므로 컴파일 에러입니다. 클로저 호출 한 번을 끝내려면 레이블 붙은 return인 `return@each`(4.3.12)를 씁니다. 함수에서 빠져나가려면 호출 대신 `for` 루프를 씁니다. help는 `return@each`를 수정 제안과 함께 알려 줍니다.

```rok
fn main() {
    each(xs) |x| {
        if x == 2 { return; }     // 에러: 이 호출만 끝내게 됨
        println(f"{x}");
    }
    for x in xs {
        if x == 2 { return; }     // main 에서 빠져나감
        println(f"{x}");
    }
}
```

괄호 안에 넘긴 클로저, 호출이 문장이 아닌 트레일링 클로저(`let n = first(xs) |x| { .. }`), 트레일링 클로저 안에 중첩된 클로저의 `return`은 그대로 허용합니다. 그 클로저 자신의 `return`으로 읽히기 때문입니다.

중첩 함수와 클로저 정의의 자세한 내용은 [클로저](405-closure.md)에서 다룹니다.

## 4.3.12. 레이블 붙은 return

`return@name`은 가장 안쪽 클로저가 `name` 호출의 트레일링 클로저일 때 그 클로저 호출 한 번을 끝냅니다. 레이블은 호출 대상 경로의 마지막 마디나 메서드 이름입니다. `each(xs) |x| { .. }`와 `m::each(xs) |x| { .. }`는 `@each`, `xs.visit() |x| { .. }`는 `@visit`을 씁니다. `@`는 `return`에 띄우지 않고 붙여 씁니다. 값은 `return`처럼 뒤에 씁니다(`return@first true`).

```
labeled_return ::= "return" "@" identifier expression?
```

```rok
fn main() {
    each(xs) |x| {
        if x == 2 { return@each; }    // 이 호출만 끝냄. 3 은 그대로 출력
        println(f"{x}");
    }
    let n = first(xs) |x| { if x > 1 { return@first true; } false };
}
```

레이블이 가장 안쪽 클로저를 가리키지 않으면 컴파일 에러입니다. 클로저는 자기를 둘러싼 다른 클로저에서 빠져나갈 수 없습니다. 중첩 클로저, `async` / `yield` 블록, 중첩 함수는 각각 `return`의 범위를 새로 시작합니다.

```rok
each(xs) |x| {
    let g = || { return@each; };      // 에러: 가장 안쪽 클로저는 g
}
```
