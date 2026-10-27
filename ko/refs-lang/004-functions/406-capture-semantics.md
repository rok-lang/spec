# 4.6. 캡처 의미론

[클로저](405-closure.md) 는 외부 환경의 변수를 캡처할 수 있으며, 본 장에서는 캡처가 값을 복사하는지 아니면 참조를 잡는지를 결정하는 규칙을 정의합니다.

## 4.6.1. 기본: 값 복사

> [!IMPORTANT]
> 캡처의 기본 의미는 **값 복사**입니다. 클로저가 생성되는 시점에 외부 변수의 값을 스냅샷으로 떠 둡니다.

```rok
let count = 0;
let f = || { count + 1 };         // count 의 값 0 이 복사됨
count = 10;
f();                               // 1 (f 는 0 을 가짐)
```

이 결정에는 세 가지 동기가 있습니다. 먼저 명시성으로, 캡처가 어떤 모드인지 코드 표면에 분명히 드러납니다. 다음으로 부작용 차단으로, 클로저가 외부 변수를 조용히 변경해 버리는 함정을 막습니다. 마지막으로 과학·수치 코드 친화성으로, Julia 가 가진 100배 박싱 함정 (참조 캡처가 동적 슬롯에 박싱되어 성능이 저하되는 문제) 을 회피합니다.

```
capture_list    ::= "[" capture_item ("," capture_item)* ","? "]"
capture_item    ::= "ref" identifier | identifier
```

## 4.6.2. 참조 캡처: `[ref x]` 캡처 리스트

외부 변수의 바인딩에 대한 참조를 캡처하려면 명시적인 캡처 리스트를 둡니다.

```rok
let count = 0;
let inc = [ref count] || { count = count + 1 };
let dec = [ref count] || { count = count - 1 };

inc(); inc(); dec();
// count = 1
```

형태는 클로저 정의 앞에 `[ref 변수, ref 변수, ...]` 처럼 둡니다. 리스트 안의 변수는 공유 셀에 담기며 (boxing), 바깥 스코프와 클로저가 같은 셀을 쓰므로 클로저 안에서 재할당하면 외부 변수까지 반영됩니다. 셀은 GC 객체라서 클로저가 변수의 스코프보다 오래 살아도 됩니다. 리스트에 들어 있지 않은 변수는 일반 값 캡처로 처리됩니다.

### 다중 캡처와 혼합 캡처

```rok
let counter = 0;
let config = load_config();

let handler = [ref counter] || {
    counter = counter + 1;        // ref: 외부 변수에 반영
    print(config.version);         // 값: 캡처 시점의 config 스냅샷
};
```

캡처 리스트에 명시한 변수만 `ref` 캡처가 되고, 나머지는 자동으로 값 캡처로 처리됩니다.

## 4.6.3. 캡처 리스트의 토큰화

`[ref x] || body` 의 토큰화는 다음과 같이 이루어집니다. `[`, `ref`, `x`, `]` 는 대괄호 안의 캡처 리스트가 되고, `||` 또는 `|x|` 는 클로저의 시작을 나타냅니다.

[구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md) 의 최장 일치 규칙과 문맥을 통해 모호함이 제거됩니다. `[...]` 뒤에 `||` 나 `|x|` 가 바로 이어진다면 캡처 리스트로 해석되고, 그렇지 않으면 일반 인덱싱 또는 배열 리터럴로 해석됩니다.

```rok
let xs = [1, 2, 3];               // List 리터럴
let f = [ref count] || { ... };   // 캡처 리스트 + 클로저
arr[ref count]                     // ERROR: 인덱싱에 ref 못 씀
```

## 4.6.4. 본체에서 외부 변수 재할당 시도 → `ref` 강제

값으로 캡처된 외부 변수를 클로저 본체에서 재할당하려고 시도하면 컴파일 에러가 발생합니다.

```rok
let count = 0;
let f = || { count = count + 1 };
// ERROR: 'count' is value-captured.
//        To mutate the outer variable, add `[ref count]` capture list.
```

이 강제는 사용자가 의도적으로 `ref` 캡처를 명시하도록 안내하기 위함입니다.

## 4.6.5. `ref` 단축 (없음)

```rok
ref || { ... }                    // 채택 안 함
```

전체를 `ref` 캡처하는 단축 형태는 도입하지 않습니다. 캡처는 항상 명시적인 리스트를 통해서만 `ref` 가 될 수 있으며, 어떤 변수가 참조로 잡히는지를 코드에 분명히 드러내기 위한 결정입니다.

## 4.6.6. 루프 변수 자동 처리

값 캡처가 기본이기 때문에, 루프 변수가 반복마다 자체 값을 갖는 동작이 자연스럽게 보장됩니다.

```rok
let fns: List<fn() -> i32> = [];
for i in 0i32..5i32 {
    fns.push(|| i);               // 각 반복에서 i 값 복사
}
fns[0]();                          // 0
fns[4]();                          // 4
```

Go 1.22 의 per-iteration scope 같은 별도의 규칙을 둘 필요가 없으며, 값 캡처가 자동으로 의도한 의미를 만들어 줍니다.

