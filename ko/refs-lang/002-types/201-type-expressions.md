# 2.1. 타입 표현식

타입 표현식은 소스 코드에서 타입을 가리키는 표기입니다. 변수 선언의 `:` 뒤, 함수 매개변수의 `:` 뒤, 반환 타입의 `->` 뒤, 제네릭 인자, 형 변환의 `as` 뒤 등 여러 자리에 나옵니다.

이 문서는 형식만 정의합니다. 어떤 이름이 어떤 타입을 가리키는지는 표준 라이브러리가 정합니다.

## 2.1.1. 기본 형태

타입 표현식의 기본 문법은 다음과 같습니다.

```
type_expr   ::= sum_type
sum_type    ::= unary_type ("|" unary_type)*
unary_type  ::= base_type "?"*                  // T?: optional (2.1.5a), `??` 는 `?` 두 개
base_type   ::= path_type
              | fn_type
              | tuple_type
              | array_type
              | any_type
              | self_type
              | qualified_type
              | macro_type
              | "none"                          // none 타입 (2.6)
              | "!"                             // never (4.3.7)
              | "(" type_expr ")"               // 그룹
self_type   ::= "Self" ("::" identifier ("::" identifier)* type_args?)?   // Self, Self::Item
qualified_type ::= "<" type_expr "as" trait_ref ">" "::" identifier       // 2.13.4
macro_type  ::= identifier "!" ( "(" token* ")" | "[" token* "]" | "{" token* "}" )   // 6.8.12
any_type    ::= "any" ( trait_ref | "(" trait_ref ("+" trait_ref)* ")" )?
trait_ref   ::= identifier ("::" identifier)* trait_args?
trait_args  ::= "<" trait_arg ("," trait_arg)* ","? ">"
trait_arg   ::= type_expr | identifier "=" type_expr | const_arg
path_type   ::= identifier ("::" identifier)* type_args?
type_args   ::= "<" trait_arg ("," trait_arg)* ","? ">"
const_arg   ::= literal                         // 정수, 문자, true / false
              | "-" literal
              | "{" expr "}"                    // 2.11.4
              | string_lit                      // 예: MLIR op 이름
fn_type     ::= "unsafe"? ("extern" string_lit)? "fn" "( (type_expr ("," type_expr)*)? ")" ("throws" ("(" type_expr ")")?)? ("->" unary_type)? fn_bound*
fn_bound    ::= "+" ("Send" | "Sync")
tuple_type  ::= "(" ")"
              | "(" type_expr "," ")"
              | "(" type_expr ("," type_expr)+ ","? ")"
array_type  ::= "[" type_expr ";" const_expr "]"
```

