# 2.10. 제네릭

제네릭은 함수, 타입, 트레잇, `impl` 블록에 타입 매개변수를 두는 메커니즘입니다. 호출이나 인스턴스화 지점의 구체 타입 튜플마다 별개의 컴파일 결과를 생성하며, 이러한 과정을 단형화라고 부릅니다.

```
generics        ::= "<" generic_param ("," generic_param)* ","? ">"
generic_param   ::= type_param | const_param
type_param      ::= identifier (":" type_bounds)? ("=" type_expr)?
const_param     ::= "const" identifier ":" type_expr ("=" const_expr)?
type_bounds     ::= type_expr ("+" type_expr)*
where_clause    ::= "where" where_pred ("," where_pred)* ","?
where_pred      ::= type_expr ":" type_bounds
```

> [!NOTE]
> `where` 는 함수 시그니처, `impl` 헤더, `struct` / `enum` / `union` / 타입 별칭 헤더 뒤에서만 키워드인 문맥 키워드입니다 ([1.5.4](../001-lexical/105-keywords.md#154-문맥-키워드)).

## 2.10.1. 매개변수 선언 위치

제네릭 매개변수는 다음과 같은 위치에서 등장합니다.

```rok
fn id<T>(x: T) -> T { x }                       // 함수
struct Pair<A, B> { first: A; second: B; }      // 구조체
enum Result<T, E> { Ok(T), Err(E) }              // enum
union Bits<T> { v: T; raw: u64; }               // union
trait Convert<Target> { ... }                    // 트레잇
impl<T> List<T> { ... }                          // impl
type Maybe<T> = T | none;                        // 타입 별칭
```

각 위치에서 매개변수 선언 형태는 모두 동일합니다. 이름 뒤의 `<...>` 안에 매개변수 이름들을 콤마로 구분해 나열하는 방식입니다.

## 2.10.2. 매개변수의 종류

### 타입 매개변수

```rok
fn first<T>(pair: (T, T)) -> T { pair.0 }
```

매개변수 이름은 [식별자](../001-lexical/104-identifiers.md) 형식을 따릅니다. 관례적으로 `T`, `U`, `K`, `V` 와 같이 한 글자 대문자를 쓰거나, 짧은 PascalCase 를 사용하지만 강제되는 규칙은 아닙니다.

### `const` 매개변수

값을 매개변수로 받는 형태입니다. 정확한 규칙은 [`const` 제네릭](211-const-generics.md) 에서 따로 정의합니다.

### 타입 인자 추론

호출 지점에서 타입 인자를 적지 않으면 컴파일러가 추론합니다. 인자의 타입뿐 아니라 기대 타입 (`let` 의 타입 주석, 인자 자리의 매개변수 타입, 반환 자리의 반환 타입) 도 추론에 쓰입니다.

```rok
fn empty<T>() -> List<T> { [] }

let xs: List<i32> = empty();        // T = i32 - let 의 타입 주석에서
takes_strs(empty());                // T = str - 매개변수 타입 List<str> 에서
```

기대 타입이 없는 `let` 은 나중의 쓰임에서 추론합니다. 주석 없는 `let x = f();` 에서 `f` 의 제네릭 반환 타입이 호출만으로 정해지지 않으면, 이후의 쓰임이 그 타입을 정합니다. 시그니처는 항상 명시합니다.

- **범위**: 그 변수의 스코프, 즉 블록의 나머지입니다. 그 안의 클로저와 async 본문도 포함합니다.
- **타입을 정하는 쓰임**: 값을 타입이 정해진 자리로 넘기는 쓰임입니다. 함수 / 메서드 인자, 대입, 주석 있는 `let`, struct 필드, `return` 이 그렇습니다. 연산자, f-string 구멍, 패턴, 필드 읽기는 타입을 정하지 않습니다.
- **서로 정하기**: 열린 `let` 여럿이 서로를 정할 수 있습니다 (`let a = List::new(); let b = List::new(); a.push(b); b.push(1);` → `b: List<i32>`, `a: List<List<i32>>`).
- **리터럴의 기본 타입**: `i32` / `f64` 는 같은 종류 (정수 / 실수) 의 이후 구체 쓰임에 양보합니다. `ys.push(1); sum_i64(ys)` 의 `ys` 는 `List<i64>` 입니다.
- **충돌**: 서로 다른 두 쓰임은 합 타입이 되지 않고 에러입니다 (`xs.push(none); xs.push(1)`, [5.1.7](../005-modules/501-variable-declaration.md#517-타입-명시와-타입-추론) 의 분기 규칙과 같음). `let v = none;` 만으로는 여전히 에러입니다.
- 클로저 매개변수 ([4.5.2](../004-functions/405-closure.md)) 와 메서드 경로 값 ([3.5.3](../003-expressions/305-member-access-and-method-call.md#메서드-경로-값)) 도 문맥이 없으면 이후 호출이 정합니다 (`let f = |x| x * 2; f(4)`, `let f = Box::get; f(b)`).

진단은 다음과 같습니다.

| 경우         | 위치                                         | 메시지 예                                                                  |
|--------------|----------------------------------------------|----------------------------------------------------------------------------|
| 끝까지 안 정해짐 | 그 `let`                                    | `` type annotations needed: cannot infer the type of `d` (`Dict<_, _>`) `` |
| 충돌         | 쓰임 자리, 타입을 정한 자리를 보조 레이블로 | `` conflicting uses of `c` (`List<_>`): `str` here, but `i32` fixed at 8:5 `` |

```rok
let xs = empty();                   // 아직 List<?>
takes_ints(xs);                     // 여기서 T = i32 - 매개변수 List<i32>
let zs = List::new();
zs.push(1);                         // zs: List<i32> - 이후 메서드 인자에서
let ys = empty();                   // ERROR - type annotations needed (구체 쓰임 없음)
```

인자가 없는 정적 트레잇 메서드도 같은 방식으로 `Self` 를 정합니다. 타입 매개변수 위의 호출 `T::zero()` 와 트레잇 경로 호출 `Tr::m()` 은 기대 타입에서 `Self` 를 가져옵니다.

```rok
let z: i32 = Zero::zero();          // Self = i32 - rok::core::num::Zero
fn sum<T: Zero + Add<Output = T>>(xs: List<T>) -> T {
    let acc: T = T::zero();          // Self = T
    ...
}
```

제네릭 `enum` 의 변형 생성자는 먼저 인자에서, 다음으로 타입 매개변수의 기본값에서, 그 다음 그것을 받는 `let` 추론 변수에서 인스턴스를 정합니다. 그래도 정해지지 않으면 ``type annotations needed`` 에러입니다.

```rok
enum Pair<A, B> { One(A), Two(A, B) }
let p = Pair::Two(1, "a");          // Pair<i32, str> - 인자에서
```

## 2.10.3. 트레잇 바운드

매개변수가 충족해야 할 트레잇을 적는 방식입니다.

```rok
fn add_all<T: Add<T>>(items: List<T>) -> T { ... }
fn print_all<T: Display>(items: List<T>) { ... }
fn sort_all<T: Ord + Clone>(ref items: List<T>) { ... }
```

`매개변수: 트레잇` 형태로 단일 바운드를 적으며, 여러 바운드가 필요하면 `+` 로 묶어서 표현합니다. 만약 바운드가 없으면 매개변수에는 모든 타입이 지원하는 것만 적용됩니다: 이동, `Reflect`, `any` 로의 저장, `none` 과의 합, `is` (2.10.9).

bound, `where` 절, `impl`, supertrait 목록, `any Tr` 에 적은 트레잇은 다른 이름과 똑같이 해소합니다: 다른 모듈의 트레잇은 `use` 나 경로가 필요합니다 ([5.5](../005-modules/505-use.md)). 그렇지 않으면 적은 자리를 알리는 cannot-find 에러이며 (``cannot find trait `Shape` (named in a bound)``, `(named in an impl)`, `(named as a supertrait)`), 더할 `use` 를 함께 알립니다.

## 2.10.4. where 절

바운드가 길어지면 `where` 절로 분리하는 편이 좋습니다.

```rok
fn complex<T, U, V>(a: T, b: U) -> V
where
    T: Add<U, Output = V>,
    V: Display + Clone,
{
    ...
}
```

매개변수 선언과 본체 사이에 `where` 키워드를 두고, 그 뒤에 콤마로 구분된 바운드를 나열합니다. 또한 연관 타입에 대한 제약도 이 자리에서 함께 적을 수 있습니다 (예: `T::Output: Foo`).

타입 별칭의 `where` 절 ([2.6.9](206-sum-types.md#269-type-alias-와-결합)) 은 별칭을 쓰는 자리마다 성립해야 하며, 타입 매개변수는 그 자리의 범위에 있는 bound 로 그것을 만족합니다.

## 2.10.5. 기본값 (default)

타입 매개변수에는 기본값을 지정할 수 있습니다.

```rok
trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}
```

호출이나 인스턴스화 시 매개변수를 생략하면 자동으로 기본값이 사용됩니다. 다만 기본값을 가진 매개변수 뒤에는 기본값이 없는 매개변수가 올 수 없으며, 뒤로 갈수록 기본값을 갖춰야 한다는 점에 유의해야 합니다.

### 인자 개수

제네릭 `struct`, `union`, `enum`, 타입 별칭은 매개변수 개수보다 많은 인자를 받지 않으며, 기본값 없는 매개변수 개수보다 적은 인자도 받지 않습니다. 그 밖은 선언을 알리는 에러입니다.

```rok
struct P<T, U> { a: T; b: U; }
struct D<T, U = i32> { a: T; b: U; }

let p: P<i64> = P { a: 1, b: 2 };          // ERROR - `P` takes 2 generic arguments but 1 was given
                                           //   help: `P` is declared `P<T, U>`; add one for `U`
let d: D<i32, i32, i32> = D { a: 1, b: 2 }; // ERROR - `D` takes 1 to 2 generic arguments but 3 were given
let e: D<i32> = D { a: 1, b: 2 };          // OK - `U` 는 `i32`
```

함수나 메서드에는 매개변수보다 적은 명시 인자를 줄 수 있으며 (`f::<i32>(..)`), 나머지는 추론합니다. 더 많이 주면 같은 에러이며, 트레잇 bound 에 트레잇이 가진 것보다 많은 인자를 줘도 그렇습니다 (`` `Tr` takes at most 1 generic argument but 2 were given ``). enum 의 경로에 적은 인자는 그 인스턴스를 가리킵니다: `E::<i64>::A(1)` 은 `E<i64>` 입니다.

타입이 올 자리에 인자 없이 적은 제네릭 타입 (`struct S<T>` 에 대한 `x: S`) 은 기본값 없는 매개변수가 있으면 에러입니다 (매개변수가 모두 기본값을 가진 타입은 인자 없이 적어도 됨): `` `S` needs generic arguments ``, help `` `S` is declared `S<T>`; write it with them, e.g. `S<i32>` ``.

## 2.10.6. 단형화 (monomorphization)

제네릭 호출과 인스턴스화는 구체 타입 튜플마다 별개의 컴파일 결과를 만듭니다.

```rok
fn id<T>(x: T) -> T { x }

let a = id(42i32);          // id<i32> 가 만들어짐 - i32 → i32 함수
let b = id(3.14f64);        // id<f64> 가 만들어짐 - f64 → f64 함수
let c = id("hello");         // id<str> 가 만들어짐
```

같은 타입 튜플의 호출은 같은 결과를 공유하므로 한 번만 컴파일됩니다. 그리고 그 결과는 직접 호출과 동일한 성능을 보이며, 간접 호출 비용이 전혀 들지 않습니다. 다만 컴파일 시간과 바이너리 크기라는 비용이 발생하지만, 이는 MLIR 과 LLVM 의 dead-code-elimination 으로 대부분 상쇄됩니다.

### 단형화의 캐시 모델

[AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 빌드에서는 호출 지점을 수집해 한 번에 [specialize](../000-meta/007-glossary.md#specialize) 합니다. 반면 [JIT](../000-meta/007-glossary.md#jit-just-in-time) 모드에서는 처음 호출되는 타입 튜플에 대해 specialize 한 뒤 그 결과를 캐시해 둡니다. 두 모드 모두 같은 specialize 기계를 공유하며, 이는 [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 의 공통 파이프라인에 해당합니다.

## 2.10.7. 점진과의 결합

[`any` 슬롯](202-any-slot.md) 에서 살펴본 함수 매개변수의 점진성은 다음과 같이 처리됩니다.

```rok
fn compute(x, y) { ... }
```

위 코드는 암묵 제네릭 `fn compute<T, U>(x: T, y: U)` 로 변환됩니다. 바운드는 합성하지 않습니다 ([4.2.2](../004-functions/402-parameters.md#422-타입-생략)): 인스턴스마다 호출 지점에서 단형화하고 거기서 본체를 검사하며, 에러는 호출 지점도 가리킵니다.

한편 명시적 `any` 가 매개변수에 적히면 단형화되지 않고 런타임 동적 슬롯이 됩니다. 바로 이 차이가 점진 모드의 핵심을 이룹니다.

## 2.10.8. 제네릭의 종류 일관성

제네릭 매개변수는 한 호출 안에서 하나의 타입만을 가리킵니다. 다른 호출에서 다른 타입을 가리키는 것은 허용되지만, 같은 호출 안에서 타입이 바뀌는 것은 허용되지 않습니다.

```rok
fn pair<T>(a: T, b: T) -> (T, T) { (a, b) }

let p = pair(1i32, 2i32);     // OK - 두 인자 모두 T = i32
let q = pair(1i32, 3.14f64);  // ERROR - T 가 i32 인지 f64 인지 모호
```

다중 매개변수 (`pair<A, B>`) 는 각각 독립적으로 추론됩니다.

## 2.10.9. 본체 검사

제네릭 함수의 본체는 바운드를 통해 알 수 있는 동작만 사용할 수 있습니다.

```rok
fn double<T>(x: T) -> T {
    x + x       // ERROR - T 에 Add 바운드가 없다
}

fn double<T: Add<T, Output = T>>(x: T) -> T {
    x + x       // OK
}
```

이러한 규칙 덕분에 제네릭 본체의 검사는 호출 지점과 독립적으로 이루어집니다. 그래서 호출에서 어떤 구체 타입을 넘기든 본체는 항상 컴파일이 보장됩니다. 이 규칙은 다음과 같이 확장됩니다.

- 트레잇의 기본 메서드 본체도 제네릭 본체입니다: `Self` 는 불투명하고 그 트레잇을 바운드로 가지며, 트레잇의 매개변수는 불투명합니다.
- 바운드가 있든 없든 모든 타입은 이동, `Reflect` ([6.10.1](../006-meta-system/610-reflection.md#6101-reflect---자동-구현-트레잇)), `any` 로의 저장, `none` 과의 합, `is` 를 지원합니다.
- 묵시 매개변수 ([4.2.2](../004-functions/402-parameters.md#422-타입-생략)) 가 하나라도 있는 함수는 통째로 예외입니다.
- const 매개변수는 값을 모르는 채로 검사하며, 두 타입은 그 const 표현식이 값으로 같을 때 같습니다 ([2.11.5](211-const-generics.md#2115-const-표현식의-산술)).
- 매개변수가 아닌 타입에 대한 `where` 절은 본체에서 성립한다고 가정합니다.
- 검사에 실패한 본체의 인스턴스는 자기 에러를 따로 더하지 않습니다.
- 타입에 타입 매개변수가 들어간 값에 대한 타입 검사 (`is`, 타입 패턴) 는 반박 가능한 것으로 검사하며, 인스턴스마다 결정됩니다.

## 2.10.10. 제네릭과 트레잇-as-타입의 차이

제네릭은 컴파일 타임 다형성에 해당합니다. 단형화를 통해 직접 호출되는 방식입니다.

반면 트레잇-as-타입은 런타임 다형성으로, vtable 디스패치를 사용합니다. 그러나 rok 은 이를 [타입 시스템 개요](README.md) 에서 금지합니다.

따라서 다형성이 필요한 상황에서는 항상 제네릭 (본 문서) 또는 합 타입 ([합 타입](206-sum-types.md)) 을 사용하면 됩니다.
