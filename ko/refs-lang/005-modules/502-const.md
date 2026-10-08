# 5.2. `const`

`const` 키워드 ([키워드](../001-lexical/105-keywords.md)) 는 컴파일 타임에 결정되는 값을 선언한다.

```
const_decl      ::= "const" identifier ":" type_expr "=" const_expr ";"
const_fn        ::= "const" fn_decl
```

## 5.2.1. 기본 형태

```nok
const PI: f64 = 3.14159f64;
const MAX_RETRY: i32 = 3i32;
const NAME: str = "nok";
```

- `const` + 이름 + `:` + 타입 + `=` + `const` 표현식.
- 타입은 필수다. `let` 과 달리 추론하지 않는다.
- 값은 `const` 표현식만 허용한다. 컴파일 타임에 평가할 수 있는 식이다.

## 5.2.2. 가능한 위치

`const` 는 다음 위치 모두에서 선언할 수 있다.

```nok
// 모듈 수준
const PI: f64 = 3.14159f64;

// 함수 본체 안
fn area(r: f64) -> f64 {
    const FACTOR: f64 = 3.14159f64;
    FACTOR * r * r
}

// impl 블록 안
impl Circle {
    const UNIT_AREA: f64 = 3.14159f64;
    fn unit() -> Circle { Circle { r: 1.0f64 } }
}

// 트레잇 안 — 연관 const
trait Numbers {
    const ZERO: Self;
    const ONE: Self;
}
```

## 5.2.3. `const` 의 의미: 사용처마다 inline

`const` 는 값이지 주소가 아니다. 사용처마다 값이 인라인된다.

```nok
const X: i32 = 42i32;

fn f() {
    let a = X;                    // a = 42i32 (X 가 인라인)
    let b = X + 1i32;              // b = 43i32 (X 가 인라인 후 평가)
}
```

`const` 의 결과는 어떤 메모리 자리에도 살지 않는다. 주소를 잡을 수 없다.

```nok
let p = &X;                       // ERROR — const 는 주소 없음 (개념적으로)
```

(실제 nok 에 `&` 같은 직접 주소 연산자가 있는지는 [타입 표현식](../002-types/201-type-expressions.md) 의 포인터 타입과 [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 사양에서 정의한다.)

## 5.2.4. `const` 표현식의 허용 범위

`const` 자리에 올 수 있는 식.

- 리터럴: 모든 [숫자 리터럴](../001-lexical/106-numeric-literals.md)부터 [진리값과 센티넬 리터럴](../001-lexical/109-other-literals.md) 까지의 리터럴.
- 다른 `const` 의 참조: `const TWICE_PI: f64 = PI * 2.0f64;`.
- `const` 함수 호출: 아래에서 정의한다.
- `const` 표현식의 산술 / 비교 / 논리: `+`, `-`, `*`, `<`, `&&` 등.
- `const` [generic](../000-meta/007-glossary.md#generic-/-제네릭) 매개변수: [const 제네릭](../002-types/211-const-generics.md).
- 튜플 / `struct` / `enum` 인스턴스: 모든 필드가 `const` 인 경우.

다음은 `const` 자리에 올 수 없다.

- 런타임 함수 호출 (현재 시각, 난수, I/O).
- 전역 mutable 상태 ([`static`](503-static.md)) 의 읽기.
- `let` 변수의 참조.

## 5.2.5. `const fn`: 컴파일 타임 평가 가능한 함수

```nok
const fn double(x: i32) -> i32 {
    x * 2i32
}

const fn gcd(a: i64, b: i64) -> i64 {
    if b == 0i64 { a } else { gcd(b, a.mod(b)) }
}

const TWO: i32 = double(1i32);                  // CTE
const TWELVE: i64 = gcd(24i64, 36i64);          // CTE

let y = double(user_input);                      // 런타임 호출도 OK
```

- `fn` 앞에 `const` 키워드를 붙인다.
- 본체는 제어 흐름이 자유롭다. `if`, `while`, `for`, `match` 모두 사용 가능.
- 순수 (pure) 해야 한다. I/O 와 전역 mutable 이 없다.
- CTE 결과는 런타임 결과와 bit-identical 이다.
- 다른 `const fn` 만 호출할 수 있다 (런타임 함수는 호출할 수 없다).
- 일반 함수처럼 런타임에서도 호출할 수 있다.

### `const fn` 의 제약

- type 인자 금지: `const fn sizeof(T: type) { ... }` 는 ERROR. 타입 introspection 은 메타 블록 영역이다.
- 반환할 owned 값의 allocation 은 OK: `String`, `List<T>` 같은 반환도 가능하다.
- 다른 `const fn` 만 호출 가능: `const` 컨텍스트에서의 일반 함수 호출은 ERROR.

### 제네릭 `const fn`

```nok
const fn id<T>(x: T) -> T { x }

const A: i32 = id(42i32);                        // T=i32 단형화 + CTE
```

[제네릭](../002-types/210-generics.md) 의 일반 제네릭 함수에 `const` 를 붙일 수 있다. 단형화 후 본체가 `const` 평가가 가능하면 `const` 호출도 가능하다.

## 5.2.6. `const` 와 다른 메커니즘의 직교성

```
generic <T>     = 타입 파라미터화 (call-site specialize)
meta block      = 코드 생성 (impl, struct 등 AST 레벨)
const fn        = 값 계산 (compile-time pure 함수)
```

세 기계는 직교한다. 서로 겹치지 않으며 자유롭게 결합할 수 있다.

## 5.2.7. `const` 의 평가 시점

- 모듈 / `impl` 수준 `const`: 패키지 빌드 시 한 번 평가한다.
- 함수 본체 안 `const`: 함수가 단형화되는 시점에 평가한다.
- 제네릭 매개변수에 의존하는 `const`: 단형화마다 평가한다 (결과는 캐시).

평가 결과는 바이너리에 박힌다. 런타임 비용이 없다.

## 5.2.8. `struct` 필드 기본값과의 관계

[`struct`](../002-types/203-struct.md) 의 필드 기본값은 `const` 표현식 자리다. 이 문서의 `const` 규칙을 그대로 따른다. `const fn` 호출은 되지만, 런타임 호출은 안 된다.

## 5.2.9. `const` generic 과의 관계

[const 제네릭](../002-types/211-const-generics.md) 의 `const` 매개변수는 타입의 일부가 되는 `const` 값이다. 이 문서의 일반 `const` 와는 직교하지만, 같은 컴파일 타임 평가 가능 영역에 속한다.
