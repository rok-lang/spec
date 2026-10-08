# 2.8. `trait`

`trait` 는 공통 동작 묶음을 선언하는 컨테이너다. 어떤 타입이 그 동작을 구현하면 그 타입은 그 트레잇을 충족한다 (satisfies).

본 문서는 트레잇의 선언 형태만 정의한다. 트레잇을 구현하는 `impl` 블록은 [`impl`](209-impl.md) 에서 정의한다.

## 2.8.1. 기본 형태

```
trait_decl  ::= "trait" identifier generics? supertraits? where_clause? "{" trait_item* "}"
supertraits ::= ":" type_expr ("+" type_expr)*
trait_item  ::= method_sig
              | assoc_type_decl
              | method_default
method_sig  ::= "fn" identifier "(" param_list? ")" return_type? ";"
method_default ::= "fn" identifier "(" param_list? ")" return_type? block
assoc_type_decl ::= "type" identifier where_clause? ";"
```

```nok
trait Display {
    fn display(self) -> str;
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

- 본체는 `{ ... }` 로 둘러싼다.
- 본체에는 메서드 시그니처, 연관 타입 (associated type), 기본 메서드 본체를 둘 수 있다.

## 2.8.2. 메서드 시그니처

```nok
fn 이름(매개변수들) -> 반환타입;
```

- 본체 없이 시그니처만 적으면 트레잇이 요구하는 메서드다. 구현체가 본체를 제공해야 한다.
- 본체가 있으면 기본 메서드다. 구현체가 덮어쓰지 않으면 그 본체를 쓴다.
- `self` / `ref self` 매개변수의 의미는 함수 사양에서 정의한다.

```nok
trait Greeter {
    fn greet(self) -> str;                          // 요구
    fn greet_loudly(self) -> str { self.greet() }    // 기본 — 덮어쓰기 가능
}
```

## 2.8.3. 연관 타입 (associated type)

트레잇이 구현 별로 다른 타입을 가지도록 선언한다.

```nok
trait Iterator {
    type Item;
    fn next(ref self) -> Item | none;
}
```

- `type 이름;` 형태로 선언한다.
- 구현체는 그 자리에 구체 타입을 채운다.
- 한 구현체에서 한 연관 타입은 정확히 하나의 구체 타입으로 정해진다 (한 구현이 여러 가능성을 동시에 가질 수 없다).

연관 타입은 같은 트레잇 안의 시그니처에서 쓸 수 있다. 위의 `Item | none` 처럼.

### 외부에서의 연관 타입 참조. `T::AssocType`

트레잇 매개변수의 연관 타입을 외부에서 참조할 때는 `T::AssocType` 표기를 쓴다.

```nok
fn first_item<T: Iterator>(ref it: T) -> T::Item | none { it.next() }
```

이 표기는 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md) 의 `::` 경로 구분자를 쓴 연관 항목 접근이다. 경로의 일반 규칙 (모듈 경로·turbofish·완전 자격 형태) 은 [타입 경로](213-type-paths.md) 에서 정의한다.

## 2.8.4. 슈퍼트레잇 (supertrait)

어떤 트레잇이 다른 트레잇을 충족하는 타입에만 구현될 수 있도록 요구할 수 있다.

```nok
trait Eq {
    fn eq(self, other: Self) -> bool;
}

trait Ord: Eq {
    fn cmp(self, other: Self) -> Ordering;
}
```

- `:` 뒤에 슈퍼트레잇을 적는다.
- `Ord` 를 구현하려면 `Eq` 도 함께 구현해야 한다.
- 여러 슈퍼트레잇은 `+` 로 묶는다: `trait Foo: Bar + Baz`.

## 2.8.5. 제네릭 트레잇

```nok
trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}

trait Convert<Target> {
    fn convert(self) -> Target;
}
```

- 트레잇 이름 뒤 `<...>` 로 타입 매개변수를 선언한다.
- 매개변수에 기본값을 줄 수 있다 (`Rhs = Self`).
- `Self` 는 트레잇이 구현될 타입을 가리키는 키워드다 ([키워드](../001-lexical/105-keywords.md)).

제네릭 트레잇의 정확한 단형화 규칙은 [제네릭](210-generics.md) 에서 정의한다.

## 2.8.6. where 절. 매개변수에 추가 제약

트레잇 선언이나 메서드 시그니처에 추가 트레잇 바운드를 적을 수 있다.

```nok
trait Sortable<T> where T: Ord {
    fn sort(ref self);
}

trait Container {
    type Item where Item: Clone;
    fn first(self) -> Item;
}
```

- `where` 키워드 뒤에 `타입: 트레잇` 형태의 제약을 콤마로 구분해 적는다.
- 트레잇 자체에 부착하면 전체 트레잇의 제약, 메서드에 부착하면 그 메서드만의 제약이다.

## 2.8.7. 트레잇 메서드의 디스패치

트레잇 메서드를 호출하면 컴파일러는 호출 지점의 수신자 타입을 보고 해당 타입에 구현된 메서드를 정적으로 호출한다.

```nok
let p = Point { x: 1.0, y: 2.0 };
let s = p.display();        // Point 의 Display 구현이 정적으로 호출됨
```

이 디스패치는 항상 정적이다. 트레잇 이름을 변수, 매개변수, 반환 타입에 적는 트레잇-as-타입이 [타입 시스템 개요](README.md) 에서 금지되기 때문이다.

다형성이 필요하면 제네릭 ([제네릭](210-generics.md)) 또는 합 타입 ([합 타입](206-sum-types.md)) 을 쓴다.

## 2.8.8. 가시성

트레잇 자체와 그 메서드의 가시성은 따로 조절할 수 있다. 정확한 규칙은 [가시성](../005-modules/506-visibility.md) 에서 정의한다.

## 2.8.9. 동일 모듈 내 중복 정의 금지

같은 모듈 안에서 같은 이름의 트레잇을 두 번 선언하면 컴파일 에러다.
