# 3.5. 멤버 접근과 메서드 호출

`.` 토큰([구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md))은 값에 딸린 항목에 접근하는 두 가지 형태에 쓰입니다.

```
member_expr     ::= primary "." identifier
                  | primary "." int_lit               // tuple position
method_call     ::= primary "." identifier ("::" "<" type_expr ("," type_expr)* ">")?
                    "(" arg_list? ")"
arg_list        ::= expr ("," expr)* ","?
```

## 3.5.1. 멤버 접근: `obj.field`

```rok
let p = Point { x: 1.0f64, y: 2.0f64 };
let xv = p.x;               // 1.0
let yv = p.y;               // 2.0
```

`obj`는 임의의 표현식이고, `field`는 [식별자](../001-lexical/104-identifiers.md)의 형식을 따릅니다. 결과는 그 필드의 값입니다.

### tuple `struct`와 튜플의 위치 접근

```rok
let pair = Pair(1i32, 2i32);
let first = pair.0;
let second = pair.1;
```

점 뒤에 10진 숫자가 오면 위치 인덱스로 해석합니다. [숫자 리터럴](../001-lexical/106-numeric-literals.md)의 `1.foo` 규칙과 같은 토큰화입니다. 예를 들어 `pair.0`은 식별자 `pair`, `.`, 정수 `0`의 세 토큰으로 나뉘고, 파서가 이를 위치 접근으로 해석합니다.

### 정적 결정과 동적 결정

좌변의 정적 타입을 알면 필드 위치는 컴파일 타임에 정해지고, 직접 메모리 접근으로 컴파일됩니다. 좌변이 [`any` 슬롯](../002-types/202-any-slot.md)이면 동적 디스패치가 일어납니다. 런타임에 태그를 검사하고 알맞은 필드 위치를 찾습니다.

## 3.5.2. 메서드 호출: `obj.method(args)`

```rok
let s = "hello";
let n = s.len();
let p = Point { x: 1.0, y: 2.0 };
let d = p.distance(other);
```

형식은 `expr . 메서드이름 ( 인자들 )`입니다. 메서드 해소(resolution)는 좌변의 정적 타입을 봅니다. 그 타입에 정의된 메서드나, 그 타입이 충족하는 트레잇의 메서드를 찾습니다.

### 메서드 해소 순서

같은 이름의 메서드가 여러 곳에 정의되어 있을 수 있습니다. 컴파일러는 다음 순서로 후보를 찾습니다.

1. 먼저 고유 `impl`([`impl`](../002-types/209-impl.md))에 정의된 메서드를 찾습니다.
2. 다음으로 트레잇 `impl`의 메서드를 찾습니다. 좌변 타입이 충족하는 트레잇 중 현재 모듈에서 보이는 트레잇만 대상입니다. 트레잇이 보이는 경우는 자동으로 보이는 `rok::**`의 트레잇, `use`로 가져온 트레잇, 같은 모듈에서 정의한 트레잇입니다.

트레잇 후보가 여럿이고 그중 하나만 그 모듈에서 정의했거나 `use`로 가져온 트레잇이면 그 후보를 고릅니다(고유 메서드는 언제나 이깁니다). 그 밖에 후보가 동시에 여럿 있으면 모호하다는 컴파일 에러입니다. 이때는 [타입 경로](../002-types/213-type-paths.md)의 완전 자격 형태로 직접 해소해야 합니다.

0.1 계열에서는 트레잇이 보이지 않아도 그 트레잇 메서드가 해소되고, 추가할 `use`를 알려 주는 경고가 나옵니다(`trait_not_in_scope`, [6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)): ``method `hi` comes from trait `a::Greet`, which is not in scope``, help ``add `use a::Greet;` ``.

### 메서드 수신자의 자동 `ref`

메서드 호출 자리의 수신자(`obj`)는 메서드의 `self` 종류(`self` / `ref self`)에 따라 자동으로 처리됩니다. 호출하는 쪽에서 `ref`를 쓰지 않아도 됩니다.

```rok
p.distance(other);          // self
p.replace_with(other);      // ref self: 호출자가 'ref' 명시하지 않음
```

이 규칙은 메서드 수신자에만 적용하는 특수 규칙입니다. 일반 함수 호출에서 `ref`를 명시하는 규칙과는 다릅니다. 자세한 내용은 함수 사양에서 정의합니다.

