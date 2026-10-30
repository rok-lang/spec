# 4.6. 캡처 의미론

[클로저](405-closure.md)는 바깥 환경의 변수를 캡처할 수 있습니다. 이 절은 캡처가 값을 복사하는지, 참조를 잡는지를 정하는 규칙을 정의합니다.

## 4.6.1. 기본: 값 복사

> [!IMPORTANT]
> 캡처의 기본 의미는 **값 복사**입니다. 클로저를 만드는 시점에 바깥 변수의 값을 스냅샷으로 떠 둡니다.

```rok
let count = 0;
let f = || { count + 1 };         // count 의 값 0 이 복사됨
count = 10;
f();                               // 1 (f 는 0 을 가짐)
```

이렇게 정한 이유는 세 가지입니다. 첫째, 명시성입니다. 캡처 모드가 코드에 분명히 드러납니다. 둘째, 부작용 차단입니다. 클로저가 바깥 변수를 몰래 바꾸는 함정을 막습니다. 셋째, 과학·수치 코드에 맞습니다. Julia의 '100배 박싱 함정'을 피합니다. 참조 캡처가 동적 슬롯에 박싱되어 성능이 떨어지는 문제입니다.

```
capture_list    ::= "[" capture_item ("," capture_item)* ","? "]"
capture_item    ::= "ref" identifier | identifier
```

## 4.6.2. 참조 캡처: `[ref x]` 캡처 리스트

바깥 변수의 바인딩을 참조로 캡처하려면 명시적인 캡처 리스트를 둡니다.

```rok
let count = 0;
let inc = [ref count] || { count = count + 1 };
let dec = [ref count] || { count = count - 1 };

inc(); inc(); dec();
// count = 1
```

클로저 정의 앞에 `[ref 변수, ref 변수, ...]`처럼 둡니다. 리스트 안의 변수는 공유 셀에 담깁니다(boxing). 바깥 스코프와 클로저가 같은 셀을 쓰므로 클로저 안에서 재할당하면 바깥 변수에도 반영됩니다. 셀은 GC 객체여서 클로저가 변수의 스코프보다 오래 살아도 됩니다. 리스트에 없는 변수는 일반 값 캡처입니다.

### 다중 캡처와 혼합 캡처

```rok
let counter = 0;
let config = load_config();

let handler = [ref counter] || {
    counter = counter + 1;        // ref - 외부 변수에 반영
    print(config.version);         // 값 - 캡처 시점의 config 스냅샷
};
```

캡처 리스트에 명시한 변수만 `ref` 캡처가 되고, 나머지는 자동으로 값 캡처입니다.

## 4.6.3. 캡처 리스트의 토큰화

`[ref x] || body`는 다음과 같이 토큰화합니다. `[`, `ref`, `x`, `]`는 대괄호 안의 캡처 리스트가 되고, `||` 또는 `|x|`는 클로저의 시작입니다.

[구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)의 최장 일치 규칙과 문맥으로 모호함을 없앱니다. `[...]` 바로 뒤에 `||`나 `|x|`가 오면 캡처 리스트로 해석하고, 그렇지 않으면 일반 인덱싱이나 배열 리터럴로 해석합니다.

```rok
let xs = [1, 2, 3];               // List 리터럴
let f = [ref count] || { ... };   // 캡처 리스트 + 클로저
arr[ref count]                     // ERROR - 인덱싱에 ref 못 씀
```

## 4.6.4. 본체에서 외부 변수 재할당 시도 → `ref` 강제

값으로 캡처한 바깥 변수를 클로저 본체에서 재할당하려고 하면 컴파일 에러입니다.

```rok
let count = 0;
let f = || { count = count + 1 };
// ERROR: 'count' is value-captured.
//        To mutate the outer variable, add `[ref count]` capture list.
```

이 규칙은 `ref` 캡처를 일부러 적도록 이끕니다.

## 4.6.5. `ref` 단축(없음)

```rok
ref || { ... }                    // 채택 안 함
```

모든 변수를 `ref` 캡처하는 단축 형태는 두지 않습니다. 캡처는 명시적인 리스트로만 `ref`가 될 수 있습니다. 어느 변수가 참조로 잡히는지 코드에 분명히 드러내기 위해서입니다.

## 4.6.6. 루프 변수 자동 처리

값 캡처가 기본이므로 각 반복의 루프 변수는 자기 값을 유지합니다.

```rok
let fns: List<fn() -> i32> = [];
for i in 0i32..5i32 {
    fns.push(|| i);               // 각 반복에서 i 값 복사
}
fns[0]();                          // 0
fns[4]();                          // 4
```

Go 1.22의 반복별 스코프(per-iteration scope) 같은 별도 규칙이 필요 없습니다. 값 캡처만으로 의도한 의미가 됩니다.

## 4.6.7. struct와 참조 핸들의 캡처

