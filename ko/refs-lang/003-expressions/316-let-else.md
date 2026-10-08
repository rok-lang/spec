# 3.16. `let-else`

`let` 의 좌변에 반증 가능 패턴을 두고, 매칭이 실패할 때의 발산 (divergent) 분기를 동시에 적는 형태.

## 3.16.1. 형태

```
let_else_stmt   ::= "let" pattern (":" type_expr)? "=" expr "else" block ";"
```

```nok
let 패턴 = 식 else { 발산 };
```

- `let` + 패턴 + `=` + 식 + `else` + 블록.
- `else` 본체는 [블록 표현식](309-block-expressions.md) 의 divergent 블록이어야 한다 (`return`, `break`, `continue`, `throw` 등으로 빠져나가는 흐름이어야 한다).

## 3.16.2. 동작

```nok
let value: T = compute() else {       // compute(): T | none
    return default;
};
// 여기서 value 는 T 타입의 값 (none 이면 위 else 로 빠짐)
```

- 식의 결과를 패턴에 매칭.
- 매칭 성공이면 패턴의 바인딩이 현재 스코프에 도입되고, 평가가 다음 줄로 진행된다.
- 매칭 실패면 `else` 본체가 평가된다. `else` 가 divergent 라서 그 시점에 흐름이 빠져나간다.

## 3.16.3. divergent 강제

`else` 본체는 반드시 divergent 여야 한다. 정상 종료하는 본체는 컴파일 에러다.

```nok
let v: T = compute() else {
    print("no value");          // ERROR: divergent 가 아님
};
```

이 강제는 매칭 실패 후 바인딩 변수가 없는 상태로 평가가 진행되는 함정을 차단한다. `else` 본체가 제대로 빠져나가야 다음 줄에서 바인딩이 보장된다.

## 3.16.4. 합 타입과 `enum` 의 자연스러운 풀기

가장 흔한 사용처는 `T | none` 이나 `enum` 변형의 한 가지만 받고 나머지 경우에는 일찍 빠져나가는 패턴이다.

```nok
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

이 패턴은 `match` 보다 수직적 흐름이 깔끔하다. 주된 흐름과 예외를 명시적으로 분리할 수 있다.

## 3.16.5. 가시 범위

매칭 성공 시 패턴의 바인딩 변수는 `let-else` 가 등장한 스코프에 도입된다 (`else` 본체 안이 아니다).

```nok
{
    let v: T = compute() else { return; };
    // 여기서 v 가시
    use(v);
}
// 여기서는 v 가시 안 됨 (스코프 종료)
```

`else` 본체에서는 바인딩 변수가 보이지 않는다. 매칭 실패 후라 값이 없다.

## 3.16.6. 가드 (없음)

```
let v: T = expr if v > 0 else { ... };       // 채택 안 함
```

`let-else` 자리에 가드 (`if 조건`) 를 두는 형태는 도입하지 않는다. 가드가 필요하면 본체 안에서 검사한다.

```nok
let v: T = compute() else { return; };
if v <= 0 { return; }
```

## 3.16.7. `else` 의 종료 키워드

`else` 본체에 흔히 등장하는 종료 키워드와 그 의미:

| 키워드                        | 효과                                  |
| ----------------------------- | ------------------------------------- |
| `return val;`                 | 함수에서 반환                         |
| `break;` 와 `break value;`    | 둘러싸는 루프에서 빠져나감            |
| `continue;`                   | 루프의 다음 반복으로                  |
| `throw err;`                  | 예외 발생                             |
| `panic("msg")`                | 라이브러리 panic 함수, 복구 불가 종료 |
| `loop { ... }` (`break` 없음) | divergent 루프                        |

이 중 무엇이든, 컴파일러가 divergent 로 분류하는 모든 형태가 허용된다.

## 3.16.8. `match` 와의 비교

```nok
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

같은 의미의 두 표현이다. `let-else` 는 주된 흐름이 평탄하여 가독성이 높을 때 선택한다.

## 3.16.9. 평가 순서

`let-else` 의 평가:

1. 우변 식 평가.
2. 결과를 패턴에 매칭.
3. 매칭 성공이면 바인딩 후 다음 줄로 진행.
4. 매칭 실패면 `else` 본체 평가. divergent 이므로 흐름이 빠져나간다.

우변 식에 부작용이 있으면 한 번 평가된다.
