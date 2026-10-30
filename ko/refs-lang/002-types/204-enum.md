# 2.4. `enum`

`enum`은 여러 변형(variant) 중 하나의 값을 갖는 타입을 정의합니다. 합 타입과 비슷하지만 이름이 있는 컨테이너라는 점이 다릅니다.

## 2.4.1. 기본 형태

`enum`의 문법은 다음과 같습니다.

```
enum_decl   ::= visibility? "enum" identifier generics? where_clause? "{" variant_list? "}"
variant_list ::= variant ("," variant)* ","?
variant     ::= visibility? identifier                                  // unit
              | visibility? identifier "(" type_expr ("," type_expr)* ","? ")"  // tuple
              | visibility? identifier "{" named_field_list? "}"        // struct
              | visibility? identifier "=" const_expr                   // discriminant
```

가장 단순한 형태는 다음과 같습니다.

```rok
enum Direction {
    North,
    East,
    South,
    West,
}
```

본체는 `{ ... }`로 감싸고, 변형은 콤마로 구분합니다. 후행 콤마도 허용합니다.

## 2.4.2. 네 가지 변형 종류

`enum`의 변형은 데이터를 담는 방식에 따라 네 가지로 나뉩니다.

### unit variant. 데이터 없음

가장 단순한 unit 변형은 데이터를 담지 않습니다.

```rok
enum Color {
    Red,
    Green,
    Blue,
}
```

데이터 없이 라벨만으로 이루어집니다.

### tuple variant. 위치 기반 필드

tuple 변형은 위치 기반 필드를 갖습니다.

```rok
enum Shape {
    Circle(f64),
    Rectangle(f64, f64),
}
```

[`struct`](203-struct.md)의 tuple `struct`와 같은 위치 기반 필드입니다.

### named variant. 이름 있는 필드

named 변형은 이름 있는 필드를 갖습니다.

```rok
enum Event {
    Click { x: i32; y: i32; },
    Scroll { dy: i32; },
    Resize { w: i32; h: i32; },
}
```

각 변형은 named `struct`와 같은 모양의 필드를 갖습니다. 필드는 `;`로 끝나며, 이 점도 [`struct`](203-struct.md)와 같습니다.

### value variant. 명시적 정수 값

value 변형은 변형에 정수 값을 직접 붙입니다.

```rok
enum Status {
    Ok = 0,
    NotFound = 404,
    InternalError = 500,
}
```

붙이는 값은 `const` 정수 표현식이어야 합니다. 두 변형이 같은 값을 가질 수 없으며, 모든 값은 tag 타입 범위에 들어가야 합니다([6.2.2](../006-meta-system/602-built-in-attributes.md#622-repr-메모리-표현-명시)).

이 형태는 C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 호환, 외부 프로토콜 매핑, 비트 플래그 등에 씁니다. value 변형은 다른 종류의 변형과 섞을 수 없습니다. value 변형이 하나라도 있으면 `enum` 전체가 value 변형으로만 이루어져야 합니다.

### 정수로의 cast

payload가 없는 `enum`(unit 변형만 있거나 value 변형만 있는 경우)은 `as`로 정수로 cast할 수 있습니다. 결과는 그 변형의 값이고, unit 변형이면 tag(0부터 센 선언 순번)입니다.

```rok
let code = Status::NotFound as i32;   // 404
```

## 2.4.3. 변형 종류 혼합

unit, tuple, named 변형은 한 `enum` 안에 자유롭게 섞을 수 있습니다. value 변형은 앞에서 정한 대로 단독으로만 쓸 수 있습니다.

다음 예는 세 종류를 함께 씁니다.

```rok
enum Result {
    Ok(i32),                    // tuple
    Err { code: i32; msg: str; }, // named
    Pending,                    // unit
}
```

## 2.4.4. 가시성

변형은 자체 가시성이 없고 `enum`의 가시성을 따릅니다. `pub enum`의 변형은 모두 공개입니다([5.6.9](../005-modules/506-visibility.md#569-enum-변형의-가시성)).

## 2.4.5. 메모리 레이아웃

기본 레이아웃은 컴파일러에 맡깁니다. 태그와 페이로드의 정렬과 패킹은 컴파일러가 정할 수 있습니다. 이 자유가 있어서 [niche 최적화](207-niche-optimization.md)를 할 수 있습니다.

C ABI 호환이 필요하면 `#[repr(...)]` 속성으로 표현을 강제할 수 있습니다. 정확한 의미는 [내장 속성](../006-meta-system/602-built-in-attributes.md)에서 정의합니다.

## 2.4.6. 동일 모듈 내 중복 정의 금지

같은 모듈에서 같은 이름의 `enum`을 두 번 선언하면 컴파일 에러입니다. 한 `enum` 안에서 변형 이름이 겹쳐도 에러입니다.

## 2.4.7. `enum` vs 합 타입 vs `union` 미리보기

세 형태를 비교하면 다음과 같습니다.

| 형태              | 명명                                   | 태그 | 변형이 같은 메모리를 공유?      |
|-------------------|----------------------------------------|------|---------------------------------|
| `enum`            | 명명된 변형이 컨테이너 안에 정의됨     | 있음 | 아니오 (별개 페이로드)          |
| 합 타입 (`A \| B`) | 변형 자체가 기존 타입                  | 있음 | 아니오 (인라인 태그 + 페이로드) |
| `union`           | 비태그. 같은 메모리를 여러 형으로 해석 | 없음 | 예 (의도된 overlap)             |

자세한 차이는 각 컨테이너 문서에서 정의합니다.
