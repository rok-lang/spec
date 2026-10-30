# 3.4. 대입과 복합 대입

대입(`=`)과 복합 대입(`+=`, `-=` 등)은 변수, 필드, 인덱싱된 자리의 값을 갱신합니다. 둘 다 문장입니다. 단순 대입은 3.4.5의 형태로 표현식 자리에도 올 수 있지만, 복합 대입은 올 수 없습니다. 문법은 다음과 같습니다.

```
assign_stmt     ::= lvalue "=" expr ";"
compound_stmt   ::= lvalue compound_op expr ";"
assign_expr     ::= lvalue "=" expr                    // 괄호 안 또는 arm 본체 (3.4.5)
compound_op     ::= "+=" | "-=" | "*=" | "/=" | "%=" | "%%=" | "&=" | "|="
                  | "^=" | "<<=" | ">>="
lvalue          ::= identifier
                  | lvalue "." identifier
                  | lvalue "." int_lit                 // tuple position
                  | lvalue "[" expr "]"                // index
```

## 3.4.1. 단순 대입

단순 대입은 다음과 같은 모양입니다.

```rok
x = expr;
obj.field = expr;
arr[i] = expr;
```

좌변(lvalue)은 재할당할 수 있는 자리입니다. 우변(rvalue)에는 아무 표현식이나 옵니다. 결과 값의 타입은 unit(`()`)입니다. 대입은 보통 문장이고, 표현식 자리에 올 때(3.4.5) 결과는 unit입니다. 그래서 C의 `if (x = read())`처럼 거의 쓸 일이 없는 패턴은 unit 결과 때문에 막힙니다.

### 좌변(lvalue)의 종류

좌변에 올 수 있는 형태와 의미는 다음과 같습니다.

| 형태           | 예                | 의미                                       |
|----------------|-------------------|--------------------------------------------|
| 단순 변수      | `x = ...`         | 변수의 슬롯에 값을 씁니다                  |
| 필드 접근      | `obj.field = ...` | 객체의 필드에 값을 씁니다                  |
| 튜플 위치 접근 | `pair.0 = ...`    | tuple `struct` / 튜플의 위치에 값을 씁니다 |
| 인덱싱         | `arr[i] = ...`    | 인덱싱 자리에 값을 씁니다                  |

인덱싱 대입은 [인덱싱과 슬라이싱](306-indexing-and-slicing.md)의 쓰기 트레잇 호출로 풀립니다. 슬라이스는 좌변이 될 수 없으므로 `arr[1:3] = other;`는 거부됩니다([3.6](306-indexing-and-slicing.md)).

### 좌변이 될 수 없는 자리

다음은 좌변이 될 수 없습니다.

```
1 = 2;                    // 리터럴은 좌변 아님
foo() = 2;                // 호출의 결과는 일반적으로 좌변 아님
(x + 1) = 2;              // 산술 결과는 좌변 아님
(a, b) = (b, a);          // 튜플은 좌변 아님. let (a, b) = (b, a); 로 적음
```

컴파일러는 좌변이 될 수 있는지를 컴파일 타임에 형태만 보고 정합니다. 런타임 검사는 하지 않습니다.

## 3.4.2. 변수의 새 선언과 재할당

[키워드](../001-lexical/105-keywords.md) `let`의 유무로 다음 두 형태를 구분합니다.

```rok
let x = 1;       // 새 선언, 'let' 필수
x = 2;           // 재할당, 'let' 없이
```

새 변수를 만들려면 `let`을 붙여야 하고, 기존 변수의 슬롯에 다시 쓸 때는 `let` 없이 씁니다. 선언하지 않은 이름에 `let` 없이 대입하면 컴파일 에러입니다. 정확한 규칙은 [변수 선언](../005-modules/501-variable-declaration.md)에서 정의합니다.

```
counter = counter + 1;     // counter 가 선언되어 있어야 OK
coutner = ...;             // ERROR: `coutner` is not declared; a variable is created only by `let`
```

이 규칙은 오타가 몰래 새 변수를 만드는 함정(Julia와 Python에서 흔한 버그)을 막습니다.

## 3.4.3. 복합 대입

복합 대입은 연산과 대입을 한 형태로 합칩니다.

```rok
x += 1;            // x = x + 1 과 의미상 동일
x -= 1;
x *= 2;
arr[i] += 1;
```

