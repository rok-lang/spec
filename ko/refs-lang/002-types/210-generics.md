# 2.10. 제네릭

제네릭은 함수, 타입, 트레잇, `impl` 블록에 타입 매개변수를 두는 메커니즘이다. 호출이나 인스턴스화 지점의 구체 타입 튜플마다 별개의 컴파일 결과를 생성한다 (단형화).

```
generics        ::= "<" generic_param ("," generic_param)* ","? ">"
generic_param   ::= type_param | const_param
type_param      ::= identifier (":" type_bounds)? ("=" type_expr)?
const_param     ::= "const" identifier ":" type_expr ("=" const_expr)?
type_bounds     ::= type_expr ("+" type_expr)*
where_clause    ::= "where" where_pred ("," where_pred)* ","?
where_pred      ::= type_expr ":" type_bounds
```

## 2.10.1. 매개변수 선언 위치

제네릭 매개변수는 다음 위치에서 등장한다.

```nok
fn id<T>(x: T) -> T { x }                       // 함수
struct Pair<A, B> { first: A; second: B; }      // 구조체
enum Result<T, E> { Ok(T), Err(E) }              // enum
trait Convert<Target> { ... }                    // 트레잇
impl<T> List<T> { ... }                          // impl
type Maybe<T> = T | none;                        // 타입 별칭
```

각 위치에서 매개변수 선언 형태는 같다. 이름 뒤 `<...>` 안에 콤마로 구분한 매개변수 이름들.

## 2.10.2. 매개변수의 종류

### 타입 매개변수

```nok
fn first<T>(pair: (T, T)) -> T { pair.0 }
```

- 매개변수 이름은 [식별자](../001-lexical/104-identifiers.md) 형식을 따른다.
- 관례적으로 한 글자 대문자 (`T`, `U`, `K`, `V`) 또는 짧은 PascalCase 를 쓴다. 강제는 아니다.

### `const` 매개변수

값을 매개변수로 받는 형태. 정확한 규칙은 [`const` 제네릭](211-const-generics.md) 에서 정의한다.

## 2.10.3. 트레잇 바운드

매개변수가 충족해야 할 트레잇을 적는다.

```nok
fn add_all<T: Add<T>>(items: List<T>) -> T { ... }
fn print_all<T: Display>(items: List<T>) { ... }
fn sort_all<T: Ord + Clone>(ref items: List<T>) { ... }
```

- `매개변수: 트레잇` 으로 단일 바운드.
- `+` 로 여러 바운드를 묶는다.
- 바운드가 없으면 매개변수에 어떤 연산도 가할 수 없다. 그저 값을 옮기는 것만 가능하다.

## 2.10.4. where 절

긴 바운드는 `where` 절로 분리한다.

```nok
fn complex<T, U, V>(a: T, b: U) -> V
where
    T: Add<U, Output = V>,
    V: Display + Clone,
{
    ...
}
```

- 매개변수 선언과 본체 사이에 `where` 키워드.
- 콤마로 구분된 바운드.
- 연관 타입의 제약도 여기서 적을 수 있다 (`T::Output: Foo`).

## 2.10.5. 기본값 (default)

타입 매개변수에 기본값을 줄 수 있다.

```nok
trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}
```

- 호출이나 인스턴스화 시 매개변수를 생략하면 기본값을 쓴다.
- 기본값을 가진 매개변수 뒤에는 기본값 없는 매개변수가 올 수 없다. 뒤로 갈수록 기본값이 있어야 한다.

## 2.10.6. 단형화 (monomorphization)

제네릭 호출과 인스턴스화는 구체 타입 튜플마다 별개의 컴파일 결과를 만든다.

```nok
fn id<T>(x: T) -> T { x }

let a = id(42i32);          // id<i32> 가 만들어짐 — i32 → i32 함수
let b = id(3.14f64);        // id<f64> 가 만들어짐 — f64 → f64 함수
let c = id("hello");         // id<str> 가 만들어짐
```

규칙:

- 같은 타입 튜플의 호출은 같은 결과를 공유한다 (한 번만 컴파일).
- 결과는 직접 호출과 같은 성능. 간접 호출 비용 0.
- 비용은 컴파일 시간 + 바이너리 크기. MLIR + LLVM 의 dead-code-elimination 으로 대부분 상쇄된다.

### 단형화의 캐시 모델

- [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 빌드. 호출 지점을 수집해 한 번에 [specialize](../000-meta/007-glossary.md#specialize).
- [JIT](../000-meta/007-glossary.md#jit-just-in-time) 모드. 처음 호출되는 타입 튜플에 대해 specialize 후 캐시.
- 두 모드 모두 같은 specialize 기계를 공유한다. [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 의 공통 파이프라인.

## 2.10.7. 점진과의 결합

[`any` 슬롯](202-any-slot.md) 에서 본 함수 매개변수의 점진성은 다음과 같이 처리된다.

```nok
fn compute(x, y) { ... }
```

위는 암묵 제네릭으로 변환된다. 컴파일러가 본체의 사용으로부터 필요한 바운드를 추론해 명시 제네릭으로 다시 적은 것과 똑같이 처리한다 (예: 본체에 `x + y` 가 있으면 `T: Add<U>` 바운드, 반환 타입은 그 트레잇의 연관 타입).

- 본체의 사용으로부터 필요한 바운드를 추론한다.
- 호출 지점의 타입에 따라 단형화된다.

명시적 `any` 가 매개변수에 적히면 단형화되지 않고 런타임 동적 슬롯이 된다. 이 차이가 점진 모드의 핵심이다.

## 2.10.8. 제네릭의 종류 일관성

제네릭 매개변수는 한 호출에서 한 타입을 가리킨다. 다른 호출에서 다른 타입을 가리키는 것은 허용하지만, 같은 호출 안에서 바뀌는 것은 허용하지 않는다.

```nok
fn pair<T>(a: T, b: T) -> (T, T) { (a, b) }

let p = pair(1i32, 2i32);     // OK — 두 인자 모두 T = i32
let q = pair(1i32, 3.14f64);  // ERROR — T 가 i32 인지 f64 인지 모호
```

다중 매개변수 (`pair<A, B>`) 는 각각 독립적으로 추론된다.

## 2.10.9. 본체 검사

제네릭 함수의 본체는 바운드를 통해 알 수 있는 동작만 쓸 수 있다.

```nok
fn double<T>(x: T) -> T {
    x + x       // ERROR — T 에 Add 바운드가 없다
}

fn double<T: Add<T, Output = T>>(x: T) -> T {
    x + x       // OK
}
```

이 규칙 덕분에 제네릭 본체의 검사는 호출 지점과 독립이다. 호출에서 어떤 구체 타입을 넘기든 본체는 항상 컴파일된다.

## 2.10.10. 제네릭과 트레잇-as-타입의 차이

제네릭은 컴파일 타임 다형성이다. 단형화로 직접 호출.

트레잇-as-타입은 런타임 다형성이다. vtable 디스패치. nok 은 이를 [타입 시스템 개요](README.md) 에서 금지한다.

다형성이 필요하면 항상 제네릭 (본 문서) 또는 합 타입 ([합 타입](206-sum-types.md)) 을 쓴다.
