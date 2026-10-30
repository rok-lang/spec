# 2.7. iter: 반복 트레잇

`rok::core::iter` 모듈은 `for x in xs`가 어떻게 풀리는지 정하는 **권위**입니다. 모든 `for` 루프는 이 장의 트레잇으로 디스패치됩니다.

[refs-lang 3.13 루프 표현식](../../refs-lang/003-expressions/313-loop-expressions.md)의 `for` 문법은 `IntoIterator::into_iter`와 `Iterator::next` 호출을 조합한 형태로 풀립니다.

## 2.7.1. `Iterator`: 단계 진행

첫 트레잇은 `Iterator`이며, 시그니처는 다음과 같습니다.

```rok
#[lang("iterator_trait")]
trait Iterator {
    type Item;

    fn next(ref self) -> Item | none;
}
```

동작은 단순합니다. `next`를 한 번 호출하면 다음 원소 하나를 반환합니다. 원소가 더 없으면 `none`을 돌려줍니다.

시그니처는 `ref self`입니다. 반복자(iterator) 자체가 **상태**를 들고 있고, 호출할 때마다 그 상태를 제자리에서 갱신하기 때문입니다. 반복자는 값입니다. 복사한 `ListIter`는 따로 진행하고, `next(ref self)`는 그 사본만 진행합니다.

### 사용 예(사용자 코드)

직접 호출하는 모습은 다음과 같습니다.

```rok
let xs: List<i32> = [10, 20, 30];
let it = xs.into_iter();         // IntoIterator 가 만든 iterator
let v1 = it.next();              //→ 10
let v2 = it.next();              //→ 20
let v3 = it.next();              //→ 30
let v4 = it.next();              //→ none
```

실제 코드에서는 보통 직접 호출하지 않고 `for`를 씁니다.

```rok
for x in xs {
    use(x);
}
// → 컴파일러가 다음으로 desugar:
//   let __it = xs.into_iter();
//   loop {
//       match __it.next() {
//           v: i32 => use(v),
//           none => break,
//       }
//   }
```

### 메서드: 풍부한 기본 impl

`Iterator`에서 직접 구현해야 하는 메서드는 `next(ref self)` 하나뿐입니다. 나머지 메서드는 모두 `next` 위에 쌓은 기본 구현입니다.

> [!NOTE]
> 아래 시그니처의 `fn(Item) -> U`는 [refs-lang 4.4 함수 타입](../../refs-lang/004-functions/404-function-type.md)이 정의하는 통합 호출 가능 타입입니다. 이 타입은 **함수와 클로저를 모두** 받습니다. 따라서 `it.map(some_fn)`도, `it.map(|x| x * 2)`도 같은 자리에 들어갑니다. Rust의 `fn`/`Fn`/`FnMut`/`FnOnce`와 달리 형태가 하나뿐입니다.