`extern "C" fn(A, B) -> R`은 C 함수 포인터 타입입니다([5.9.4](../005-modules/509-extern-block.md#c-함수-포인터-타입)). ABI 문자열에는 `"C"`만 쓸 수 있고, 다른 값은 에러입니다.

형태별로 정리하면 다음과 같습니다.

| 형태          | 예                                          |
|---------------|---------------------------------------------|
| 단순 이름     | `i32`, `bool`, `Point`                      |
| 경로          | `rok::std::time::Duration`                  |
| 제네릭 적용   | `List<i32>`, `Dict<str, f64>`, `Pair<A, B>` |
| 합 타입 (`\|`) | `i32 \| f32`, `T \| none`                     |
| optional (`?`)  | `i32?`, `(i32 \| str)?`, `List<T>?`           |
| 함수 타입     | `fn(i32, i32) -> i32`, `fn() -> T + Send`   |
| 튜플 타입     | `(i32, str)`, `(i32,)`                      |
| 배열 타입     | `[T; N]` (고정 크기)                        |
| 동적 슬롯     | `any`, `any Error`, `any (Display + Hash)`  |
| `none` / never | `none`, `!`                                |
| `Self` 와 그 항목 | `Self`, `Self::Item`                     |
| 한정 경로     | `<T as Iterator>::Item`                     |
| 매크로 호출   | `name!(..)`, 타입으로 펼쳐짐 ([6.8.12](../006-meta-system/608-meta-blocks.md#6812-macro-형태)) |
| const 인자    | `Buf<16>`, `Buf<-1>`, `Buf<{ N + 1 }>`      |

각 형태의 정확한 의미는 아래에서 차례로 정의합니다.

`T::Assoc` 형태의 associated type projection(`I::Item`, `F::Output`)도 타입입니다. 경로 형태(`path_type`)의 하나이며, `T`가 그 associated type을 가진 트레잇으로 제약된 타입 매개변수일 때 유효합니다([타입 경로](213-type-paths.md)).

```rok
fn zip<I: Iterator, J: Iterator>(a: I, b: J) -> List<(I::Item, J::Item)> { ... }
```

## 2.1.2. 단순 이름

단순 이름은 식별자 하나로 이루어지며 [식별자](../001-lexical/104-identifiers.md) 형식을 따릅니다.

```rok
i32
bool
Point
변수타입
```

이 이름이 어떤 항목을 가리키는지는 이름 해소(name resolution) 단계에서 정해집니다. 현재 모듈에 정의된 항목, `use`로 가져온 항목 등이 후보입니다.

## 2.1.3. 경로

식별자 여럿을 `::`로 이은 형태도 타입 표현식으로 쓸 수 있습니다.

```rok
rok::std::time::Duration
rok::core::collections::List
pkg::geom::Point
```

여기서 `::` 토큰은 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)에 정의된 경로 구분자입니다. 경로의 마지막 토큰이 타입 이름이고, 앞부분은 그 타입을 어디서 찾을지 알려 주는 모듈 경로 또는 항목 경로입니다.

경로의 의미와 해석 규칙은 [타입 경로](213-type-paths.md)에서 정의합니다. 이 문서는 경로가 타입 표현식의 한 형태로 나올 수 있다는 사실만 밝힙니다.

## 2.1.4. 제네릭 적용

타입 이름 뒤에 `<...>`를 붙여 타입 인자를 적을 수 있습니다.

```rok
List<i32>
Dict<str, f64>
Pair<A, B>
Result<T, E>
Array<f64, 3>          // const 제네릭 (값 인자)
```

규칙은 다음과 같습니다. 인자는 콤마로 구분합니다. 인자로는 다른 타입 표현식이 올 수 있고, `const` 제네릭에는 `const` 표현식도 올 수 있습니다. 빈 `<>`는 허용되지 않습니다. 인자가 없으면 `<>`를 아예 적지 않습니다. 후행 콤마(`<A, B,>`)는 허용됩니다.

값 인자(예: `Array<f64, 3>`)의 정확한 의미는 [`const` 제네릭](211-const-generics.md)에서 정의합니다.

### 표현식 위치에서의 turbofish

표현식 안에서 제네릭 인자를 적을 때는 `::<...>` 형태를 쓰며, 이 형태는 표현식 사양에서 정의합니다. 이 문서는 타입 표현식 위치의 `<...>`만 다룹니다.

## 2.1.5. 합 타입 (`|`)

여러 타입 중 하나의 값을 갖는 합 타입은 `|`로 적습니다.

```rok
i32 | f32
T | none
A | B | C | D
```

`|`로 구분한 각 항은 서로 다른 타입 표현식이고, 그 결과인 합 타입은 이 항 중 하나의 값을 갖습니다. 의미와 메모리 표현은 [합 타입](206-sum-types.md)에서 정의합니다.

`|` 토큰은 다른 타입 결합자보다 약하게 묶입니다. 제네릭 적용이 합 타입보다 강하게 묶이므로 `i32 | List<f32>`는 `i32 | (List<f32>)`로 묶입니다.

## 2.1.5a. optional 타입 (`T?`)

후위 `?`는 optional 타입을 만듭니다. `T?`는 `Maybe<T>`(`type Maybe<T> = T | none;`)와 똑같이 동작합니다. `?`의 피연산자는 제네릭 항이며, [합 타입의 층 규칙](206-sum-types.md#264-집합-의미론)이 적용됩니다.

```rok
i32?                // i32 | none
(i32 | str)?        // i32 | str | none (겹침 없음, 펼침)
(i32 | none)?       // 두 층: 바깥 none ≠ 안쪽 none
i32??               // 두 층: (i32 | none)? 과 같음
i32 | none          // 적은 합: 늘 평평한 i32 | none
```

- 후위 `?`는 `|`보다, 그리고 함수 타입의 `->`보다 강하게 묶입니다(2.1.10). `i32 | str?`는 `i32 | (str?)`입니다.
- `fn() -> i32?`는 optional을 돌려주는 함수이고, `(fn() -> i32)?`는 optional 함수입니다.
- 타입 자리에서 `??` 토큰은 `?` 두 개입니다(`i32??`).
- 식 `x as T?` / `x is T?`에서 끝의 `?`는 타입에 속합니다. 변환 결과에 `?` 연산자([3.14](../003-expressions/314-error-control-flow.md))를 쓰려면 `(x as T)?`처럼 괄호로 감쌉니다. `x as T?`의 변환이 실패하면 진단이 "전파하려면 `(x as T)?`로 쓰라"는 힌트를 줍니다.
- 단, `as` / `is`의 대상 뒤에 오는 `??`는 null 합체 연산자입니다(`x as i32 ?? 0`은 `(x as i32) ?? 0`). 두 층 optional로 변환하려면 `x as (i32??)`로 씁니다.

## 2.1.6. 함수 타입

함수 자체를 가리키는 타입은 다음과 같이 적습니다.

```rok
fn(i32, i32) -> i32
fn() -> str
fn(T) -> T
fn(i32)                  // 반환 타입 생략 가능 (단위 타입)
```

함수 타입은 `fn` 토큰으로 시작합니다. 괄호 안에는 매개변수 타입만 적고 이름은 적지 않습니다. 반환 타입은 `->` 뒤에 적으며, 생략하면 반환값이 없다는 뜻입니다(정확한 의미는 함수 사양에서 정의합니다).

## 2.1.7. 튜플 타입

여러 타입을 묶은 튜플 타입은 다음과 같이 적습니다.

```rok
(i32, str)
(f64, f64, f64)
(i32,)                  // 한 항짜리 튜플은 후행 콤마 필수
()                      // 빈 튜플 (단위 타입)
```

규칙은 다음과 같습니다. 항이 둘 이상이면 콤마로 구분합니다. 항이 하나인 튜플에는 반드시 후행 콤마(`(T,)`)를 붙여야 합니다. 콤마가 없으면 `T`를 괄호로 묶은 것으로 해석합니다. 항이 없는 튜플(`()`)은 단위 타입을 가리킵니다.

## 2.1.8. 배열 타입

고정 크기 배열 타입은 대괄호로 적습니다.

```rok
[T; N]
[i32; 16]
[f64; 1024]
[u8; SIZE]
```

배열 타입은 대괄호 안에 원소 타입, 세미콜론, 길이 순으로 적습니다. 길이는 `const` 표현식이며 컴파일 타임에 정해져야 합니다. 길이 0(`[T; 0]`, 빈 배열 타입)도 허용됩니다.

## 2.1.9. 동적 슬롯

동적 타입 슬롯은 `any` 키워드로 적습니다.

```rok
any
any Error
any (Display + Hash)
any Iterator<Item = i32>
any Future<Output = User>
```

`any`는 [키워드](../001-lexical/105-keywords.md)에 예약된 타입 이름 슬롯 키워드이며, 의미는 [`any` 슬롯](202-any-slot.md)에서 정의합니다. `any` 뒤에 트레잇 경로 하나, 또는 `+`로 이어 괄호로 묶은 트레잇 경로 여러 개를 붙이면, 그 트레잇을 구현하는 값만 담는 제약 붙은 동적 슬롯이 됩니다([2.2.10](202-any-slot.md#2210-제약-붙은-any-any-trait)). 트레잇 경로의 `<...>` 안에는 타입 인자 말고도 `Item = T`처럼 associated type을 고정하는 바인딩이 올 수 있습니다(`trait_arg`). 트레잇 경로가 둘 이상이면 괄호가 필요합니다. `any`는 바로 뒤의 트레잇 하나만 묶으므로 `any Error | none`은 `(any Error) | none`입니다. 트레잇 경로는 `any` 뒤에서만 타입 자리에 올 수 있습니다([트레잇-as-타입 금지](212-trait-as-type-banned.md)).

## 2.1.10. 우선순위 / 결합

타입 표현식 안에서 결합자가 섞일 때의 우선순위(강한 결합 ↔ 약한 결합)는 다음과 같습니다.

```
strong  ─→  weak

[T; N]   (...)   <...>   ::   .  →  T?  →  fn(...) -> ...   →   |
```

경로(`::`), 제네릭 적용(`<...>`), 배열(`[...]`), 그룹화(`(...)`)가 가장 강하게 묶입니다. 그다음이 후위 optional(`?`), 그다음이 함수 타입(`fn(...) -> ...`)이고, 합 타입(`|`)이 가장 약하게 묶입니다.

따라서 `i32 | List<f32>`는 `i32 | (List<f32>)`로, `fn(A | B) -> C`는 `fn((A|B)) -> C`로 묶입니다.

그룹을 명시해야 하면 괄호로 감쌉니다.

```rok
fn(i32) -> (i32 | str)         // 반환 타입이 합 타입
(fn(i32) -> i32) | (fn(f64) -> f64)   // 합 타입이 두 함수 타입을 항으로
```