## 4.6.7. 구조체와 참조 핸들의 캡처

값 캡처는 값을 클로저 안으로 복사합니다. [`struct`](../002-types/203-struct.md) 는 값 타입이므로 클로저는 자기 사본을 가집니다. 클로저 안의 필드 쓰기는 바깥 변수에 닿지 않으며, 그러려면 `[ref p]` 를 씁니다.

컬렉션 (`List`, `Dict`, `Set`, `StrBuf`, ...) 은 참조 핸들입니다 ([stdlib 2.9](../../refs-standard-libraries/200-core/209-collections.md#참조-핸들)). 값 캡처는 핸들을 복사하므로 객체는 공유됩니다.

```rok
let xs = [1, 2, 3];
let f = || { xs.push(4) };        // 핸들 복사 (같은 List)
f();
// xs == [1, 2, 3, 4]

let p = Point { x: 0.0, y: 0.0 };
let g = [ref p] || { p.x = 1.0 }; // ref 캡처: 바깥 p 에 씀
g();
// p.x == 1.0
```

컬렉션의 독립된 사본이 필요하면 `.clone()` 을 호출합니다.

## 4.6.8. `ref` 캡처 vs 객체 변경의 차이

```rok
let xs = [1, 2, 3];

// 값 캡처: 객체 변경은 OK, 바인딩 재할당은 안 됨
let f = || {
    xs.push(4);                   // OK: 객체 변경
    // xs = [];                    // ERROR: 바인딩 재할당
};

// ref 캡처: 바인딩 재할당까지 가능
let g = [ref xs] || {
    xs = [];                      // OK: 외부 xs 가 [] 가 됨
};
```

참조 핸들은 객체가 공유되므로 값 캡처와 객체 변경이 함께 가능합니다. 반면 `ref` 캡처는 바인딩 자체를 재할당해야 할 때 사용합니다.

이 구분은 [`impl`](../002-types/209-impl.md) 의 `self` 와 `ref self` 모델과 동일합니다.

## 4.6.9. 병렬과 동시성 문맥의 추가 제약

`fn() + Send` 클로저 (`task::spawn`, `par::for_each`, `thread::spawn` 등에 넘기는 것) 의 캡처는 모두 `Send + Sync` 여야 합니다 ([4.4](404-function-type.md#send--sync-bound)). `[ref x]` 셀은 클로저와 그것을 만든 코드가 함께 쓰므로, `x` 의 타입과 무관하게 결코 `Send` 도 `Sync` 도 아닙니다: `[ref]` 캡처가 있는 클로저 (또는 그런 클로저를 캡처하는 클로저, `async [ref x] ||` 클로저가 만든 future) 는 어떤 `+ Send` / `+ Sync` 자리에도 들어가지 않습니다. 같은 스레드 안의 사용 (`spawn_local`, 일반 호출) 은 영향이 없습니다. 컬렉션은 `!Sync` 이므로 ([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---스레드-경계-marker)) 값 캡처로도 이렇게 넘길 수 없습니다.

```rok
let buf: List<i32> = [];
par::for_each(xs, [ref buf] |x| { buf.push(x) });
// ERROR: a [ref buf] cell cannot cross threads
```

대신 reduce 를 쓰거나 (`par::reduce(xs, 0, |a, b| a + b)`), `Mutex` / `RwLock` / `Atomic` / 채널로 공유하거나, 스레드에서 값을 돌려받습니다 (`join()`).

이렇게 공유한 값은 새 소유자의 것입니다: `Mutex::new(buf)` 나 `tx.send(buf)` 뒤에 `buf` 를 쓰면 컴파일 에러입니다 ([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---스레드-경계-marker)). lock 을 통해 쓰거나 `buf.clone()` 을 넘깁니다.

```rok
let m = Mutex::new(buf);
buf.push(1);
// ERROR: `buf` was moved into the `Mutex`; use it through the lock: using g = m.lock() { g.get().push(1) }
```

자세한 병렬·동시성 규칙은 표준 라이브러리 사양이 정의하며, 본 장은 언어 수준의 캡처 메커니즘까지만 다룹니다.

## 4.6.10. 캡처 변수의 가시 범위

캡처된 변수는 클로저 값이 살아 있는 동안 함께 유지됩니다. 클로저가 외부 스코프를 벗어나더라도 캡처된 값은 클로저 객체 안에 보존됩니다.

```rok
fn make_counter() -> fn() -> i32 {
    let count = 0;
    [ref count] || { count = count + 1; count }
    // count 는 ref 로 캡처됨. 클로저가 사라질 때까지 유지
}

let c = make_counter();
c();                               // 1
c();                               // 2
c();                               // 3
```

위 예시에서 `count` 는 외부 함수가 종료된 뒤에도 클로저 객체에 부속되어 살아 있습니다. [GC](../000-meta/007-glossary.md#gc-garbage-collector) 가 이 수명을 관리합니다 ([7. 런타임 모델](../007-runtime/README.md)).
