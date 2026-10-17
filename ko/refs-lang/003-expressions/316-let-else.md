# 3.16. `let-else`

`let-else` 는 `let` 의 좌변에 반증 가능 패턴을 두고, 매칭이 실패할 때의 발산 (divergent) 분기를 동시에 적는 형태입니다.

## 3.16.1. 형태

문법은 다음과 같이 정의됩니다.

```
let_else_stmt   ::= "let" pattern (":" type_expr)? "=" expr "else" block ";"
```

```rok
let 패턴 = 식 else { 발산 };
```

구성을 살펴보면 `let` 다음에 패턴이 오고, `=` 으로 식을 연결한 뒤, `else` 와 블록이 따라옵니다. 이때 `else` 본체는 [블록 표현식](309-block-expressions.md) 의 divergent 블록이어야 하므로, `return`, `break`, `continue`, `throw` 등으로 빠져나가는 흐름을 반드시 갖추어야 합니다.

## 3.16.2. 동작

```rok
let value: T = compute() else {       // compute(): T | none
    return default;
};
// 여기서 value 는 T 타입의 값 (none 이면 위 else 로 빠짐)
```

평가 과정은 다음과 같이 진행됩니다. 먼저 식의 결과를 패턴에 매칭하며, 매칭이 성공하면 패턴의 바인딩이 현재 스코프에 도입되고 평가가 다음 줄로 진행됩니다. 반대로 매칭이 실패하면 `else` 본체가 평가되는데, `else` 가 divergent 이기 때문에 그 시점에 흐름이 밖으로 나가게 됩니다.

## 3.16.3. divergent 강제

`else` 본체는 반드시 divergent 여야 합니다. 정상 종료하는 본체는 컴파일 에러로 처리됩니다.

```rok
let v: T = compute() else {
    print("no value");          // ERROR: divergent 가 아님
};
```

이러한 강제는 매칭 실패 후에도 바인딩 변수가 없는 상태로 평가가 진행되는 함정을 차단하기 위한 장치입니다. `else` 본체가 제대로 빠져나가야 다음 줄에서 바인딩의 존재가 보장될 수 있습니다.

## 3.16.4. 합 타입과 `enum` 의 자연스러운 풀기

가장 흔한 사용처는 `T | none` 이나 `enum` 변형 가운데 한 가지만 받아들이고 나머지 경우에는 일찍 빠져나가는 패턴입니다.

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

이 패턴은 `match` 보다 수직적 흐름이 간결하다는 장점이 있어, 주된 흐름과 예외를 명시적으로 분리하고자 할 때 유용합니다.

> [!NOTE]
> `let 식별자 : 타입 = expr else { ... };` 형태에서의 `: 타입` 은 **반증 가능한 type-check 패턴** ([패턴](311-patterns.md) 의 `type_check_pattern`) 으로 해석됩니다. 일반 `let` 의 슬롯 타입 명시와 달리 매칭 실패가 가능하기 때문에 `else` 분기가 의미를 갖게 됩니다. 즉 `let n: i32 = x else { ... };` 는 `match x { v: i32 => v, _ => { ... } }` 와 동등하며, `: i32` 는 `n` 의 슬롯 타입이 아니라 `x` 가 통과해야 하는 검사를 나타냅니다.

## 3.16.5. 가시 범위

매칭이 성공하는 경우, 패턴의 바인딩 변수는 `let-else` 가 등장한 스코프에 도입됩니다. `else` 본체 안이 아니라는 점에 유의해야 합니다.

```rok
{
    let v: T = compute() else { return; };
    // 여기서 v 가시
    use(v);
}
// 여기서는 v 가시 안 됨 (스코프 종료)
```

`else` 본체에서는 바인딩 변수가 보이지 않는데, 매칭에 실패한 직후이므로 값이 존재하지 않기 때문입니다.

## 3.16.6. 가드 (없음)

```
let v: T = expr if v > 0 else { ... };       // 채택 안 함
```

`let-else` 자리에 가드 (`if 조건`) 를 두는 형태는 도입하지 않습니다. 가드가 필요한 경우라면 본체 안에서 별도로 검사하면 됩니다.

```rok
let v: T = compute() else { return; };
if v <= 0 { return; }
```

## 3.16.7. `else` 의 종료 키워드

`else` 본체에 흔히 등장하는 종료 키워드와 그 의미는 다음 표와 같습니다.

| 키워드                        | 효과                                  |
|-------------------------------|---------------------------------------|
| `return val;`                 | 함수에서 반환                         |
| `break;` 와 `break value;`    | 둘러싸는 루프에서 빠져나감            |
| `continue;`                   | 루프의 다음 반복으로                  |
| `throw err;`                  | 예외 발생                             |
| `panic("msg")`                | 라이브러리 panic 함수, 복구 불가 종료 |
| `loop { ... }` (`break` 없음) | divergent 루프                        |

이 가운데 무엇이든, 컴파일러가 divergent 로 분류하는 모든 형태가 허용됩니다.

## 3.16.8. `match` 와의 비교

```rok
// let-else
let v: T = compute() else { return; };       // compute(): T | none
use(v);

// match 등가
let v = match compute() {
    v: T => v,
    none => return,
};
use(v);
```

두 코드는 같은 의미를 표현합니다. 그중 `let-else` 는 주된 흐름이 평탄하여 가독성이 더 높을 때 선택하면 좋습니다.

## 3.16.9. 평가 순서

`let-else` 의 평가는 다음 순서로 이루어집니다.

1. 우변 식을 평가합니다.
2. 결과를 패턴에 매칭합니다.
3. 매칭이 성공하면 바인딩한 뒤 다음 줄로 진행합니다.
4. 매칭이 실패하면 `else` 본체를 평가하며, divergent 이므로 흐름이 밖으로 나갑니다.

우변 식에 부작용이 있는 경우에는 한 번만 평가됩니다.
