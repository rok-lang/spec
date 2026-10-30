# 2.3. `struct`

`struct`는 여러 필드를 하나로 묶어 곱 타입을 정의하는 컨테이너 문법입니다. 이 장은 `struct`의 형태와 struct 규칙(인스턴스 생성, 기본값, spread, 가시성, 메모리 레이아웃)을 다룹니다.

## 2.3.1. 세 가지 형태

`struct` 선언은 다음과 같은 문법을 따릅니다.

```
struct_decl     ::= "struct" identifier generics? where_clause? "{" named_field_list? "}"
                  | "struct" identifier generics? "(" tuple_field_list? ")" where_clause? ";"   // tuple struct: where 는 필드 목록 뒤
                  | ";"
named_field_list ::= named_field (";" named_field)* ";"?
named_field     ::= identifier ":" type_expr
tuple_field_list ::= type_expr ("," type_expr)* ","?
```

본체의 모양에 따라 세 가지 형태로 나뉩니다. 아래에서 하나씩 다룹니다.

### named `struct`. 이름 있는 필드

가장 흔한 형태는 필드마다 이름을 붙이는 named `struct`입니다.

```rok
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

규칙은 다음과 같습니다. 본체는 `{ ... }`로 감쌉니다. 각 필드는 `이름: 타입;` 형태로 씁니다. 필드 끝의 `;`는 [문장 종결자](../001-lexical/103-statement-terminator.md)와 같은 종결자입니다. 필드 타입은 생략할 수 없습니다. [`any` 슬롯](202-any-slot.md)에서 정한 대로 struct 필드에는 점진 타이핑을 쓸 수 없기 때문입니다.

### tuple `struct`. 위치 기반 필드

tuple `struct`는 이름 없이 위치만으로 필드를 구별합니다.

```rok
struct Pair(i32, i32);
struct Wrapper(str);
struct Color(u8, u8, u8);
```

본체는 `( ... )`로 감싸고, 각 항에는 이름 없이 타입만 적습니다. 필드는 `.0`, `.1` 같은 위치 인덱스로 접근합니다. 정확한 문법은 표현식 사양에서 다룹니다. 필드가 하나인 `struct W(T);`도 쓸 수 있고, newtype 패턴으로 씁니다.

### unit `struct`. 필드 없는 마커

unit `struct`에는 필드가 하나도 없습니다.

```rok
struct Marker;
struct Sentinel;
```

본체가 없고, 선언 바로 뒤에 `;`로 끝납니다. 주로 인스턴스가 하나만 있으면 되는 마커 타입에 씁니다.

## 2.3.2. 인스턴스 생성: 미리 보기

형태별 인스턴스 생성 문법은 표현식 사양에서 정식으로 정의합니다. 이 문서는 각 형태가 어떤 모양인지만 미리 보여 줍니다.

```rok
let p = Point { x: 1.0f64, y: 2.0f64 };    // named
let q = Pair(1i32, 2i32);                   // tuple
let m = Marker;                              // unit
```

## 2.3.3. 필드 기본값

named `struct`의 필드에는 기본값을 줄 수 있습니다.

```rok
struct Config {
    retry: i32 = 3i32;
    timeout: f64 = 30.0f64;
    verbose: bool = false;
}
```

기본값에는 다음 제약이 있습니다. `const` 표현식만 쓸 수 있습니다. 리터럴, `const` 변수, `const` 함수 호출 정도를 쓸 수 있습니다. `const` 표현식의 정의는 [`const`](../005-modules/502-const.md)에 있습니다. 현재 시각이나 난수처럼 런타임 호출이 필요한 식은 쓸 수 없습니다. 그런 코드는 컴파일 에러입니다. 같은 struct의 다른 필드도 참조할 수 없습니다. 기본값은 인스턴스를 만들 때 그 필드를 생략한 경우에만 평가하고, 인스턴스마다 평가합니다. 평가가 결정론적이므로 컴파일러는 캐시 최적화를 적용할 수 있습니다.

이 제약은 결정론과 예측 가능한 비용, SemVer 안전성을 보장합니다. 런타임 초기화가 필요하면 따로 팩토리 함수를 두는 `fn new() -> T` 패턴을 쓰세요. 필드 기본값 안의 이름은 값을 만드는 자리가 아니라 struct를 선언한 모듈의 스코프에서 해소합니다.

## 2.3.4. 필드 spread

기존 인스턴스의 필드를 복사하되 일부 필드만 바꾼 새 인스턴스가 필요할 때가 있습니다.

```rok
let base = Point { x: 0.0f64, y: 0.0f64 };
let p = Point { x: 5.0f64, ..base };
```

여기서 `..base`는 base의 필드를 모두 가져오고, 직접 적은 필드만 새 값으로 덮어쓴다는 뜻입니다.

문법 규칙은 다음과 같습니다. `..` 토큰([구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)) 뒤에 인스턴스 값을 적습니다. 인스턴스 생성식 하나에 `..`는 한 번만 나올 수 있습니다. spread를 둘 이상 쓸 수 없습니다. spread 항은 직접 적은 필드 뒤에 두는 것이 관례입니다. 적은 필드를 필드 순서대로 먼저 평가하고, 그다음 base를 평가합니다.

## 2.3.5. 가시성

각 필드는 [키워드](../001-lexical/105-keywords.md)에서 정의한 가시성 키워드로 외부 노출 여부를 정할 수 있습니다.

```rok
struct Account {
    pub id: i64;        // 외부에서 읽기 가능
    pub balance: f64;
        secret: str;    // 같은 모듈 내부에서만 (기본 = 모듈 비공개)
}
```

tuple struct도 필드마다 가시성이 있고, 기본은 비공개입니다(`struct S(pub i32, i32);`). 생성, `x.N` 접근, 패턴에 적용하는 규칙과 가시성 단계의 의미는 [5.6.8](../005-modules/506-visibility.md#568-struct-필드의-가시성)에서 정합니다.

## 2.3.6. 메모리 레이아웃

컴파일러는 필드를 선언 순서대로 자연 정렬에 맞춰 배치합니다. 이 배치는 `#[repr(C)]`와 같습니다([6.2.2](../006-meta-system/602-built-in-attributes.md#622-repr-메모리-표현-명시)). `packed`와 `align(N)`으로 이 배치를 바꿀 수 있습니다.

