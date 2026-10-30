# 2.14. 형 변환

이 절은 값을 한 타입에서 다른 타입으로 옮기는 구문을 정의합니다. `as`는 언어에 내장되어 있습니다. `.into()`와 `.try_into()`는 표준 라이브러리 트레잇의 일반 메서드 호출입니다. 그 impl이 모여 표준 라이브러리의 변환 매트릭스를 이룹니다.

```
cast_expr   ::= expr "as" type_expr
into_expr   ::= expr "." "into" ("::" "<" type_expr ">")? "(" ")"
try_into    ::= expr "." "try_into" ("::" "<" type_expr ">")? "(" ")"
```

## 2.14.1. 세 가지 구문

세 구문과 각 구문이 실행하는 동작은 다음과 같습니다.

| 구문               | 형태                   | 실행                        |
|--------------------|------------------------|-----------------------------|
| `as` 캐스트        | `expr as 타입`         | 내장 캐스트 (트레잇 없음)   |
| `.into()` 호출     | `expr.into()`          | `Target::from` (`Into::into`) |
| `.try_into()` 호출 | `expr.try_into::<T>()` | `Target::try_from` (`TryInto::try_into`) |

`Cast` 트레잇은 없습니다. 직접 정의한 타입에 `as` 쌍을 추가할 수도 없습니다. `From` / `Into`와 `TryFrom` / `TryInto`의 정확한 시그니처, 그리고 어떤 타입 쌍에 구현되어 있는지는 표준 라이브러리의 변환 매트릭스에서 정의합니다.

## 2.14.2. `as` 캐스트

첫째 구문은 `as` 캐스트입니다.

```rok
expr as 타입
```

`as`는 [키워드](../001-lexical/105-keywords.md)입니다. 왼쪽에는 아무 표현식이나, 오른쪽에는 [타입 표현식](201-type-expressions.md)이 옵니다. 컴파일러가 캐스트를 직접 낮추고, 트레잇 메서드는 호출하지 않습니다. 비트 자르기, 부호 확장, 표현 변환처럼 값이 보존되지 않을 수 있는 변환에 씁니다. 결과는 `.wrapping_into::<T>()`와 같습니다([stdlib 2.11.5](../../refs-standard-libraries/200-core/211-conversion.md)).

```rok
let x: i32 = 256i32;
let y: u8 = x as u8;          // 0
let z: f64 = x as f64;        // 256.0
```

허용하는 쌍은 정해져 있습니다.

