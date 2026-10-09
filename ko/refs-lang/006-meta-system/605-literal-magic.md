# 6.5. `#[literal(...)]` 매직

`#[literal("...")]` 속성은 사용자가 직접 적은 문법 패턴을 라이브러리 핸들러로 디스패치되는 리터럴로 등록해 주는 도구입니다.

## 6.5.1. 동기

[숫자 리터럴](../001-lexical/106-numeric-literals.md) 부터 [진리값과 센티넬 리터럴](../001-lexical/109-other-literals.md) 까지의 어휘 리터럴, 즉 숫자, 문자열, 문자, 진리값 외에도 rok 은 컬렉션, 범위, 슬라이스 같은 복합 리터럴을 표준 라이브러리에서 등록하고 있습니다.

```
[1, 2, 3]                 → List<T> 인스턴스 (기대 타입이 [T; N] 이면 고정 배열)
{"a": 1, "b": 2}          → Dict<K, V>
{1, 2, 3}                 → Set<T>
(1, "hi", true)            → Tuple
0..5                       → Range<T>
0..=5                      → RangeInclusive<T>
1:3                        → Slice<T>
2i                         → Complex<T>
```

이 리터럴들의 문법 자체는 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md) 의 토큰과 [타입 표현식](../002-types/201-type-expressions.md) 등의 형태이지만, 어떤 라이브러리 타입이 만들어지는지는 라이브러리가 결정하게 됩니다.

`#[literal]` 매직이 바로 그 결정을 트레잇이나 함수에 부착해 주는 역할을 맡습니다.

