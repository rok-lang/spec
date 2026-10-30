# 2.8. `trait`

`trait`는 공통 동작 묶음을 선언하는 컨테이너입니다. 어떤 타입이 그 동작을 구현하면 그 타입은 트레잇을 충족(satisfy)합니다.

이 문서는 트레잇의 선언 형태만 정의합니다. 트레잇을 구현하는 `impl` 블록은 [`impl`](209-impl.md)에서 정의합니다.

## 2.8.1. 기본 형태

트레잇 선언의 문법은 다음과 같습니다.

```
trait_decl  ::= visibility? "trait" identifier generics? supertraits? where_clause? "{" trait_item* "}"
supertraits ::= ":" type_expr ("+" type_expr)*
trait_item  ::= visibility? method_sig
              | visibility? assoc_type_decl
              | visibility? method_default
              | visibility? assoc_const_decl
method_sig  ::= "fn" identifier "(" param_list? ")" throws_clause? return_type? ";"
method_default ::= "fn" identifier "(" param_list? ")" throws_clause? return_type? block
assoc_type_decl ::= "type" identifier (":" type_expr ("+" type_expr)*)? where_clause? ("=" type_expr)? ";"
assoc_const_decl ::= "const" identifier ":" type_expr ("=" const_expr)? ";"
```

트레잇 선언의 예는 다음과 같습니다.

```rok
trait Describe {
    fn describe(self) -> str;
}

trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}

trait Iterator {
    type Item;
    fn next(ref self) -> Item | none;
}
```

트레잇의 본체는 `{ ... }`로 감쌉니다. 본체 안에는 메서드 시그니처, 연관 타입(associated type), 기본 메서드 본체를 둘 수 있습니다.

## 2.8.2. 메서드 시그니처

메서드 시그니처는 다음과 같이 적습니다.

```rok
fn 이름(매개변수들) -> 반환타입;
```

본체 없이 시그니처만 적으면 트레잇이 요구하는 메서드입니다. 구현체가 본체를 적어야 합니다. 본체가 있으면 기본 메서드입니다. 구현체가 덮어쓰지 않으면 그 본체를 그대로 씁니다. `self`와 `ref self` 매개변수의 의미는 함수 사양에서 정의합니다.

요구 메서드와 기본 메서드를 함께 쓴 예는 다음과 같습니다.

```rok
trait Greeter {
    fn greet(self) -> str;                          // 요구
    fn greet_loudly(self) -> str { self.greet() }    // 기본 - 덮어쓰기 가능
}
```

## 2.8.3. 연관 타입(associated type)

연관 타입을 쓰면 트레잇의 구현마다 다른 타입을 정할 수 있습니다.

```rok
trait Iterator {
    type Item;
    fn next(ref self) -> Item | none;
}
```

연관 타입은 `type 이름;` 형태로 선언하고, 구현체가 그 자리에 구체 타입을 채웁니다. 한 구현체에서 연관 타입 하나는 구체 타입 하나로 정해집니다. 한 구현이 여러 타입을 동시에 고를 수는 없습니다.

연관 타입은 같은 트레잇 안의 시그니처에서 쓸 수 있습니다. 위의 `Item | none`이 그 예입니다. `throws (Self::X)`처럼 throws 절에도 쓸 수 있습니다. 트레잇 시그니처와 impl 시그니처 모두에서 쓸 수 있습니다.

### 기본값

연관 타입에는 `type X: Bounds = T;`로 기본값을 줄 수 있습니다. impl은 그 연관 타입을 생략해 기본값을 쓰거나, `type X = U;`로 덮어씁니다. 기본값은 트레잇을 정의한 자리에서 트레잇의 매개변수를 대입해 해소합니다.

```rok
trait TryFrom<U> {
    type Error = ConversionError;
    fn try_from(value: U) throws (Self::Error) -> Self;
}
impl TryFrom<RawPath> for Path { type Error = NonUtf8Error; ... }   // 덮어씀
impl TryFrom<Celsius> for Kelvin { ... }                           // Error = ConversionError
```

트레잇 안과 모든 impl(트레잇 impl과 고유 impl) 안에서 맨 이름 `Item`과 `Self::Item`은 둘 다 그 연관 타입을 가리킵니다. 메서드 시그니처와 본문에서 모두 쓸 수 있습니다. impl에서는 그 impl이 정한 구체 타입을 가리킵니다. 연관 타입이 바깥에 있는 같은 이름의 타입(모듈의 `struct Item`, 원시 타입 등)을 가리면 컴파일러가 경고합니다. 같은 이름의 트레잇은 타입이 아니므로 경고하지 않습니다(`trait IntoFuture { type IntoFuture; }`는 괜찮습니다).

한 타입이 이름이 같은 연관 타입이 있는 트레잇 둘을 구현할 수 있습니다. 그러면 그 타입의 impl 본문에서 맨 `X` / `Self::X`는 어느 쪽인지 모호하므로 에러입니다. 이때는 `<Self as Tr>::X`로 트레잇을 밝힙니다.

```rok
impl Iterator for Stream { type Item = i32; ... }
impl Source for Stream { type Item = str; ... }

impl Stream {
    fn first(self) -> Self::Item { ... }                   // ERROR - Iterator::Item? Source::Item?
    fn first(self) -> <Self as Iterator>::Item { ... }     // OK
}
```

### 외부에서 연관 타입 참조: `T::AssocType`

트레잇으로 제약된 매개변수의 연관 타입을 외부에서 참조할 때는 `T::AssocType` 표기를 씁니다.

