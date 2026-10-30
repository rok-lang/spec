# 1.5. 키워드

키워드는 식별자처럼 보이지만 언어가 예약한 토큰입니다. 식별자 위치에 키워드가 나타나면 컴파일러는 식별자가 아니라 키워드 토큰을 만듭니다.

이 절은 언어가 직접 예약하는 키워드 목록만 정의합니다. 각 키워드의 정확한 의미는 사양의 뒷부분에서 다룹니다. 이 절은 형태와 분류만 밝힙니다.

## 1.5.1. 예약 키워드 목록

키워드는 두 부류로 나뉩니다. 원칙은 다음 한 줄입니다.

> [!IMPORTANT]
> 문장이나 식을 시작해 "식별자 + 뒤따르는 것"과 헷갈릴 수 있는 단어는 **예약 키워드**입니다. 나타나는 위치가 고정된 단어는 **문맥 키워드**(1.5.4)입니다.

예약 키워드는 어디에 나타나든 키워드로 토큰화하며 식별자로 쓸 수 없습니다. 예외는 1.5.5의 이름 자리뿐입니다.

```
let       fn        if        else      match     while     loop      for
return    break     continue  try       throw
async     await     yield     unsafe    using
struct    enum      trait     impl      pub       mod       use
as        is        ref
true      false     none      self      Self
```

