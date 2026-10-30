# 3.8. 옵셔널 체이닝과 null 합체

[합 타입](../002-types/206-sum-types.md)의 `T | none` 값에서 체이닝과 기본값 처리를 간결하게 쓰도록 다음 두 표기를 정의합니다.

```
optional_chain  ::= primary "?." identifier                       // optional field
                  | primary "?." identifier "(" arg_list? ")"     // optional method
coalesce_expr   ::= expr "??" expr
```

## 3.8.1. 옵셔널 체이닝: `?.`

```rok
x?.field
x?.method()
```

`?.` 토큰은 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)에서 정의합니다. 좌변 `x`의 타입이 `T | none`일 때 값이 있으면 점 접근이나 메서드 호출을 수행하고, 값이 없으면(`none`) 전체 결과가 `none`입니다.

### desugar

```
x?.field            →   match x { v: T => v.field, none => none }
x?.method(args)     →   match x { v: T => v.method(args), none => none }
```

여기서 `T`는 좌변 합 타입에서 `none`을 뺀 변형입니다. 타입 검사 패턴(`v: T`)으로 합 타입의 none이 아닌 분기를 받아 좁힌 뒤, 점 접근이나 메서드 호출을 이어서 수행합니다.

### 옵셔널이 아닌 수신자

좌변의 타입에 `none`이 없으면 `?.`는 컴파일 에러입니다. 이때는 `.`을 씁니다.

```rok
let p: Point = ...;
p?.x;          // ERROR: p 는 옵셔널이 아니므로 . 를 쓰세요
```

## 3.8.2. 체이닝

`?.`는 여러 번 이어 쓸 수 있습니다.

```rok
let name = user?.profile?.name;
```

어느 단계에서든 `none`이 나오면 그 자리에서 전체 표현식이 `none`으로 끝납니다. `user`가 `none`이면 전체 결과가 `none`이고, `user.profile`이 `none`이어도 전체가 `none`입니다. 모든 값이 있을 때만 `name`의 값이 나옵니다.

## 3.8.3. 결과 타입

`?.`의 결과 타입은 체인 끝까지 전파됩니다. 마지막 식이 `T`여도 체인의 결과 타입은 항상 `T | none`입니다.

```rok
let user: User | none = ...;
let name = user?.profile?.name;
// name: str | none
```

마지막 단계의 평탄 타입(`str`)이 합 타입(`str | none`)으로 자동 승급되는 것이 아닙니다. 체인 자체가 `T | none`이라는 결과 타입을 만듭니다.

## 3.8.4. null 합체: `??`

```rok
x ?? y
```

좌변 `x`가 `T | none`일 때 `??`는 값이 있으면 그 값을, 없으면 `y`를 반환합니다.

### desugar

```
x ?? y          →   match x { v: T => v, none => y }
```

### 결과 타입

`??`의 결과 타입은 [표현식 개요](README.md#분기-결과-타입)의 분기 결과 규칙을 따릅니다. 두 분기(`T`와 `typeof(y)`)의 타입이 같으면 그 타입이 결과 타입입니다. 다르면 받는 쪽에 타입이 있어야 하며, 합 타입으로 자동으로 넓히지 않습니다.

```rok
let a = foo() ?? 0;                           // i32 (foo(): i32 | none)
let r = foo() ?? "default";                   // ERROR: i32 와 str, 받는 쪽 타입 없음
let r2: i32 | str = foo() ?? "default";       // OK
```

`y`도 `T | none`이면 결과는 `T | none`입니다(`T | (T | none)`을 합친 것). 그래서 `??`를 이어 쓰면 마지막 대안이 `none`이 아닐 때 비로소 `none`이 빠집니다.

```rok
let a: i32 | none = ...;
let b: i32 | none = ...;
let r = a ?? b;              // r: i32 | none
let s = a ?? b ?? 0i32;      // s: i32
```

## 3.8.5. 우선순위

[우선순위와 결합성](302-precedence-and-associativity.md)의 등급에서 `?.`는 멤버 접근(`.`)과 우선순위가 같고 매우 강하게 묶입니다. `??`(prec=1)는 논리 OR(`||`)보다 낮고 범위 연산자(`..`, prec=0)보다 높으며, 우결합입니다.

```
a ?? b ?? c     →   a ?? (b ?? c)              // 우결합
foo() ?? bar    →   (foo()) ?? (bar)           // 호출이 ?? 보다 강함
a + b ?? c      →   (a + b) ?? c               // + 가 ?? 보다 강함
a ?? 0..3       →   (a ?? 0)..3                // ?? 가 .. 보다 강함
```

## 3.8.6. 체이닝과 합체의 결합

```rok
let name = user?.profile?.name ?? "anonymous";
```

`user?.profile?.name`의 결과는 `str | none`이고, 뒤의 `?? "anonymous"`가 `none`인 경우를 채우므로 최종 결과는 `str`입니다. 없을 수도 있는 값을 안전하게 꺼내면서 기본값을 적용하는 표준 형태입니다.

## 3.8.7. `?.` 와 `?` 의 구분

`?.`는 옵셔널 체이닝입니다. `T | none` 값에 붙는 후위 `?`는 [3.14.10](314-error-control-flow.md)에서 정의합니다.

`?.`와 `?` `.`가 서로 다른 토큰이라는 점은 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)의 최장 일치 규칙이 보장합니다.

## 3.8.8. 옵셔널 체이닝의 단축 평가

```
x?.foo().bar()
```

위 표현식에서 `x`가 `none`이면 `foo()`와 `bar()` 모두 호출되지 않습니다. 단축 평가는 보장됩니다.

desugar한 `match`가 `none` 분기에서 바로 `none`을 반환하기 때문입니다.

따라서 부작용이 있는 메서드를 체인에 두면, 수신자가 `none`일 때 그 메서드는 호출되지 않습니다.

체인은 `?.` 뒤에 이어지는 후위 연산(필드 접근, 메서드 호출, `?.[i]`를 포함한 인덱싱)입니다. 괄호는 체인을 끝냅니다. `(x?.a).b`에서 `.b`는 `none`일 수 있는 `x?.a`의 결과에 적용됩니다.

## 3.8.9. 깊은 점 접근의 대안

체인이 깊고 복잡해지면 `??`와 함께 쓰거나 명시적인 매칭([`match` 표현식](312-match-expression.md))을 쓰는 편이 더 명확할 수 있습니다.

```rok
// 체이닝
let x = a?.b?.c?.d ?? default;

// 명시적 (a: A | none, b: B | none, c: C | none 의 경우)
let x = match a {
    av: A => match av.b {
        bv: B => match bv.c {
            cv: C => cv.d,
            none => default,
        },
        none => default,
    },
    none => default,
};
```

두 형태는 의미가 같으므로 가독성을 보고 고르면 됩니다.
