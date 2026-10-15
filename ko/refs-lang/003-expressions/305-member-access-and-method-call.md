# 3.5. 멤버 접근과 메서드 호출

`.` 토큰 ([구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)) 은 값에 부속된 항목에 접근하는 두 가지 형태에 쓰입니다.

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

여기서 `obj` 는 임의의 표현식이며, `field` 는 [식별자](../001-lexical/104-identifiers.md) 의 형식을 따릅니다. 결과는 그 필드의 값이 됩니다.

### tuple `struct` 와 튜플의 위치 접근

```rok
let pair = Pair(1i32, 2i32);
let first = pair.0;
let second = pair.1;
```

점 뒤에 10진 자릿수가 오면 위치 인덱스로 해석되는데, 이는 [숫자 리터럴](../001-lexical/106-numeric-literals.md) 의 `1.foo` 규칙과 동일한 토큰화 결과입니다. 예를 들어 `pair.0` 의 토큰화는 식별자 `pair`, `.`, 정수 `0` 의 세 토큰으로 이루어지며, 파서가 이를 위치 접근으로 해석합니다.

### 정적 결정과 동적 결정

좌변의 정적 타입이 알려진 경우에는 컴파일 타임에 필드 위치가 결정되어 직접 메모리 접근으로 컴파일됩니다. 반면 좌변이 [`any` 슬롯](../002-types/202-any-slot.md) 인 경우에는 런타임에 태그를 검사한 뒤 적절한 필드 위치를 찾는 동적 디스패치가 일어납니다.

## 3.5.2. 메서드 호출: `obj.method(args)`

```rok
let s = "hello";
let n = s.len();
let p = Point { x: 1.0, y: 2.0 };
let d = p.distance(other);
```

형식은 `expr . 메서드이름 ( 인자들 )` 의 모양을 갖습니다. 메서드 해소 (resolution) 는 좌변의 정적 타입을 보고, 그 타입에 정의된 메서드 또는 그 타입이 충족하는 트레잇의 메서드를 찾아 이루어집니다.

### 메서드 해소 순서

같은 이름의 메서드가 여러 곳에 정의되어 있을 수 있습니다. 이때 컴파일러는 다음 순서로 후보를 찾습니다.

1. 고유 `impl` ([`impl`](../002-types/209-impl.md)) 에 정의된 메서드를 먼저 살핍니다.
2. 그 다음으로 트레잇 `impl` 의 메서드를 살피며, 이는 좌변의 타입이 충족하는 트레잇이 현재 모듈에서 가시한 경우에 한정됩니다. 가시성은 자동 가시인 `rok::**` 의 트레잇, `use` 로 가져온 트레잇, 또는 같은 모듈에서 정의된 트레잇으로부터 비롯됩니다.

여러 후보가 동시에 존재하면 모호하다는 컴파일 에러가 발생합니다. 이 경우 사용자가 [타입 경로](../002-types/213-type-paths.md) 의 완전 자격 형태로 명시 해소해 주어야 합니다.

### 메서드 receiver 의 `ref` 자동

메서드 호출 자리의 수신자 (`obj`) 는 메서드의 `self` 종류 (`self` / `ref self`) 에 따라 자동으로 처리됩니다. 따라서 호출 시점에 `ref` 를 명시하지 않아도 됩니다.

```rok
p.distance(other);          // self
p.replace_with(other);      // ref self — 호출자가 'ref' 명시하지 않음
```

이는 일반 함수 호출의 `ref` 명시 규칙과는 다른, 메서드 receiver 에 적용되는 특수 규칙입니다. 자세한 내용은 함수 사양에서 정의됩니다.

### 메서드 호출의 desugar

```
obj.method(a, b)
```

위 표현은 의미적으로 다음과 동등합니다.

```
<typeof(obj) as TraitContaining_method>::method(obj, a, b)
```

(또는 고유 `impl` 에 정의된 경우라면 `Type::method(obj, a, b)` 가 됩니다.)

