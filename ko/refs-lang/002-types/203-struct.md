# 2.3. `struct`

`struct` 는 여러 필드를 하나로 묶어 곱 타입을 정의하는 컨테이너 문법입니다. 이 장에서는 `struct` 가 가질 수 있는 형태와, 인스턴스 생성·기본값·spread·가시성·메모리 레이아웃에 이르기까지 구조체와 관련된 규칙을 차례로 살펴봅니다.

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

본체의 모양에 따라 세 가지 형태가 갈리며, 아래에서 각 형태를 하나씩 다룹니다.

### named `struct`. 이름 있는 필드

가장 흔하게 쓰이는 형태는 필드마다 이름을 붙이는 named `struct` 입니다.

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

이 형태에는 몇 가지 규칙이 따릅니다. 우선 본체는 `{ ... }` 로 둘러쌉니다. 각 필드는 `이름: 타입;` 형태로 쓰며, 필드 끝의 `;` 는 [문장 종결자](../001-lexical/103-statement-terminator.md) 와 같은 종결자에 해당합니다. 또한 필드 타입 명시는 필수인데, [`any` 슬롯](202-any-slot.md) 에서 정한 대로 구조체 필드에는 점진이 허용되지 않기 때문입니다.

### tuple `struct`. 위치 기반 필드

다음으로, 이름 대신 위치만으로 필드를 식별하는 tuple `struct` 가 있습니다.

```rok
struct Pair(i32, i32);
struct Wrapper(str);
struct Color(u8, u8, u8);
```

이 형태에서는 본체를 `( ... )` 로 둘러싸며, 각 항에는 타입만 적고 이름은 적지 않습니다. 필드 접근은 `.0`, `.1` 과 같은 위치 인덱스로 이루어지고, 구체적인 문법은 표현식 사양에서 다룹니다. 한편 한 필드짜리 형태인 `struct W(T);` 도 허용되며, 이는 newtype 패턴으로 활용할 수 있습니다.

### unit `struct`. 필드 없는 마커

마지막으로, 필드가 전혀 없는 unit `struct` 형태도 있습니다.

```rok
struct Marker;
struct Sentinel;
```

이 형태는 본체가 없고 선언 직후 곧바로 `;` 로 종료됩니다. 단일 인스턴스만 의미가 있는 마커 타입을 표현할 때 주로 사용합니다.

## 2.3.2. 인스턴스 생성. 미리보기

세 형태 각각의 인스턴스 생성 문법은 표현식 사양에서 정식으로 정의합니다. 본 문서에서는 형태별로 어떤 모양이 되는지를 미리 보여 드리는 정도로만 다룹니다.

```rok
let p = Point { x: 1.0f64, y: 2.0f64 };    // named
let q = Pair(1i32, 2i32);                   // tuple
let m = Marker;                              // unit
```

## 2.3.3. 필드 기본값

named `struct` 의 필드는 기본값을 가질 수 있습니다.

```rok
struct Config {
    retry: i32 = 3i32;
    timeout: f64 = 30.0f64;
    verbose: bool = false;
}
```

다만 기본값에는 다음과 같은 제약이 따릅니다. 먼저 `const` 표현식만 허용되며, 리터럴이나 `const` 변수, `const` 함수 호출 정도가 사용 가능합니다. `const` 표현식의 정의는 [`const`](../005-modules/502-const.md) 에서 자세히 다룹니다. 또한 현재 시각이나 난수처럼 런타임 호출이 필요한 식은 쓸 수 없으며, 그러한 코드는 컴파일 에러가 됩니다. 같은 구조체 안의 다른 필드를 참조하는 것도 금지됩니다. 평가 시점은 인스턴스 생성 시 해당 필드를 생략한 경우로 한정되며, 매 인스턴스마다 평가가 수행됩니다. 이때 평가가 결정론적이므로 컴파일러는 캐시 최적화를 적용할 수 있습니다.

이러한 제약을 두는 이유는 결정론과 예측 가능한 비용, 그리고 SemVer 안전성을 확보하기 위해서입니다. 만약 런타임 초기화가 필요하다면 별도의 팩토리 함수, 즉 `fn new() -> T` 패턴을 활용하는 것이 좋습니다. 필드 기본값 안의 이름은 값을 만드는 자리가 아니라 구조체를 선언한 모듈의 스코프에서 해소합니다.

## 2.3.4. 필드 spread

