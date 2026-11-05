# 3.13. 루프 표현식

반복 실행은 `loop`, `while`, `for` 세 형태로 씁니다. 셋 모두 표현식이며, 이 문서는 각 형태의 문법과 값 평가 규칙을 정의합니다.

```
loop_expr   ::= label? "loop" block
              | label? "while" expr block
              | label? "for" pattern "in" expr block
              | label? "for" "await" pattern "in" expr block
              | label? "for" "ref" identifier "in" expr block
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

`while`이 정상적으로 끝났을 때(`cond`가 `false`) 반환할 값으로 자연스러운 선택지가 없기 때문입니다. 이 형태는 `loop { break v; }`처럼 명시적 `break value`로 값을 내는 형태가 아닙니다. 대상이 `while`(또는 `for`)인 `break`에 값을 붙이면 컴파일 에러(E0059)입니다.

값을 반환하려면 `loop { ... break v; ... }`를 쓰세요.

### `while let`: 패턴 매칭과 결합

```rok
while let v = compute() {        // compute(): T | none; v: T
    consume(v);
}
```

```
while_let ::= "while" "let" pattern (":" type_expr)? "=" expr block
```

반복할 때마다 식을 평가하고 `if let`([3.10.5](310-if-expression.md#3105-if-let-패턴-매칭과-결합))과 같은 규칙으로 매칭합니다. 그래서 이름 하나는 `none`을 뺀 값을 받습니다. 매칭되면 본체를 실행하고, 아니면 끝납니다. 패턴의 바인딩은 본체에서 보입니다.

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

값이 `IntoIterator`를 구현하면 루프는 그 `Iterator`를 진행합니다. 그렇지 않고 `TryIterator`([stdlib 2.7.13](../../refs-standard-libraries/200-core/207-iter.md#2713-tryiterator-실패할-수-있는-반복))를 구현하면 그 값을 바로 진행합니다. 이때 `next`가 던지므로 루프는 `try for`(또는 `try! for` / `try? for`)로 써야 합니다. 그 밖의 값은 컴파일 에러입니다.

```
try for pat in xs { body }
```

는 의미상 다음과 같습니다.

```
let it = try xs;                                     // 반복 대상을 덮음
loop {
    match try <typeof(it) as TryIterator>::next(ref it) {   // 단계마다 덮음
        pat: T => body,                              // 본체는 덮지 않음
        none   => break,
    }
}
```

`try for`의 `try`는 반복만 덮습니다. 반복 대상 식과 각 `next`입니다. 본체는 덮지 않습니다. 본체 안에서 에러를 던지는 호출은 다른 곳과 똑같이 자기 `try`를 붙입니다([3.14.2](314-error-control-flow.md#덮는-범위)). 단계에서 난 에러는 루프의 `try`가 보내는 곳으로 갑니다. `try for`면 함수 밖으로, `try! for`면 패닉으로, `try? for`면 루프 끝으로 가며 이때 식의 값은 `none`입니다. 루프 뒤의 `catch`는 `try for` 식 전체에 붙습니다([4.14.10](../004-functions/414-throws.md#try-for)). `TryStream`을 도는 `try for await`도 같은 규칙을 따릅니다([stdlib 2.8.11](../../refs-standard-libraries/200-core/208-stream.md#2811-trystream-실패할-수-있는-스트림)).

변수나 변수의 필드에 든 `TryIterator`는 그 자리에서 진행합니다. 위의 `it`은 복사본이 아니라 그 자리이며, `for await`의 스트림을 그 자리에서 폴링하는 것과 같습니다([4.13.5](../004-functions/413-stream.md#4135-for-await-형태)). 그런 자리를 `match` 갈래(또는 `if let`)로 묶은 바인딩도 같습니다. 바인딩은 그 자리를 가리키므로, 반복이 멈추면 그 자리도 멈춘 곳에 머뭅니다.

변수에 든 `Iterator`를 도는 일반 `for`는 복사본을 진행합니다. 반복자의 `into_iter`는 반복자 자신이고, 위의 `it`은 새 변수입니다. 변수는 자기 위치를 그대로 가지므로 `let r = 0..3; for i in r { .. } for i in r { .. }`는 `r`을 두 번 돕니다. 반복이 멈춘 곳에서 이어 가려면 반복자를 직접 진행하세요(`while let x: T = it.next() { .. }`). 이렇게 하면 변수 자체가 진행합니다.

`List`의 반복자는 단계마다 그 순간의 리스트를 읽습니다. 그래서 본문이 도는 리스트의 길이를 바꾸면 루프도 달라집니다. `push`하면 루프가 계속 이어지고, `remove`하면 지운 자리로 옮겨 온 원소를 건너뜁니다. 경고 린트 `resize_while_iterating`([6.2](../006-meta-system/602-built-in-attributes.md))은 본문에서 도는 변수에 `push`, `pop`, `insert`, `remove`, `clear`, `truncate`, `drain`, `retain`, `dedup`, `extend`, `append`를 부르는 곳을 알려 줍니다. 같은 블록에서 그 뒤에 `break`, `return`, `throw`가 오면 알리지 않습니다. 리스트를 바꾸려면 복사본을 돌거나(`for x in xs.clone()`) 바꿀 내용을 모아 두었다가 루프가 끝난 뒤에 적용하세요.

### 결과 값

`for`의 결과 값은 항상 unit입니다(`while`과 같은 이유). 대상이 `for`인 `break`에 값을 붙이면 컴파일 에러입니다.

### 패턴 자리

[패턴](311-patterns.md) 중 반증 불가능 패턴(이름, 와일드카드, 튜플 분해 등)만 쓸 수 있습니다.

```rok
for x in 0..10 { ... }                    // 단순 이름
for (i, v) in xs.enumerate() { ... }      // 튜플 분해
for Point { x, y } in points { ... }      // struct 분해
```

반증 가능 패턴(예: `enum` 변형 매칭)은 매칭에 실패했을 때의 처리가 모호하므로 `for`의 패턴 자리에 쓸 수 없습니다. 그런 형태가 필요하면 `while let`을 쓰거나 `for`와 `if let`을 함께 쓰세요.

### `for ref`: 원소를 자리로

```rok
struct Point { x: f64; y: f64; }