## 2.3.7. 같은 모듈 내 중복 정의 금지

같은 모듈에서 같은 이름의 struct를 두 번 선언하면 컴파일 에러입니다. 한 struct 안에서 필드 이름이 겹쳐도 에러입니다. 이 규칙은 가시성 단계와 상관없이 적용됩니다. tuple struct와 unit struct의 생성자는 함수, `const`, `static`과 같은 이름공간에 있는 값입니다([4.1.10](../004-functions/401-function-declaration.md#4110-같은-모듈-내-중복-정의-금지)). 같은 모듈에 `struct Point(i32, i32);`와 이름이 `Point`인 `fn`, `const`, `static`이 함께 있으면 중복 정의(E200)입니다.

## 2.3.8. 값 의미

`struct`는 값 타입입니다. 대입, 인자 전달, 반환, 원소 읽기(`dict[k]`, `list[i]`, `entry(k).or_insert_with(..)`)는 struct를 복사합니다. 바꾼 내용이 다른 곳에서 보이려면 `ref` 매개변수나 `ref self`([4.7](../004-functions/407-ref-keyword.md))를 거치거나 값을 되써야 합니다.

```rok
let a = Point { x: 1.0, y: 2.0 };
let b = a;
b.x = 5.0;                        // a.x 는 그대로

let ps: List<Point> = [a];
let q = ps[0];
q.x = 9.0;                        // ps[0].x 는 그대로
ps[0] = q;                        // 되씀
```

값을 공유하려면 참조 핸들을 씁니다. 컬렉션(`List`, `Dict`, `Set`, `StrBuf`, `Deque` 등, [stdlib 2.9](../../refs-standard-libraries/200-core/209-collections.md#참조-핸들)), `Mutex`, `RwLock`, atomic, 채널, `[ref x]` 캡처 셀이 참조 핸들입니다. 핸들을 복사하면 같은 객체를 공유합니다.

|                     | 값 타입 (`struct`, `enum`, 튜플, 원시 타입) | 참조 핸들 (`List`, `Dict`, `Mutex`, ...) |
|---------------------|---------------------------------------------|----------------------------------------|
| `let b = a;`        | 독립된 사본                                 | 같은 객체                              |
| 필드 / 원소 쓰기    | 사본에만                                    | 모든 핸들에서 보임                     |
| 독립된 사본         | 대입 자체                                   | `.clone()`                             |

struct 필드가 핸들이면 핸들만 복사됩니다. 그래서 struct를 복사해도 핸들이 가리키는 객체는 공유됩니다.
