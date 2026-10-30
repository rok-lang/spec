# 2.10. 제네릭

제네릭은 함수, 타입, 트레잇, `impl` 블록에 타입 매개변수를 두는 메커니즘입니다. 호출 지점이나 인스턴스화 지점의 구체 타입 튜플마다 별도의 컴파일 결과를 만들며, 이 과정을 단형화라고 합니다.

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
> `where`는 함수 시그니처, `impl` 헤더, `struct` / `enum` / `union` / 타입 별칭 헤더 뒤에서만 키워드인 문맥 키워드입니다([1.5.4](../001-lexical/105-keywords.md#154-문맥-키워드)).

## 2.10.1. 매개변수 선언 위치

제네릭 매개변수는 다음 위치에 나옵니다.

```rok
fn id<T>(x: T) -> T { x }                       // 함수
struct Pair<A, B> { first: A; second: B; }      // 구조체
enum Result<T, E> { Ok(T), Err(E) }              // enum
union Bits<T> { v: T; raw: u64; }               // union
trait Convert<Target> { ... }                    // 트레잇
impl<T> List<T> { ... }                          // impl
type Maybe<T> = T | none;                        // 타입 별칭
```

매개변수를 선언하는 형태는 어느 위치에서나 같습니다. 이름 뒤의 `<...>` 안에 매개변수 이름을 콤마로 구분해 나열합니다.

## 2.10.2. 매개변수의 종류

### 타입 매개변수

```rok
fn first<T>(pair: (T, T)) -> T { pair.0 }
```

매개변수 이름은 [식별자](../001-lexical/104-identifiers.md) 형식을 따릅니다. 관례상 `T`, `U`, `K`, `V` 같은 대문자 한 글자나 짧은 PascalCase 이름을 쓰지만, 강제하지는 않습니다.

### `const` 매개변수

값을 매개변수로 받는 형태입니다. 정확한 규칙은 [`const` 제네릭](211-const-generics.md)에서 따로 정의합니다.

### 타입 인자 추론

호출 지점에서 타입 인자를 적지 않으면 컴파일러가 추론합니다. 인자의 타입뿐 아니라 기대 타입(`let`의 타입 주석, 인자 자리의 매개변수 타입, 반환 자리의 반환 타입)도 추론에 씁니다.

```rok
fn empty<T>() -> List<T> { [] }

let xs: List<i32> = empty();        // T = i32, let 의 타입 주석에서
takes_strs(empty());                // T = str, 매개변수 타입 List<str> 에서
```

기대 타입이 없는 `let`은 나중의 쓰임에서 추론합니다. 주석 없는 `let x = f();`에서 `f`의 제네릭 반환 타입이 호출만으로 정해지지 않으면, 이후의 쓰임이 그 타입을 정합니다. 시그니처는 항상 명시합니다.

- **범위**: 그 변수의 스코프, 곧 블록의 나머지입니다. 그 안의 클로저와 async 본문도 포함합니다.
- **타입을 정하는 쓰임**: 타입이 정해진 자리로 값을 넘기는 쓰임입니다. 함수 / 메서드 인자, 대입, 주석 있는 `let`, struct 필드, `return`이 여기에 속합니다. 연산자, f-string 구멍, 패턴, 필드 읽기는 타입을 정하지 않습니다.
- **서로 정하기**: 아직 열린 `let` 여럿이 서로의 타입을 정할 수 있습니다(`let a = List::new(); let b = List::new(); a.push(b); b.push(1);` → `b: List<i32>`, `a: List<List<i32>>`).
- **리터럴의 기본 타입**: `i32` / `f64`는 같은 종류(정수 / 실수)의 이후 구체 쓰임에 양보합니다. `ys.push(1); sum_i64(ys)`의 `ys`는 `List<i64>`입니다.
- **충돌**: 서로 다른 두 쓰임은 합 타입이 되지 않고 에러입니다(`xs.push(none); xs.push(1)`, [5.1.7](../005-modules/501-variable-declaration.md#517-타입-명시와-타입-추론)의 분기 규칙과 같음). `let v = none;`만으로는 여전히 에러입니다.
- 클로저 매개변수([4.5.2](../004-functions/405-closure.md))와 메서드 경로 값([3.5.3](../003-expressions/305-member-access-and-method-call.md#메서드-경로-값))도 문맥이 없으면 이후 호출이 타입을 정합니다(`let f = |x| x * 2; f(4)`, `let f = Box::get; f(b)`).

진단은 다음과 같습니다.

| 경우         | 위치                                         | 메시지 예                                                                  |
|--------------|----------------------------------------------|----------------------------------------------------------------------------|
| 끝까지 안 정해짐 | 그 `let`                                    | `` type annotations needed: cannot infer the type of `d` (`Dict<_, _>`) `` |
| 충돌         | 쓰임 자리, 타입을 정한 자리를 보조 레이블로 | `` conflicting uses of `c` (`List<_>`): `str` here, but `i32` fixed at 8:5 `` |

```rok
let xs = empty();                   // 아직 List<?>
takes_ints(xs);                     // 여기서 T = i32 (매개변수 List<i32>)
let zs = List::new();
zs.push(1);                         // zs: List<i32>, 이후 메서드 인자에서
let ys = empty();                   // ERROR: type annotations needed (구체 쓰임 없음)
```

인자가 없는 정적 트레잇 메서드도 같은 방식으로 `Self`를 정합니다. 타입 매개변수에 대한 호출 `T::zero()`와 트레잇 경로 호출 `Tr::m()`은 기대 타입에서 `Self`를 가져옵니다.

```rok
let z: i32 = Zero::zero();          // Self = i32 (rok::core::num::Zero)
fn sum<T: Zero + Add<Output = T>>(xs: List<T>) -> T {
    let acc: T = T::zero();          // Self = T
    ...
}
```

제네릭 `enum`의 변형 생성자는 인스턴스를 먼저 인자에서, 다음으로 타입 매개변수의 기본값에서, 그다음 그 값을 받는 `let` 추론 변수에서 정합니다. 그래도 정해지지 않으면 ``type annotations needed`` 에러입니다.

```rok
enum Pair<A, B> { One(A), Two(A, B) }
let p = Pair::Two(1, "a");          // Pair<i32, str>, 인자에서
```

## 2.10.3. 트레잇 바운드

바운드는 매개변수가 충족해야 할 트레잇을 적는 방식입니다.

```rok
fn add_all<T: Add<T>>(items: List<T>) -> T { ... }
fn print_all<T: Display>(items: List<T>) { ... }
fn sort_all<T: Ord + Clone>(ref items: List<T>) { ... }
```

바운드 하나는 `매개변수: 트레잇` 형태로 적고, 여럿이면 `+`로 잇습니다. 바운드가 없으면 매개변수에는 모든 타입이 지원하는 동작만 쓸 수 있습니다. 이동, `Reflect`, `any`에 저장하기, `none`과의 합, `is`(2.10.9)입니다.

bound, `where` 절, `impl`, supertrait 목록, `any Tr`에 적은 트레잇은 다른 이름과 똑같이 해소합니다. 다른 모듈의 트레잇에는 `use`나 경로가 필요합니다([5.5](../005-modules/505-use.md)). 그렇지 않으면 적은 자리를 알려 주는 cannot-find 에러가 나며(``cannot find trait `Shape` (named in a bound)``, `(named in an impl)`, `(named as a supertrait)`), 추가할 `use`도 함께 알려 줍니다.

## 2.10.4. where 절

바운드가 길어지면 `where` 절로 빼는 편이 좋습니다.

```rok
fn complex<T, U, V>(a: T, b: U) -> V
where
    T: Add<U, Output = V>,
    V: Display + Clone,
{
    ...
}
```

매개변수 선언과 본체 사이에 `where` 키워드를 두고, 그 뒤에 바운드를 콤마로 구분해 나열합니다. 연관 타입에 대한 제약도 이 자리에 적을 수 있습니다(예: `T::Output: Foo`).

연관 타입 바인딩에 나오는 타입 매개변수(`I: Iterator<Item = (A, B)>`, 또는 `where` 절의 `Self: Iterator<Item = (A, B)>`)는 인자의 그 연관 타입에서 추론하고, `Self`이면 수신자의 연관 타입에서 추론합니다. 바인딩의 타입은 구조를 가질 수 있습니다. `(A, B)`, `(A, List<B>)`, `fn(A) -> B`는 안에 든 매개변수를 각각 정합니다. 연관 타입이 다른 형태이면 그 호출은 에러이며, 에러는 항목이 어떤 모양이어야 하는지 알려 줍니다: ``` `unzip` needs `Item` of the form `(A, B)`, but `ListIter<i32>` has `Item = i32` ```.

```rok
fn unzip2<I: Iterator<Item = (A, B)>, A, B>(it: I) -> (List<A>, List<B>) { ... }
unzip2([(1, "a"), (2, "b")].iter());   // A = i32, B = str
```

타입 별칭의 `where` 절([2.6.9](206-sum-types.md#269-type-alias-와-결합))은 별칭을 쓰는 자리마다 성립해야 합니다. 타입 매개변수는 그 자리에서 유효한 bound로 이 절을 만족합니다.

## 2.10.5. 기본값 (default)

타입 매개변수에는 기본값을 줄 수 있습니다.

```rok
trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}
```

호출하거나 인스턴스화할 때 매개변수를 생략하면 기본값을 씁니다. 기본값이 있는 매개변수 뒤에는 기본값이 없는 매개변수가 올 수 없습니다. 기본값은 뒤쪽 매개변수에 몰려 있어야 합니다.

### 인자 개수

제네릭 `struct`, `union`, `enum`, 타입 별칭은 매개변수 개수보다 많은 인자를 받지 않고, 기본값 없는 매개변수 개수보다 적은 인자도 받지 않습니다. 그 밖의 경우는 에러이며, 에러는 선언을 함께 보여 줍니다.

```rok
struct P<T, U> { a: T; b: U; }
struct D<T, U = i32> { a: T; b: U; }

let p: P<i64> = P { a: 1, b: 2 };          // ERROR: `P` takes 2 generic arguments but 1 was given
                                           //   help: `P` is declared `P<T, U>`; add one for `U`
let d: D<i32, i32, i32> = D { a: 1, b: 2 }; // ERROR: `D` takes 1 to 2 generic arguments but 3 were given
let e: D<i32> = D { a: 1, b: 2 };          // OK: `U` 는 `i32`
```

함수나 메서드에는 매개변수보다 적은 수의 인자를 명시할 수 있으며(`f::<i32>(..)`), 나머지는 추론합니다. 더 많이 주면 같은 에러입니다. 트레잇 bound에 트레잇의 매개변수보다 많은 인자를 줘도 마찬가지입니다(`` `Tr` takes at most 1 generic argument but 2 were given ``). enum 경로에 적은 인자는 그 인스턴스를 가리킵니다. `E::<i64>::A(1)`은 `E<i64>`입니다.

타입 자리에 제네릭 타입을 인자 없이 적으면(`struct S<T>`에 대해 `x: S`), 기본값 없는 매개변수가 있을 때 에러입니다(매개변수가 모두 기본값을 가진 타입은 인자 없이 적어도 됩니다): `` `S` needs generic arguments ``, help `` `S` is declared `S<T>`; write it with them, e.g. `S<i32>` ``.

## 2.10.6. 단형화 (monomorphization)

제네릭 호출과 인스턴스화는 구체 타입 튜플마다 별도의 컴파일 결과를 만듭니다.

```rok
fn id<T>(x: T) -> T { x }

let a = id(42i32);          // id<i32> 가 만들어짐: i32 → i32 함수
let b = id(3.14f64);        // id<f64> 가 만들어짐: f64 → f64 함수
let c = id("hello");         // id<str> 가 만들어짐
```

타입 튜플이 같은 호출은 결과를 공유하므로 한 번만 컴파일됩니다. 그 결과는 직접 호출과 성능이 같고, 간접 호출 비용이 없습니다. 대신 컴파일 시간과 바이너리 크기라는 비용이 들지만, MLIR과 LLVM의 dead-code elimination이 대부분 상쇄합니다.

### 단형화의 캐시 모델

[AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 빌드는 호출 지점을 모아 한 번에 [specialize](../000-meta/007-glossary.md#specialize)합니다. [JIT](../000-meta/007-glossary.md#jit-just-in-time) 모드는 타입 튜플이 처음 호출될 때 specialize하고 그 결과를 캐시합니다. 두 모드는 같은 specialize 기계를 공유하며, 이 기계는 [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md)의 공통 파이프라인에 속합니다.

## 2.10.7. 점진과의 결합

[`any` 슬롯](202-any-slot.md)에서 설명한 함수 매개변수의 점진성은 다음과 같이 처리합니다.

```rok
fn compute(x, y) { ... }
```

위 코드는 암묵 제네릭 `fn compute<T, U>(x: T, y: U)`로 바뀝니다. 바운드는 합성하지 않습니다([4.2.2](../004-functions/402-parameters.md#422-타입-생략)). 인스턴스마다 호출 지점에서 단형화하고 그 자리에서 본체를 검사하며, 에러는 호출 지점도 함께 가리킵니다.

매개변수에 `any`를 명시하면 단형화하지 않고 런타임 동적 슬롯이 됩니다. 이 차이가 점진 모드의 핵심입니다.

## 2.10.8. 제네릭의 종류 일관성

제네릭 매개변수는 한 호출 안에서 한 타입만 가리킵니다. 호출마다 다른 타입을 가리킬 수는 있지만, 한 호출 안에서 타입이 바뀔 수는 없습니다.

```rok
fn pair<T>(a: T, b: T) -> (T, T) { (a, b) }

let p = pair(1i32, 2i32);     // OK: 두 인자 모두 T = i32
let q = pair(1i32, 3.14f64);  // ERROR: T 가 i32 인지 f64 인지 모호
```

매개변수가 여럿이면(`pair<A, B>`) 각각 따로 추론합니다.

## 2.10.9. 본체 검사

제네릭 함수의 본체는 바운드로 알 수 있는 동작만 쓸 수 있습니다.

```rok
fn double<T>(x: T) -> T {
    x + x       // ERROR: T 에 Add 바운드가 없다
}

fn double<T: Add<T, Output = T>>(x: T) -> T {
    x + x       // OK
}
```

이 규칙 덕분에 제네릭 본체는 호출 지점과 상관없이 검사합니다. 그래서 호출에서 어떤 구체 타입을 넘기든 본체는 반드시 컴파일됩니다. 이 규칙은 다음과 같이 확장됩니다.

- 트레잇의 기본 메서드 본체도 제네릭 본체입니다. `Self`는 불투명하며 그 트레잇을 바운드로 가지고, 트레잇의 매개변수도 불투명합니다.
- 바운드가 있든 없든 모든 타입은 이동, `Reflect`([6.10.1](../006-meta-system/610-reflection.md#6101-reflect---자동-구현-트레잇)), `any`에 저장하기, `none`과의 합, `is`를 지원합니다.
- 묵시 매개변수([4.2.2](../004-functions/402-parameters.md#422-타입-생략))가 하나라도 있는 함수는 함수 전체가 예외입니다.
- const 매개변수는 값을 모르는 상태로 검사하며, 두 타입은 const 표현식의 값이 같을 때 같습니다([2.11.5](211-const-generics.md#2115-const-표현식의-산술)).
- 매개변수가 아닌 타입에 대한 `where` 절은 본체에서 성립한다고 가정합니다.
- 검사에 실패한 본체의 인스턴스는 자기 에러를 따로 추가하지 않습니다.
- 타입 매개변수가 들어간 타입의 값에 대한 타입 검사(`is`, 타입 패턴)는 반박 가능한 검사로 다루며, 인스턴스마다 결과가 정해집니다.

## 2.10.10. 제네릭과 트레잇-as-타입의 차이

제네릭은 컴파일 타임 다형성입니다. 단형화를 거쳐 직접 호출됩니다.

트레잇-as-타입은 vtable 디스패치를 쓰는 런타임 다형성이며, Rok은 [타입 시스템 개요](README.md)에서 이를 금지합니다.

그래서 다형성이 필요할 때는 언제나 제네릭(이 문서)이나 합 타입([합 타입](206-sum-types.md))을 씁니다.
