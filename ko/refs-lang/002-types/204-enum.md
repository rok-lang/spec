# 2.4. `enum`

`enum` 은 여러 변형 (variant) 중 하나의 값을 갖는 타입을 정의한다. 합 타입과 비슷하지만, 명명된 컨테이너라는 점에서 구별된다.

## 2.4.1. 기본 형태

```
enum_decl   ::= "enum" identifier generics? "{" variant_list? "}"
variant_list ::= variant ("," variant)* ","?
variant     ::= identifier                                  // unit
              | identifier "(" type_expr ("," type_expr)* ","? ")"  // tuple
              | identifier "{" named_field_list? "}"        // struct
              | identifier "=" const_expr                   // discriminant
```

```nok
enum Direction {
    North,
    East,
    South,
    West,
}
```

- 본체는 `{ ... }` 로 둘러싼다.
- 각 변형은 콤마로 구분한다.
- 마지막 후행 콤마 허용.

## 2.4.2. 네 가지 변형 종류

### unit variant. 데이터 없음

```nok
enum Color {
    Red,
    Green,
    Blue,
}
```

데이터가 없는 단순 라벨.

### tuple variant. 위치 기반 필드

```nok
enum Shape {
    Circle(f64),
    Rectangle(f64, f64),
}
```

이 변형은 [`struct`](203-struct.md) 의 tuple `struct` 와 같은 위치 기반 필드를 갖는다.

### named variant. 이름 있는 필드

```nok
enum Event {
    Click { x: i32; y: i32; },
    Scroll { dy: i32; },
    Resize { w: i32; h: i32; },
}
```

각 변형이 named `struct` 와 같은 모양의 필드를 갖는다. 필드 끝은 `;` 로 종결한다 ([`struct`](203-struct.md) 와 일관).

### value variant. 명시적 정수 값

```nok
enum Status {
    Ok = 0,
    NotFound = 404,
    InternalError = 500,
}
```

변형에 정수 값을 직접 부여한다. 값은 `const` 정수 표현식이어야 한다.

이 형태는 C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 호환·외부 프로토콜 매핑·비트 플래그 등에 쓴다. value variant 는 다른 변형 종류와 섞일 수 없다. value 변형이 하나라도 있으면 `enum` 전체가 value 변형으로만 구성되어야 한다.

## 2.4.3. 변형 종류 혼합

unit / tuple / named 변형은 한 `enum` 안에 자유롭게 섞을 수 있다. 단 value 변형은 위에서 정한 대로 단독 사용만 허용한다.

```nok
enum Result {
    Ok(i32),                    // tuple
    Err { code: i32; msg: str; }, // named
    Pending,                    // unit
}
```

## 2.4.4. 가시성

전체 `enum` 의 가시성과 각 변형의 가시성은 따로 조절할 수 있다. 정확한 규칙은 가시성 사양에서 정의한다.

## 2.4.5. 메모리 레이아웃

기본 레이아웃은 컴파일러 자유다. 태그 + 페이로드의 정렬·패킹을 컴파일러가 결정할 수 있다. 이 자유 덕분에 [niche 최적화](207-niche-optimization.md) 가 가능하다.

C ABI 호환이 필요하면 `#[repr(...)]` 속성으로 표현 방식을 강제할 수 있다. 정확한 의미는 [내장 속성](../006-meta-system/602-built-in-attributes.md) 에서 정의한다.

## 2.4.6. 동일 모듈 내 중복 정의 금지

같은 모듈 안에서 같은 이름의 `enum` 을 두 번 선언하면 컴파일 에러다.

## 2.4.7. `enum` vs 합 타입 vs `union` 미리보기

| 형태               | 명명                                   | 태그 | 변형이 같은 메모리를 공유?      |
| ------------------ | -------------------------------------- | ---- | ------------------------------- |
| `enum`             | 명명된 변형이 컨테이너 안에 정의됨     | 있음 | 아니오 (별개 페이로드)          |
| 합 타입 (`A \| B`) | 변형 자체가 기존 타입                  | 있음 | 아니오 (인라인 태그 + 페이로드) |
| `union`            | 비태그. 같은 메모리를 여러 형으로 해석 | 없음 | 예 (의도된 overlap)             |

자세한 차이는 각 컨테이너 문서에서 정의한다.
