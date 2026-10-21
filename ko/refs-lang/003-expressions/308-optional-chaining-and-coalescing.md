# 3.8. 옵셔널 체이닝과 null 합체

[합 타입](../002-types/206-sum-types.md) 의 `T | none` 값을 다룰 때 체이닝과 기본값 처리를 간결하게 표현하기 위해, 다음 두 가지 표기를 정의합니다.

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

여기서 사용되는 `?.` 토큰은 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md) 에 정의됩니다. 좌변 `x` 가 `T | none` 타입일 때 값이 존재하면 점 접근이나 메서드 호출을 수행하고, 값이 없으면 (즉 `none` 인 경우) 전체 결과를 그대로 `none` 으로 둡니다.

### desugar

```
x?.field            →   match x { v: T => v.field, none => none }
x?.method(args)     →   match x { v: T => v.method(args), none => none }
```

여기서 `T` 는 좌변의 합 타입에서 `none` 을 제외한 변형을 가리킵니다. 타입 검사 패턴 (`v: T`) 으로 합 타입의 non-none 분기를 받아 좁힌 뒤에 점 접근이나 메서드 호출을 이어 수행합니다.

### 옵셔널이 아닌 수신자

좌변의 타입이 `none` 을 포함하지 않으면 `?.` 는 컴파일 에러입니다. 이때는 `.` 를 씁니다.

```rok
let p: Point = ...;
p?.x;          // ERROR — p 는 옵셔널이 아니므로 . 를 쓰세요
```

## 3.8.2. 체이닝

`?.` 는 여러 번 이어서 사용할 수 있습니다.

```rok
let name = user?.profile?.name;
```

각 단계에서 `none` 이 반환되면 그 시점에 전체 표현식이 `none` 으로 마무리됩니다. 즉 `user` 가 `none` 이면 전체 결과가 `none` 이 되고, `user.profile` 이 `none` 이면 역시 전체가 `none` 이 되며, 모두 값이 존재하는 경우에만 최종적으로 `name` 의 값이 산출됩니다.

## 3.8.3. 결과 타입

`?.` 의 결과 타입은 체이닝의 끝까지 그대로 propagate 됩니다. 끝의 식이 `T` 라 하더라도 체이닝의 결과 타입은 항상 `T | none` 이 됩니다.

```rok
let user: User | none = ...;
let name = user?.profile?.name;
// name: str | none
```

마지막 단계의 평탄 타입 (`str`) 이 자동으로 합 타입 (`str | none`) 으로 승급되는 방식이 아니라, 체이닝 그 자체가 `T | none` 이라는 결과 타입을 만들어 낸다는 점에 유의합니다.

## 3.8.4. null 합체: `??`

```rok
x ?? y
```

여기서 사용되는 `??` 토큰은 좌변 `x` 가 `T | none` 일 때 값이 있으면 그 값을 그대로 반환하고, 값이 없으면 `y` 를 반환합니다.

### desugar

```
x ?? y          →   match x { v: T => v, none => y }
```

### 결과 타입

`??` 의 결과 타입은 [표현식 개요](README.md#분기-결과-타입) 의 분기 결과 규칙을 따릅니다. 두 분기 (`T` 와 `typeof(y)`) 가 같은 타입이면 그 타입입니다. 다르면 받는 쪽의 타입이 있어야 하며, 합으로 자동으로 넓히지 않습니다.

```rok
let a = foo() ?? 0;                           // i32 (foo(): i32 | none)
let r = foo() ?? "default";                   // ERROR — i32 와 str, 받는 쪽 타입 없음
let r2: i32 | str = foo() ?? "default";       // OK
```

`y` 도 `T | none` 이면 결과는 `T | none` 입니다 (`T | (T | none)` 을 합친 것). 그래서 `??` 를 이어 쓰면 마지막 대안이 `none` 이 아닐 때 비로소 `none` 이 빠집니다.

```rok
let a: i32 | none = ...;
let b: i32 | none = ...;
let r = a ?? b;              // r: i32 | none
let s = a ?? b ?? 0i32;      // s: i32
```

## 3.8.5. 우선순위

[우선순위와 결합성](302-precedence-and-associativity.md) 의 등급에서 `?.` 는 멤버 접근 (`.`) 과 같은 우선순위를 가지며 매우 강하게 묶입니다. 반면 `??` (prec=1) 는 논리 OR (`||`) 보다는 낮고 범위 연산자 (`..`, prec=0) 보다는 높은 우선순위에 위치하며 우결합으로 처리됩니다.

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

위 예에서 `user?.profile?.name` 의 결과는 `str | none` 이 되고, 뒤이은 `?? "anonymous"` 가 `none` 자리를 채워 주어 최종 결과가 `str` 로 확정됩니다. 이 패턴은 없을 수도 있는 값을 안전하게 꺼내면서 동시에 기본값을 적용하는 표준적인 형태이므로 자주 활용해 주시면 됩니다.

## 3.8.7. `?.` 와 `?` 의 구분

`?.` 는 옵셔널 체이닝입니다. `T | none` 값에 붙는 후위 `?` 는 [3.14.10](314-error-control-flow.md) 에서 정의합니다.

`?.` 와 `?` `.` 가 서로 다른 토큰으로 구분된다는 점은 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md) 의 최장 일치 규칙에 의해 보장됩니다.

## 3.8.8. 옵셔널 체이닝의 단축 평가

```
x?.foo().bar()
```

위 표현식에서 `x` 가 `none` 이면 `foo()` 와 `bar()` 어느 쪽도 호출되지 않습니다. 단축 평가가 보장된다고 이해하면 됩니다.

이는 desugar 의 `match` 가 `none` 분기에서 즉시 `none` 을 반환하기 때문입니다.

따라서 부작용이 있는 메서드를 체인에 두는 경우, `none` 일 때는 그 메서드가 호출되지 않는다는 점을 미리 이해해 두어야 합니다.

체인은 `?.` 뒤에 이어지는 후위 연산 (필드 접근, 메서드 호출, `?.[i]` 를 포함한 인덱싱) 입니다. 괄호는 체인을 끝냅니다: `(x?.a).b` 에서 `.b` 는 `none` 일 수 있는 `x?.a` 의 결과에 적용됩니다.

## 3.8.9. 깊은 점 접근의 대안

체이닝이 깊고 복잡해지는 경우에는 `??` 와 결합하거나 명시적인 매칭 ([`match` 표현식](312-match-expression.md)) 을 사용하는 편이 더 명확할 수 있습니다.

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

두 형태는 의미상 동일하므로, 사용자의 가독성 판단에 따라 선택하면 됩니다.
