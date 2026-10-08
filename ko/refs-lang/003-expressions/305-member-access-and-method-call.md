# 3.5. 멤버 접근과 메서드 호출

`.` 토큰 ([구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)) 은 값에 부속된 항목에 접근하는 두 가지 형태에 쓴다.

```
member_expr     ::= primary "." identifier
                  | primary "." int_lit               // tuple position
method_call     ::= primary "." identifier ("::" "<" type_expr ("," type_expr)* ">")?
                    "(" arg_list? ")"
arg_list        ::= expr ("," expr)* ","?
```

## 3.5.1. 멤버 접근: `obj.field`

```nok
let p = Point { x: 1.0f64, y: 2.0f64 };
let xv = p.x;               // 1.0
let yv = p.y;               // 2.0
```

- `obj` 는 임의의 표현식.
- `field` 는 [식별자](../001-lexical/104-identifiers.md) 의 형식.
- 결과는 그 필드의 값.

### tuple `struct` 와 튜플의 위치 접근

```nok
let pair = Pair(1i32, 2i32);
let first = pair.0;
let second = pair.1;
```

- 점 뒤가 10진 자릿수면 위치 인덱스다 ([숫자 리터럴](../001-lexical/106-numeric-literals.md) 의 `1.foo` 규칙과 같은 토큰화 결과).
- `pair.0` 의 토큰화는 식별자 `pair`, `.`, 정수 `0` 으로 세 토큰이며, 파서가 위치 접근으로 해석한다.

### 정적 결정과 동적 결정

- 좌변의 정적 타입이 알려진 경우. 컴파일 타임에 필드 위치가 결정되어 직접 메모리 접근으로 컴파일된다.
- 좌변이 [`any` 슬롯](../002-types/202-any-slot.md) 인 경우. 런타임에 태그를 검사한 뒤 적절한 필드 위치를 찾는다 (동적 디스패치).

## 3.5.2. 메서드 호출: `obj.method(args)`

```nok
let s = "hello";
let n = s.len();
let p = Point { x: 1.0, y: 2.0 };
let d = p.distance(other);
```

- 형식: `expr . 메서드이름 ( 인자들 )`.
- 메서드 해소 (resolution) 는 좌변의 정적 타입을 보고, 그 타입에 정의된 메서드 또는 그 타입이 충족하는 트레잇의 메서드를 찾는다.

### 메서드 해소 순서

같은 이름의 메서드가 여러 곳에 정의되어 있을 수 있다. 컴파일러는 다음 순서로 후보를 찾는다.

1. 본질 `impl` ([`impl`](../002-types/209-impl.md)) 에 정의된 메서드.
2. 트레잇 `impl` 의 메서드. 좌변의 타입이 충족하는 트레잇.
3. 점진 대상의 자동 가시 prelude.

여러 후보가 동시에 있으면 모호하다는 컴파일 에러가 난다. 사용자가 [타입 경로](../002-types/213-type-paths.md) 의 완전 자격 형태로 명시 해소해야 한다.

### 메서드 receiver 의 `ref` 자동

메서드 호출 자리의 수신자 (`obj`) 는 메서드의 `self` 종류 (`self` / `ref self`) 에 따라 자동 처리된다. 호출 시점에 `ref` 를 명시하지 않아도 된다.

```nok
p.distance(other);          // self
p.replace_with(other);      // ref self — 호출자가 'ref' 명시하지 않음
```

이는 일반 함수 호출의 `ref` 명시 규칙과는 다른, 메서드 receiver 의 특수 규칙이다. 함수 사양에서 자세히 정의한다.

### 메서드 호출의 desugar

```
obj.method(a, b)
```

는 의미적으로 다음과 동등하다.

```
<typeof(obj) as TraitContaining_method>::method(obj, a, b)
```

(또는 본질 `impl` 에 정의된 경우 `Type::method(obj, a, b)`.)

이 desugar 는 [연산자 디스패치](301-operator-dispatch.md) 의 트레잇 메서드 매핑과 같은 메커니즘이다. `.` 토큰이 `#[infix(".")]` 식의 매핑을 갖는 것은 아니지만, 결과 호출의 형태가 같다.

### turbofish: 제네릭 메서드의 명시

[타입 경로](../002-types/213-type-paths.md) 의 turbofish 형태로 메서드의 제네릭 인자를 명시할 수 있다.

```nok
let v = expr.parse::<i32>();
let m = arr.collect::<List<f64>>();
```

`.method::<...>(args)` 의 형태로, `.` 뒤에 메서드 이름, 그 뒤에 turbofish, 그 뒤에 인자 괄호가 온다.

## 3.5.3. 정적 항목 접근과의 차이

`Type::item` ([타입 경로](../002-types/213-type-paths.md)) 은 타입에 부속된 항목의 접근이다.

```
Type::item       // 정적 메서드 / 연관 const / 연관 타입 — 인스턴스 없이
obj.item         // 인스턴스에 부속된 멤버 / 메서드 — 인스턴스 필요
```

두 형태는 완전히 다른 종류의 호출이다. 토큰도 다르고 (`::` vs `.`), 의미도 다르다.

## 3.5.4. `.method` (호출 없이)

`.method` 만 적고 괄호를 두지 않으면, nok 은 이를 함수 값으로 해석하지 않는다. 부분 적용이나 메서드 참조 같은 별도 의미가 없다.

```
let f = obj.method;        // ERROR: 메서드 호출이 아니라 식이 미완성
let f = || obj.method();   // 명시적 closure 로 감싸기
```

이 결정은 암묵적 부분 적용의 모호함을 차단한다. 사용자가 의도를 명시한다.

## 3.5.5. 안전한 멤버 접근: `?.`

값이 없을 수도 있는 경우의 점 접근은 옵셔널 체이닝으로 적는다. [옵셔널 체이닝과 null 합체](308-optional-chaining-and-coalescing.md) 에서 정의한다.