이 desugar 는 [연산자 디스패치](301-operator-dispatch.md) 의 트레잇 메서드 매핑과 동일한 메커니즘을 따릅니다. `.` 토큰이 `#[infix(".")]` 식의 매핑을 직접 갖는 것은 아니지만, 결과 호출의 형태는 같습니다.

### turbofish: 제네릭 메서드의 명시

[타입 경로](../002-types/213-type-paths.md) 의 turbofish 형태를 통해 메서드의 제네릭 인자를 명시할 수 있습니다.

```rok
let v = expr.parse::<i32>();
let m = arr.collect::<List<f64>>();
```

`.method::<...>(args)` 형태로 적으며, `.` 뒤에 메서드 이름이 오고, 그 뒤에 turbofish 가 따르며, 마지막에 인자 괄호가 옵니다.

## 3.5.3. 정적 항목 접근과의 차이

`Type::item` ([타입 경로](../002-types/213-type-paths.md)) 은 타입에 부속된 항목의 접근을 의미합니다.

```
Type::item       // 정적 메서드 / 연관 const / 연관 타입 — 인스턴스 없이
obj.item         // 인스턴스에 부속된 멤버 / 메서드 — 인스턴스 필요
```

이 두 형태는 완전히 다른 종류의 호출입니다. 토큰도 `::` 와 `.` 로 다르고, 의미 또한 서로 다릅니다.

### 메서드 경로 값

괄호 없이 적은 메서드 경로 (`T::method`, `Trait::method`, `<T as Tr>::method`, `i32::add`, `S::new`) 는 함수 값입니다.

| 메서드 종류 | 함수 값의 타입                         |
|-------------|----------------------------------------|
| 정적 메서드 | `fn(args) -> R`                        |
| 인스턴스 메서드 | `fn(Self, args) -> R` (`self` 가 첫 인자) |

```rok
let mk = Point::new;               // fn(f64, f64) -> Point
let add = i32::add;                // fn(i32, i32) -> i32
xs.map(Point::norm);               // fn(Point) -> f64
```

- `ref self` 나 `ref` 매개변수를 가진 메서드는 값이 될 수 없습니다 ([4.7.6](../004-functions/407-ref-keyword.md#476-ref-의-타입-측면)).
- throwing 메서드는 throwing 함수 타입 (`fn(..) throws (E) -> R`) 을 줍니다.
- `Tr::m` 이나 `Self` 가 정해지지 않은 제네릭 타입의 메서드는 기대 타입에서 함수 타입을 정합니다. 기대 타입이 없으면 이후의 호출이 정하며 (`let f = Box::get; f(b)`, [2.10.2](../002-types/210-generics.md#타입-인자-추론)), 그래도 정해지지 않으면 "type annotations needed" 에러입니다.
- 제네릭 메서드의 타입 인자는 turbofish 나 기대 함수 타입에서 정합니다.
- `#[intrinsic]` 메서드도 값이 됩니다.

`obj.method` 처럼 인스턴스에 붙인 형태는 함수 값이 아닙니다 (3.5.4).

## 3.5.4. `.method` (호출 없이)

`.method` 만 적고 괄호를 두지 않는 경우, rok 은 이를 함수 값으로 해석하지 않습니다. 부분 적용이나 메서드 참조와 같은 별도의 의미는 부여되어 있지 않습니다.

```
let f = obj.method;        // ERROR: 메서드 호출이 아니라 식이 미완성
let f = || obj.method();   // 명시적 closure 로 감싸기
```

이러한 결정은 암묵적 부분 적용에서 비롯되는 모호함을 차단하기 위한 것입니다. 사용자가 의도를 직접 명시하도록 합니다.

## 3.5.5. 안전한 멤버 접근: `?.`

값이 없을 수도 있는 경우의 점 접근은 옵셔널 체이닝으로 적습니다. 이에 대해서는 [옵셔널 체이닝과 null 합체](308-optional-chaining-and-coalescing.md) 에서 정의됩니다.
