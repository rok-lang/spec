# 4.6. 캡처 의미론

[클로저](405-closure.md) 는 외부 환경의 변수를 캡처할 수 있다. 본 문서는 캡처가 값을 복사하는지, 참조를 잡는지를 결정하는 규칙을 정의한다.

## 4.6.1. 기본: 값 복사

> [!IMPORTANT]
> 캡처의 기본 의미는 **값 복사** 다. 클로저가 생성되는 시점에 외부 변수의 값을 스냅샷한다.

```nok
let count = 0;
let f = || { count + 1 };         // count 의 값 0 이 복사됨
count = 10;
f();                               // 1 (f 는 0 을 가짐)
```

이 결정의 동기:

- 명시성. 캡처가 어떤 모드인지 코드에 분명히 드러난다.
- 부작용 차단. 클로저가 외부 변수를 조용히 변경하는 함정을 막는다.
- 과학·수치 코드 친화. Julia 의 100x 박싱 함정 (참조 캡처가 동적 슬롯에 박싱되어 성능 저하) 을 회피한다.

```
capture_list    ::= "[" capture_item ("," capture_item)* ","? "]"
capture_item    ::= "ref" identifier | identifier
```

## 4.6.2. 참조 캡처: `[ref x]` 캡처 리스트

외부 변수의 바인딩에 대한 참조를 캡처하려면 명시적 캡처 리스트를 둔다.

```nok
let count = 0;
let inc = [ref count] || { count = count + 1 };
let dec = [ref count] || { count = count - 1 };

inc(); inc(); dec();
// count = 1
```

- 클로저 정의 앞에 `[ref 변수, ref 변수, ...]` 형태.
- 리스트 안의 변수는 참조로 캡처되어, 클로저 안에서 재할당하면 외부 변수에 반영된다.
- 리스트에 없는 변수는 일반 값 캡처.

### 다중 캡처와 혼합 캡처

```nok
let counter = 0;
let config = load_config();

let handler = [ref counter] || {
    counter = counter + 1;        // ref — 외부 변수에 반영
    print(config.version);         // 값 — 캡처 시점의 config 스냅샷
};
```

캡처 리스트에 명시한 변수만 `ref` 캡처, 나머지는 자동 값 캡처.

## 4.6.3. 캡처 리스트의 토큰화

`[ref x] || body` 의 토큰화는:

- `[` `ref` `x` `]`: 대괄호 안의 캡처 리스트.
- `||` 또는 `|x|`: 클로저 시작.

[구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md) 의 최장 일치와 문맥으로 모호함을 막는다. `[...]` 뒤에 `||` 나 `|x|` 가 바로 오면 캡처 리스트, 그렇지 않으면 일반 인덱싱이나 배열 리터럴.

```nok
let xs = [1, 2, 3];               // 배열 리터럴
let f = [ref count] || { ... };   // 캡처 리스트 + 클로저
arr[ref count]                     // ERROR — 인덱싱에 ref 못 씀
```

## 4.6.4. 본체에서 외부 변수 재할당 시도 → `ref` 강제

값 캡처된 외부 변수를 클로저 본체에서 재할당 시도하면 컴파일 에러.

```nok
let count = 0;
let f = || { count = count + 1 };
// ERROR: 'count' is value-captured.
//        To mutate the outer variable, add `[ref count]` capture list.
```

이 강제는 사용자가 의도적으로 `ref` 캡처를 적도록 안내한다.

## 4.6.5. `ref` 단축 (없음)

```nok
ref || { ... }                    // 채택 안 함
```

전체 `ref` 캡처의 단축 형태는 도입하지 않는다. 캡처는 항상 명시적 리스트를 통해서만 `ref` 가 된다. 어떤 변수가 참조로 잡히는지 코드에 드러나도록 한다.

## 4.6.6. 루프 변수 자동 처리

값 캡처가 기본이라, 루프 변수의 반복마다 자체 값을 갖는 것이 자연스럽게 보장된다.

```nok
let fns = [];
for i in 0..5 {
    fns.push(|| i);               // 각 반복에서 i 값 복사
}
fns[0]();                          // 0
fns[4]();                          // 4
```

Go 1.22 의 per-iteration scope 같은 별도 규칙이 필요 없다. 값 캡처가 자동으로 의도된 의미를 만든다.

## 4.6.7. 힙 객체의 캡처

힙에 있는 객체 (`struct`, List, Dict 등) 는 바인딩 자체가 객체의 참조다. 값 캡처는 이 참조를 복사하는 것이며, 객체 자체는 공유된다.

```nok
let xs = [1, 2, 3];
let f = || { xs.push(4) };        // xs 의 참조를 캡처 — 객체 변경 가능
f();
// xs == [1, 2, 3, 4]
```

이 동작은 [`struct`](../002-types/203-struct.md) 와 컬렉션의 공유 참조 의미와 일관된다. 값 캡처가 의미하는 것은 바인딩 (포인터나 참조) 의 복사이지, 객체 자체의 복사가 아니다.

객체 자체를 복사하려면 명시적으로 `.clone()` 같은 메서드를 호출 (표준 라이브러리 결정).

## 4.6.8. `ref` 캡처 vs 객체 변경의 차이

```nok
let xs = [1, 2, 3];

// 값 캡처 — 객체 변경은 OK, 바인딩 재할당은 안 됨
let f = || {
    xs.push(4);                   // OK — 객체 변경
    // xs = [];                    // ERROR — 바인딩 재할당
};

// ref 캡처 — 바인딩 재할당까지 가능
let g = [ref xs] || {
    xs = [];                      // OK — 외부 xs 가 [] 가 됨
};
```

- 값 캡처 + 객체 변경. 항상 가능 (공유 참조에서 자연스럽게 따라오는 결과).
- `ref` 캡처. 바인딩 자체를 재할당하고 싶을 때.

이 구분은 [`impl`](../002-types/209-impl.md) 의 `self` 와 `ref self` 와 같은 모델이다.

## 4.6.9. 병렬과 동시성 문맥의 추가 제약

`task::spawn` 이나 `par::for_each` 같은 병렬 실행의 클로저에서는 `[ref x]` 캡처가 거부된다. 병렬 분기가 같은 외부 변수를 동시에 갱신하면 데이터 경쟁이 발생한다.

```nok
let buf = [];
par::for_each(xs, [ref buf] |x| { buf.push(x) });
// ERROR: ref capture rejected in parallel context.
```

해결 패턴:

- 함수형 reduce. `xs.par_iter().reduce(0, |acc, x| acc + x)`.
- 동시성 안전 wrapper. `Mutex`, `Atomic`, `Channel` 등 표준 라이브러리 타입.

자세한 병렬과 동시성 규칙은 표준 라이브러리 사양에서 정의한다. 본 문서는 언어 수준의 캡처 메커니즘까지만 다룬다.

## 4.6.10. 캡처 변수의 가시 범위

캡처된 변수는 클로저 값이 살아 있는 동안 유지된다. 클로저가 외부 스코프를 벗어나도 캡처된 값은 클로저 객체 안에 보존된다.

```nok
fn make_counter() -> fn() -> i32 {
    let count = 0;
    [ref count] || { count = count + 1; count }
    // count 는 ref 로 캡처됨 — 클로저가 사라질 때까지 유지
}

let c = make_counter();
c();                               // 1
c();                               // 2
c();                               // 3
```

위 예에서 `count` 는 외부 함수가 종료되어도 클로저 객체에 부속되어 살아 있다. [GC](../000-meta/007-glossary.md#gc-garbage-collector) 가 관리한다 ([7. 런타임 모델](../007-runtime/README.md)).