```rok
fn first_item<T: Iterator>(ref it: T) -> T::Item | none { it.next() }
```

바운드가 제네릭 트레잇 `T: Tr<A>`이면 `T::X`는 `Tr<A>` impl의 `X`입니다. 한 타입이 `Tr<A>`와 `Tr<B>`를 서로 다른 `X`로 구현할 수 있기 때문입니다. `<T as Tr<A>>::X`도 같습니다([2.13.4](213-type-paths.md#2134-완전-한정-경로fully-qualified-path)). 바운드가 `T: Tr<A> + Tr<B>`이거나 바운드의 두 트레잇이 모두 `X`를 선언하면 맨 `T::X`는 모호하므로 에러입니다. 이때는 `<T as Tr<A>>::X`로 적습니다. 연관 상수도 마찬가지입니다(2.8.3a).

이 표기는 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)의 `::` 경로 구분자로 연관 항목에 접근하는 것입니다. 경로의 일반 규칙(모듈 경로, turbofish, 완전 자격 형태)은 [타입 경로](213-type-paths.md)에서 정의합니다.

## 2.8.3a. 연관 상수(associated const)

트레잇은 `const N: T;`로 연관 상수를 선언할 수 있고, `= 값`으로 기본값을 줄 수 있습니다. 값은 각 `impl`이 정합니다. 기본값이 있으면 생략할 수 있습니다. 그 트레잇으로 제약된 타입 매개변수에서는 `T::N`으로 읽습니다.

```rok
trait Shape {
    const SIDES: u32;
    const NAME: str = "shape";
}

impl Shape for Square { const SIDES: u32 = 4; }

fn sides<T: Shape>() -> u32 { T::SIDES }      // <T as Shape>::SIDES 와 같음
```

제네릭 타입의 `impl`에서는 연관 상수의 값이 인스턴스화마다 다를 수 있습니다(`impl<const N: usize> Shape for Poly<N> { const SIDES: u32 = N as u32; }`). 모호하면 완전 자격 형태 `<T as Tr>::N`을 씁니다([2.13.4](213-type-paths.md#2134-완전-한정-경로fully-qualified-path)).

## 2.8.4. 슈퍼트레잇(supertrait)

트레잇은 다른 트레잇을 충족하는 타입에만 구현하도록 요구할 수 있습니다.

```rok
trait Eq {
    fn eq(self, other: Self) -> bool;
}

trait Ord: Eq {
    fn cmp(self, other: Self) -> Ordering;
}
```

슈퍼트레잇은 `:` 뒤에 적습니다. 따라서 `Ord`를 구현하려면 `Eq`도 구현해야 합니다. 슈퍼트레잇 여럿을 요구할 때는 `+`로 이어 `trait Foo: Bar + Baz`처럼 적습니다. 제네릭 impl은 바운드가 허용하는 모든 인스턴스에서 대상 타입이 슈퍼트레잇을 갖추게 해야 합니다. 컴파일러는 이 조건을 impl에서 검사합니다.

## 2.8.5. 제네릭 트레잇

트레잇에도 타입 매개변수를 둘 수 있습니다.

```rok
trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}

trait Convert<Target> {
    fn convert(self) -> Target;
}
```

타입 매개변수는 트레잇 이름 뒤 `<...>` 안에 선언합니다. `Rhs = Self`처럼 기본값을 줄 수도 있습니다. `Self`는 트레잇을 구현하는 타입을 가리키는 키워드입니다([키워드](../001-lexical/105-keywords.md)).

제네릭 트레잇의 정확한 단형화 규칙은 [제네릭](210-generics.md)에서 정의합니다.

## 2.8.6. where 절: 매개변수에 추가 제약

트레잇 선언이나 메서드 시그니처에 추가 트레잇 바운드를 적을 수 있습니다.

```rok
trait Sortable<T> where T: Ord {
    fn sort(ref self);
}

trait Container {
    type Item where Item: Clone;
    fn first(self) -> Item;
}
```

제약은 `where` 키워드 뒤에 `타입: 트레잇` 형태로, 콤마로 구분해 적습니다. 트레잇 자체에 붙이면 트레잇 전체의 제약이 되고, 메서드에 붙이면 그 메서드만의 제약이 됩니다.

## 2.8.7. 트레잇 메서드의 디스패치

트레잇 메서드를 호출하면 컴파일러는 호출 지점의 수신자 타입을 보고, 그 타입에 구현된 메서드를 정적으로 호출합니다.

```rok
let p = Point { x: 1.0, y: 2.0 };
let s = p.describe();       // Point 의 Describe 구현이 정적으로 호출됨
```

이 디스패치는 항상 정적입니다. 트레잇 이름을 변수, 매개변수, 반환 타입에 적는 트레잇-as-타입을 [타입 시스템 개요](README.md)에서 금지하기 때문입니다.

그래서 다형성이 필요하면 제네릭([제네릭](210-generics.md))이나 합 타입([합 타입](206-sum-types.md))을 씁니다.

## 2.8.8. 가시성

트레잇 자체의 가시성과 메서드의 가시성은 따로 정할 수 있습니다. 정확한 규칙은 [가시성](../005-modules/506-visibility.md)에서 정의합니다.

## 2.8.9. 동일 모듈 내 중복 정의 금지

같은 모듈에서 같은 이름의 트레잇을 두 번 선언하면 컴파일 에러입니다. 한 트레잇에 같은 이름의 메서드를 두 번 선언해도 컴파일 에러입니다(오버로딩 없음).
