# 2.4. `enum`

`enum` 은 여러 변형 (variant) 중 하나의 값을 갖는 타입을 정의합니다. 합 타입과 비슷하지만, 명명된 컨테이너라는 점에서 구별됩니다.

## 2.4.1. 기본 형태

`enum` 의 문법은 다음과 같이 정의됩니다.

```
enum_decl   ::= visibility? "enum" identifier generics? "{" variant_list? "}"
variant_list ::= variant ("," variant)* ","?
variant     ::= visibility? identifier                                  // unit
              | visibility? identifier "(" type_expr ("," type_expr)* ","? ")"  // tuple
              | visibility? identifier "{" named_field_list? "}"        // struct
              | visibility? identifier "=" const_expr                   // discriminant
```

가장 단순한 형태의 예를 들면 다음과 같습니다.

```rok
enum Direction {
    North,
    East,
    South,
    West,
}
```

이 선언에서 볼 수 있듯이 본체는 `{ ... }` 로 둘러싸며, 각 변형은 콤마로 구분합니다. 또한 마지막 후행 콤마도 허용합니다.

## 2.4.2. 네 가지 변형 종류

`enum` 의 변형은 데이터를 어떻게 담는지에 따라 네 가지로 나뉩니다.

### unit variant. 데이터 없음

먼저 가장 단순한 unit 변형은 데이터를 담지 않습니다.

```rok
enum Color {
    Red,
    Green,
    Blue,
}
```

이처럼 데이터가 없는 단순 라벨로만 구성됩니다.

### tuple variant. 위치 기반 필드

다음으로 tuple 변형은 위치 기반 필드를 갖습니다.

```rok
enum Shape {
    Circle(f64),
    Rectangle(f64, f64),
}
```

이 변형은 [`struct`](203-struct.md) 의 tuple `struct` 와 같은 위치 기반 필드를 갖습니다.

### named variant. 이름 있는 필드

이어서 named 변형은 이름이 붙은 필드를 갖습니다.

```rok
enum Event {
    Click { x: i32; y: i32; },
    Scroll { dy: i32; },
    Resize { w: i32; h: i32; },
}
```

각 변형이 named `struct` 와 같은 모양의 필드를 갖습니다. 필드 끝은 `;` 로 종결하여 [`struct`](203-struct.md) 와 일관된 형태를 유지합니다.

### value variant. 명시적 정수 값

마지막으로 value 변형은 변형에 정수 값을 직접 부여합니다.

```rok
enum Status {
    Ok = 0,
    NotFound = 404,
    InternalError = 500,
}
```

이때 부여하는 값은 `const` 정수 표현식이어야 합니다.

이 형태는 C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 호환이나 외부 프로토콜 매핑, 비트 플래그 등에 사용합니다. 다만 value variant 는 다른 변형 종류와 섞을 수 없습니다. value 변형이 하나라도 있으면 `enum` 전체가 value 변형으로만 구성되어야 합니다.

## 2.4.3. 변형 종류 혼합

unit, tuple, named 변형은 한 `enum` 안에 자유롭게 섞을 수 있습니다. 단 value 변형은 앞서 정한 대로 단독 사용만 허용합니다.

다음 예시는 세 가지 변형을 함께 사용한 모습을 보여줍니다.

```rok
enum Result {
    Ok(i32),                    // tuple
    Err { code: i32; msg: str; }, // named
    Pending,                    // unit
}
```

## 2.4.4. 가시성

전체 `enum` 의 가시성과 각 변형의 가시성은 따로 조절할 수 있습니다. 정확한 규칙은 가시성 사양에서 정의합니다.

## 2.4.5. 메모리 레이아웃

기본 레이아웃은 컴파일러의 자유에 맡깁니다. 즉 태그와 페이로드의 정렬과 패킹을 컴파일러가 결정할 수 있습니다. 이러한 자유 덕분에 [niche 최적화](207-niche-optimization.md) 가 가능합니다.

만약 C ABI 호환이 필요하다면 `#[repr(...)]` 속성으로 표현 방식을 강제할 수 있습니다. 정확한 의미는 [내장 속성](../006-meta-system/602-built-in-attributes.md) 에서 정의합니다.

## 2.4.6. 동일 모듈 내 중복 정의 금지

같은 모듈 안에서 같은 이름의 `enum` 을 두 번 선언하면 컴파일 에러가 발생합니다.

## 2.4.7. `enum` vs 합 타입 vs `union` 미리보기

세 가지 형태의 차이를 한눈에 비교하면 다음과 같습니다.

| 형태              | 명명                                   | 태그 | 변형이 같은 메모리를 공유?      |
|-------------------|----------------------------------------|------|---------------------------------|
| `enum`            | 명명된 변형이 컨테이너 안에 정의됨     | 있음 | 아니오 (별개 페이로드)          |
| 합 타입 (`A \| B`) | 변형 자체가 기존 타입                  | 있음 | 아니오 (인라인 태그 + 페이로드) |
| `union`           | 비태그. 같은 메모리를 여러 형으로 해석 | 없음 | 예 (의도된 overlap)             |

자세한 차이는 각 컨테이너 문서에서 정의합니다.