값 캡처는 값을 클로저 안으로 복사합니다. [`struct`](../002-types/203-struct.md)는 값 타입이므로 클로저 안에는 자기 사본이 있습니다. 클로저 안에서 필드에 써도 바깥 변수는 바뀌지 않습니다. 바깥 변수를 바꾸려면 `[ref p]`를 씁니다.

컬렉션(`List`, `Dict`, `Set`, `StrBuf` 등)은 참조 핸들입니다([stdlib 2.9](../../refs-standard-libraries/200-core/209-collections.md#참조-핸들)). 값 캡처는 핸들을 복사하므로 객체는 공유됩니다.

```rok
let xs = [1, 2, 3];
let f = || { xs.push(4) };        // 핸들 복사 - 같은 List
f();
// xs == [1, 2, 3, 4]

let p = Point { x: 0.0, y: 0.0 };
let g = [ref p] || { p.x = 1.0 }; // ref 캡처 - 바깥 p 에 씀
g();
// p.x == 1.0
```

컬렉션의 독립된 사본이 필요하면 `.clone()`을 호출합니다.

## 4.6.8. `ref` 캡처와 객체 변경의 차이

```rok
let xs = [1, 2, 3];

// 값 캡처 - 객체 변경은 OK, 바인딩 재할당은 안 됨
let f = || {
    xs.push(4);                   // OK - 객체 변경
    // xs = [];                    // ERROR - 바인딩 재할당
};

// ref 캡처 - 바인딩 재할당까지 가능
let g = [ref xs] || {
    xs = [];                      // OK - 외부 xs 가 [] 가 됨
};
```

참조 핸들은 객체를 공유하므로 값으로 캡처해도 객체를 바꿀 수 있습니다. `ref` 캡처는 바인딩 자체를 재할당해야 할 때 씁니다.

이 구분은 [`impl`](../002-types/209-impl.md)의 `self`와 `ref self` 모델과 같습니다.

## 4.6.9. 병렬과 동시성 문맥의 추가 제약

`fn() + Send` 클로저(`task::spawn`, `par::for_each`, `thread::spawn` 등에 넘기는 클로저)의 캡처는 모두 `Send + Sync`여야 합니다([4.4](404-function-type.md#send--sync-바운드)). `[ref x]` 셀은 클로저와 그 클로저를 만든 코드가 함께 씁니다. 그래서 이 셀은 `x`의 타입과 상관없이 `Send`도 `Sync`도 아닙니다. 따라서 `[ref]` 캡처가 있는 클로저는 어떤 `+ Send` / `+ Sync` 자리에도 들어가지 않습니다. 그런 클로저를 캡처한 클로저와 `async [ref x] ||` 클로저가 만든 future도 마찬가지입니다. 같은 스레드 안에서 쓸 때(`spawn_local`, 일반 호출)는 영향이 없습니다. 컬렉션은 `!Sync`이므로([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync-스레드-경계-마커)) 값 캡처로도 이렇게 넘길 수 없습니다.

```rok
let buf: List<i32> = [];
par::for_each(xs, [ref buf] |x| { buf.push(x) });
// ERROR: a [ref buf] cell cannot cross threads
```

대신 reduce를 쓰거나(`par::reduce(xs, 0, |a, b| a + b)`), `Mutex` / `RwLock` / `Atomic` / 채널로 공유하거나, 스레드에서 값을 돌려받습니다(`join()`).

이렇게 공유한 값은 새 소유자의 것입니다. `Mutex::new(buf)`나 `tx.send(buf)` 뒤에 `buf`를 쓰면 컴파일 에러입니다([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync-스레드-경계-마커)). 락을 거쳐 쓰거나 `buf.clone()`을 넘깁니다.

```rok
let m = Mutex::new(buf);
buf.push(1);
// ERROR: `buf` was moved into the `Mutex`; use it through the lock: using g = m.lock() { g.get().push(1) }
```

병렬·동시성 규칙의 자세한 내용은 표준 라이브러리 사양이 정의합니다. 이 절은 언어 수준의 캡처 메커니즘만 다룹니다.

## 4.6.10. 캡처 변수의 수명

캡처한 변수는 클로저 값이 살아 있는 동안 함께 남습니다. 클로저가 바깥 스코프를 벗어나도 캡처한 값은 클로저 객체 안에 남아 있습니다.

```rok
fn make_counter() -> fn() -> i32 {
    let count = 0;
    [ref count] || { count = count + 1; count }
    // count 는 ref 로 캡처됨 - 클로저가 사라질 때까지 유지
}

let c = make_counter();
c();                               // 1
c();                               // 2
c();                               // 3
```

위 예에서 `count`는 바깥 함수가 끝난 뒤에도 클로저 객체에 붙어 살아 있습니다. [GC](../000-meta/007-glossary.md#gc-garbage-collector)가 이 수명을 관리합니다([7. 런타임 모델](../007-runtime/README.md)).
