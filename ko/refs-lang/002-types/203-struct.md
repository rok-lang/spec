# 2.3. `struct`

`struct` 는 여러 필드를 묶은 곱 타입을 정의하는 컨테이너 문법이다.

## 2.3.1. 세 가지 형태

```
struct_decl     ::= "struct" identifier generics? struct_body
struct_body     ::= "{" named_field_list? "}"
                  | "(" tuple_field_list? ")" ";"
                  | ";"
named_field_list ::= named_field (";" named_field)* ";"?
named_field     ::= identifier ":" type_expr
tuple_field_list ::= type_expr ("," type_expr)* ","?
```

### named `struct`. 이름 있는 필드

```nok
struct Point {
    x: f64;
    y: f64;
}

struct User {
    id: i64;
    name: str;
    age: i32;
}
```

규칙:

- 본체는 `{ ... }` 로 둘러싼다.
- 각 필드는 `이름: 타입;` 형태다. 필드 끝의 `;` 는 [문장 종결자](../001-lexical/103-statement-terminator.md) 와 같은 종결자다.
- 필드 타입 명시는 필수다. [`any` 슬롯](202-any-slot.md) 에서 정한 대로 구조체 필드에는 점진이 허용되지 않는다.

### tuple `struct`. 위치 기반 필드

```nok
struct Pair(i32, i32);
struct Wrapper(str);
struct Color(u8, u8, u8);
```

규칙:

- 본체는 `( ... )` 로 둘러싸고, 각 항은 타입만 적는다 (이름 없음).
- 필드 접근은 `.0`, `.1`, ... 위치 인덱스로 한다 (구체 문법은 표현식 사양).
- 한 필드짜리 (`struct W(T);`) 도 허용된다. newtype 패턴.

### unit `struct`. 필드 없는 마커

```nok
struct Marker;
struct Sentinel;
```

규칙:

- 본체가 없고, 선언 직후 `;` 로 종료된다.
- 단일 인스턴스만 의미가 있는 마커 타입으로 쓴다.

## 2.3.2. 인스턴스 생성. 미리보기

세 형태 각각의 인스턴스 생성 문법은 표현식 사양에서 정의한다. 본 문서에서는 형태별로 어떤 모양인지만 보인다.

```nok
let p = Point { x: 1.0f64, y: 2.0f64 };    // named
let q = Pair(1i32, 2i32);                   // tuple
let m = Marker;                              // unit
```

## 2.3.3. 필드 기본값

named `struct` 의 필드는 기본값을 가질 수 있다.

```nok
struct Config {
    retry: i32 = 3i32;
    timeout: f64 = 30.0f64;
    verbose: bool = false;
}
```

기본값의 제약:

- `const` 표현식만 허용한다. 리터럴, `const` 변수, `const` 함수 호출이 가능하다. `const` 표현식의 정의는 [`const`](../005-modules/502-const.md) 에서 한다.
- 런타임 호출 (예: 현재 시각·난수) 은 쓸 수 없다. 컴파일 에러.
- 같은 구조체의 다른 필드를 참조할 수 없다.
- 인스턴스 생성 시 해당 필드를 생략한 경우에만 평가한다.
- 매 인스턴스마다 평가한다. 결정론적이므로 컴파일러가 캐시 최적화 가능.

이 제약은 결정론·예측 가능 비용·SemVer 안전을 위한 것이다. 런타임 초기화가 필요하면 별도 팩토리 함수 (`fn new() -> T`) 패턴을 쓴다.

## 2.3.4. 필드 spread

기존 인스턴스 필드를 복사하면서 일부만 바꿔 새 인스턴스를 만들 수 있다.

```nok
let base = Point { x: 0.0f64, y: 0.0f64 };
let p = Point { x: 5.0f64, ..base };
```

`..base` 는 base 의 모든 필드를 가져오되 명시한 필드는 덮어쓴다는 뜻이다.

규칙:

- `..` 토큰 ([구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)) 뒤에 인스턴스 값을 적는다.
- 한 인스턴스 생성식에서 `..` 는 한 번만 등장한다. 두 개 이상 spread 는 허용하지 않는다.
- spread 항은 명시 필드들 뒤에 둔다 (관용).

## 2.3.5. 가시성

각 필드는 [키워드](../001-lexical/105-keywords.md) 의 가시성 키워드로 외부 노출 여부를 조절할 수 있다.

```nok
struct Account {
    pub id: i64;        // 외부에서 읽기 가능
    pub balance: f64;
        secret: str;    // 같은 모듈 내부에서만 (기본 = 모듈 비공개)
}
```

가시성 단계의 정확한 의미는 [가시성](../005-modules/506-visibility.md) 에서 정의한다.

## 2.3.6. 메모리 레이아웃

기본 레이아웃은 컴파일러 자유다. 필드 순서·정렬·패킹을 컴파일러가 결정할 수 있다. 이 자유 덕분에 niche 최적화·캐시 친화 정렬 등이 가능하다.

C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 호환이 필요하면 `#[repr(C)]` 속성을 부착한다. 정확한 의미는 [내장 속성](../006-meta-system/602-built-in-attributes.md) 에서 정의한다. 본 문서는 기본 레이아웃이 자유라는 사실만 명시한다.

## 2.3.7. 같은 모듈 내 중복 정의 금지

같은 모듈 안에서 같은 이름의 구조체를 두 번 선언하면 컴파일 에러다. 가시성 단계와 무관하다.
