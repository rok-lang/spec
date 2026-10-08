# 3.8. 옵셔널 체이닝과 null 합체

[합 타입](../002-types/206-sum-types.md) 의 `T | none` 값을 다룰 때, 체이닝과 기본값 처리를 간결하게 하는 두 표기를 정의한다.

```
optional_chain  ::= primary "?." identifier                       // optional field
                  | primary "?." identifier "(" arg_list? ")"     // optional method
coalesce_expr   ::= expr "??" expr
```

## 3.8.1. 옵셔널 체이닝: `?.`

```nok
x?.field
x?.method()
```

- `?.` 토큰 ([구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)).
- 좌변 `x` 가 `T | none` 타입일 때, 값이 있으면 점 접근이나 메서드 호출을 수행하고, 없으면 (`none` 이면) 전체 결과를 `none` 으로 둔다.

### desugar

```
x?.field            →   match x { v: T => v.field, none => none }
x?.method(args)     →   match x { v: T => v.method(args), none => none }
```

여기서 `T` 는 좌변의 합 타입에서 `none` 을 제외한 변형이다. 타입 검사 패턴 (`v: T`) 으로 합 타입의 non-none 분기를 받아 좁힌 뒤 점 접근이나 메서드 호출을 수행한다.

## 3.8.2. 체이닝

`?.` 는 여러 번 이어 쓸 수 있다.

```nok
let name = user?.profile?.name;
```

각 단계가 `none` 을 반환하면 그 시점에 전체 표현식이 `none` 으로 끝난다.

- `user` 가 `none` → 전체 `none`.
- `user.profile` 이 `none` → 전체 `none`.
- 모두 있으면 → `name` 의 값.

## 3.8.3. 결과 타입

`?.` 의 결과 타입은 체이닝 끝까지 propagate 된다. 끝의 식이 `T` 라도 체이닝의 결과는 항상 `T | none` 이다.

```nok
let user: User | none = ...;
let name = user?.profile?.name;
// name: str | none
```

마지막 단계의 평탄 타입 (`str`) 이 자동으로 합 타입 (`str | none`) 으로 승급되는 것이 아니다. 체이닝 자체가 `T | none` 결과 타입을 만든다.

## 3.8.4. null 합체: `??`

```nok
x ?? y
```

- `??` 토큰.
- `x` 가 `T | none` 일 때, 값이 있으면 그 값을, 없으면 `y` 를 반환한다.

### desugar

```
x ?? y          →   match x { v: T => v, none => y }
```

### 결과 타입

`??` 의 결과 타입은 [표현식 개요](README.md) 의 분기 결과 규칙을 따른다. 두 분기 (non-none 변형 값 vs `y`) 의 합 타입.

```
typeof(x ?? y)  =  T_non_none | typeof(y)     // 다르면 합 타입
typeof(x ?? y)  =  T                          // 양쪽 같은 타입이면 그 타입
```

```nok
let r = foo() ?? "default";
// 식 결과: i32 | str (foo(): i32 | none, "default": str)
// 슬롯 r 은 명시 없으니 any
let r2: i32 | str = foo() ?? "default";       // 명시 합 타입 슬롯 OK
```

## 3.8.5. 우선순위

[우선순위와 결합성](302-precedence-and-associativity.md) 의 등급에서:

- `?.`. 멤버 접근 (`.`) 과 같은 우선순위. 매우 강하게 묶임.
- `??`. 논리 OR (`||`) 보다 낮고 대입 (`=`) 보다 높다. 우결합.

```
a ?? b ?? c     →   a ?? (b ?? c)              // 우결합
foo() ?? bar    →   (foo()) ?? (bar)           // 호출이 ?? 보다 강함
a + b ?? c      →   (a + b) ?? c               // + 가 ?? 보다 강함
```

## 3.8.6. 체이닝과 합체의 결합

```nok
let name = user?.profile?.name ?? "anonymous";
```

- `user?.profile?.name` 의 결과는 `str | none`.
- `?? "anonymous"` 가 `none` 자리를 채워 결과는 `str` 로 확정.

이 패턴은 없을 수 있는 값을 안전하게 꺼내고 기본값을 적용하는 표준 형태다.

## 3.8.7. `?.` 와 `?` 의 구분

`?.` 는 옵셔널 체이닝. 단독 `?` 는 별도 의미를 갖는다 (후속 사양에서 정의).

`?.` 와 `?` `.` 가 다른 토큰임은 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md) 의 최장 일치 규칙으로 보장된다.

## 3.8.8. 옵셔널 체이닝의 단축 평가

```
x?.foo().bar()
```

위에서 `x` 가 `none` 이면 `foo()`, `bar()` 둘 다 호출되지 않는다. 단축 평가가 보장된다.

이는 desugar 의 `match` 가 `none` 분기에서 즉시 `none` 을 반환하기 때문이다.

부작용이 있는 메서드를 체인에 두는 경우, `none` 일 때 호출되지 않는다는 점을 사용자가 이해해야 한다.

## 3.8.9. 깊은 점 접근의 대안

체이닝이 깊고 복잡한 경우, `??` 와 결합하거나 명시적 매칭 ([`match` 표현식](312-match-expression.md)) 을 쓰는 것이 더 명확할 수 있다.

```nok
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

두 형태는 같은 의미다. 사용자의 가독성 판단에 따라 선택한다.
