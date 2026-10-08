# 5.1. 변수 선언

`let` 키워드 ([키워드](../001-lexical/105-keywords.md)) 로 새 변수를 선언한다. 이 문서는 그 형태와, 재할당과 새 선언을 구분하는 규칙을 정의한다.

## 5.1.1. 새 선언: `let`

```
let_stmt    ::= "let" identifier (":" type_expr)? "=" expr ";"
```

```nok
let x = 42i32;
let y: i32 = 10i32;
let z;                            // ERROR — 초기값 없음
let z: i32;                        // ERROR — 초기값 없음
```

- `let` + 이름 + (선택적 `: 타입`) + `=` + 초기값.
- 초기값은 필수다. 미초기화 변수는 도입하지 않는다.
- 타입을 적지 않으면 [`any` 슬롯](../002-types/202-any-slot.md) 이 된다.
- 타입을 적으면 그 타입의 정적 슬롯이 된다.

값이 없는 슬롯이 읽히는 함정 (Rust 의 borrow checker 가 차단하는 종류) 을 원천 차단한다. 초기값을 미리 정할 수 없으면 [합 타입](../002-types/206-sum-types.md) 의 `T | none` 으로 부재를 명시한다.

```nok
let cache: Data | none = none;     // "아직 없음" 을 명시
cache = compute();                  // 나중에 채움
```

## 5.1.2. 재할당: `let` 없이

이미 선언된 변수에 다른 값을 대입할 때는 `let` 을 빼고 적는다.

```nok
let counter: i32 = 0i32;
counter = counter + 1i32;         // OK — 재할당
counter = counter + 1i32;         // OK — 또 재할당
```

선언되지 않은 이름에 `let` 없이 대입하면 컴파일 에러다.

```nok
let counter: i32 = 0i32;
coutner = counter + 1i32;         // ERROR: 'coutner' not declared.
                                   //        Did you mean 'counter'?
```

Julia 와 Python 에서 오타가 새 변수를 만드는 함정을 차단한다.

## 5.1.3. 새 선언과 재할당의 결정

| 형태              | 의미                                    |
| ----------------- | --------------------------------------- |
| `let name = ...;` | 새 선언. 그 스코프에 새 바인딩.         |
| `name = ...;`     | 재할당. 기존 바인딩의 슬롯에 다시 쓰기. |

오타 시 컴파일 에러를 내어 명시성을 우선하기 위함이다.

## 5.1.4. 슬롯 모드 전환 금지

같은 스코프 안에서 같은 이름의 재선언은 컴파일 에러다.

```nok
let x = 1;                         // any 슬롯
let x: i32 = 5i32;                  // ERROR — 같은 스코프에 x 가 이미 있음
```

이 규칙은 슬롯 모드 (점진 / 정적) 전환도 차단한다. 변수의 슬롯 종류는 선언 시점에 고정된다.

## 5.1.5. 스코프: 블록 단위

변수의 가시 범위는 [블록 표현식](../003-expressions/309-block-expressions.md) 의 블록 단위다.

```nok
let y = 1;
if cond {
    y = "hi";                      // 외부 y 재할당 (같은 슬롯)
    let z = 2;                     // 블록 로컬 z 선언
}
// 여기서 z 는 접근 불가
```

- 블록 안의 `let` 은 그 블록 로컬의 새 바인딩이다.
- 블록 밖에서는 그 변수가 보이지 않는다.
- 외부 변수는 같은 이름의 안쪽 `let` 이 그 블록 안에서만 가린다.

## 5.1.6. 중첩 스코프: 별개 바인딩

같은 이름의 변수가 내부 블록에 새로 선언되면 별개 바인딩이 된다 (shadowing 이 아니다).

```nok
let a = 1;
{
    let a: i32 = 42i32;            // 블록 로컬 a — 외부 a 와 다른 슬롯
    a = 10i32;                     // 블록 안 a 재할당 (외부 a 영향 없음)
}
// 여기서 a = 1 (외부 a 그대로)
```

내부 블록의 `let` 은 완전히 새로운 바인딩을 만든다. 외부의 같은 이름 변수와는 독립된 슬롯이다.

블록을 벗어나면 내부 변수가 사라진다는 자연스러운 의미와 외부 변수 보호를 함께 살리려는 설계다.

### shadowing 과의 차이

Rust 식 shadowing 은 같은 스코프에서 같은 이름을 다시 선언할 수 있다. nok 은 같은 스코프 안에서는 재선언을 금지한다 (위의 슬롯 모드 전환 규칙과 결합).

내부 블록에서 같은 이름을 새로 선언하는 것은 shadowing 이 아니라 새 스코프의 별개 바인딩이다. 효과는 같아 (블록 안에서 외부 이름이 가려짐) 보여도 의미론은 다르다.

## 5.1.7. 타입 명시와 타입 추론

```nok
let a = 42i32;                     // a: any (값은 i32)
let b: i32 = 42i32;                // b: i32 (정적)
let c: any = 42i32;                // c: any (명시)
let d = if cond { 1i32 } else { 2i32 };   // d: any (값은 i32)
```

타입 명시가 있으면 그 타입의 정적 슬롯이다. 없으면 [`any` 슬롯](../002-types/202-any-slot.md) 이 된다.

식의 결과 타입은 슬롯 호환 검사에만 쓴다. [표현식 개요](../003-expressions/README.md) 의 분기 결과 규칙을 참조한다.

## 5.1.8. 동시 선언 (없음)

```
let a = 1, b = 2;                  // 채택 안 함
```

C 식 콤마로 여러 변수를 동시 선언하는 형태는 도입하지 않는다. 한 줄에 하나씩 적거나, [분해 대입](../003-expressions/315-destructuring.md) 의 튜플 분해를 쓴다.

```nok
let (a, b) = (1, 2);               // 분해 대입 — OK
let a = 1; let b = 2;              // 두 줄로 — OK
```

## 5.1.9. `let` 의 자리

`let` 은 문장 (statement) 이다 ([표현식 개요](../003-expressions/README.md) 의 변환 규칙).

- 모듈 수준: `let` 은 허용되지 않는다 (모듈 수준의 값 슬롯은 [`const`](502-const.md) 또는 [`static`](503-static.md)).
- 함수 본체 / 블록 안: OK.
- `if` / `match` / `loop` 등 제어 구조 본체: 그 본체가 블록이므로 OK.
- 식 자리: `let` 은 식이 아니라 문장이므로 식 자리에 직접 등장할 수 없다.

## 5.1.10. `let` 으로 선언된 변수의 수명

선언 시점부터 그 변수가 도입된 스코프가 끝날 때까지 살아있다.

`ref` 캡처 ([캡처 의미론](../004-functions/406-capture-semantics.md)) 로 클로저에 잡힌 변수는 클로저 객체가 살아있는 동안 추가로 보존된다. [GC](../000-meta/007-glossary.md#gc-garbage-collector) 가 관리하므로 사용자가 직접 해제할 필요는 없다.

## 5.1.11. `let` 의 좌변과 [분해 대입](../003-expressions/315-destructuring.md) 의 연결

`let` 좌변에 [패턴](../003-expressions/311-patterns.md) 의 반증 불가능 패턴을 둘 수 있다. [분해 대입 (destructuring)](../003-expressions/315-destructuring.md) 이 그것이다.

```nok
let (x, y) = compute();
let Point { x, y } = origin;
```

분해 대입의 의미는 [분해 대입 (destructuring)](../003-expressions/315-destructuring.md) 에서 정의한다. 각 부분 패턴이 해당 변수에 바인딩되며, 모두 한 스코프에 도입된다.