```rok
impl Iterator {
    fn next(ref self) -> Item | none;       // 사용자 정의

    // 변환 어댑터 (lazy)
    fn map<U>(self, f: fn(Item) -> U) -> Map<Self, U>;
    fn filter(self, pred: fn(Item) -> bool) -> Filter<Self>;
    fn enumerate(self) -> Enumerate<Self>;        // (index, item) 쌍
    fn zip<J: Iterator>(self, other: J) -> Zip<Self, J>;    // Item = (Item, J::Item)
    fn chain<I: Iterator<Item = Item>>(self, other: I) -> Chain<Self, I>;
    fn take(self, n: usize) -> Take<Self>;
    fn skip(self, n: usize) -> Skip<Self>;
    fn step_by(self, step: usize) -> StepBy<Self>;
    fn rev(self) -> Rev<Self>;                    // DoubleEndedIterator 만
    fn flatten(self) -> Flatten<Self> where Item: IntoIterator;
    fn flat_map<U: IntoIterator>(self, f: fn(Item) -> U) -> FlatMap<Self, U>;
    fn peekable(self) -> Peekable<Self>;          // 다음 원소 미리 보기
    fn take_while(self, pred: fn(Item) -> bool) -> TakeWhile<Self>;
    fn skip_while(self, pred: fn(Item) -> bool) -> SkipWhile<Self>;
    fn scan<S>(self, init: S, f: fn(S, Item) -> S) -> Scan<Self, S>;   // Item = S
    fn inspect(self, f: fn(Item)) -> InspectIter<Self>;    // f 가 각 원소를 봄
    fn cycle(self) -> Cycle<Self>;                // 끝없이 반복
    fn chunks(self, size: usize) -> Chunks<Self>;     // Item = List<Item>
    fn windows(self, size: usize) -> Windows<Self>;   // Item = List<Item>

    fn exact_len(self) -> usize | none;           // 진행 없이 아는 길이, 모르면 none

    // 종결 메서드 (eager)
    fn count(self) -> usize;
    fn sum(self) -> Item where Item: Add<Output = Item> + Zero;     // 비면 0
    fn product(self) -> Item where Item: Mul<Output = Item> + One;  // 비면 1
    fn fold<B>(self, init: B, f: fn(B, Item) -> B) -> B;
    fn try_fold<B, E>(self, init: B, f: fn(B, Item) throws (E) -> B) throws (E) -> B;
    fn try_for_each<E>(self, f: fn(Item) throws (E)) throws (E);
    fn partition(self, pred: fn(Item) -> bool) -> (List<Item>, List<Item>);
    fn unzip<A, B>(self) -> (List<A>, List<B>) where Self: Iterator<Item = (A, B)>;
    fn reduce(self, f: fn(Item, Item) -> Item) -> Item | none;
    fn any(self, pred: fn(Item) -> bool) -> bool;
    fn all(self, pred: fn(Item) -> bool) -> bool;
    fn find(self, pred: fn(Item) -> bool) -> Item | none;
    fn position(self, pred: fn(Item) -> bool) -> usize | none;
    fn min(self) -> Item | none where Item: Ord;
    fn max(self) -> Item | none where Item: Ord;
    fn min_by_key<K: Ord>(self, key: fn(Item) -> K) -> Item | none;
    fn max_by_key<K: Ord>(self, key: fn(Item) -> K) -> Item | none;
    fn collect<C: FromIterator<Item>>(self) -> C;
    fn for_each(self, f: fn(Item));               // side effect 한정
    fn last(self) -> Item | none;
    fn nth(ref self, n: usize) -> Item | none;
}
```

### `sum` / `product`와 `Zero` / `One`

`sum`은 `Zero::zero()`에서 시작해 `+`로 누적하고, `product`는 `One::one()`에서 시작해 `*`로 누적합니다. 따라서 빈 반복자의 합은 0, 곱은 1입니다. `Zero` / `One`은 `rok::core::num`의 트레잇입니다([2.1](201-numbers.md)).

### 멈춤, 누적 상태, 묶음

