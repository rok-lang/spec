# 5.2. `const`

`const` 키워드 ([키워드](../001-lexical/105-keywords.md)) 는 컴파일 타임에 결정되는 값을 선언할 때 사용합니다. 문법은 다음과 같이 정의됩니다.

```
const_decl      ::= "const" identifier ":" type_expr "=" const_expr ";"
const_fn        ::= "const" fn_decl
```

## 5.2.1. 기본 형태

가장 단순한 형태부터 살펴보겠습니다.

```rok
const PI: f64 = 3.14159f64;
const MAX_RETRY: i32 = 3i32;
const NAME: str = "rok";
```

선언은 `const` 키워드로 시작하여 이름, `:`, 타입, `=`, 그리고 `const` 표현식 순서로 이어집니다. 타입은 반드시 명시해야 하며, `let` 과 달리 추론에 의존하지 않습니다. 값 자리에는 `const` 표현식, 즉 컴파일 타임에 평가할 수 있는 식만 올 수 있습니다.

## 5.2.2. 가능한 위치

`const` 는 아래 모든 위치에서 자유롭게 선언할 수 있습니다.

```rok
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

`const` 는 본질적으로 값이지 주소가 아닙니다. 따라서 사용되는 자리마다 그 값이 인라인되어 들어갑니다.

```rok
const X: i32 = 42i32;

fn f() {
    let a = X;                    // a = 42i32 (X 가 인라인)
    let b = X + 1i32;              // b = 43i32 (X 가 인라인 후 평가)
}
```

`const` 의 결과는 어떤 메모리 자리에도 머무르지 않으므로 주소를 가져올 수 없습니다.

```rok
let p = &X;                       // ERROR — const 는 주소 없음 (개념적으로)
```

실제 rok 에서 `&` 같은 직접 주소 연산자가 존재하는지에 대해서는 [타입 표현식](../002-types/201-type-expressions.md) 의 포인터 타입과 [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 사양에서 정의하고 있습니다.

## 5.2.4. `const` 표현식의 허용 범위

`const` 자리에 올 수 있는 식은 다음과 같습니다.

- 리터럴: 모든 [숫자 리터럴](../001-lexical/106-numeric-literals.md)부터 [진리값과 센티넬 리터럴](../001-lexical/109-other-literals.md) 까지의 리터럴이 해당합니다.
- 다른 `const` 의 참조: 예를 들어 `const TWICE_PI: f64 = PI * 2.0f64;` 와 같은 형태입니다.
- `const` 함수 호출: 아래에서 자세히 정의합니다.
- `const` 표현식의 산술 / 비교 / 논리 연산: `+`, `-`, `*`, `<`, `&&` 등이 가능합니다.
- `const` [generic](../000-meta/007-glossary.md#generic--제네릭) 매개변수: 자세한 내용은 [const 제네릭](../002-types/211-const-generics.md) 을 참고해 주세요.
- 튜플 / `struct` / `enum` 인스턴스: 모든 필드가 `const` 인 경우에 한해 허용됩니다.

반대로 다음과 같은 것들은 `const` 자리에 올 수 없습니다.

- 런타임 함수 호출 (현재 시각, 난수, I/O 등).
- 전역 mutable 상태 ([`static`](503-static.md)) 의 읽기.
- `let` 변수의 참조.

## 5.2.5. `const fn`: 컴파일 타임 평가 가능한 함수

`const fn` 은 컴파일 타임에 평가할 수 있는 함수를 선언하는 방법입니다.

```rok
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

선언할 때는 `fn` 앞에 `const` 키워드를 붙이기만 하면 됩니다. 본체에서는 제어 흐름을 자유롭게 사용할 수 있어서 `if`, `while`, `for`, `match` 가 모두 가능합니다. 다만 함수는 순수 (pure) 해야 하며, I/O 나 전역 mutable 접근이 들어가서는 안 됩니다. CTE 결과는 런타임 결과와 비트 단위로 동일 (bit-identical) 함을 보장합니다. `const fn` 안에서는 다른 `const fn` 만 호출할 수 있고 런타임 함수는 호출할 수 없으며, 동시에 일반 함수처럼 런타임에서도 자유롭게 호출할 수 있습니다.

### `const fn` 의 제약

몇 가지 제약을 함께 알아두면 좋습니다. 우선 type 인자는 금지됩니다. 즉 `const fn sizeof(T: type) { ... }` 같은 형태는 ERROR 이며, 타입 introspection 은 메타 블록 영역에 속합니다. 반환할 owned 값에 대한 allocation 은 허용되므로 `String`, `List<T>` 같은 값을 반환하는 것도 가능합니다. 호출 측면에서는 다른 `const fn` 만 호출할 수 있고, `const` 컨텍스트에서 일반 함수를 호출하면 ERROR 가 발생합니다.

### 제네릭 `const fn`

`const fn` 은 제네릭과도 자연스럽게 결합됩니다.

```rok
const fn id<T>(x: T) -> T { x }

const A: i32 = id(42i32);                        // T=i32 단형화 + CTE
```

[제네릭](../002-types/210-generics.md) 의 일반 제네릭 함수에 `const` 를 붙여 사용할 수 있으며, 단형화된 후 본체가 `const` 평가가 가능하면 `const` 호출도 정상적으로 이루어집니다.

## 5.2.6. `const` 와 다른 메커니즘의 직교성

```
generic <T>     = 타입 파라미터화 (call-site specialize)
meta block      = 코드 생성 (impl, struct 등 AST 레벨)
const fn        = 값 계산 (compile-time pure 함수)
```

세 가지 기계 장치는 서로 직교합니다. 영역이 겹치지 않으므로 자유롭게 결합하여 사용할 수 있습니다.

## 5.2.7. `const` 의 평가 시점

`const` 의 평가 시점은 선언 위치와 의존 관계에 따라 달라집니다. 모듈 수준이나 `impl` 수준의 `const` 는 패키지 빌드 시에 한 번 평가됩니다. 함수 본체 안의 `const` 는 함수가 단형화되는 시점에 평가됩니다. 제네릭 매개변수에 의존하는 `const` 는 단형화될 때마다 평가되며, 그 결과는 캐시됩니다. 어떤 경우든 평가 결과는 바이너리에 박혀 들어가므로 런타임 비용이 발생하지 않습니다.

## 5.2.8. `struct` 필드 기본값과의 관계

[`struct`](../002-types/203-struct.md) 의 필드 기본값은 `const` 표현식이 들어가는 자리이므로 이 문서에서 설명한 `const` 규칙을 그대로 따릅니다. 따라서 `const fn` 호출은 가능하지만, 런타임 호출은 허용되지 않습니다.

## 5.2.9. `const` generic 과의 관계

[const 제네릭](../002-types/211-const-generics.md) 의 `const` 매개변수는 타입의 일부가 되는 `const` 값을 의미합니다. 이 문서에서 다루는 일반 `const` 와는 직교하는 개념이지만, 같은 컴파일 타임 평가 가능 영역에 함께 속합니다.
