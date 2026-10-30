# 3.16. `let-else`

`let-else`는 `let`의 좌변에 반증 가능 패턴을 두고, 매칭에 실패했을 때의 발산(divergent) 분기를 함께 쓰는 형태입니다.

## 3.16.1. 형태

문법은 다음과 같습니다.

```
let_else_stmt   ::= "let" pattern (":" type_expr)? "=" expr "else" block ";"
```

```rok
let 패턴 = 식 else { 발산 };
```

`let` 다음에 패턴, `=`와 식, 그리고 `else`와 블록이 옵니다. 반증 가능한 패턴이면 무엇이든 됩니다. `else` 본체는 [블록 표현식](309-block-expressions.md)의 divergent 블록이어야 하므로 `return`, `break`, `continue`, `throw` 등으로 반드시 빠져나가야 합니다.

## 3.16.2. 동작

```rok
let value: T = compute() else {       // compute(): T | none
    return default;
};
// 여기서 value 는 T 타입의 값 (none 이면 위 else 로 빠짐)
```

평가는 다음과 같이 진행됩니다. 식의 결과를 패턴에 매칭합니다. 매칭에 성공하면 패턴의 바인딩이 현재 스코프에 도입되고 다음 줄로 넘어갑니다. 실패하면 `else` 본체를 평가하고, `else`는 divergent이므로 그 자리에서 흐름이 밖으로 나갑니다.

## 3.16.3. divergent 강제

`else` 본체는 반드시 divergent여야 합니다. 정상적으로 끝나는 본체는 컴파일 에러입니다.

```rok
let v: T = compute() else {
    print("no value");          // ERROR: divergent 가 아님
};
```

이 규칙은 매칭에 실패한 뒤 바인딩 변수 없이 평가가 계속되는 함정을 막습니다. `else` 본체가 빠져나가야만 다음 줄에서 바인딩이 있다고 보장할 수 있습니다.

## 3.16.4. 합 타입과 `enum` 의 자연스러운 풀기

가장 흔한 쓰임은 `T | none`이나 `enum` 변형 중 한 경우만 받아들이고, 나머지 경우에는 일찍 빠져나가는 것입니다.

```rok
// 합 타입에서 한 항만 풀기
let n: i32 = x else {                    // x: i32 | f32
    panic("expected i32");
};
// 여기서 n: i32

// enum 변형 풀기
let Ok(data) = parse(input) else {
    return Err(ParseFailed);
};
// data 는 Ok 변형의 내용
```

이 패턴은 `match`보다 수직 흐름이 간결해서, 주 흐름과 예외 경우를 명시적으로 나눌 때 유용합니다.

> [!NOTE]
> `let 식별자 : 타입 = expr else { ... };` 형태의 `: 타입`은 **반증 가능한 type-check 패턴**([패턴](311-patterns.md)의 `type_check_pattern`)으로 해석합니다. 일반 `let`의 슬롯 타입 명시와 달리 매칭에 실패할 수 있으므로 `else` 분기에 의미가 생깁니다. `let n: i32 = x else { ... };`는 `match x { v: i32 => v, _ => { ... } }`와 같으며, `: i32`는 `n`의 슬롯 타입이 아니라 `x`가 통과해야 하는 검사입니다.

## 3.16.5. 가시 범위

매칭에 성공하면 패턴의 바인딩 변수는 `else` 본체 안이 아니라 `let-else`가 있는 스코프에 도입됩니다.

```rok
{
    let v: T = compute() else { return; };
    // 여기서 v 가시
    consume(v);
}
// 여기서는 v 가시 안 됨 (스코프 종료)
```

`else` 본체에서는 바인딩 변수가 보이지 않습니다. 매칭에 실패한 직후라 값이 없기 때문입니다.

## 3.16.6. 가드 (없음)

```
let v: T = expr if v > 0 else { ... };       // 채택 안 함
```

`let-else`에 가드(`if 조건`)를 두는 형태는 채택하지 않습니다. 가드가 필요하면 뒤에서 따로 검사하세요.

```rok
let v: T = compute() else { return; };
if v <= 0 { return; }
```

## 3.16.7. `else` 의 종료 키워드

`else` 본체에 흔히 쓰는 종료 키워드와 그 효과는 다음과 같습니다.

| 키워드                        | 효과                                  |
|-------------------------------|---------------------------------------|
| `return val;`                 | 함수에서 반환                         |
| `break;`와 `break value;`     | 둘러싸는 루프에서 빠져나감            |
| `continue;`                   | 루프의 다음 반복으로                  |
| `throw err;`                  | 예외 발생                             |
| `panic("msg")`                | 라이브러리 panic 함수, 복구 불가 종료 |
| `loop { ... }` (`break` 없음) | divergent 루프                        |

이 중 어느 것이든, 그리고 컴파일러가 divergent로 분류하는 모든 형태를 쓸 수 있습니다.

## 3.16.8. `match` 와의 비교

```rok
// let-else
let v: T = compute() else { return; };       // compute(): T | none
consume(v);

// match 등가
let v = match compute() {
    v: T => v,
    none => return,
};
consume(v);
```

두 코드는 의미가 같습니다. 주 흐름이 평탄해서 더 읽기 쉬울 때 `let-else`를 고르세요.

## 3.16.9. 평가 순서

`let-else`는 다음 순서로 평가합니다.

1. 우변 식을 평가합니다.
2. 결과를 패턴에 매칭합니다.
3. 매칭에 성공하면 바인딩한 뒤 다음 줄로 넘어갑니다.
4. 매칭에 실패하면 `else` 본체를 평가하고, divergent이므로 흐름이 밖으로 나갑니다.

우변 식에 부작용이 있어도 한 번만 평가합니다.
