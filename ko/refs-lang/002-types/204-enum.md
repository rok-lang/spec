# 2.4. `enum`

`enum`은 여러 변형(variant) 중 하나를 값으로 담는 타입을 정의합니다. 합 타입과 비슷하지만 이름이 있는 컨테이너라는 점이 다릅니다.

## 2.4.1. 기본 형태

`enum`의 문법은 다음과 같습니다.

```
enum_decl   ::= visibility? "enum" identifier generics? where_clause? "{" variant_list? "}"
variant_list ::= variant ("," variant)* ","?
variant     ::= identifier                                  // unit
              | identifier "(" type_expr ("," type_expr)* ","? ")"  // tuple
              | identifier "{" named_field_list? "}"        // struct
              | identifier "=" const_expr                   // discriminant
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

본체는 `{ ... }`로 감싸고, 변형은 콤마로 구분합니다. 후행 콤마를 써도 됩니다.

## 2.4.2. 네 가지 변형 종류

`enum`의 변형은 데이터를 담는 방식에 따라 네 가지로 나뉩니다.

### unit variant: 데이터 없음

가장 단순한 unit 변형은 데이터를 담지 않습니다.

```rok
enum Color {
    Red,
    Green,
    Blue,
}
```

데이터 없이 라벨만으로 이루어집니다.

### tuple variant: 위치 기반 필드

tuple 변형에는 위치 기반 필드가 있습니다.

```rok
enum Shape {
    Circle(f64),
    Rectangle(f64, f64),
}
```

[`struct`](203-struct.md)의 tuple `struct`와 같은 위치 기반 필드입니다.

### named variant: 이름 있는 필드

named 변형에는 이름 있는 필드가 있습니다.

```rok
enum Event {
    Click { x: i32; y: i32; },
    Scroll { dy: i32; },
    Resize { w: i32; h: i32; },
}
```

각 변형의 필드는 named `struct`의 필드와 모양이 같습니다. 필드는 `;`로 끝나고, 이 점도 [`struct`](203-struct.md)와 같습니다.

### value variant: 명시적 정수 값

value 변형은 변형에 정수 값을 직접 붙입니다.

```rok
enum Status {
    Ok = 0,
    NotFound = 404,
    InternalError = 500,
}
```

붙이는 값은 `const` 정수 표현식이어야 합니다. 두 변형에 같은 값을 붙일 수 없고, 모든 값은 태그 타입의 범위에 들어가야 합니다([6.2.2](../006-meta-system/602-built-in-attributes.md#622-repr-메모리-표현-명시)).

이 형태는 C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 호환, 외부 프로토콜 대응, 비트 플래그 등에 씁니다. value 변형은 다른 종류의 변형과 섞을 수 없습니다. value 변형이 하나라도 있으면 `enum` 전체가 value 변형으로만 이루어져야 합니다.

### 정수로 캐스트

페이로드가 없는 `enum`은 `as`로 정수로 캐스트할 수 있습니다. unit 변형만 있거나 value 변형만 있는 경우입니다. 결과는 그 변형의 값이고, unit 변형이면 태그(0부터 센 선언 순번)입니다.

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

기본 레이아웃은 컴파일러에 맡깁니다. 컴파일러는 태그와 페이로드의 정렬과 패킹을 정할 수 있습니다. 이 자유가 있어서 [niche 최적화](207-niche-optimization.md)를 할 수 있습니다.

C ABI 호환이 필요하면 `#[repr(...)]` 속성으로 표현을 강제할 수 있습니다. 정확한 의미는 [내장 속성](../006-meta-system/602-built-in-attributes.md)에서 정의합니다.

## 2.4.6. 동일 모듈 내 중복 정의 금지

같은 모듈에서 같은 이름의 `enum`을 두 번 선언하면 컴파일 에러입니다. 한 `enum` 안에서 변형 이름이 겹쳐도 에러입니다.

## 2.4.7. `enum`, 합 타입, `union` 비교: 미리 보기

세 형태를 비교하면 다음과 같습니다.

| 형태              | 명명                                   | 태그 | 변형끼리 메모리 공유            |
|-------------------|----------------------------------------|------|---------------------------------|
| `enum`            | 명명된 변형이 컨테이너 안에 정의됨     | 있음 | 아니오 (별개 페이로드)          |
| 합 타입 (`A \| B`) | 변형 자체가 기존 타입                  | 있음 | 아니오 (인라인 태그 + 페이로드) |
| `union`           | 비태그. 같은 메모리를 여러 형으로 해석 | 없음 | 예 (의도한 겹침)                |

자세한 차이는 각 컨테이너 문서에서 정의합니다.

## 2.4.8. 기대 타입으로 정하는 맨 변형

변형은 enum의 경로와 함께 쓰거나(`Ordering::Less`, `Shape::Circle`), 그 자리가 기대하는 타입이 enum을 정해 줄 때는 맨 이름으로 씁니다. 보통의 이름 찾기([5.7.1](../005-modules/507-paths.md#571-경로의-시작-식별자))로 해소되지 않는 맨 이름은, 기대 타입이 변형 `V`를 가진 enum `E`이거나 변형 `V`를 가진 enum 멤버가 정확히 하나인 합 타입일 때 그 변형 `V`를 가리킵니다. 사용자의 enum이든 표준 라이브러리의 enum이든 똑같이 `use` 없이 적용됩니다.

기대 타입은 다음에서 옵니다.

- 타입을 적은 `let`
- 매개변수(메서드 포함)와 매개변수의 기본값
- `return`과 함수의 꼬리 식
- 구조체 리터럴의 필드
- 원소 타입을 아는 리스트와 딕셔너리 리터럴의 원소
- 왼쪽이 `E` 타입인 `==` / `!=`의 오른쪽
- `E` 타입 검사 대상에 대한 `match`, `if let`, `let`의 패턴([3.11](../003-expressions/311-patterns.md))

```rok
enum Shape { Circle, Square, Tri(i32) }

fn area(s: Shape = Square) -> i32 {
    match s { Circle => 1, Square => 2, Tri(n) => n }
}

fn main() {
    let s: Shape = Tri(3);
    let o: Ordering = Less;
    let n = Atomic::new(0);
    n.store(1, SeqCst);                     // StoreOrdering::SeqCst
    let lt = 1.cmp(2) == Less;
    println(f"{area()} {area(Circle)} {lt}");
}
```

지역 변수, 항목, `use`가 먼저입니다. 기대 타입은 다른 무엇으로도 해소되지 않는 이름에만 쓰는 대체 규칙이므로, 나중에 core enum에 변형이 생겨도 사용자의 이름을 빼앗지 않습니다. 제네릭 enum의 인스턴스는 기대 타입의 것입니다(`-> Result<i32, str>` 함수 안의 `Ok(1)`은 `Result<i32, str>::Ok`).

기대 타입이 없으면 맨 변형은 에러이고, 에러는 그 변형을 가장 짧은 경로로 알려 줍니다.

```rok
let o = Less;   // ERROR: `Less` is not in scope: it is the variant `Ordering::Less`
                // help: write `Ordering::Less`
```