- 임의의 타입에서 자기 자신으로(아무 일도 하지 않음).
- 정수 타입, 부동소수 타입, `bool`, `char`, `c_char` 중 임의의 두 타입 사이. 제약은 대상이 `char`일 때 하나뿐입니다. `u8 as char`와 `char as char`만 쓸 수 있습니다. 더 넓은 값은 서로게이트이거나 `U+10FFFF`를 넘을 수 있기 때문입니다. 그 밖의 정수, 부동소수, `bool`, `c_char` → `char`는 타입 에러입니다. 에러 메시지는 ``only `u8` can be cast as `char`, not `u32`; use `char::from_u32(n)` (`char | none`) for a wider value``입니다.
- 페이로드가 없는 `enum`에서 정수 타입으로. 결과는 그 태그나 값입니다([2.4.2](204-enum.md#정수로-캐스트)).
- 임의 정밀도 표준 타입 `int` / `float`로, 또는 그 타입에서. 그 타입으로 캐스트하면 무손실 `From` impl을 실행합니다. 그래서 그런 impl이 없는 쌍(`f64 as int`)은 에러입니다. 그 타입에서 캐스트하면 대상의 `WrappingFrom` impl을 실행합니다.

그 밖의 쌍은 컴파일 에러입니다(``cannot cast `str` as `i32` (only numeric / bool / char casts)``). 허용된 `as`는 항상 성공합니다. 실패할 수 있는 변환에는 `.try_into()`를 씁니다.

## 2.14.3. `.into()` 호출

둘째 구문은 `.into()` 호출입니다.

```rok
expr.into()
```

이 구문은 `Into::into::<Target>(expr)`, 곧 `Target::from(expr)`로 풀립니다. `Target: From<typeof(expr)>` 구현이 있어야 합니다. 대상 타입은 변수의 선언 타입이나 매개변수 타입 같은 문맥에서 추론합니다.

```rok
let x: i64 = 100i64;
let y: int = x.into();          // int::from(x)
```

대상 타입을 추론할 수 없는 자리에서는 turbofish로 적을 수 있습니다.

```rok
let y = x.into::<int>();
```

`.into()`는 표준 라이브러리가 값 보존 변환, 곧 정보 손실이 없는 변환으로 등록한 경우에만 쓸 수 있습니다. 보통 좁은 타입을 넓은 타입으로 변환할 때 씁니다.

## 2.14.4. `.try_into()` 호출

셋째 구문은 `.try_into()` 호출입니다.

```rok
expr.try_into::<타입>()
```

이 구문은 `Target::try_from(expr)`로 풀립니다(`Target: TryFrom<typeof(expr)>`). `try_into`는 [`throws`](../004-functions/414-throws.md) 메서드입니다. 성공하면 대상 타입의 값을 돌려주고, 실패하면 에러를 던집니다. 에러는 대상의 `TryFrom::Error`이고, 숫자 변환이면 `OverflowError`입니다([stdlib 2.11.4](../../refs-standard-libraries/200-core/211-conversion.md#2114-tryfromu-트레잇-실패할-수-있는-변환)). 그래서 호출에 `try`를 표시해야 합니다. 큰 정수를 작은 정수로, 부동소수를 정수로 옮기는 경우처럼 변환이 실패할 수 있을 때 씁니다.

```rok
let big: i64 = 10_000_000_000i64;
let n = try big.try_into::<i32>() catch e {
    ...
};
let m: i32 | none = try? big.try_into::<i32>();
```

## 2.14.5. 세 구문의 차이 요약

세 구문의 차이는 다음과 같습니다.

| 구문          | 실패할 수 있음     | 정보를 잃을 수 있음     | 트레잇       |
|---------------|--------------------|-------------------------|--------------|
| `as`          | 아니요             | 가능                    | 없음 (내장)  |
| `.into()`     | 아니요             | 아니요 (값 보존)        | `From<S>`    |
| `.try_into()` | 가능 (에러를 던짐) | 아니요 (성공 시)        | `TryFrom<S>` |

비트나 정밀도를 일부러 바꾸려면 `as`를 씁니다. 정보를 잃지 않는 안전한 확장만 필요하면 `.into()`를 씁니다. 실패 가능성을 직접 다루려면 `.try_into()`를 씁니다.

## 2.14.6. 문맥 기반 자동 변환

접미사 없는 숫자 리터럴은 변환되지 않고, 기대 타입에 따라 타입이 정해집니다. `let x: f64 = 1`의 `1`은 처음부터 `f64`입니다. 정확한 규칙은 [숫자 리터럴](../001-lexical/106-numeric-literals.md#163-접미사-없는-리터럴)에서 정의합니다.

반면 런타임 변수는 자동으로 변환하지 않습니다. 항상 위 세 구문 중 하나를 적어야 합니다.

```rok
let a: i64 = 100i64;
let b: i32 = a;                 // ERROR: 명시 변환 필요
let b: i32 = a as i32;          // OK
let b: i32 = try! a.try_into::<i32>();     // OK: 실패하면 panic
```

변환을 직접 적게 하는 이유는 모르는 사이에 정밀도를 잃지 않게 하려는 것입니다.

## 2.14.7. 변환 트레잇의 위치

`From<U>` / `Into`와 `TryFrom<U>` / `TryInto`는 `rok::core::convert`에 정의되어 있습니다([표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md)). 어떤 타입 쌍에 어떤 트레잇이 구현되어 있는지 보여 주는 전체 매트릭스는 표준 라이브러리 사양에서 정의합니다. `as`에는 트레잇이 없습니다. 허용하는 쌍은 [2.14.2](#2142-as-캐스트)에 정해진 목록입니다.
