# 5.2. `const`

`const` 키워드 (항목 시작에서만 키워드인 문맥 키워드, [1.5.4](../001-lexical/105-keywords.md#154-문맥-키워드)) 는 컴파일 타임에 결정되는 값을 선언할 때 사용합니다. 문법은 다음과 같이 정의됩니다.

```
const_decl      ::= "const" identifier ":" type_expr "=" const_expr ";"
const_fn        ::= "const" fn_decl
```

## 5.2.1. 기본 형태

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

// 트레잇 안: 연관 const
trait Numbers {
    const ZERO: Self;
    const ONE: Self;
}
```

함수 본체의 `const` 는 그 블록의 항목입니다 ([3.9.1](../003-expressions/309-block-expressions.md#391-형태)): 블록 전체와 그 안에 선언한 항목에서 보이며, 둘러싼 코드의 지역 변수, 매개변수, 제네릭 매개변수를 읽지 않습니다.

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
let p = &X;                       // ERROR: const 는 주소 없음 (개념적으로)
```

실제 Rok에서 `&` 같은 직접 주소 연산자가 존재하는지에 대해서는 [타입 표현식](../002-types/201-type-expressions.md) 의 포인터 타입과 [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 사양에서 정의합니다.

## 5.2.4. 상수 식과 평가기

`const` 항목, 배열 길이, const 제네릭 인자, `struct` 필드 기본값, `static` 초기값, 상수 인자를 받은 `const fn` 리터럴 핸들러 ([6.5.8](../006-meta-system/605-literal-magic.md#658-리터럴-핸들러의-const-호환)) 는 같은 컴파일 타임 평가기를 씁니다.

평가할 수 있는 것:

- 값: 모든 폭의 정수, 부동소수, `bool`, `char`, `str`, 튜플, `struct`, `enum`, 합 타입, 고정 크기 배열.
- 식: 리터럴, 다른 `const` 의 참조, const generic 매개변수, 연산자, `const fn` 호출.
- 제어 흐름: `if`, `match`, `loop` / `while` / `for`, 라벨 `break`, `return`, `throw`. 재귀와 제네릭 `const fn` 도 됩니다.

평가할 수 없는 것: `const fn` 이 아닌 함수 호출 (I/O, 시각, 난수 등), `static` 읽기, 바깥 `let` 변수의 참조.

다음은 컴파일 에러입니다.

- 정수 overflow (리터럴과 상수만으로 된 식을 접은 결과 포함, `let m: u8 = 2 * 200`), 0 으로 나누기, `MIN / -1`, 비트 폭 이상의 shift, 범위 밖의 부동소수 → 정수 변환. 감싸는 산술은 `wrapping_*` 만입니다.
- 평가 중의 panic 과 잡히지 않은 `throw`.
- 평가 단계 수나 호출 깊이의 한도 초과. 한도는 구현이 정합니다 (rok: 4,000,000 단계, 깊이 128).
- `const` 사이의 순환 참조.

## 5.2.5. `const fn`: 컴파일 타임 평가 가능한 함수

```rok
const fn double(x: i32) -> i32 {
    x * 2i32
}

const fn gcd(a: i64, b: i64) -> i64 {
    if b == 0i64 { a } else { gcd(b, a % b) }
}

const TWO: i32 = double(1i32);                  // 컴파일 타임
const TWELVE: i64 = gcd(24i64, 36i64);          // 컴파일 타임

let y = double(user_input);                      // 런타임 호출
```

- `const fn` 은 `const fn` 만 부를 수 있습니다. `const fn` 안에서 `fn` 값이나 클로저를 통한 호출은 정적 검사가 따라갈 수 없으므로 평가될 때에만 검사합니다. 표준 라이브러리는 순수 함수를 `const fn` 으로 표시합니다.
- 본체는 순수해야 합니다. I/O 와 전역 mutable 접근은 에러입니다.
- 컴파일 타임 평가에 성공한 결과는 같은 인자의 런타임 결과와 같습니다. 런타임에 overflow 할 연산은 컴파일 타임에 에러입니다 (5.2.4).
- 타입을 인자로 받을 수 없습니다 (`const fn sizeof(T: type)` 는 에러). 타입 introspection 은 메타 블록의 영역입니다.
- `const fn` 은 런타임에도 일반 함수처럼 부를 수 있습니다.

### 제네릭 `const fn`

```rok
const fn id<T>(x: T) -> T { x }

const A: i32 = id(42i32);                        // T = i32 단형화 후 평가
```

### heap 을 쓰는 결과

결과가 heap 할당 (`List`, 버퍼, 클로저 등) 을 담으면 바이너리에 넣지 않습니다. 그런 `const` 는 컴파일 타임에 평가해 검사하고, 사용처에서 런타임에 만듭니다.

## 5.2.6. `const` 와 다른 메커니즘의 직교성

```
generic <T>     = 타입 파라미터화 (call-site specialize)
meta block      = 코드 생성 (impl, struct 등 AST 레벨)
const fn        = 값 계산 (compile-time pure 함수)
```

세 가지 기계 장치는 서로 직교합니다. 영역이 겹치지 않으므로 자유롭게 결합하여 사용할 수 있습니다.

## 5.2.7. `const` 의 평가 시점

모듈 수준, 함수 본체, `impl` 수준의 `const` 는 패키지 빌드 때 한 번 평가합니다. 제네릭 매개변수에 의존하는 `const` (제네릭 `impl` 의 연관 `const`) 는 단형화마다 평가합니다. heap 을 쓰지 않는 결과는 바이너리에 들어가며 런타임 비용이 없습니다.

## 5.2.8. `struct` 필드 기본값과의 관계

[`struct`](../002-types/203-struct.md) 의 필드 기본값은 상수 식입니다 (5.2.4).

## 5.2.9. `const` generic 과의 관계

[const 제네릭](../002-types/211-const-generics.md) 의 인자는 타입의 일부가 되는 상수이며, 같은 평가기로 평가합니다 (5.2.4).
