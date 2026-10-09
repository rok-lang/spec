# 3.13. 루프 표현식

`loop`, `while`, `for` 의 세 형태로 반복 실행을 표현할 수 있습니다. 셋 모두 표현식이며, 본 문서에서는 각 형태의 문법과 값 평가 규칙을 정의합니다.

```
loop_expr   ::= label? "loop" block
              | label? "while" expr block
              | label? "for" pattern "in" expr block
              | label? "for" "await" pattern "in" expr block
label       ::= "'" identifier ":"
break_stmt  ::= "break" label_target? expr? ";"
continue_stmt ::= "continue" label_target? ";"
label_target ::= "'" identifier
```

> [!NOTE]
> `in` 은 `for pat in` 자리에서만 키워드인 문맥 키워드입니다 ([1.5.4](../001-lexical/105-keywords.md#154-문맥-키워드)).

`for await` 의 의미와 사용처는 [스트림](../004-functions/413-stream.md) 에서 정의합니다. 비동기 스트림에서 한 항목씩 받아 본체를 실행하는 형태입니다.

라벨 (`'name:`) 의 의미는 [3.13.4 라벨](#3134-라벨-중첩-루프-제어) 에서 정의합니다. 라벨이 붙은 **블록** (`'b: { ... }`) 은 [블록 표현식](309-block-expressions.md) 에서 정의합니다.

## 3.13.1. `loop`: 무한 반복

```rok
loop {
    ...
}
```

`loop` 키워드 뒤에 블록 본체를 두는 형태이며, 본체를 무한히 반복합니다. 종료는 `break`, `return`, `throw` 로만 가능합니다.

### `break value` 로 값 반환

```rok
let v = loop {
    let x = compute();
    if x.converged { break x; }
};
```

`break` 뒤에 식을 두면 그 식의 값이 `loop` 표현식의 결과가 됩니다. `break;` 처럼 값 없이 종료하면 `loop` 의 결과는 unit 입니다. 한 `loop` (또는 라벨 붙은 블록) 의 모든 `break` 값은 한 타입이어야 하며, 받는 쪽에 타입 주석이 있으면 그 타입이어야 합니다. `break;` 와 `break v` 를 섞으면 컴파일 에러입니다.

### 결과 타입

`break value` 가 한 번이라도 등장하는 `loop` 는 그 값의 타입이 결과 타입이 됩니다. `break;` 만 있는 `loop` 는 결과 타입이 unit 입니다. `break` 가 전혀 없는 `loop` 는 divergent 로, 절대 끝나지 않습니다.

## 3.13.2. `while`: 조건 기반 반복

```rok
while cond {
    ...
}
```

`while` 키워드 뒤에 bool 조건과 블록 본체를 두는 형태입니다. 매 반복 시작 시 `cond` 를 평가해서 `true` 면 본체를 실행하고, `false` 면 종료합니다.

### 결과 값

`while` 의 결과 값은 항상 unit 입니다.

```rok
let x = while cond { ... };     // x: unit
```

이렇게 결정한 근거는, `while` 이 자연스럽게 끝났을 때 (`cond` 가 `false`) 어떤 값을 반환할지에 자연스러운 선택이 없다는 점입니다. `loop { break v; }` 처럼 명시적 `break value` 를 위한 형태가 아니라는 뜻입니다.

값을 반환하고 싶다면 `loop { ... break v; ... }` 를 사용하면 됩니다.

### `while let`: 패턴 매칭과의 결합

```rok
while let v: T = compute() {     // compute(): T | none
    use(v);
}
```

매 반복마다 식을 평가해서 패턴에 매칭합니다. 매칭에 성공하면 본체를 실행하고, 실패하면 종료합니다. 패턴의 변수 바인딩은 본체에서 가시 영역에 들어옵니다.

[패턴](311-patterns.md) 의 모든 패턴 형태를 사용할 수 있고, 보통 `enum` 이나 합 타입 변형을 매칭할 때 활용합니다.

## 3.13.3. `for`: 반복자 기반 반복

```rok
for 패턴 in 반복식 {
    ...
}
```

`for` 키워드 뒤에 패턴, `in`, 반복식, 블록 본체를 차례로 두는 형태입니다. 반복식은 반복자이거나 반복 가능 (Iterable) 한 값이어야 합니다. 매 반복마다 다음 원소를 패턴에 매칭해서 본체에 바인딩합니다.

### 반복식의 desugar

```
for pat in xs { body }
```

는 의미적으로 다음과 동등합니다.

```
let it = <typeof(xs) as IntoIterator>::into_iter(xs);
loop {
    match <typeof(it) as Iterator>::next(ref it) {     // 반환: T | none
        pat: T => body,
        none   => break,
    }
}
```

(정확한 트레잇과 메서드 이름은 표준 라이브러리에서 정의합니다.)

### 결과 값

`for` 의 결과 값은 항상 unit 입니다 (`while` 과 같은 이유입니다).

### 패턴 자리

[패턴](311-patterns.md) 중 반증 불가능 패턴만 사용할 수 있습니다 (이름, 와일드카드, 튜플 분해 등입니다).

```rok
for x in 0..10 { ... }                    // 단순 이름
for (i, v) in xs.enumerate() { ... }      // 튜플 분해
for Point { x, y } in points { ... }      // struct 분해
```

반증 가능 패턴 (예: `enum` 변형 매칭) 은 매칭 실패 시 처리가 모호해서 `for` 의 패턴 자리에서는 거부됩니다. 그러한 형태가 필요하면 `while let` 이나 `for` 와 `if let` 의 조합을 사용하면 됩니다.

## 3.13.4. 라벨: 중첩 루프 제어

```rok
'outer: loop {
    for x in xs {
        if cond { break 'outer; }
        if other { continue 'outer; }
    }
}
```

루프 앞에 `'이름:` 형태로 라벨을 붙일 수 있습니다. `break 'name` 이나 `continue 'name` 을 사용하면 해당 라벨의 루프를 직접 제어할 수 있습니다.

### 라벨 토큰

라벨은 작은따옴표, 식별자, 콜론으로 이루어집니다 (`'outer:`). 작은따옴표는 [문자 리터럴](../001-lexical/108-character-literals.md) 의 시작과 같은 토큰이지만, 콜론과 결합한 위치에서는 라벨로 인식됩니다. 정확한 토큰화 규칙은 어휘 사양의 보강 정의에 해당하고, 본 문서에서는 언어 수준의 형태만 명시합니다.

### `break label value`

`loop` 라벨에서는 값과 결합할 수 있습니다.

```rok
'outer: loop {
    for x in xs {
        if found(x) { break 'outer x; }
    }
}
```

`break 'outer x` 는 라벨 outer 의 `loop` 에서 값 x 를 들고 빠져나간다는 뜻입니다.

## 3.13.5. `break` 와 `continue` 기본 동작

라벨이 없으면 가장 안쪽 루프를 가리킵니다.

```rok
loop {
    for x in xs {
        if cond { break; }       // for 를 빠져나감
    }
    // 여기서 계속
}
```

## 3.13.6. 평가 순서

각 루프의 평가는 위에서 정의한 의미를 따릅니다. 본체의 식과 문장은 [블록 표현식](309-block-expressions.md) 의 순서대로 평가되며, 부작용도 그 순서를 따릅니다.

루프 종료 조건의 검사는 다음과 같이 이루어집니다. `loop` 는 검사 없이 외부의 `break` 로만 종료됩니다. `while` 은 매 반복 시작 시 `cond` 를 평가합니다. `for` 는 매 반복 시작 시 `Iterator::next` 를 호출합니다.

## 3.13.7. divergent 루프

`break`, `return`, `throw` 등으로도 종료될 가능성이 없는 루프 (예: `loop { ... }` 안에 `break` 가 없고 외부로 빠져나가는 흐름도 없는 경우) 는 divergent 로 분류됩니다. 그러한 루프 표현식의 타입은 임의의 타입과 호환됩니다.