### 메서드 호출 풀어 쓰기

```
obj.method(a, b)
```

위 표현식은 의미상 다음과 같습니다.

```
<typeof(obj) as TraitContaining_method>::method(obj, a, b)
```

(고유 `impl`에 정의된 메서드라면 `Type::method(obj, a, b)`입니다.)

이렇게 풀어 쓰는 방식은 [연산자 디스패치](301-operator-dispatch.md)의 트레잇 메서드 매핑과 같은 메커니즘을 따릅니다. `.` 토큰 자체에 `#[infix(".")]` 같은 매핑이 있지는 않지만, 결과 호출의 모양은 같습니다.

### turbofish: 제네릭 메서드 명시

[타입 경로](../002-types/213-type-paths.md)의 turbofish 형태로 메서드의 제네릭 인자를 명시할 수 있습니다.

```rok
let v = expr.parse::<i32>();
let m = arr.collect::<List<f64>>();
```

형태는 `.method::<...>(args)`입니다. `.` 뒤에 메서드 이름, 그다음 turbofish, 마지막으로 인자 괄호가 옵니다.

## 3.5.3. 정적 항목 접근과 다른 점

`Type::item`([타입 경로](../002-types/213-type-paths.md))은 타입에 딸린 항목에 접근합니다.

```
Type::item       // 정적 메서드 / 연관 const / 연관 타입 (인스턴스 없이)
obj.item         // 인스턴스에 부속된 멤버 / 메서드 (인스턴스 필요)
```

두 형태는 완전히 다른 종류의 접근입니다. 토큰(`::`와 `.`)도 다르고 의미도 다릅니다.

### 메서드 경로 값

괄호 없이 쓴 메서드 경로(`T::method`, `Trait::method`, `<T as Tr>::method`, `i32::add`, `S::new`)는 함수 값입니다.

| 메서드 종류 | 함수 값의 타입                         |
|-------------|----------------------------------------|
| 정적 메서드 | `fn(args) -> R`                        |
| 인스턴스 메서드 | `fn(Self, args) -> R` (`self`가 첫 인자)  |

```rok
let mk = Point::new;               // fn(f64, f64) -> Point
let add = i32::add;                // fn(i32, i32) -> i32
xs.map(Point::norm);               // fn(Point) -> f64
```

- `ref self`나 `ref` 매개변수가 있는 메서드는 값이 될 수 없습니다([4.7.6](../004-functions/407-ref-keyword.md#476-ref와-타입)).
- 에러를 던지는 메서드는 에러를 던지는 함수 타입(`fn(..) throws (E) -> R`)이 됩니다.
- `Tr::m`이나 `Self`가 정해지지 않은 제네릭 타입의 메서드는 기대 타입으로 함수 타입을 정합니다. 기대 타입이 없으면 뒤에 오는 호출이 정합니다(`let f = Box::get; f(b)`, [2.10.2](../002-types/210-generics.md#타입-인자-추론)). 그래도 정해지지 않으면 "type annotations needed" 에러입니다.
- 제네릭 메서드의 타입 인자는 turbofish나 기대 함수 타입으로 정합니다.
- `#[intrinsic]` 메서드도 값이 될 수 있습니다.

`obj.method`처럼 인스턴스에 붙인 형태는 함수 값이 아닙니다(3.5.4).

## 3.5.4. `.method`(호출 없이)

괄호 없이 `.method`만 쓰면 Rok은 이를 함수 값으로 해석하지 않습니다. 부분 적용이나 메서드 참조 같은 별도의 의미도 없습니다.

```
let f = obj.method;        // ERROR: 메서드 호출이 아니라 식이 미완성
let f = || obj.method();   // 명시적 closure 로 감싸기
```

암묵적 부분 적용이 낳는 모호함을 막고, 의도를 코드에 직접 쓰게 하려는 결정입니다.

## 3.5.5. 안전한 멤버 접근: `?.`

값이 없을 수 있는 자리의 점 접근은 옵셔널 체이닝으로 씁니다. 옵셔널 체이닝은 [옵셔널 체이닝과 null 합체](308-optional-chaining-and-coalescing.md)에서 정의합니다.
