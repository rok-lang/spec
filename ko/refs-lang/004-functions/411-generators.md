# 4.11. 제너레이터(`yield`)

`yield` 키워드([키워드](../001-lexical/105-keywords.md))는 값을 하나씩 만들어 내는 함수, 곧 제너레이터를 정의하는 세 가지 형태에 씁니다.

## 4.11.1. 세 가지 형태

제너레이터의 문법은 다음과 같습니다.

```
generator_fn    ::= "yield" fn_decl
generator_block ::= "yield" block
generator_closure ::= "yield" closure_expr
yield_stmt      ::= "yield" expr? ";"
```

```rok
yield fn name(args) -> R { ... yield expr; ... }     // 명명된 제너레이터 함수
yield { ... yield expr; ... }                         // 인라인 블록 (값)
yield || { ... yield expr; ... }                      // 클로저 수식어
```

세 형태 모두 결과는 라이브러리 타입 `GenIter<T>`(`Iterator<Item = T>` 구현)의 값입니다. `throws (E)`이면 `TryGenIter<T, E>`입니다. 이 규칙은 [4.1.11](401-function-declaration.md#코루틴-함수의-결과-타입)에서 정의합니다.

## 4.11.2. `yield` 문: 값 내보내기

`yield` 문은 값과 함께 쓸 수도 있고 값 없이 쓸 수도 있습니다.

```rok
yield expr;
yield;
```

`yield expr;`은 `expr`의 값을 현재 반복 단계의 산출값으로 내보내고 함수나 블록의 평가를 멈춥니다. 다음 반복에서 평가를 이어 갑니다. `yield;`는 값 없이(unit) 한 단계만 나아갑니다.

## 4.11.3. 명명된 제너레이터 함수

이름 있는 제너레이터 함수는 다음과 같이 씁니다.

```rok
yield fn count_up_to(n: i32) -> i32 {
    let i = 0;
    while i < n {
        yield i;
        i = i + 1;
    }
}
```

`yield fn`은 제너레이터임을 표시한 함수 선언입니다. 본체 안에서 `yield expr;`으로 값을 내보냅니다. `-> R` 자리에는 `yield`로 내보내는 값의 타입을 적고, 호출 결과는 `GenIter<R>`입니다. `-> R`은 적어야 합니다. 컴파일러는 원소 타입을 본체에서 추론하지 않습니다. 호출 방식은 일반 함수와 같고, 결과는 반복자 객체입니다.

```rok
let it = count_up_to(5i32);       // it: `GenIter<i32>`
for x in it { print(f"{x} "); }   // 0 1 2 3 4
```

## 4.11.4. 인라인 `yield` 블록: 값

식 자리에서 바로 반복자를 만들려면 인라인 `yield` 블록을 씁니다.

```rok
let it = yield {
    yield 1i32;
    yield 2i32;
    yield 3i32;
};
```

이 형태는 식 자리에 오고 바로 `GenIter<T>`를 만듭니다. 본체는 [블록 표현식](../003-expressions/309-block-expressions.md)의 블록입니다. 짧은 제너레이터를 이름 없이 그 자리에 적을 때 씁니다.

## 4.11.5. `yield` 클로저

`yield` 클로저는 호출할 수 있는 제너레이터를 만듭니다.

```rok
let make_iter = yield || {
    yield 1i32;
    yield 2i32;
    yield 3i32;
};

let it = make_iter();             // 호출 시 새 Iterator 생성
```

`yield ||`는 호출할 수 있는 제너레이터 클로저를 만듭니다. 호출할 때마다 새 반복자가 생깁니다. 바깥 변수 캡처는 [캡처 의미론](406-capture-semantics.md)의 규칙을 따릅니다.

## 4.11.6. 세 형태의 대응

제너레이터의 세 형태는 [함수 선언](401-function-declaration.md), [클로저](405-closure.md), [블록 표현식](../003-expressions/309-block-expressions.md)의 세 형태(이름 있는 함수, 블록, 클로저)에 일대일로 대응합니다.

```
일반 함수    fn f() { ... }       ↔  || { ... }       ↔  { ... } (블록)
yield 형     yield fn f() { ... } ↔  yield || { ... } ↔  yield { ... }
```

곧 `yield`는 함수, 블록, 클로저 형태에 수식어로 붙어 그 형태를 제너레이터로 바꿉니다.

## 4.11.7. `yield` 토큰의 네 가지 등장 위치

`yield` 토큰이 올 수 있는 위치는 네 가지입니다.

```rok
yield fn name(...) { ... }        // declaration modifier (선언 수식어)
yield { ... }                      // block marker (블록 마커)
yield || { ... }                   // closure modifier (클로저 수식어)
yield expr;                         // statement (문장)
```

네 위치는 문맥으로 분명히 구분됩니다. 파서는 다음 토큰을 보고 어느 형태인지 정합니다.

| `yield` 다음 토큰 | 의미                                                             |
|-------------------|------------------------------------------------------------------|
| `fn`              | 선언 수식어                                                      |
| `{`               | [block](../000-meta/007-glossary.md#block-블록) marker (식 자리) |
| `\|\|` 또는 `\|`     | 클로저 수식어                                                    |
| 식 시작 토큰      | `yield` 문                                                       |

## 4.11.8. Iterator 트레잇과 결합

제너레이터 함수의 결과는 표준 라이브러리의 `Iterator<Item = T>` 트레잇을 구현하는 값입니다. 그래서 [루프](../003-expressions/313-loop-expressions.md)의 `for` 루프에서 바로 쓸 수 있습니다.

```rok
yield fn naturals() -> i32 {
    let i = 0;
    loop {
        yield i;
        i = i + 1;
    }
}

for n in naturals() {
    if n > 100 { break; }
    print(n);
}
```

`Iterator` 트레잇의 정확한 정의와 메서드(`map`, `filter`, `take`, `collect` 등)는 표준 라이브러리에서 정의합니다.

## 4.11.9. 무한 제너레이터

위 예의 `naturals()`는 끝없이 값을 내는 무한 제너레이터입니다. 이런 제너레이터는 `for` 루프의 `break`나 라이브러리의 `take(n)` 메서드 등으로 끝냅니다.

## 4.11.10. 제너레이터의 종료

본체가 정상 종료하면 반복자도 끝나며, 다음 `next()` 호출은 `none`을 반환합니다.

```rok
yield fn finite() -> i32 {
    yield 1i32;
    yield 2i32;
    yield 3i32;
    // 함수 정상 종료 → 반복자 종료
}
```

`return;`으로 명시적으로 끝낼 수도 있습니다.

```rok
yield fn early_stop(stop: i32) -> i32 {
    let i = 0;
    loop {
        if i >= stop { return; }
        yield i;
        i = i + 1;
    }
}
```

`for` 루프가 일찍 떠난 제너레이터는 닫힙니다. 루프 본문이 `break`(레이블 붙은 `break`나 바깥 루프의 `continue`도 포함), `return`, 던진 에러, `try` 전파, task의 취소, 바깥 제너레이터의 닫힘으로 루프를 떠날 때입니다. 닫으면 제너레이터는 기다리던 `yield`에서 재개되고, 거기서 `return`처럼 본체를 떠납니다. 그 `yield`를 둘러싼 `finally` 블록과 `using`의 drop이 안쪽부터 실행됩니다. 닫는 중에 만난 `yield`도 같은 방식으로 제너레이터를 끝냅니다. 그 값은 버려집니다. 첫 `next()` 전에 닫힌 제너레이터는 본체를 전혀 실행하지 않습니다. 닫힌 뒤의 `next()`는 `none`을 반환합니다.

```rok
yield fn lines() -> str {
    using f = open_file() {          // 아래 루프가 break하면 drop됨
        for l in f.lines() { yield l; }
    }
}

for l in lines() {
    if l == "" { break; }            // 제너레이터를 닫음: `f`가 drop됨
}
```

루프 자신이 쥔 제너레이터만 닫힙니다. 곧 `for`의 식이 제너레이터 값(호출, `yield` 블록)인 경우입니다. 변수, 필드, 원소에 든 제너레이터는 닫히지 않으므로 루프가 끝난 뒤에도 반복을 이어 갈 수 있습니다.

```rok
let g = lines();
for header in g { if header == "" { break; } }   // g는 열린 채로 남음
for body in g { print(body); }                    // 빈 줄 다음부터 이어 감
```

닫히지도 끝나지도 않은 제너레이터의 정리는 GC가 회수할 때까지 남아 있다가, 회수될 때에도 실행되지 않습니다.

## 4.11.11. 제너레이터의 상태

제너레이터는 호출 사이에 내부 상태를 유지합니다. 본체의 지역 변수, 반복 위치, 코루틴 프레임이 다음 `next()` 호출까지 남아 있습니다.

이 상태는 컴파일러가 만든 상태 기계(stackless)로 동작합니다. 상태 기계는 [7. 런타임 모델](../007-runtime/README.md)의 코루틴 인프라 위에서 돕니다. 자세한 구현은 런타임 사양에서 정의합니다.

## 4.11.12. `yield` 함수의 매개변수와 반환

`yield fn`은 제너레이터이므로 `-> T`는 본체에서 `yield`로 내보내는 값의 타입입니다. 호출 결과는 `GenIter<T>`입니다.

```rok
yield fn f() -> i32 { yield 1i32; }
// 호출 결과: `GenIter<i32>`
```

매개변수 사용 방식은 일반 함수와 같습니다.

## 4.11.13. 제너레이터 메서드

[메서드](409-method.md)의 `impl` 블록 안에서도 `yield fn`으로 제너레이터 메서드를 정의할 수 있습니다.

```rok
impl<T> List<T> {
    yield fn iter(self) -> T {
        for x in self.elements { yield x; }
    }
}
```

호출 방식과 동작은 일반 인스턴스 메서드와 같습니다.
