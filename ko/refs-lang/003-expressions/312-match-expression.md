# 3.12. `match` 표현식

`match` 는 값을 여러 패턴에 비교하여, 처음 매칭하는 가지를 평가하는 표현식이다.

## 3.12.1. 형태

```
match_expr  ::= "match" expr "{" match_arm ("," match_arm)* ","? "}"
match_arm   ::= pattern ("if" expr)? "=>" expr
```

```nok
match expr {
    패턴1 => 식1,
    패턴2 => 식2,
    ...
    패턴N => 식N,
}
```

- `match` 키워드 + 피검사 식 + `{ ... }`.
- 본체에 0 개 이상의 가지 (arm) 가 콤마로 구분되어 등장한다.
- 각 가지는 패턴 + `=>` 토큰 ([구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)) + 식.
- 마지막 후행 콤마 허용.

## 3.12.2. 가지의 본체

```nok
match v {
    1 => "one",
    2 => "two",
    _ => "other",
}
```

- `=>` 뒤에 단일 식이 오는 형태.
- 식이 [블록 표현식](309-block-expressions.md) 이면 여러 문장을 두기 자연스럽다.

```nok
match v {                        // v: i32 | none
    n: i32 => {
        let x = n * 2;
        x + 1
    },
    none => 0,
}
```

블록 가지의 마지막 식 값이 그 가지의 결과다.

## 3.12.3. 매칭 순서

가지는 위에서 아래로 검사되며, 처음 매칭하는 가지가 평가된다. 그 가지의 식 값이 `match` 표현식의 결과다.

## 3.12.4. 분기 결과 타입

[표현식 개요](README.md) 의 분기 결과 규칙에 따라, `match` 식의 결과 타입은 모든 가지 결과 식의 합 타입이 된다. 받는 슬롯의 타입은 별개이며 [`any` 슬롯](../002-types/202-any-slot.md) 의 변수 선언 규칙을 따른다.

```nok
// 슬롯에 타입 명시 없음 → any
let x = match v {
    1 => "one",
    _ => "other",
};                      // 식 결과 str, 슬롯 x: any

let y = match v {
    1 => 1i32,
    _ => "other",
};                      // 식 결과 i32|str, 슬롯 y: any

// 슬롯에 단일 타입 명시 → 식 결과가 그 타입이어야
let z: i32 = match v {
    1 => 1i32,
    _ => "other",
};                      // ERROR: 식 결과 i32|str
```

## 3.12.5. 완전성 검사 (exhaustiveness)

가지의 패턴 집합이 피검사 식의 가능한 모든 값을 덮어야 한다. 그렇지 않으면 컴파일 에러다.

```nok
let x: bool = ...;
match x {
    true => 1,
    // ERROR: false 가 빠짐
}
```

검사 규칙:

- 닫힌 합 타입 (`A | B`) 의 모든 항을 덮으면 완전.
- `enum` 의 모든 변형을 덮으면 완전.
- `bool` 의 `true` 와 `false` 둘 다 덮으면 완전.
- 원시 정수, 문자열, 부동소수. 가능한 값이 무한이므로, 반드시 와일드카드 (`_`) 또는 식별자 패턴 (`x`) 같은 catch-all 가지로 덮어야 완전하다.

[`any` 슬롯](../002-types/202-any-slot.md) 의 동적 슬롯에 대한 매칭은 어떤 타입이 들어올지 컴파일러가 모르므로 항상 catch-all 가지가 필요하다.

## 3.12.6. 도달 불가능 가지

매칭 순서상 절대 도달할 수 없는 가지는 컴파일 에러 (또는 강한 경고) 다.

```nok
match v {
    _ => 0,
    1 => 1,         // ERROR: 위의 _ 가 항상 매칭
}
```

## 3.12.7. or 패턴

[패턴](311-patterns.md) 의 or 패턴 (`|`) 으로 같은 본체를 갖는 여러 패턴을 묶을 수 있다.

```nok
match v {
    1 | 2 | 3 => "small",
    4 | 5 | 6 => "medium",
    _         => "large",
}
```

## 3.12.8. 가드

[패턴](311-patterns.md) 의 가드 (`if 조건`) 가 가지에 부착될 수 있다.

```nok
match (x, y) {
    (a, b) if a == b => "equal",
    (a, b) if a > b  => "first bigger",
    _                => "second bigger",
}
```

가드가 있는 가지는 완전성 검사에서 catch-all 로 인정되지 않는다. 가드가 `false` 일 수 있기 때문이다. 따라서 가드가 있는 가지만으로는 완전성이 보장되지 않는다.

```nok
let x: bool = ...;
match x {
    a if a == true  => 1,
    a if a == false => 2,
    // ERROR: 가드 가지만으로는 완전성이 보장되지 않음, _ 필요
}
```

## 3.12.9. 매칭 결과의 좁히기

타입 패턴이나 `enum` 패턴이 매칭한 가지 안에서, 컴파일러는 피검사 식의 타입을 좁힌다.

```nok
let v: i32 | str = ...;
match v {
    n: i32 => use_int(n),       // 이 가지에서 n: i32, 정적 디스패치 가능
    s: str => use_str(s),
}
```

이 좁히기는 [`any` 슬롯](../002-types/202-any-slot.md) 과 [합 타입](../002-types/206-sum-types.md) 의 좁히기와 같은 메커니즘이다.

## 3.12.10. 평가 순서

`match` 의 평가는:

1. 피검사 식 평가 (한 번만).
2. 가지를 위에서 아래로 검사.
3. 처음 매칭하는 가지의 식 평가.

피검사 식에 부작용이 있으면 그 부작용은 한 번만 발생한다.

가드가 있는 가지의 가드는 그 가지에 도달했을 때만 평가된다.

## 3.12.11. 매칭 식 자체의 결과

`match` 표현식 자체의 결과는 위 분기 결과 규칙에 따라 모든 가지의 합 타입이다 (가지가 모두 같은 타입이면 그 타입, 다르면 합 타입).

피검사 식이 divergent 면 (예: `match return 0 { ... }`) `match` 도 divergent 다.

## 3.12.12. divergent 가지

가지의 식이 [블록 표현식](309-block-expressions.md) 의 divergent 형태 (`return`, `break`, `throw` 등) 면, 그 가지는 분기 결과 타입 검사에서 임의의 타입과 호환된다.

```nok
let x: i32 = match v {           // v: i32 | none
    n: i32  => n,
    none    => return 0,         // divergent, i32 자리에서 OK
};
```
