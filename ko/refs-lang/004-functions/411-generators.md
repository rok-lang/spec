# 4.11. 제너레이터 (`yield`)

`yield` 키워드 ([키워드](../001-lexical/105-keywords.md)) 는 값을 하나씩 만들어내는 함수, 즉 제너레이터를 정의하는 세 가지 형태에서 사용됩니다.

## 4.11.1. 세 가지 형태

제너레이터는 다음과 같은 문법으로 표현됩니다.

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

세 형태 모두 결과는 **Iterator<T>** 라이브러리 타입의 인스턴스가 됩니다.

## 4.11.2. `yield` statement: 값을 내보내기

`yield` 문은 값과 함께 또는 값 없이 사용할 수 있습니다.

```rok
yield expr;
yield;
```

`yield expr;` 형태는 `expr` 의 값을 현재 반복 단계의 산출값으로 내보내고, 함수나 블록의 평가를 일시 중단합니다. 다음 반복에서는 이어서 평가를 진행합니다. 한편 `yield;` 형태는 값 없이 (unit) 단계만 진행시킵니다.

## 4.11.3. 명명된 제너레이터 함수

명명된 제너레이터 함수는 다음과 같이 작성합니다.

```rok
yield fn count_up_to(n: i32) -> i32 {
    let i = 0;
    while i < n {
        yield i;
        i = i + 1;
    }
}
```

`yield fn` 은 제너레이터임을 표시하는 함수 선언이며, 본체 안에서는 `yield expr;` 으로 값을 내보냅니다. `-> R` 자리에는 `yield` 되는 값의 타입을 적으며, 함수의 실제 반환 타입은 자동으로 **Iterator<R>** 로 wrapping 됩니다. R 을 생략하면 본체의 `yield` 식에서 타입을 추론합니다. 호출 방식은 일반 함수와 동일하며, 결과는 반복자 객체가 됩니다.

```rok
let it = count_up_to(5i32);       // it: Iterator<i32>
for x in it { print(x); }         // 0 1 2 3 4
```

## 4.11.4. 인라인 `yield` 블록: 값

식 자리에서 즉시 반복자를 만들고 싶다면 인라인 `yield` 블록을 사용합니다.

```rok
let it = yield {
    yield 1i32;
    yield 2i32;
    yield 3i32;
};
```

이 형태는 식 자리에 등장하며, 즉시 **Iterator<T>** 값을 만들어냅니다. 본체는 [블록 표현식](../003-expressions/309-block-expressions.md) 의 블록입니다. 짧은 제너레이터를 익명 인라인으로 적기 위한 형태라고 할 수 있습니다.

## 4.11.5. `yield` 클로저

`yield` 클로저는 호출 가능한 제너레이터를 만들어냅니다.

```rok
let make_iter = yield || {
    yield 1i32;
    yield 2i32;
    yield 3i32;
};

let it = make_iter();             // 호출 시 새 Iterator 생성
```

`yield ||` 는 호출 가능한 제너레이터 클로저를 만들며, 호출할 때마다 새 반복자가 생성됩니다. 외부 캡처는 [캡처 의미론](406-capture-semantics.md) 의 규칙을 따릅니다.

## 4.11.6. 세 형태의 대응

제너레이터의 세 형태는 [함수 선언](401-function-declaration.md), [클로저](405-closure.md), [블록 표현식](../003-expressions/309-block-expressions.md) 의 세 형태 (명명, 블록, 클로저) 에 일대일로 대응됩니다.

```
일반 함수    fn f() { ... }       ↔  || { ... }       ↔  { ... } (블록)
yield 형     yield fn f() { ... } ↔  yield || { ... } ↔  yield { ... }
```

즉, `yield` 가 동일한 함수, 블록, 클로저 형태에 수식어로 붙어 그것을 제너레이터로 변환하는 셈입니다.

## 4.11.7. `yield` 토큰의 네 가지 등장 위치

`yield` 토큰은 네 가지 위치에서 등장할 수 있습니다.

```rok
yield fn name(...) { ... }        // declaration modifier (선언 수식어)
yield { ... }                      // block marker (블록 마커)
yield || { ... }                   // closure modifier (클로저 수식어)
yield expr;                         // statement (문장)
```

이 네 위치는 문맥으로 명확히 구분되며, 파서는 다음 토큰을 보고 어느 형태인지 결정합니다.

| `yield` 다음 토큰 | 의미                                                             |
|-------------------|------------------------------------------------------------------|
| `fn`              | declaration modifier                                             |
| `{`               | [block](../000-meta/007-glossary.md#block-블록) marker (식 자리) |
| `\|\|` 또는 `\|`     | closure modifier                                                 |
| 식 시작 토큰      | `yield` statement                                                |

## 4.11.8. Iterator 트레잇과의 결합

제너레이터 함수의 결과는 표준 라이브러리의 `Iterator<T>` 트레잇을 충족하는 객체이므로, [루프](../003-expressions/313-loop-expressions.md) 의 `for` 루프에서 그대로 사용할 수 있습니다.

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

`Iterator` 트레잇의 정확한 정의와 메서드 (`map`, `filter`, `take`, `collect` 등) 는 표준 라이브러리에서 정의됩니다.

## 4.11.9. 무한 제너레이터

위 예의 `naturals()` 는 끝없이 값을 내는 무한 제너레이터에 해당합니다. 이러한 제너레이터는 `for` 루프의 `break` 또는 라이브러리의 `take(n)` 메서드 등으로 종료시킬 수 있습니다.

## 4.11.10. 제너레이터의 종료

본체가 정상 종료하면 반복자 역시 종료되며, 다음 `next()` 호출에서는 `none` 이 반환됩니다.

```rok
yield fn finite() -> i32 {
    yield 1i32;
    yield 2i32;
    yield 3i32;
    // 함수 정상 종료 → 반복자 종료
}
```

또한 `return;` 으로 명시적으로 종료시킬 수도 있습니다.

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

## 4.11.11. 제너레이터의 상태

제너레이터는 호출 사이에 내부 상태를 유지합니다. 본체의 지역 변수, 반복 위치, 코루틴 프레임이 다음 `next()` 호출까지 보존됩니다.

이 상태는 [7. 런타임 모델](../007-runtime/README.md) 의 코루틴 인프라 위에서 작동하며, LLVM coroutine intrinsic 기반의 stackless 구현으로 동작합니다. 자세한 구현 메커니즘은 런타임 사양에서 정의됩니다.

## 4.11.12. `yield` 함수의 매개변수와 반환

`yield fn` 은 제너레이터이므로, 반환 타입은 본체에서 `yield` 되는 값의 타입을 가리키며, 자동으로 `Iterator<T>` 로 wrapping 됩니다.

```rok
yield fn f() -> i32 { yield 1i32; }
// 실제 호출 결과 타입: Iterator<i32>
```

매개변수의 사용 방식은 일반 함수와 동일합니다.

## 4.11.13. 메서드로서의 제너레이터

[메서드](409-method.md) 의 `impl` 블록 안에서도 `yield fn` 을 사용해 제너레이터 메서드를 정의할 수 있습니다.

```rok
impl<T> List<T> {
    yield fn iter(self) -> T {
        for x in self.elements { yield x; }
    }
}
```

호출 방식과 동작은 일반 인스턴스 메서드와 동일합니다.
