# 3.4. 대입과 복합 대입

대입 (`=`) 과 복합 대입 (`+=`, `-=` 등) 은 변수 / 필드 / 인덱싱된 자리의 값을 갱신하는 표현식이다.

```
assign_expr     ::= lvalue assign_op expr
assign_op       ::= "=" | "+=" | "-=" | "*=" | "/=" | "%=" | "&=" | "|="
                  | "^=" | "<<=" | ">>="
lvalue          ::= identifier
                  | lvalue "." identifier
                  | lvalue "." int_lit                 // tuple position
                  | lvalue "[" expr "]"                // index
                  | lvalue "[" range_expr "]"          // slice
```

## 3.4.1. 단순 대입

```nok
x = expr;
obj.field = expr;
arr[i] = expr;
```

- 좌변 (lvalue) 은 재할당 가능한 자리.
- 우변 (rvalue) 은 임의의 표현식.
- 결과 값은 단위 타입 (`()`).

대입 자체가 표현식이지만 결과가 unit 이라 보통 문장으로 쓴다. C 의 `if (x = read())` 같은 패턴은 사용 빈도가 낮으므로 unit 결과로 자연스럽게 차단된다.

### 좌변 (lvalue) 의 종류

| 형태           | 예                | 의미                                     |
| -------------- | ----------------- | ---------------------------------------- |
| 단순 변수      | `x = ...`         | 변수의 슬롯에 값을 쓴다                  |
| 필드 접근      | `obj.field = ...` | 객체의 필드에 값을 쓴다                  |
| 튜플 위치 접근 | `pair.0 = ...`    | tuple `struct` / 튜플의 위치에 값을 쓴다 |
| 인덱싱         | `arr[i] = ...`    | 인덱싱 자리에 값을 쓴다                  |
| 슬라이싱       | `arr[1:3] = ...`  | 슬라이스 영역에 값을 쓴다                |

인덱싱 / 슬라이싱 대입은 [인덱싱과 슬라이싱](306-indexing-and-slicing.md) 의 쓰기 트레잇으로 desugar 된다.

### 좌변이 될 수 없는 자리

```
1 = 2;                    // 리터럴은 좌변 아님
foo() = 2;                // 호출의 결과는 일반적으로 좌변 아님
(x + 1) = 2;              // 산술 결과는 좌변 아님
```

좌변이 될 수 있는지는 컴파일 타임에 검사되며, 형태 (모양) 만으로 결정된다. 런타임 검사는 없다.

## 3.4.2. 변수의 새 선언과 재할당

[키워드](../001-lexical/105-keywords.md) 의 `let` 키워드와 결합하여 다음 두 형태가 구분된다.

```nok
let x = 1;       // 새 선언, 'let' 필수
x = 2;           // 재할당, 'let' 없이
```

- 새 변수를 만들려면 `let`.
- 기존 변수의 슬롯에 다시 쓰려면 `let` 없이.

`let` 없이 선언되지 않은 이름에 대입하면 컴파일 에러다. 정확한 규칙은 [변수 선언](../005-modules/501-variable-declaration.md) 에서 정의한다.

```
counter = counter + 1;     // counter 가 선언되어 있어야 OK
coutner = ...;             // ERROR: 'coutner' not declared. Did you mean 'counter'?
```

이 강제는 오타가 새 변수를 만들어 버리는 함정 (Julia / Python 의 흔한 버그) 을 차단한다.

## 3.4.3. 복합 대입

```nok
x += 1;            // x = x + 1 과 의미상 동일
x -= 1;
x *= 2;
arr[i] += 1;
```

[구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md) 의 복합 대입 토큰 (`+=`, `-=`, `*=`, `/=`, `%=`, `&=`, `|=`, `^=`, `<<=`, `>>=`) 모두 같은 패턴을 따른다.

### desugar 와 좌변 평가

```
lvalue OP= rhs       →   lvalue = (lvalue OP rhs)
```

좌변이 두 번 평가되지 않도록 컴파일러가 보장한다.

```nok
arr[expensive_index()] += 1;
// expensive_index() 는 한 번만 호출됨
```

- 좌변이 순수한 변수나 필드면 그대로 두 번 참조한다.
- 좌변이 호출이나 계산을 포함하면 그 부분을 임시 변수로 묶어 한 번만 평가한다.

이 보장은 부작용이 있는 인덱스나 메서드가 의도치 않게 두 번 호출되는 함정을 차단한다.

### 트레잇 매핑

각 복합 대입은 별개의 _Assign_ 트레잇 패밀리로 매핑된다.

```
x += y       →   <typeof(x) as AddAssign<typeof(y)>>::add_assign(ref x, y)
x -= y       →   <typeof(x) as SubAssign<typeof(y)>>::sub_assign(ref x, y)
```

(`AddAssign`, `SubAssign`, `MulAssign`, `DivAssign`, ... 등.)

이 트레잇은 제자리 갱신 (in-place update) 으로 구현될 수 있어, `x = x + y` 보다 효율적인 구현이 가능하다. 예를 들어 큰 컬렉션의 원소 추가가 새 컬렉션 생성 없이 수행된다.

`Add` 와 `AddAssign` 두 트레잇 모두를 구현하는 것은 사용자의 자유이며, 한쪽만 구현된 경우 대응 표기를 쓰면 컴파일 에러다.

### 시그니처

`AddAssign` 등의 시그니처는 다음 모양을 갖는다.

```nok
trait AddAssign<Rhs = Self> {
    #[infix("+=")]
    fn add_assign(ref self, rhs: Rhs);
}
```

- `ref self` 로 좌변의 슬롯을 참조한다.
- 반환 타입 없음. 결과는 unit.

`ref self` 의 정확한 의미와 좌변 슬롯의 갱신 의미는 함수 사양에서 정의한다.

## 3.4.4. 다중 대입

C / Python 식의 연쇄 대입 (`a = b = c = 1`) 은 nok 에서 허용되지 않는다.

```
a = b = c = 1;             // ERROR
```

대입의 결과 값이 unit 이므로, `b = c = 1` 의 결과가 unit 이라 `a = (unit)` 이 되어 의미가 없다. 의도한 의미를 적으려면 분리한다.

```nok
c = 1; b = 1; a = 1;
```

또는 분해 대입을 활용한다.

## 3.4.5. 표현식 안에서의 대입

대입은 표현식이고 결과는 unit 이므로, 표현식 자리에 끼워 넣을 수 있지만 그렇게 할 가치는 거의 없다.

```nok
let x = (y = 1);           // x = unit (의미가 거의 없음)
```

이런 패턴은 권장되지 않으며, 도구 / 린터가 경고할 수 있다.
