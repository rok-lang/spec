# 6.5. `#[literal(...)]` 매직

`#[literal("...")]` 속성은 사용자가 직접 적은 문법 패턴을 라이브러리 핸들러로 디스패치되는 리터럴로 등록한다.

## 6.5.1. 동기

[숫자 리터럴](../001-lexical/106-numeric-literals.md) 부터 [진리값과 센티넬 리터럴](../001-lexical/109-other-literals.md) 까지의 어휘 리터럴 (숫자, 문자열, 문자, 진리값) 외에, nok 은 컬렉션, 범위, 슬라이스 같은 복합 리터럴을 표준 라이브러리에서 등록한다.

```
[1, 2, 3]                 → List<T> 인스턴스
{"a": 1, "b": 2}          → Dict<K, V>
{1, 2, 3}                 → Set<T>
(1, "hi", true)            → Tuple
0..5                       → Range<T>
0..=5                      → RangeInclusive<T>
1:3                        → Slice<T>
2i                         → Complex<T>
```

이 리터럴들의 문법 자체는 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md) 의 토큰과 [타입 표현식](../002-types/201-type-expressions.md) 등의
형태이지만, 어떤 라이브러리 타입이 만들어지는지는 라이브러리가 결정한다.

`#[literal]` 매직이 그 결정을 트레잇이나 함수에 부착한다.

## 6.5.2. 형태

```
literal_attr    ::= "#" "[" "literal" "(" pattern_string ")" "]"
pattern_string  ::= string_lit                  // 패턴 미니 언어 — 본문 참조
```

```nok
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

- `#[literal("문법 패턴")]` 형태이다. 문법 패턴 (query) 을 인자로 받는다.
- 함수나 메서드에 부착하면 그 함수가 패턴에 매칭되는 리터럴의 핸들러가 된다.

## 6.5.3. 문법 패턴의 형식

문법 패턴은 작은 query 언어로 구성된다.

| 토큰       | 의미                                          |
| ---------- | --------------------------------------------- |
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

예시 분해는 다음과 같다.

```
"[" {expr} % "," * "]"
 │   │      │       │
 │   │      │       └── 닫기 토큰
 │   │      └── 0 회 이상 반복
 │   ├── 콤마로 구분된
 │   └── expr 그룹
 └── 여는 토큰
```

이 패턴은 `[]`, `[1]`, `[1, 2]`, `[1, 2, 3]` 등 모든 콤마 구분 리스트에 매칭된다.

## 6.5.4. 매칭과 디스패치

소스에 `[1, 2, 3]` 이 등장하면 다음 단계로 처리된다.

1. 파서가 등록된 `#[literal]` 패턴 중 매칭하는 것을 찾는다.
2. 매칭하면 그 핸들러 함수에 매칭된 부분 표현식들이 인자로 전달된다.
3. 핸들러 함수가 값을 만들어 반환한다.

```nok
[1, 2, 3]
   ↓
list_literal(1, 2, 3)
   ↓
List<i32> 인스턴스
```

## 6.5.5. 다중 핸들러 (모호성)

같은 입력에 여러 핸들러가 매칭하면 모호하다. 컴파일러는 다음 규칙으로 해소한다.

1. 문맥의 타입 정보. 받는 슬롯이 `Set<T>` 면 set 핸들러를 우선한다.
2. 우선순위 명시. `#[literal(..., priority = N)]` 같은 추가 인자를 사용한다 (정확한 형태는 표준 라이브러리 사양에 따른다).
3. 모호함이 남으면 컴파일 에러를 낸다.

## 6.5.6. Dict 와 Set 의 모호성

```
{ "a": 1 }              // Dict (`:` 패턴 매칭)
{ 1, 2 }                 // Set (값만)
{ 1 }                    // Set (단일 항)? 또는 block?
{ stmt; stmt; expr }    // block ([블록 표현식](../003-expressions/309-block-expressions.md))
```

`{ ... }` 는 [블록 표현식](../003-expressions/309-block-expressions.md) 으로도 쓰인다. 다음 규칙으로 구분한다.

- `{ expr ; ... }`: [block](../000-meta/007-glossary.md#block-블록) (statement 가 있으면).
- `{ expr : expr , ... }`: Dict (`:` 가 있으면).
- `{ expr , expr , ... }`: Set (콤마만 있고, 단일 항이 아닌 경우).
- `{ expr }`: 문맥에 따라. 받는 슬롯이 Set 이면 Set, 아니면 block.

이 규칙은 파서가 내용을 보고 결정하는 형태다. 명시적 접두 (set 표시용 `s{1, 2}` 같은 것) 는
가독성을 위해 도입하지 않는다.

## 6.5.7. 사용자 정의 리터럴

사용자도 자신의 리터럴을 등록할 수 있다 (드물게).

```nok
struct Coord { x: f64; y: f64; }

#[literal( "(" expr "," expr ")" "@" )]
fn coord_literal(x: f64, y: f64) -> Coord { Coord { x, y } }

let p = (3.14, 2.71)@;        // → coord_literal(3.14, 2.71) → Coord
```

위는 가상의 예시다. `()@` 같은 임의 패턴을 등록할 수 있다. 다만 기본 토큰과 충돌하면 컴파일 에러다.

## 6.5.8. 리터럴 핸들러의 `const` 호환

리터럴 핸들러는 `const` `fn` 이거나 컴파일 타임에 평가 가능해야 한다 (가능한 경우).

```nok
#[literal("[" {expr} % "," * "]")]
const fn list_literal<T>(items: ...T) -> List<T> { ... }
```

`const` `fn` 이면 리터럴이 컴파일 타임에 평가되어 결과가 바이너리에 박힌다.

이 최적화는 사용자에게 친화적이다. 사용자가 `[1, 2, 3]` 을 자주 쓰는데 매번 런타임 비용이 들면 곤란하기 때문이다.

## 6.5.9. 슬라이스 패턴 (특수 형태)

슬라이스 (`s[1:3]`) 는 기존 식 안에 등장하므로 일반 `[리터럴]` 과는 다른 형태다.

```nok
#[literal( expr? ":" expr? (":" expr?)? )]
fn slice_literal(start: any | none, end: any | none, step: any | none) -> Slice { ... }
```

이는 대괄호 안의 패턴이며, 인덱싱 (`arr[i]`) 의 자리에 등장한다.

자세한 슬라이스와 인덱싱 디스패치는 [인덱싱과 슬라이싱](../003-expressions/306-indexing-and-slicing.md) 의 트레잇 매핑과 결합된다.

## 6.5.10. 평가 시점

`#[literal]` 매핑은 컴파일 타임에 일어난다.

1. 파서가 입력을 토큰화한다.
2. 입력의 식 자리에서 등록된 패턴들을 시도한다.
3. 매칭하면 핸들러 호출이 식 자리에 들어간다.
4. 후속 패스 (이름 해소, 타입 검사) 가 그 결과를 처리한다.

따라서 `[1, 2, 3]` 이 어떤 함수 호출인지 컴파일 타임에 명확해진다.

## 6.5.11. 매직과 표준 라이브러리

`#[literal]` 은 거의 표준 라이브러리만 사용한다. core 의 컬렉션 리터럴이 등록 위치다.

사용자가 자기 라이브러리에 등록할 수도 있지만, 기본 토큰과 충돌하지 않는 패턴을 신중히 골라야 한다.