- `take_while(pred)`는 `pred`가 처음 거부하는 원소 전까지 내놓습니다. 그 원소는 안쪽 반복자에서 꺼낸 뒤 버립니다. 그 뒤로는 아무것도 꺼내지 않으므로 멈춘 지점 뒤의 부수 효과는 실행되지 않습니다. `skip_while(pred)`는 `pred`가 처음 거부하는 원소 전까지 버리고, 그 원소부터 그 뒤 원소를 검사 없이 모두 내놓습니다.
- `scan(init, f)`는 누적 상태를 내놓습니다. 먼저 `f(init, x0)`, 그다음 그 값과 `x1`로 계산한 `f`를 내놓는 식입니다. 단계마다 값을 내놓는 `fold`이므로 안쪽 반복자와 같은 개수를 내놓습니다.
- `inspect(f)`는 각 원소를 `f`에 넘긴 뒤 그대로 내놓습니다.
- `cycle()`은 다 돌 때마다 `cycle`이 받은 반복자의 복사본에서 다시 시작합니다. 그래서 반복자가 비어 있지 않으면 끝이 없고, 비어 있으면 아무것도 내놓지 않습니다. 제너레이터의 복사본은 한 위치를 공유하므로(2.7.10a) 제너레이터를 순환 반복자로 감싸도 원소가 반복되지 않습니다.
- `chunks(size)`는 `size`개씩 순서대로 묶은 리스트를 내놓고, 원소가 모자라면 마지막 것이 더 짧습니다. `windows(size)`는 연속한 `size`개를 하나씩 밀면서 리스트로 내놓습니다. 원소가 `size`개보다 적으면 아무것도 내놓지 않습니다. 둘 다 `size`가 0이면 panic이 일어납니다. `List`와 `Slice`에는 뷰를 내놓는 자체 `chunks` / `windows`가 있습니다([2.9.7](209-collections.md#297-slicet-뷰)).
- `try_fold`와 `try_for_each`는 에러를 던지는 `f`를 받습니다. 첫 에러에서 루프가 멈추고 그 뒤 원소는 방문하지 않습니다. 그 에러는 호출한 쪽으로 던집니다.
- `partition(pred)`는 `pred`가 받아들인 원소와 나머지를 두 리스트로 돌려줍니다. 두 리스트 모두 원래 순서를 지킵니다. `unzip()`은 쌍 `(A, B)`의 반복자를 `List<A>`와 `List<B>`로 나눕니다. 원소는 쌍이어야 합니다. `A`와 `B`는 `Item = (A, B)`에서 추론하고, 다른 형태의 원소는 호출 지점에서 에러입니다.

```rok
let xs = [1, 2, 3, 4];
let small: List<i32> = xs.iter().take_while(|x| x < 3).collect();   //→ [1, 2]
let sums: List<i32> = xs.iter().scan(0, |acc, x| acc + x).collect(); //→ [1, 3, 6, 10]
let pairs: List<List<i32>> = xs.iter().windows(2).collect();        //→ [[1, 2], [2, 3], [3, 4]]
let (even, odd) = xs.iter().partition(|x| x % 2 == 0);              //→ [2, 4], [1, 3]
let (ns, ss) = [(1, "a"), (2, "b")].iter().unzip();                 //→ [1, 2], ["a", "b"]
```

### lazy와 eager

이 메서드는 두 부류로 나뉩니다. `map`, `filter`, `take` 같은 변환 어댑터는 **지연(lazy)** 평가합니다. 새 반복자를 반환할 뿐 실제 계산은 하지 않습니다. `count`, `sum`, `collect`, `for_each` 같은 종결 메서드를 호출해야 비로소 진행이 시작됩니다.

```rok
let xs = [1, 2, 3, 4, 5];
let r = xs.into_iter()
    .map(|x| x * 2)            // lazy: 아직 계산 안 함
    .filter(|x| x > 4)         // lazy
    .sum();                    // eager: 여기서 단계 진행
//→ 6 + 8 + 10 = 24
```

### 사용자 정의 Iterator

반복자를 직접 정의하는 예입니다.

```rok
struct Counter { n: i32; }

impl Iterator for Counter {
    type Item = i32;

    fn next(ref self) -> i32 | none {
        if self.n < 5 {
            let cur = self.n;
            self.n = self.n + 1;
            cur
        } else {
            none
        }
    }
}

let c = Counter { n: 0 };
for x in c {
    rok_print_i32(x);          //→ 0, 1, 2, 3, 4
}
```

## 2.7.2. `IntoIterator`: `for`의 시작

다음은 `for` 루프의 출발점인 `IntoIterator` 트레잇입니다.

```rok
#[lang("into_iterator_trait")]
trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Item>;

    fn into_iter(self) -> IntoIter;
}
```

`IntoIterator`는 컬렉션에서 반복자를 만드는 어댑터입니다. `for x in xs`는 **xs 자체**를 반복자로 다루지 않습니다. `xs.into_iter()`로 반복자를 만든 뒤 그 반복자의 `next`를 호출합니다.

### 분리되어 있는 이유

두 트레잇을 나눈 이유는 두 관점이 근본적으로 다르기 때문입니다.

- 컬렉션 자체(List, Dict 등)는 상태 없는 컨테이너이므로 어떤 위치를 가리키는지 알지 못합니다.
- 반복자에는 진행 상태가 있어 어디까지 봤는지 추적합니다.

따라서 같은 컬렉션을 두 번 순회하려면 매번 새 반복자가 필요합니다. 그 반복자를 `into_iter`가 만듭니다.

### 자동 impl

`Iterator` 자체도 `IntoIterator`를 자동으로 구현합니다. 이때 `into_iter`는 self를 그대로 반환합니다.

```rok
// 모든 Iterator 는 자동으로 IntoIterator
impl<I: Iterator> IntoIterator for I {
    type Item = I::Item;
    type IntoIter = I;
    fn into_iter(self) -> I { self }
}
```

덕분에 `for x in iter`도 그대로 동작합니다. iter가 이미 반복자라면 `into_iter`가 self를 돌려주기 때문입니다.

### `for` 풀어 쓰기의 정확한 형태

`for` 루프가 풀리는 형태를 좀 더 정확히 적으면 다음과 같습니다.

```rok
for x in expr {
    body
}
```

위 코드는 다음과 같이 풀립니다.

```rok
let __it = expr.into_iter();
loop {
    match __it.next() {
        v: T => {
            let x = v;
            body;
        }
        none => break,
    }
}
```

자세한 규칙은 [refs-lang 3.13](../../refs-lang/003-expressions/313-loop-expressions.md)에 있습니다.

## 2.7.3. `Step`: 한 칸 진행 추상화

`Range<T>`의 순회는 "다음 값으로 한 칸 이동"이라는 단순한 동작에 기대고 있습니다. 이 동작은 일반 산술(`Add`)과 의미가 다릅니다. 예를 들어 `char`의 다음 값은 코드 포인트 + 1이지만 `char + char` 자체는 정의되지 않습니다. 그래서 Rok은 한 칸 진행을 별도의 트레잇으로 나눕니다.

```rok
trait Step: PartialOrd + Clone {
    fn forward_checked(self, count: usize) -> Self | none;
    fn backward_checked(self, count: usize) -> Self | none;
    fn steps_between(start: Self, end: Self) -> usize | none;
}
```

각 메서드의 의미는 다음과 같습니다.

- `forward_checked(self, n)`: `self`에서 `n`만큼 앞으로 이동한 값을 반환합니다. 타입 범위를 넘으면 `none`을 돌려줍니다.
- `backward_checked(self, n)`: 뒤로 이동한 값을 반환합니다. 마찬가지로 범위를 넘으면 `none`을 돌려줍니다.
- `steps_between(start, end)`: `start`에서 `end`까지 몇 칸인지 계산합니다. 표현할 수 없으면(예: 음수, 오버플로) `none`을 돌려줍니다.

### 산술 트레잇과 분리한 이유

`Step`을 `Add`에서 분리한 이유는 세 가지입니다.

- **의미 분리**: `+1`은 단순한 1 증가이지 임의 산술이 아닙니다. `MyVec3 + MyVec3`는 의미가 있지만 `Range<MyVec3>`는 의미가 없습니다. `Step`을 따로 구현하지 않으면 컴파일러가 거부합니다.
- **부동소수 배제**: `f32` / `f64`에는 `Step` 구현이 없습니다. 따라서 `Range<f64>`는 컴파일 에러입니다. `0.0..1.0`처럼 의미가 모호한 표기는 사양 수준에서 사라집니다.
- **효율적인 건너뛰기**: `Iterator::nth(n)`과 `step_by(n)`은 `forward_checked(_, n)`을 한 번 호출해 O(1)로 동작합니다. `succ`만 있다면 같은 동작이 O(n)입니다.

### Step impl 매트릭스

표준 라이브러리는 정수 원시 타입과 `char`에만 `Step`을 구현합니다.

| 타입                 | 구현     | 비고                                                 |
|----------------------|----------|------------------------------------------------------|
| `i8`..`i64`, `isize` | **있음** | 오버플로 시 `forward_checked`가 `none` 반환         |
| `u8`..`u64`, `usize` | **있음** | 언더플로 시 `backward_checked`가 `none` 반환        |
| `int`                | **없음** | 표준 라이브러리 구현 없음                            |
| `char`               | **있음** | 코드 포인트 진행, 서로게이트(`0xD800..=0xDFFF`) 건너뜀 |
| `bool`               | **없음** | 표준 라이브러리 구현 없음                            |
| `f32`, `f64`         | **없음** | `Range<f64>` 거부: 의미 모호                         |
| `float`              | **없음** | (위와 같음)                                          |

사용자 정의 타입에는 `impl Step for MyType`을 자유롭게 추가해도 됩니다. 그러면 바로 `Range<MyType>`을 쓸 수 있습니다.

```rok
enum Weekday { Mon, Tue, Wed, Thu, Fri, Sat, Sun }

impl Step for Weekday {
    fn forward_checked(self, n: usize) -> Weekday | none { ... }
    fn backward_checked(self, n: usize) -> Weekday | none { ... }
    fn steps_between(s: Weekday, e: Weekday) -> usize | none { ... }
}

for d in Weekday::Mon..=Weekday::Fri {
    schedule(d);     // 평일 5 일
}
```

## 2.7.4. `Range<T>` / `RangeInclusive<T>` / `RangeFrom<T>` / `RangeTo<T>` / `RangeToInclusive<T>` / `RangeFull`

[refs-lang 3.7 범위 표현식](../../refs-lang/003-expressions/307-range-expressions.md)의 `a..b` 문법은 이 표준 라이브러리 타입으로 풀립니다.

### `Range<T>`: `a..b`(끝 제외)

```rok
#[literal( expr ".." expr )]
#[lang("range_type")]
struct Range<T> {
    start: T;
    end: T;
}

impl<T: Step> Iterator for Range<T> {
    type Item = T;

    fn next(ref self) -> T | none {
        if self.start < self.end {
            let cur = self.start.clone();
            self.start = self.start.forward_checked(1)?;   // overflow → none
            cur
        } else {
            none
        }
    }
}
```

여기서 `#[literal( expr ".." expr )]`는 리터럴 문법을 등록합니다. 그래서 `0..5` 같은 표기가 `Range` struct로 풀립니다(권위는 [refs-lang 6.5](../../refs-lang/006-meta-system/605-literal-magic.md)).

```rok
for i in 0..5 {
    use(i);          //→ 0, 1, 2, 3, 4
}

let v: List<i32> = (0..10).collect();   //→ [0, 1, ..., 9]
```

### `RangeInclusive<T>`: `a..=b`

```rok
#[literal( expr "..=" expr )]
struct RangeInclusive<T> {
    start: T;
    end: T;
}

impl<T: Step> Iterator for RangeInclusive<T> { ... }
```

`start`와 `end` 외의 필드(예: 끝에 도달했는지 표시하는 내부 플래그)는 구현 세부이고 비공개입니다. `end`가 타입의 최댓값이어도 `a..=b`는 `b`까지 정확히 냅니다.

```rok
for i in 0..=5 {
    use(i);          //→ 0, 1, 2, 3, 4, 5
}
```

### `RangeFrom<T>`: `a..`(무한)

```rok
#[literal( expr ".." )]
struct RangeFrom<T> { start: T; }

impl<T: Step> Iterator for RangeFrom<T> {
    type Item = T;
    fn next(ref self) -> T | none {
        let cur = self.start.clone();
        self.start = self.start.forward_checked(1)?;  // overflow → none (예: i32::MAX 도달)
        cur
    }
}
```

```rok
let it = (10..).take(3);     //→ [10, 11, 12] (take 가 lazy 어댑터)
```

`RangeFrom`은 타입의 최댓값을 내지 않습니다. 최댓값의 다음 값이 없으므로 최댓값에 도달하면 그 값을 내기 전에 끝납니다. `(254u8..)`은 `254`만 내고 끝납니다.

### `RangeTo<T>` / `RangeToInclusive<T>` / `RangeFull`

```rok
#[literal( ".." expr )]
struct RangeTo<T> { end: T; }

#[literal( "..=" expr )]
struct RangeToInclusive<T> { end: T; }               // ..=b

#[literal( ".." )]
struct RangeFull;
```

`..b` / `..=b`(시작이 없는 형태)와 `..`(둘 다 없는 형태)는 슬라이스나 인덱스 문맥에서만 의미가 있고 반복자를 만들지 않습니다.

```rok
let xs = [1, 2, 3, 4, 5];
let s = xs[..3];          //→ [1, 2, 3]  (Slice: 2.3 ops 의 Slice 트레잇)
```

## 2.7.5. `FromIterator<T>`: `collect`의 디스패치

`collect`의 디스패치는 `FromIterator<T>` 트레잇이 맡습니다.

```rok
trait FromIterator<T> {
    fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Self;
}
```

`Iterator::collect`를 호출하면 대상 컬렉션의 `from_iter`가 호출됩니다.

```rok
let v: List<i32> = (0..5).collect();
// = List::from_iter((0..5).into_iter())
```

`List`, `Dict`, `Set`, `str` 등 모든 컬렉션은 `FromIterator`를 구현합니다.

## 2.7.6. 컬렉션별 IntoIterator impl

컬렉션별 `IntoIterator` 구현은 다음과 같습니다.

| 컬렉션              | `Item` 타입 | 동작                                      |
|---------------------|-------------|-------------------------------------------|
| `List<T>`           | `T`         | 첫 원소부터 마지막까지 순서대로           |
| `Dict<K, V>`        | `(K, V)`    | 순회 순서 **보장 안 됨** (HashDict)       |
| `OrderedDict<K, V>` | `(K, V)`    | 삽입 순서                                 |
| `Set<T>`            | `T`         | 순회 순서 **보장 안 됨**                  |
| `OrderedSet<T>`     | `T`         | 삽입 순서                                 |
| `Tuple<A, B, C>`    | (특수)      | 반복자 **만들지 않음**, 패턴 매칭 사용    |
| `Range<T>`          | `T`         | 시작부터 end-1까지                      |
| `Slice<T>`          | `T`         | 뷰의 시작부터 끝까지                     |

### Dict / Set의 순회 순서

기본 `Dict<K, V>`와 `Set<T>`는 **해시 기반**이므로 순회 순서가 해시 분포에 따라 정해집니다. 그래서 실행할 때나 컴파일할 때마다 순서가 달라질 수 있습니다.

순서 보장이 필요하면 삽입 순서를 보존하는 `OrderedDict<K, V>` 또는 `OrderedSet<T>`를 쓰세요.

## 2.7.7. 무한 반복자와 `take`

`RangeFrom`이나 사용자가 정의한 무한 반복자를 즉시 평가 메서드와 바로 결합하면 무한 루프에 빠집니다.

```rok
(0..).count();         // 무한 루프: count 가 끝까지 진행
(0..).sum();           // 무한 루프
```

`take(n)`으로 유한하게 만들면 해결됩니다.

```rok
(0..).take(100).count();     //→ 100
(0..).take(10).sum();        //→ 0+1+...+9 = 45
```

`Iterator`는 무한 여부를 검출하지 않습니다. 사용자가 지연 어댑터를 일부러 결합해서 처리해야 합니다.

## 2.7.8. 함수 broadcast: `f.(x)` 풀어 쓰기

[refs-lang](../../refs-lang/003-expressions/303-broadcast-operators.md)의 함수 브로드캐스트(`sin.(xs)`, `f.(a, b)`)는 별도의 트레잇 없이 `Iterator::map`으로 풀립니다.

```rok
sin.(xs)   ≡   xs.into_iter().map(sin).collect()
g.(a, b)   ≡   a.into_iter().zip(b.into_iter()).map(|(x, y)| g(x, y)).collect()
```

## 2.7.9. `DoubleEndedIterator`: 양방향 진행

양방향으로 진행할 수 있는 반복자는 `DoubleEndedIterator` 트레잇으로 나타냅니다.

```rok
trait DoubleEndedIterator: Iterator {
    fn next_back(ref self) -> Item | none;
}
```

이 트레잇은 뒤에서부터 진행할 수 있는 반복자를 나타냅니다. `rev()` 어댑터와 `rfold`, `rfind` 등 일부 메서드가 이 트레잇을 요구합니다.

`List<T>`, `Slice<T>`의 반복자와 다음 타입은 이 트레잇을 구현해야 합니다.

- `Range<T>`, `RangeInclusive<T>`
- `Rev<I>` (`I`가 양방향일 때), `Map<I, U>`, `Filter<I>`, `Chain<A, B>`, `InspectIter<I>` (구성 반복자가 양방향일 때)
- `Enumerate<I>`, `Zip<A, B>`, `Take<I>`, `Skip<I>`, `StepBy<I>`: 구성 반복자가 `DoubleEndedIterator + ExactSizeIterator`일 때. 뒤에서 세려면 길이가 필요하기 때문입니다.

반면 `Dict<K, V>`와 `Set<T>`는 순서가 없으므로 구현하지 않습니다.

```rok
let r: List<i32> = (0..5).rev().collect();    //→ [4, 3, 2, 1, 0]
```

## 2.7.10. `ExactSizeIterator`: 길이를 미리 아는 반복자

진행하기 전에 원소 개수를 알 수 있는 반복자는 `ExactSizeIterator`로 나타냅니다.

```rok
trait ExactSizeIterator: Iterator {
    fn len(self) -> usize;
}
```

`len()`은 남은 원소 수입니다. `Iterator::exact_len()`은 진행하지 않고 그 수를 알면 그 수를, 아니면 `none`(기본)을 돌려줍니다. 표준 라이브러리의 `ExactSizeIterator` 타입은 `len()`을 돌려줍니다. 그 위에 쌓은 아래 목록의 어댑터도 마찬가지입니다. 기본 `count()`는 `exact_len()`이 `none`이 아니면 그 값을 돌려줍니다. 그래서 `count()`는 `map`의 함수를 부르지 않습니다. `xs.iter().map(f).take(3).count()`는 `f`를 한 번도 부르지 않습니다. `scan`과 `inspect`의 함수도 마찬가지입니다. `Filter`, `TakeWhile`, `SkipWhile`, `(0..).take(100)`처럼 길이를 모르는 반복자는 실제로 진행합니다. 이 트레잇을 구현하는 타입은 다음과 같습니다. `List`, `Slice`, `Range`의 반복자, `List`와 `Slice`의 `windows` / `chunks` 뷰, 그리고 구성 반복자가 모두 `ExactSizeIterator`인 `Enumerate`, `Zip`, `Take`, `Skip`, `StepBy`, `Rev`, `Map`, `Scan`, `InspectIter`, `Chunks`입니다. `Windows`는 길이를 아는 반복자 위에서만 `exact_len`이 있습니다. `Cycle`은 반복자가 비어 있을 때만 그 값(0)이 있습니다.

## 2.7.10a. `GenIter<T>` / `TryGenIter<T, E>`: 제너레이터의 값

`yield fn` 호출과 `yield { .. }` 블록의 값입니다([refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#코루틴-함수의-결과-타입)).

```rok
struct GenIter<T> { /* 비공개 */ }
impl<T> Iterator for GenIter<T> {
    type Item = T;
    fn next(ref self) -> T | none;          // 본체를 다음 yield 까지 진행
}

struct TryGenIter<T, E> { /* 비공개 */ }    // yield fn .. throws (E)
impl<T, E> TryGenIter<T, E> {
    fn next(ref self) throws (E) -> T | none;
}
```

`TryGenIter`는 `next`가 던지므로 `Iterator`를 구현하지 않습니다. `try for`나 `try g.next()`로 진행합니다. 비공개 필드는 사양이 정하지 않습니다. 복사본은 같은 코루틴 프레임을 공유하므로 진행 위치도 공유합니다.

## 2.7.11. 모듈 매핑

트레잇별 lang slot과 표준 라이브러리 위치는 다음과 같습니다.

| 항목                  | lang slot             | 표준 라이브러리 위치 |
|-----------------------|-----------------------|----------------------|
| `Iterator` 트레잇     | `iterator_trait`      | 표준 라이브러리      |
| `IntoIterator` 트레잇 | `into_iterator_trait` | 표준 라이브러리      |
| `Step` 트레잇         | (없음)                | 표준 라이브러리      |
| `FromIterator` 트레잇 | (없음)                | 표준 라이브러리      |
| `DoubleEndedIterator` | (없음)                | 표준 라이브러리      |
| `ExactSizeIterator`   | (없음)                | 표준 라이브러리      |
| `Range<T>` 등 5종     | `range_type`          | `rok/range/`         |

원시 타입 자체에는 `Iterator` 구현이 없습니다. i32는 `for`의 대상이 아닙니다. 컬렉션과 Range만 대상입니다.

## 2.7.12. async: `Stream` 트레잇

비동기 반복자는 [2.8 stream](208-stream.md)에서 다룹니다. `for await x in stream` 문법([refs-lang 3.13](../../refs-lang/003-expressions/313-loop-expressions.md))은 `Stream::poll_next`로 디스패치됩니다.