> [!NOTE]
> `try?`와 `try!`는 별도의 키워드가 아닙니다. 키워드 `try`와 구두점 `?` / `!`로 이루어진 토큰 열입니다([구두점과 연산자 토큰](110-punctuation-and-operator-tokens.md#점--물음표)).

`self`(소문자)는 인스턴스 식별자, 곧 메서드의 수신자(receiver)를 가리킵니다. `Self`(대문자)는 현재 `impl`이 대상으로 하는 타입의 이름을 가리킵니다. `using`은 RAII 방식 자원 관리 블록을 여는 키워드입니다([`using` 표현식](../003-expressions/317-using-expression.md)).

## 1.5.2. 의미는 후속 문서에서

위 토큰의 의미는 여러 장에 나뉘어 있습니다. 변수, 가시성, 모듈은 5장에서 정의합니다. 타입과 컨테이너는 2장, 제어 흐름은 3장, 함수, 클로저, 코루틴은 4장, 메타와 매직은 6장에서 정의합니다.

1장은 어휘 단계에서 이 토큰을 식별자가 아니라 키워드로 분류한다는 규칙만 다룹니다.

## 1.5.3. 등록된 키워드(확장 메커니즘)

위 예약 목록 말고도 표준 라이브러리가 키워드를 더 등록하는 메커니즘이 있습니다. 이 메커니즘은 메타 시스템의 일부입니다. 등록 메커니즘과 정확한 처리 규칙은 [`#[keyword(...)]` 매직](../006-meta-system/606-keyword-magic.md)에서 정의합니다.

이 절은 다음만 약속합니다. 예약 키워드는 1.5.5의 이름 자리를 빼면 식별자가 아니라 키워드로 나뉩니다. 등록된 키워드는 그 등록이 들어 있는 컴파일 단위 안에서만 효력이 있습니다. 두 집합이 충돌하면 컴파일러는 그 등록을 거부합니다. 예약 키워드는 덮어쓸 수 없습니다.

## 1.5.4. 문맥 키워드

다음 단어는 정해진 위치에서만 키워드이고, 그 밖의 위치에서는 일반 식별자입니다.

| 단어                               | 키워드가 되는 위치                                                     | 정의 |
|------------------------------------|------------------------------------------------------------------------|------|
| `type`                             | 항목 시작에서 식별자가 오고 그 뒤에 `=` 또는 `<`가 올 때(타입 별칭)  | [2.6](../002-types/206-sum-types.md) |
| `any`                              | 타입 자리(`any`, `any Tr`)                                             | [2.2](../002-types/202-any-slot.md) |
| `in`                               | `for pat in`의 `in`                                                   | [3.13.3](../003-expressions/313-loop-expressions.md#3133-for-반복자-기반-반복) |
| `where`                            | 함수 시그니처, `impl` 헤더, `struct` / `enum` / `union` / 타입 별칭 헤더 뒤 | [2.10.4](../002-types/210-generics.md#2104-where-절) |
| `throws`, `rethrows`               | 함수 시그니처의 매개변수 `)` 뒤                                        | [4.14](../004-functions/414-throws.md) |
| `catch`, `finally`                 | 식이나 블록 뒤(후위 절)                                                | [3.14](../003-expressions/314-error-control-flow.md) |
| `const`, `static`, `extern`, `mut` | 항목 시작(`extern "C"`, `static mut` 포함)                             | [5.2](../005-modules/502-const.md), [5.3](../005-modules/503-static.md), [5.9](../005-modules/509-extern-block.md) |
| `union`                            | 항목 시작에서 바로 뒤에 식별자가 올 때                                 | [2.5](../002-types/205-union.md) |
| `pkg`, `sub`                       | `pub(` 바로 뒤                                                         | [5.6](../005-modules/506-visibility.md) |
| `pkg`, `sup`, `rok`                | 경로의 첫 토큰이고 바로 뒤에 `::`가 올 때                             | [5.7](../005-modules/507-paths.md) |

```rok
let type = 1;                       // 식별자
type Pair = (i32, i32);             // 타입 별칭 (항목 시작 + 식별자 + =)
let in = 3;  for x in xs { }        // 식별자 / for 의 in
fn catch(e: str) { }                // 식별자: 식 뒤가 아님
```

`any`라는 이름의 타입을 선언하는 것(`struct any { }`)은 컴파일 에러입니다. 타입 자리의 `any`와 구분할 수 없기 때문입니다.

### 가시성 한정자: `pkg` / `sub`

`pkg`와 `sub`는 `pub(pkg)`, `pub(sub)`의 괄호 안에서 한정자로 쓰입니다. 그 밖의 위치에서는 일반 식별자입니다.

```rok
pub(sub) fn helper() { ... }   // pub( ) 안에서: 가시성 한정자
trait Sub { fn sub(self, rhs: Self) -> Self; }  // 일반 위치에서: 식별자
```

### 경로 루트: `pkg` / `sup` / `rok`

세 단어가 경로의 첫 토큰이고 바로 뒤에 `::`가 오면 항상 경로 루트입니다([5.7 경로](../005-modules/507-paths.md)). 그 밖의 위치에서는 일반 식별자입니다.

- `pkg::xxx`: 현재 패키지 루트부터
- `sup::xxx`: 부모 모듈부터
- `rok::xxx`: 표준 라이브러리 가상 네임스페이스(`rok::core`, `rok::bare`, `rok::std`, [0.5 표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md))

`pkg::name`은 언제나 루트로 해석됩니다. 그래서 세 단어를 모듈이나 타입(`struct`, `enum`, `union`, `trait`, 타입 별칭, `mod`)의 이름으로 쓰면 컴파일 에러입니다. 변수, 함수, 필드, 메서드 이름으로는 쓸 수 있습니다.

```rok
let rok = 42;                   // 변수 이름 OK
fn sup() { }                    // 함수 이름 OK
self.rok                        // 필드 / 멤버 이름 OK
mod rok { ... }                 // ERROR: 모듈 이름으로 쓸 수 없음
struct pkg { }                  // ERROR: 타입 이름으로 쓸 수 없음
rok::core::iter::Iterator       // 경로 루트
```

## 1.5.5. 이름 자리의 키워드

다음 자리에서는 예약 키워드를 포함한 모든 키워드가 식별자입니다. 앞뒤 토큰만 봐도 이름이라는 것이 분명한 자리이기 때문입니다.

- `.` 바로 뒤(멤버 접근, 메서드 호출): `x.type`, `stream.match(p)`
- `struct` / `enum` 필드 이름 선언: `struct Token { type: Kind; }`
- `struct` 리터럴과 `struct` 패턴의 필드 이름: `Ev { match: 1 }`, `Ev { match: m } => ...`
- 이름 있는 인자의 이름: `f(type: 1)`. 이름 있는 인자는 매개변수를 가리키고, 매개변수 이름은 예약 키워드일 수 없습니다. 그래서 실제로 이 자리에는 문맥 키워드만 옵니다.

`self`는 예외여서 이 자리에서도 식별자가 될 수 없습니다. raw 식별자(Rust의 `r#fn`)는 여기서도 없습니다([1.4.4](104-identifiers.md#144-키워드와-충돌)).

```rok
struct Token { type: Kind; for: str; }   // OK: 필드 이름
let k = tok.type;                         // OK: . 뒤
make(type: Kind::Ident);                  // OK: 이름 있는 인자 (문맥 키워드)
struct S { self: i32; }                   // ERROR: self 는 예외
```
