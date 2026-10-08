# 3.13. 루프 표현식

`loop` / `while` / `for` 의 세 형태는 반복 실행을 표현한다. 셋 모두 표현식이며, 본 문서는 각 형태의 문법과 값 평가 규칙을 정의한다.

```
loop_expr   ::= "loop" block
              | "while" expr block
              | "for" pattern "in" expr block
break_stmt  ::= "break" expr? ";"
continue_stmt ::= "continue" ";"
```

## 3.13.1. `loop`: 무한 반복

```nok
loop {
    ...
}
```

- `loop` 키워드 + 블록 본체.
- 본체를 무한히 반복한다. 종료는 `break`, `return`, `throw` 로만 가능하다.

### `break value` 로 값 반환

```nok
let v = loop {
    let x = compute();
    if x.converged { break x; }
};
```

- `break` 뒤에 식을 두면 그 식의 값이 `loop` 표현식의 결과다.
- `break;` (값 없음) 로 종료하면 `loop` 의 결과는 unit.
- 같은 `loop` 안의 여러 `break` 가 서로 다른 타입이면, 결과 타입은 [표현식 개요](README.md) 의 규칙에 따라 합 타입으로 합쳐진다.

### 결과 타입

- `break value` 가 한 번이라도 등장하는 `loop`. 결과 타입은 그 값의 타입.
- `break;` 만 있는 `loop`. 결과 타입은 unit.
- `break` 가 전혀 없는 `loop`. divergent (절대 끝나지 않음).

## 3.13.2. `while`: 조건 기반 반복

```nok
while cond {
    ...
}
```

- `while` 키워드 + bool 조건 + 블록 본체.
- 매 반복 시작 시 `cond` 를 평가한다. `true` 면 본체 실행, `false` 면 종료한다.

### 결과 값

`while` 의 결과 값은 항상 unit.

```nok
let x = while cond { ... };     // x: unit
```

이 결정의 근거는, `while` 이 자연스럽게 끝났을 때 (`cond` 가 `false`) 어떤 값을 반환할지에 자연스러운 선택이 없다는 점이다. `loop { break v; }` 처럼 명시적 `break value` 를 위한 형태가 아니라는 뜻이다.

값을 반환하고 싶으면 `loop { ... break v; ... }` 를 쓴다.

### `while let`: 패턴 매칭과의 결합

```nok
while let v: T = compute() {     // compute(): T | none
    use(v);
}
```

- 매 반복마다 식을 평가하고 패턴에 매칭.
- 매칭 성공이면 본체 실행, 실패면 종료한다.
- 패턴의 변수 바인딩이 본체에서 가시.

[패턴](311-patterns.md) 의 모든 패턴 형태를 사용할 수 있으며, 보통 `enum` 이나 합 타입 변형 매칭에 쓰인다.

## 3.13.3. `for`: 반복자 기반 반복

```nok
for 패턴 in 반복식 {
    ...
}
```

- `for` 키워드 + 패턴 + `in` + 반복식 + 블록 본체.
- 반복식은 반복자 또는 반복 가능 (Iterable) 한 값.
- 매 반복마다 다음 원소를 패턴에 매칭하여 본체에 바인딩한다.

### 반복식의 desugar

```
for pat in xs { body }
```

는 의미적으로 다음과 동등하다.

```
let it = <typeof(xs) as IntoIterator>::into_iter(xs);
loop {
    match <typeof(it) as Iterator>::next(ref it) {     // 반환: T | none
        pat: T => body,
        none   => break,
    }
}
```

(정확한 트레잇 / 메서드 이름은 표준 라이브러리에서 정의.)

### 결과 값

`for` 의 결과 값은 항상 unit (`while` 과 같은 이유).

### 패턴 자리

[패턴](311-patterns.md) 의 반증 불가능 패턴만 사용 가능하다 (이름, 와일드카드, 튜플 분해 등).

```nok
for x in 0..10 { ... }                    // 단순 이름
for (i, v) in xs.enumerate() { ... }      // 튜플 분해
for Point { x, y } in points { ... }      // struct 분해
```

반증 가능 패턴 (예: `enum` 변형 매칭) 은 매칭 실패 시 처리가 모호하므로 `for` 의 패턴 자리에서 거부된다. 그런 형태가 필요하면 `while let` 이나 `for` + `if let` 조합을 쓴다.

## 3.13.4. 라벨: 중첩 루프 제어

```nok
'outer: loop {
    for x in xs {
        if cond { break 'outer; }
        if other { continue 'outer; }
    }
}
```

- 루프 앞에 `'이름:` 형태로 라벨을 붙인다.
- `break 'name` 이나 `continue 'name` 으로 해당 라벨의 루프를 직접 제어한다.

### 라벨 토큰

라벨은 작은따옴표 + 식별자 + 콜론 (`'outer:`). 작은따옴표는 [문자 리터럴](../001-lexical/108-character-literals.md) 의 시작과 같은 토큰이지만, 콜론과 결합한 위치에서 라벨로 인식된다. 정확한 토큰화 규칙은 어휘 사양의 보강 정의이며, 본 문서는 언어 수준의 형태만 명시한다.

### `break label value`

`loop` 라벨에서 값과 결합 가능.

```nok
'outer: loop {
    for x in xs {
        if found(x) { break 'outer x; }
    }
}
```

`break 'outer x` 는 라벨 outer 의 `loop` 에서 값 x 를 들고 빠져나간다.

## 3.13.5. `break` 와 `continue` 기본 동작

라벨이 없으면 가장 안쪽 루프를 가리킨다.

```nok
loop {
    for x in xs {
        if cond { break; }       // for 를 빠져나감
    }
    // 여기서 계속
}
```

## 3.13.6. 평가 순서

각 루프의 평가는 위에서 정의된 의미를 따른다. 본체의 식과 문장은 [블록 표현식](309-block-expressions.md) 의 순서대로 평가되며, 부작용도 그 순서를 따른다.

루프 종료 조건의 검사는:

- `loop`. 검사 없음 (외부 `break` 로만).
- `while`. 매 반복 시작 시 `cond` 평가.
- `for`. 매 반복 시작 시 `Iterator::next` 호출.

## 3.13.7. divergent 루프

`break`, `return`, `throw` 등으로도 종료 가능성이 없는 루프 (예: `loop { ... }` 안에 `break` 가 없고 외부로 빠져나가는 흐름도 없는 경우) 는 divergent 로 분류된다. 그 루프 표현식의 타입은 임의의 타입과 호환된다.