기존 인스턴스의 필드를 그대로 복사하면서 일부 필드만 바꿔 새로운 인스턴스를 만들고 싶을 때가 있습니다.

```rok
let base = Point { x: 0.0f64, y: 0.0f64 };
let p = Point { x: 5.0f64, ..base };
```

여기서 `..base` 는 base 의 모든 필드를 그대로 가져오되, 명시한 필드는 새 값으로 덮어쓴다는 의미입니다.

문법적으로는 다음과 같은 규칙이 적용됩니다. `..` 토큰 ([구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)) 뒤에는 인스턴스 값을 적습니다. 한 인스턴스 생성식 안에서 `..` 는 한 번만 등장할 수 있으며, 두 개 이상의 spread 는 허용되지 않습니다. 또한 spread 항은 명시 필드들 뒤에 두는 것이 관용적인 작성 방식입니다. 적은 필드를 필드 순서대로 먼저 평가하고, 그 다음 base 를 평가합니다.

## 2.3.5. 가시성

각 필드는 [키워드](../001-lexical/105-keywords.md) 에서 정의한 가시성 키워드를 활용해 외부 노출 여부를 조절할 수 있습니다.

```rok
struct Account {
    pub id: i64;        // 외부에서 읽기 가능
    pub balance: f64;
        secret: str;    // 같은 모듈 내부에서만 (기본 = 모듈 비공개)
}
```

tuple struct 의 필드도 각자 가시성을 가집니다 (`struct S(pub i32, i32);`, 기본 비공개). 생성, `x.N` 접근, 패턴에서의 규칙과 가시성 단계의 의미는 [5.6.8](../005-modules/506-visibility.md#568-struct-필드의-가시성) 이 정합니다.

## 2.3.6. 메모리 레이아웃

필드는 선언 순서와 자연 정렬로 놓이며, `#[repr(C)]` 와 같습니다 ([6.2.2](../006-meta-system/602-built-in-attributes.md#622-repr-메모리-표현-명시)). `packed` 와 `align(N)` 이 이를 바꿉니다.

## 2.3.7. 같은 모듈 내 중복 정의 금지

같은 모듈 안에서 같은 이름의 구조체를 두 번 선언하면 컴파일 에러가 발생합니다. 한 구조체 안의 필드 이름이 겹쳐도 에러입니다. 이는 가시성 단계와는 무관하게 적용되는 규칙입니다. tuple 구조체나 unit 구조체의 생성자는 함수, `const`, `static` 의 이름공간에 있는 값입니다 ([4.1.10](../004-functions/401-function-declaration.md#4110-같은-모듈-내-중복-정의-금지)): 같은 모듈의 `struct Point(i32, i32);` 와 `Point` 라는 `fn`, `const`, `static` 은 중복 정의 (E200) 입니다.

## 2.3.8. 값 의미

`struct` 는 값 타입입니다. 대입, 인자 전달, 반환, 원소 읽기 (`dict[k]`, `list[i]`, `entry(k).or_insert_with(..)`) 는 구조체를 복사합니다. 변경은 `ref` 매개변수나 `ref self` ([4.7](../004-functions/407-ref-keyword.md)) 를 거치거나 값을 되써야만 다른 곳에 보입니다.

```rok
let a = Point { x: 1.0, y: 2.0 };
let b = a;
b.x = 5.0;                        // a.x 는 그대로

let ps: List<Point> = [a];
let q = ps[0];
q.x = 9.0;                        // ps[0].x 는 그대로
ps[0] = q;                        // 되씀
```

공유는 참조 핸들로 합니다: 컬렉션 (`List`, `Dict`, `Set`, `StrBuf`, `Deque`, ..., [stdlib 2.9](../../refs-standard-libraries/200-core/209-collections.md#참조-핸들)), `Mutex`, `RwLock`, atomic, 채널, `[ref x]` 캡처 셀. 핸들을 복사하면 객체를 공유합니다.

|                     | 값 타입 (`struct`, `enum`, 튜플, 원시 타입) | 참조 핸들 (`List`, `Dict`, `Mutex`, ...) |
|---------------------|---------------------------------------------|----------------------------------------|
| `let b = a;`        | 독립된 사본                                 | 같은 객체                              |
| 필드 / 원소 쓰기    | 사본에만                                    | 모든 핸들에서 보임                     |
| 독립된 사본         | 대입 자체                                   | `.clone()`                             |

핸들을 가진 구조체 필드는 핸들이 복사됩니다: 구조체를 복사해도 그 객체는 공유됩니다.