let ps: List<Point> = [Point { x: 1.0, y: 2.0 }, Point { x: 3.0, y: 4.0 }];
for ref p in ps {
    p.x = 0.0;                            // ps 의 원소에 씀
}
println(f"{ps[1].x}");                    // 0
```

보통의 `for`는 각 원소의 사본을 묶습니다. struct는 값이므로([2.3.8](../002-types/203-struct.md#238-값-의미)) 바인딩의 필드에 써도 루프 밖은 바뀌지 않고, 아무도 읽지 않으면 컴파일 에러입니다. `for ref x in xs`는 대신 `x`를 각 원소의 자리로 묶습니다. `x`를 읽으면 원소를 읽고, `x = v`, `x.f = v`, `x.f += v`, `ref self` 메서드 호출(`x.m()`, [4.7](../004-functions/407-ref-keyword.md))은 원소에 씁니다. 패턴은 이름 하나입니다.

`xs`는 `List`, `SliceMut`, 배열 중 하나입니다. 배열은 저장된 자리에 쓰므로 임시값이 아닌 변수나 필드여야 합니다. 다른 반복 대상은 항목이 자리가 아니므로 컴파일 에러입니다. 루프는 다음과 같습니다.

```
let v = xs;                    // List: 그 SliceMut 뷰, 배열: xs 그대로
for i in 0..v.len() {          // 길이는 한 번 읽음
    body                       // x 는 v[i] 를 뜻함
}
```

리스트는 `SliceMut` 뷰로 순회하므로 뷰의 규칙을 따릅니다([stdlib 2.9.7a](../../refs-standard-libraries/200-core/209-collections.md#297a-slicemutt-쓰기-가능한-뷰)). 본문에서 `xs`의 크기를 바꾸면(`push`, `remove`, `clear` 등) 본문에 크기 변경과 이후의 `x` 사용이 보이는 곳에서는 컴파일 에러 `E0138`, 그 밖(호출한 함수 안의 크기 변경)에서는 다음 `x` 사용에서 panic입니다. 더하거나 뺄 원소를 모아 두었다가 루프가 끝난 뒤 `xs`를 바꾸세요.

클로저나 `async` 블록은 `x`를 캡처할 수 없습니다. 원소를 먼저 복사하세요(`let v = x;`). `if x is T`는 `x`를 좁히지 않고, `for await`는 `for ref`가 될 수 없습니다.

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
