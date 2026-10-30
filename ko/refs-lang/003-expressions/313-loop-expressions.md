# 3.13. 루프 표현식

반복 실행은 `loop`, `while`, `for` 세 형태로 씁니다. 셋 모두 표현식이며, 이 문서는 각 형태의 문법과 값 평가 규칙을 정의합니다.

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
> `in`은 `for pat in` 자리에서만 키워드가 되는 문맥 키워드입니다([1.5.4](../001-lexical/105-keywords.md#154-문맥-키워드)).

`for await`의 의미와 쓰임은 [스트림](../004-functions/413-stream.md)에서 정의합니다. 비동기 스트림에서 항목을 하나씩 받아 본체를 실행하는 형태입니다.

라벨(`'name:`)의 의미는 [3.13.4 라벨](#3134-라벨-중첩-루프-제어)에서 정의합니다. 라벨이 붙은 **블록**(`'b: { ... }`)은 [블록 표현식](309-block-expressions.md)에서 정의합니다.

## 3.13.1. `loop`: 무한 반복

```rok
loop {
    ...
}
```

`loop` 키워드 뒤에 블록 본체를 둡니다. 본체를 끝없이 반복합니다. `break`, `return`, `throw`로만 끝낼 수 있습니다.

### `break value`로 값 반환

```rok
let v = loop {
    let x = compute();
    if x.converged { break x; }
};
```

`break` 뒤에 식을 두면 그 식의 값이 `loop` 표현식의 결과가 됩니다. `break;`처럼 값 없이 끝내면 `loop`의 결과는 unit입니다. 한 `loop`(또는 라벨 붙은 블록)의 모든 `break` 값은 타입이 하나여야 합니다. 받는 쪽에 타입 주석이 있으면 그 타입이어야 합니다. `break;`와 `break v`를 섞으면 컴파일 에러입니다.

### 결과 타입

`break value`가 한 번이라도 나오는 `loop`는 그 값의 타입이 결과 타입입니다. `break;`만 있는 `loop`는 결과 타입이 unit입니다. `break`가 전혀 없는 `loop`는 divergent입니다. 절대 끝나지 않습니다.

## 3.13.2. `while`: 조건 기반 반복

```rok
while cond {
    ...
}
```

`while` 키워드 뒤에 bool 조건과 블록 본체를 둡니다. 반복을 시작할 때마다 `cond`를 평가해서 `true`면 본체를 실행하고, `false`면 끝냅니다.

### 결과 값

`while`의 결과 값은 항상 unit입니다.

```rok
let x = while cond { ... };     // x: unit
```

`while`이 정상적으로 끝났을 때(`cond`가 `false`) 반환할 값으로 자연스러운 선택지가 없기 때문입니다. 이 형태는 `loop { break v; }`처럼 명시적 `break value`로 값을 내는 형태가 아닙니다.

값을 반환하려면 `loop { ... break v; ... }`를 쓰세요.

### `while let`: 패턴 매칭과 결합

```rok
while let v: T = compute() {     // compute(): T | none
    use(v);
}
```

```
while_let ::= "while" "let" pattern (":" type_expr)? "=" expr block
```

반복할 때마다 식을 평가하고 `if let`([3.10.5](310-if-expression.md#3105-if-let-패턴-매칭과-결합))과 같은 규칙으로 매칭합니다. 매칭되면 본체를 실행하고, 아니면 끝납니다. 패턴의 바인딩은 본체에서 보입니다.

[패턴](311-patterns.md)의 모든 패턴 형태를 쓸 수 있고, 보통은 `enum`이나 합 타입의 변형을 매칭할 때 씁니다.

## 3.13.3. `for`: 반복자 기반 반복

```rok
for 패턴 in 반복식 {
    ...
}
```

`for` 키워드 뒤에 패턴, `in`, 반복식, 블록 본체를 차례로 둡니다. 반복식은 반복자이거나 반복 가능(Iterable)한 값이어야 합니다. 반복식과 `while` 조건의 최상위에는 struct 리터럴을 둘 수 없습니다([3.10.3](310-if-expression.md#조건-자리의-struct-리터럴)). 반복할 때마다 다음 원소를 패턴에 매칭해서 본체에 바인딩합니다.

### 반복식 풀어 쓰기

```
for pat in xs { body }
```

는 의미상 다음과 같습니다.

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

`next`가 에러를 던지는 반복자(`TryGenIter<T, E>` 등)를 도는 `for`는 에러를 던지는 식입니다. 그래서 `try for pat in it { ... }`처럼 `try`로 덮어야 합니다. `for await`도 마찬가지입니다([4.14.10](../004-functions/414-throws.md#try-for)).

### 결과 값

`for`의 결과 값은 항상 unit입니다(`while`과 같은 이유).

### 패턴 자리

[패턴](311-patterns.md) 중 반증 불가능 패턴(이름, 와일드카드, 튜플 분해 등)만 쓸 수 있습니다.

```rok
for x in 0..10 { ... }                    // 단순 이름
for (i, v) in xs.enumerate() { ... }      // 튜플 분해
for Point { x, y } in points { ... }      // struct 분해
```

반증 가능 패턴(예: `enum` 변형 매칭)은 매칭에 실패했을 때의 처리가 모호하므로 `for`의 패턴 자리에 쓸 수 없습니다. 그런 형태가 필요하면 `while let`을 쓰거나 `for`와 `if let`을 함께 쓰세요.

## 3.13.4. 라벨: 중첩 루프 제어

```rok
'outer: loop {
    for x in xs {
        if cond { break 'outer; }
        if other { continue 'outer; }
    }
}
```

루프 앞에 `'이름:` 형태의 라벨을 붙일 수 있습니다. `break 'name`이나 `continue 'name`으로 그 라벨의 루프를 직접 제어합니다.

### 라벨 토큰

라벨은 작은따옴표, 식별자, 콜론으로 이루어집니다(`'outer:`). 작은따옴표는 [문자 리터럴](../001-lexical/108-character-literals.md)을 시작하는 토큰과 같지만, 콜론과 결합한 위치에서는 라벨로 인식합니다. 정확한 토큰화 규칙은 어휘 사양을 보충하는 정의입니다. 이 문서는 언어 수준의 형태만 정합니다.

### `break label value`

`loop` 라벨에는 값을 함께 쓸 수 있습니다.

```rok
'outer: loop {
    for x in xs {
        if found(x) { break 'outer x; }
    }
}
```

`break 'outer x`는 라벨이 outer인 `loop`를 값 x와 함께 빠져나갑니다.

## 3.13.5. `break`와 `continue` 기본 동작

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

각 루프는 위에서 정의한 의미대로 평가합니다. 본체의 식과 문장은 [블록 표현식](309-block-expressions.md)의 순서대로 평가하며, 부작용도 그 순서를 따릅니다.

루프 종료 조건은 다음과 같이 검사합니다. `loop`는 검사 없이 명시적인 `break`로만 끝납니다. `while`은 반복을 시작할 때마다 `cond`를 평가합니다. `for`는 반복을 시작할 때마다 `Iterator::next`를 호출합니다.

## 3.13.7. divergent 루프

`break`, `return`, `throw` 등으로도 끝날 수 없는 루프(예: 안에 `break`가 없고 밖으로 나가는 흐름도 없는 `loop { ... }`)는 divergent로 분류합니다. 그런 루프 표현식의 타입은 어떤 타입과도 호환됩니다.