숫자 리터럴의 타입은 이 매직이 아니라 언어가 정합니다 ([1.6.2](../001-lexical/106-numeric-literals.md#162-타입-접미사), [1.6.3](../001-lexical/106-numeric-literals.md#163-접미사-없는-리터럴)). 숫자에 대해 `#[literal]` 이 다루는 것은 `2i` 처럼 언어가 정의하지 않은 접미사뿐입니다.

## 6.5.2. 형태

```
literal_attr    ::= "#" "[" "literal" "(" pattern_string ")" "]"
pattern_string  ::= string_lit                  // 패턴 미니 언어 — 본문 참조
```

```rok
#[literal("[" {expr} % "," * "]")]
fn list_literal<T>(items: ...T) -> List<T> { ... }

#[literal("{" {expr ":" expr} % "," * "}")]
fn dict_literal<K, V>(pairs: ...(K, V)) -> Dict<K, V> { ... }

#[literal("{" {expr} % "," + "}")]
fn set_literal<T>(items: ...T) -> Set<T> { ... }

#[literal("(" expr "," {expr} % "," * ")")]
fn tuple_literal(items: ...) -> Tuple<...> { ... }

#[literal( expr ".." expr )]
fn range_literal<T>(start: T, end: T) -> Range<T> { ... }

#[literal( expr "..=" expr )]
fn range_inclusive_literal<T>(start: T, end: T) -> RangeInclusive<T> { ... }

#[literal( number "i" )]
fn imaginary_literal(n: Float) -> Complex<f64> { ... }
```

기본 형태는 `#[literal("문법 패턴")]` 이며, 문법 패턴 (query) 을 인자로 받습니다. 함수나 메서드에 부착하면 그 함수가 패턴에 매칭되는 리터럴의 핸들러로 동작하게 됩니다.

## 6.5.3. 문법 패턴의 형식

문법 패턴은 작은 query 언어로 구성되어 있습니다.

| 토큰       | 의미                                          |
|------------|-----------------------------------------------|
| `"문자열"` | 리터럴 토큰 (구두점 또는 키워드). 그대로 매칭 |
| `expr`     | 임의의 표현식                                 |
| `number`   | 숫자 리터럴                                   |
| `string`   | 문자열 리터럴                                 |
| `ident`    | 식별자                                        |
| `{ ... }`  | 그룹                                          |
| `% ","`    | 구분자 (콤마 등)                              |
| `+`        | 1 회 이상 반복                                |
| `*`        | 0 회 이상 반복                                |
| `?`        | 선택적 (0 회 또는 1 회)                       |

예시를 분해해 보면 다음과 같이 읽을 수 있습니다.

```
"[" {expr} % "," * "]"
 │   │      │       │
 │   │      │       └── close token
 │   │      └── zero or more
 │   ├── separated by comma
 │   └── expr group
 └── open token
```

> open token 은 여는 토큰을, close token 은 닫기 토큰을, expr group 은 expr 그룹을, separated by comma 는 콤마로 구분된 것을, zero or more 는 0 회 이상 반복을 뜻합니다.

이 패턴은 `[]`, `[1]`, `[1, 2]`, `[1, 2, 3]` 등 모든 콤마 구분 리스트에 매칭됩니다.

## 6.5.4. 매칭과 디스패치

소스에 `[1, 2, 3]` 이 등장하면 다음 단계를 거쳐 처리됩니다.

1. 먼저 파서가 등록된 `#[literal]` 패턴 중 매칭되는 것을 찾습니다.
2. 매칭이 이루어지면 해당 핸들러 함수에 매칭된 부분 표현식들이 인자로 전달됩니다.
3. 마지막으로 핸들러 함수가 값을 만들어 반환하게 됩니다.

```rok
[1, 2, 3]
   ↓
list_literal(1, 2, 3)
   ↓
List<i32> 인스턴스
```

## 6.5.5. 다중 핸들러 (모호성)

같은 입력에 여러 핸들러가 매칭되면 모호한 상황이 발생합니다. 컴파일러는 다음 규칙을 차례로 적용해 이를 해소하게 됩니다.

1. 가장 먼저 문맥의 타입 정보를 활용합니다. 받는 슬롯이 `Set<T>` 라면 set 핸들러가 우선됩니다.
2. 그 다음으로 우선순위 명시를 따릅니다. `#[literal(..., priority = N)]` 같은 추가 인자를 사용할 수 있으며, 정확한 형태는 표준 라이브러리 사양에 따릅니다.
3. 그래도 모호함이 남는다면 컴파일 에러를 내게 됩니다.

## 6.5.6. Dict 와 Set 의 모호성

```
{ "a": 1 }              // Dict (`:` 패턴 매칭)
{ 1, 2 }                 // Set (값만)
{ 1, }                   // Set (단일 원소 — 후행 콤마)
{ 1 }                    // 항상 block
{ stmt; stmt; expr }    // block ([블록 표현식](../003-expressions/309-block-expressions.md))
```

`{ ... }` 는 [블록 표현식](../003-expressions/309-block-expressions.md) 으로도 쓰이기 때문에 다음 규칙으로 구분합니다.

- `{ expr ; ... }` 처럼 statement 가 있는 경우에는 [block](../000-meta/007-glossary.md#block-블록) 으로 봅니다.
- `{ expr : expr , ... }` 처럼 `:` 가 있으면 Dict 입니다.
- `{ expr , ... }` 처럼 콤마가 하나라도 있으면 Set 입니다. 원소가 하나인 Set 은 `{ expr , }` 로 씁니다.
- `{ expr }` 은 받는 슬롯의 타입과 관계없이 항상 block 입니다.

이 규칙은 파서가 내용을 보고 결정하는 형태입니다. set 표시용 `s{1, 2}` 같은 명시적 접두는 가독성을 위해 도입하지 않습니다.

## 6.5.7. 사용자 정의 리터럴

드물기는 하지만 사용자도 자신의 리터럴을 직접 등록할 수 있습니다.

```rok
struct Coord { x: f64; y: f64; }

#[literal( "(" expr "," expr ")" "@" )]
fn coord_literal(x: f64, y: f64) -> Coord { Coord { x, y } }

let p = (3.14, 2.71)@;        // → coord_literal(3.14, 2.71) → Coord
```

위는 가상의 예시이며, `()@` 같은 임의 패턴도 등록이 가능합니다. 다만 기본 토큰과 충돌하는 경우에는 컴파일 에러가 발생하게 됩니다.

## 6.5.8. 리터럴 핸들러의 `const` 호환

리터럴 핸들러는 가능한 경우 `const` `fn` 이거나 컴파일 타임에 평가 가능해야 합니다.

```rok
#[literal("[" {expr} % "," * "]")]
const fn list_literal<T>(items: ...T) -> List<T> { ... }
```

`const` `fn` 으로 정의되면 리터럴이 컴파일 타임에 평가되어 그 결과가 바이너리에 그대로 박히게 됩니다.

이 최적화는 사용자에게 매우 친화적입니다. 사용자가 `[1, 2, 3]` 같은 표현을 자주 사용하는데 매번 런타임 비용이 발생하면 곤란하기 때문입니다.

## 6.5.9. 슬라이스 패턴 (특수 형태)

슬라이스 (`s[1:3]`) 는 기존 식 안에 등장하기 때문에 일반 `[리터럴]` 과는 다른 형태를 띱니다.

```rok
#[literal( expr? ":" expr? (":" expr?)? )]
fn slice_literal(start: any | none, end: any | none, step: any | none) -> Slice { ... }
```

이는 대괄호 안의 패턴이며, 인덱싱 (`arr[i]`) 의 자리에 등장하게 됩니다.

자세한 슬라이스와 인덱싱 디스패치는 [인덱싱과 슬라이싱](../003-expressions/306-indexing-and-slicing.md) 의 트레잇 매핑과 결합되어 다루어집니다.

## 6.5.10. 평가 시점

`#[literal]` 매핑은 컴파일 타임에 일어나며, 다음 순서로 진행됩니다.

1. 먼저 파서가 입력을 토큰화합니다.
2. 이어서 입력의 식 자리에서 등록된 패턴들을 시도해 봅니다.
3. 매칭되는 패턴이 발견되면 핸들러 호출이 식 자리에 들어가게 됩니다.
4. 그 후 후속 패스인 이름 해소와 타입 검사가 그 결과를 처리합니다.

따라서 `[1, 2, 3]` 이 어떤 함수 호출인지 컴파일 타임에 명확히 결정됩니다.

## 6.5.11. 매직과 표준 라이브러리

`#[literal]` 은 거의 표준 라이브러리에서만 사용됩니다. rok 의 컬렉션 리터럴이 주된 등록 위치라고 할 수 있습니다.

사용자가 자기 라이브러리에 등록하는 것도 가능하지만, 기본 토큰과 충돌하지 않는 패턴을 신중히 골라야 합니다.