[구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)의 복합 대입 토큰(`+=`, `-=`, `*=`, `/=`, `%=`, `%%=`, `&=`, `|=`, `^=`, `<<=`, `>>=`)은 모두 같은 패턴을 따릅니다. 복합 대입은 `;`로 끝나는 문장입니다. 값이 없으므로 표현식 자리에 올 수 없습니다(`let y = (x += 1);`은 파싱되지 않습니다).

### 풀어 쓰기와 좌변 평가

복합 대입은 다음과 같이 풀립니다. 타입이 `OpAssign` 트레잇을 구현하면 `a += b`는 그 트레잇을 호출하고, 구현하지 않으면 `a = a + b`입니다([stdlib 2.4.1](../../refs-standard-libraries/200-core/204-ops-assign.md)). 모든 연산자가 마찬가지입니다.

```
lvalue OP= rhs       →   OpAssign::op_assign(ref lvalue, rhs)   // 구현되어 있으면
                     →   lvalue = (lvalue OP rhs)               // 아니면
```

컴파일러는 좌변을 두 번 평가하지 않도록 보장합니다.

```rok
arr[expensive_index()] += 1;
// expensive_index() 는 한 번만 호출됨
```

좌변이 단순한 변수나 필드면 그대로 두 번 참조합니다. 좌변에 호출이나 계산이 들어 있으면 그 부분을 임시 변수에 묶어 한 번만 평가합니다. 이 보장 덕분에 부작용이 있는 인덱스나 메서드가 의도치 않게 두 번 호출되지 않습니다.

### 트레잇 매핑

각 복합 대입은 별도의 **Assign** 트레잇 패밀리에 대응합니다.

```
x += y       →   <typeof(x) as AddAssign<typeof(y)>>::add_assign(ref x, y)
x -= y       →   <typeof(x) as SubAssign<typeof(y)>>::sub_assign(ref x, y)
```

`AddAssign`, `SubAssign`, `MulAssign`, `DivAssign` 등이 있습니다.

이 트레잇은 제자리 갱신(in-place update)으로 구현해도 됩니다. 그러면 `x = x + y`보다 효율적으로 구현할 수 있습니다. 예를 들어 큰 컬렉션에 원소를 추가할 때 새 컬렉션을 만들지 않아도 됩니다.

`AddAssign` 구현은 선택입니다. 구현이 없으면 `x += y`는 `x = x + y`로 바뀝니다. `x + y` 자체에는 항상 `Add`가 필요합니다.

### 시그니처

`AddAssign` 등의 시그니처는 다음과 같은 모양입니다.

```rok
trait AddAssign<Rhs = Self> {
    #[infix("+=")]
    fn add_assign(ref self, rhs: Rhs);
}
```

다른 연산자처럼 `*Assign` 트레잇도 `#[infix("op=")]`로 등록합니다([3.1](301-operator-dispatch.md)). `ref self`는 좌변의 슬롯을 참조합니다. 반환 타입이 없으므로 결과는 unit입니다. `ref self`의 정확한 의미와 좌변 슬롯을 갱신한다는 것의 의미는 함수 사양에서 정의합니다.

## 3.4.4. 다중 대입

C와 Python 식의 연쇄 대입(`a = b = c = 1`)은 Rok에서 쓸 수 없습니다.

```
a = b = c = 1;             // ERROR
```

대입의 결과 값은 unit입니다. 그래서 `b = c = 1`의 결과는 unit이고, 전체는 의미 없는 `a = (unit)`이 됩니다. 대입을 나누어 쓰세요.

```rok
c = 1; b = 1; a = 1;
```

## 3.4.5. 표현식 안의 대입

단순 대입은 표현식 자리에 넣을 수 있고 그 결과는 unit입니다. 하지만 그렇게 쓸 이유는 거의 없습니다.

```rok
let x = (y = 1);           // x = unit (의미가 거의 없음)
```

이 패턴은 권장하지 않습니다. 도구나 린트가 경고할 수 있습니다.

문장 자리가 아닌 곳에서는 대입을 괄호 `(y = 1)`로 감싸거나 match 가지의 본체로 씁니다. 괄호 없는 `let x = y = 1;`은 쓸 수 없습니다. 복합 대입에는 표현식 형태가 없습니다.
