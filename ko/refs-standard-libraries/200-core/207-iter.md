# 2.7. iter — 반복 트레잇

`rok::core::iter` 모듈은 `for x in xs` 의 **desugar 권위** 를 제공합니다. 모든 `for` 루프는 본 장의 트레잇으로 dispatch 되어 동작합니다.

이때 [refs-lang 3.13 루프 표현식](../../refs-lang/003-expressions/313-loop-expressions.md) 의 `for` syntax 가 `IntoIterator::into_iter` 와 `Iterator::next` 호출의 조합으로 desugar 됩니다.

## 2.7.1. `Iterator` — 단계 진행

가장 먼저 살펴볼 트레잇은 `Iterator` 이며, 시그니처는 다음과 같습니다.

```rok
#[lang("iterator_trait")]
trait Iterator {
    type Item;

    fn next(ref self) -> Item | none;
}
```

핵심 동작은 단순합니다. `next` 를 한 번 호출하면 다음 원소 하나를 반환하며, 더 이상 원소가 없으면 `none` 을 돌려줍니다.

시그니처가 `ref self` 인 점에 주목합니다. iterator 자체가 **상태** 를 들고 있으며, 호출이 일어날 때마다 그 상태를 in-place 로 갱신하기 때문입니다. iterator 는 값입니다: 복사한 `ListIter` 는 따로 진행하며, `next(ref self)` 는 그 사본만 진행시킵니다.

### 사용 예 (사용자 코드)

직접 호출하는 모습은 다음과 같습니다.

```rok
let xs: List<i32> = [10, 20, 30];
let it = xs.into_iter();         // IntoIterator 가 만든 iterator
let v1 = it.next();              //→ 10
let v2 = it.next();              //→ 20
let v3 = it.next();              //→ 30
let v4 = it.next();              //→ none
```

다만 실제 코드에서는 직접 호출보다 `for` 를 사용하는 편이 일반적입니다.

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

### 메서드 — default impl 풍부

`Iterator` 에서 사용자가 직접 구현해야 하는 메서드는 `next(ref self)` 하나뿐입니다. 나머지 메서드들은 모두 `next` 위에 default impl 형태로 제공됩니다.

> [!NOTE]
> 아래 시그니처에 등장하는 `fn(Item) -> U` 는 [refs-lang 4.4 함수 타입](../../refs-lang/004-functions/404-function-type.md) 이 정의하는 callable 통합 타입이며, **함수와 closure 둘 다** 받습니다. 따라서 `it.map(some_fn)` 도, `it.map(|x| x * 2)` 도 같은 자리에 들어갑니다. Rust 의 `fn`/`Fn`/`FnMut`/`FnOnce` 와는 다른 단일 형태이니 유의합니다.

```rok
impl Iterator {
    fn next(ref self) -> Item | none;       // 사용자 정의

    // 변환 어댑터 — lazy
    fn map<U>(self, f: fn(Item) -> U) -> Map<Self, U>;
    fn filter(self, pred: fn(Item) -> bool) -> Filter<Self>;
    fn enumerate(self) -> Enumerate<Self>;        // (index, item) 쌍
    fn zip<J: Iterator>(self, other: J) -> Zip<Self, J>;    // Item = (Item, J::Item)
    fn chain<I: Iterator<Item = Item>>(self, other: I) -> Chain<Self, I>;
    fn take(self, n: usize) -> Take<Self>;
    fn skip(self, n: usize) -> Skip<Self>;
    fn step_by(self, step: usize) -> StepBy<Self>;
    fn rev(self) -> Rev<Self>;                    // DoubleEndedIterator 만
    fn flatten<U: IntoIterator>(self) -> Flatten<Self> where Item: IntoIterator;
    fn flat_map<U: IntoIterator>(self, f: fn(Item) -> U) -> FlatMap<Self, U>;
    fn peekable(self) -> Peekable<Self>;          // 다음 원소 미리 보기

    // 종결 메서드 — eager
    fn count(self) -> usize;
    fn sum(self) -> Item where Item: Add<Output = Item> + Zero;     // 비면 0
    fn product(self) -> Item where Item: Mul<Output = Item> + One;  // 비면 1
    fn fold<B>(self, init: B, f: fn(B, Item) -> B) -> B;
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

### `sum` / `product` 와 `Zero` / `One`

`sum` 은 `Zero::zero()` 에서 시작해 `+` 로 누적하고, `product` 는 `One::one()` 에서 시작해 `*` 로 누적합니다. 따라서 빈 iterator 의 합은 0, 곱은 1 입니다. `Zero` / `One` 은 `rok::core::num` 의 트레잇입니다 ([2.1](201-numbers.md)).

### lazy vs eager

이 메서드들은 두 부류로 나뉩니다. 변환 어댑터인 `map`, `filter`, `take` 등은 **lazy** 로 동작하므로, 새 iterator 를 반환만 할 뿐 실제 계산은 수행하지 않습니다. 반면 `count`, `sum`, `collect`, `for_each` 같은 종결 메서드가 호출되어야 비로소 단계 진행이 시작됩니다.

```rok
let xs = [1, 2, 3, 4, 5];
let r = xs.into_iter()
    .map(|x| x * 2)            // lazy — 아직 계산 안 함
    .filter(|x| x > 4)         // lazy
    .sum();                    // eager — 여기서 단계 진행
//→ 6 + 8 + 10 = 24
```

### 사용자 정의 Iterator

직접 Iterator 를 정의하는 예시는 다음과 같습니다.

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

## 2.7.2. `IntoIterator` — `for` 의 시작

다음으로는 `for` 루프의 출발점이 되는 `IntoIterator` 트레잇을 살펴보겠습니다.

```rok
#[lang("into_iterator_trait")]
trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Item>;

    fn into_iter(self) -> IntoIter;
}
```

`IntoIterator` 는 컬렉션이 iterator 를 생성하도록 만드는 어댑터입니다. `for x in xs` 는 **xs 자체** 를 iterator 처럼 다루지 않고, `xs.into_iter()` 로 iterator 를 만든 뒤 그 위에서 `next` 를 호출하는 방식으로 동작합니다.

### 분리되어 있는 이유

이 두 트레잇이 분리되어 있는 까닭은 두 가지 관점이 본질적으로 다르기 때문입니다.

- 컬렉션 자체 (List, Dict 등) 는 상태 없는 컨테이너이므로, 어떤 위치를 가리키는지 알지 못합니다.
- iterator 는 진행 상태를 가지며, 어디까지 봤는지를 추적합니다.

따라서 같은 컬렉션을 두 번 순회하려면 매번 새 iterator 가 필요하며, `into_iter` 가 바로 그 생성 자리를 담당합니다.

### 자동 impl

`Iterator` 자체도 `IntoIterator` 의 자동 impl 을 받습니다. 이때 `into_iter` 는 self 를 그대로 반환합니다.

```rok
// 모든 Iterator 는 자동으로 IntoIterator
impl<I: Iterator> IntoIterator for I {
    type Item = I::Item;
    type IntoIter = I;
    fn into_iter(self) -> I { self }
}
```

덕분에 `for x in iter` 도 그대로 작동합니다. iter 가 이미 Iterator 라면 `into_iter` 가 self 를 돌려주기 때문입니다.

### `for` desugar 정확한 형태

`for` 루프의 desugar 형태를 좀 더 정확하게 정리해 보겠습니다.

```rok
for x in expr {
    body
}
```

위 구문은 다음과 같이 desugar 됩니다.

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

자세한 desugar 규칙은 [refs-lang 3.13](../../refs-lang/003-expressions/313-loop-expressions.md) 을 참고합니다.

## 2.7.3. `Step` — 한 칸 진행 추상화

`Range<T>` 의 iteration 은 "다음 값으로 한 칸 이동" 이라는 단순 동작에 기반합니다. 그런데 이 동작은 일반적인 산술 (`Add`) 과는 의미가 다른데, 예를 들어 `char` 의 다음 값은 codepoint + 1 이지만 `char + char` 자체는 정의되지 않습니다. 따라서 rok 은 step 진행을 별도의 트레잇으로 분리해 두고 있습니다.

```rok
trait Step: PartialOrd + Clone {
    fn forward_checked(self, count: usize) -> Self | none;
    fn backward_checked(self, count: usize) -> Self | none;
    fn steps_between(start: Self, end: Self) -> usize | none;
}
```

각 메서드의 의미는 다음과 같습니다.

- `forward_checked(self, n)` — `self` 에서 `n` 만큼 앞으로 이동한 값을 반환하며, 타입 범위를 넘으면 `none` 을 돌려줍니다.
- `backward_checked(self, n)` — 뒤로 이동한 값. 마찬가지로 범위를 넘으면 `none`.
- `steps_between(start, end)` — `start` 에서 `end` 까지 몇 칸인지 계산하며, 표현 불가일 때 `none` 을 돌려줍니다 (예: 음수, overflow).

### 산술 트레잇과 분리한 이유

`Step` 을 `Add` 에서 분리한 까닭은 세 가지로 정리할 수 있습니다.

- **의미 분리**: `+1` 은 단순 increment 이지 임의 산술이 아닙니다. `MyVec3 + MyVec3` 는 의미가 있지만 `Range<MyVec3>` 는 의미가 없으므로, `Step` 만 별도 구현하지 않으면 자연스럽게 거부됩니다.
- **부동소수 배제**: `f32` / `f64` 에는 `Step` 이 impl 되어 있지 않습니다. 따라서 `Range<f64>` 는 컴파일 에러로 잡히고, `0.0..1.0` 같은 의미 모호 표기가 사양 차원에서 사라집니다.
- **효율적 skip**: `Iterator::nth(n)` 이나 `step_by(n)` 가 `forward_checked(_, n)` 를 한 번 호출하여 O(1) 로 동작합니다. `succ` 만 있는 경우라면 같은 동작이 O(n) 가 됩니다.

### Step impl 매트릭스

표준 라이브러리는 정수 primitive 와 `char` 에만 `Step` 을 impl 합니다.

| 타입                 | impl     | 비고                                                 |
|----------------------|----------|------------------------------------------------------|
| `i8`..`i64`, `isize` | **있음** | overflow 시 `forward_checked` 가 `none` 반환         |
| `u8`..`u64`, `usize` | **있음** | underflow 시 `backward_checked` 가 `none` 반환       |
| `int`                | **없음** | 표준 라이브러리 impl 없음                            |
| `char`               | **있음** | codepoint 진행, surrogate (`0xD800..=0xDFFF`) 건너뜀 |
| `bool`               | **없음** | 표준 라이브러리 impl 없음                            |
| `f32`, `f64`         | **없음** | `Range<f64>` 거부 — 의미 모호                        |
| `float`              | **없음** | (위와 동일)                                          |

사용자 정의 타입은 `impl Step for MyType` 로 자유롭게 추가할 수 있으며, `Range<MyType>` 이 즉시 가능해집니다.

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

[refs-lang 3.7 범위 표현식](../../refs-lang/003-expressions/307-range-expressions.md) 의 `a..b` syntax 는 본 표준 라이브러리 타입으로 desugar 됩니다.

### `Range<T>` — `a..b` (exclusive)

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

여기서 `#[literal( expr ".." expr )]` 는 리터럴 문법을 등록하는 역할을 합니다. 이를 통해 `0..5` 같은 표기가 `Range` struct 로 desugar 됩니다 ([refs-lang 6.5](../../refs-lang/006-meta-system/605-literal-magic.md) 권위).

```rok
for i in 0..5 {
    use(i);          //→ 0, 1, 2, 3, 4
}

let v: List<i32> = (0..10).collect();   //→ [0, 1, ..., 9]
```

### `RangeInclusive<T>` — `a..=b`

```rok
#[literal( expr "..=" expr )]
struct RangeInclusive<T> {
    start: T;
    end: T;
}

impl<T: Step> Iterator for RangeInclusive<T> { ... }
```

`start` 와 `end` 외의 필드 (예: 끝에 도달했는지 표시하는 내부 플래그) 는 구현 세부이며 비공개입니다. `end` 가 타입의 최댓값이어도 `a..=b` 는 `b` 까지 정확히 냅니다.

```rok
for i in 0..=5 {
    use(i);          //→ 0, 1, 2, 3, 4, 5
}
```

### `RangeFrom<T>` — `a..` (무한)

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

`RangeFrom` 은 타입의 최댓값을 내지 않습니다. 최댓값의 다음 값이 없으므로, 최댓값에 도달하면 그 값을 내기 전에 끝납니다 (`(254u8..)` 은 `254` 만 내고 끝남).

### `RangeTo<T>` / `RangeToInclusive<T>` / `RangeFull`

```rok
#[literal( ".." expr )]
struct RangeTo<T> { end: T; }

#[literal( "..=" expr )]
struct RangeToInclusive<T> { end: T; }               // ..=b

#[literal( ".." )]
struct RangeFull;
```

`..b` / `..=b` (start 가 없는 형태) 와 `..` (둘 다 없는 형태) 는 slice 나 index 문맥에서만 의미를 가지며, iterator 를 만들지는 않습니다.

```rok
let xs = [1, 2, 3, 4, 5];
let s = xs[..3];          //→ [1, 2, 3]  (Slice — 2.3 ops 의 Slice 트레잇)
```

## 2.7.5. `FromIterator<T>` — `collect` 의 dispatch

`collect` 의 dispatch 를 담당하는 트레잇은 `FromIterator<T>` 입니다.

```rok
trait FromIterator<T> {
    fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Self;
}
```

`Iterator::collect` 가 호출되면 대상 컬렉션의 `from_iter` 가 호출됩니다.

```rok
let v: List<i32> = (0..5).collect();
// = List::from_iter((0..5).into_iter())
```

`List`, `Dict`, `Set`, `str` 등 각 컬렉션이 모두 `FromIterator` 를 impl 하고 있습니다.

## 2.7.6. 컬렉션 별 IntoIterator impl

각 컬렉션이 어떤 방식으로 `IntoIterator` 를 impl 하는지 표로 정리하면 다음과 같습니다.

| 컬렉션              | `Item` 타입 | 동작                                      |
|---------------------|-------------|-------------------------------------------|
| `List<T>`           | `T`         | 첫 원소부터 마지막까지 순서대로           |
| `Dict<K, V>`        | `(K, V)`    | 순회 순서 **보장 안 됨** (HashDict)       |
| `OrderedDict<K, V>` | `(K, V)`    | 삽입 순서                                 |
| `Set<T>`            | `T`         | 순회 순서 **보장 안 됨**                  |
| `OrderedSet<T>`     | `T`         | 삽입 순서                                 |
| `Tuple<A, B, C>`    | (특수)      | iterator **만들지 않음** — 패턴 매칭 사용 |
| `Range<T>`          | `T`         | start 부터 end-1 까지                     |
| `Slice<T>`          | `T`         | view 의 시작부터 끝까지                   |

### Dict / Set 의 순회 순서

기본 `Dict<K, V>` 와 `Set<T>` 는 **hash 기반** 으로 동작하므로, 순회 순서가 hash 분포에 따라 결정됩니다. 이 때문에 실행 시점이나 컴파일 시점마다 순서가 달라질 수 있습니다.

순서 보장이 필요하다면 `OrderedDict<K, V>` 또는 `OrderedSet<T>` 를 사용합니다. 이들은 삽입 순서를 보존합니다.

## 2.7.7. 무한 iterator 와 `take`

`RangeFrom` 이나 사용자 정의로 만든 무한 iterator 는 eager 메서드와 직접 결합할 경우 무한 루프에 빠지게 됩니다.

```rok
(0..).count();         // 무한 루프 — count 가 끝까지 진행
(0..).sum();           // 무한 루프
```

이 문제는 `take(n)` 으로 유한화하여 해결할 수 있습니다.

```rok
(0..).take(100).count();     //→ 100
(0..).take(10).sum();        //→ 0+1+...+9 = 45
```

`Iterator` 는 무한 여부를 검출하지 않습니다. 사용자가 의도적으로 lazy 어댑터를 결합하여 처리해야 합니다.

## 2.7.8. 함수 broadcast — `f.(x)` 의 desugar

[refs-lang](../../refs-lang/003-expressions/303-broadcast-operators.md) 의 함수 broadcast (`sin.(xs)`, `f.(a, b)`) 는 별도의 트레잇 없이 `Iterator::map` 으로 desugar 됩니다.

```rok
sin.(xs)   ≡   xs.into_iter().map(sin).collect()
g.(a, b)   ≡   a.into_iter().zip(b.into_iter()).map(|(x, y)| g(x, y)).collect()
```

## 2.7.9. `DoubleEndedIterator` — 양방향 진행

양방향 진행이 가능한 iterator 는 `DoubleEndedIterator` 트레잇으로 모델링됩니다.

```rok
trait DoubleEndedIterator: Iterator {
    fn next_back(ref self) -> Item | none;
}
```

이 트레잇은 뒤에서부터 진행 가능한 iterator 를 표현하며, `rev()` 어댑터나 `rfold`, `rfind` 등 일부 메서드가 이 트레잇을 요구합니다.

`List<T>`, `Slice<T>` 의 iterator 와 다음 타입은 반드시 이 트레잇을 impl 합니다.

- `Range<T>`, `RangeInclusive<T>`
- `Rev<I>` (`I` 가 양방향일 때), `Map<I, U>`, `Filter<I>`, `Chain<A, B>` (구성 iterator 가 양방향일 때)
- `Enumerate<I>`, `Zip<A, B>`, `Take<I>`, `Skip<I>`, `StepBy<I>` — 구성 iterator 가 `DoubleEndedIterator + ExactSizeIterator` 일 때. 뒤에서 세려면 길이가 필요하기 때문입니다.

반면 `Dict<K, V>` 와 `Set<T>` 는 순서가 없으므로 impl 하지 않습니다.

```rok
let r: List<i32> = (0..5).rev().collect();    //→ [4, 3, 2, 1, 0]
```

## 2.7.10. `ExactSizeIterator` — 길이 미리 알려진 iterator

원소 개수를 진행 전부터 알 수 있는 iterator 는 `ExactSizeIterator` 로 표현됩니다.

```rok
trait ExactSizeIterator: Iterator {
    fn len(self) -> usize;
}
```

`len()` 은 남은 원소 수입니다. 표준 라이브러리의 `ExactSizeIterator` 타입은 `count()` 를 `len()` 으로 답하므로, `count()` 는 `Map` 의 함수를 부르지 않습니다. `List`, `Slice`, `Range` 의 iterator 와 구성 iterator 가 모두 `ExactSizeIterator` 인 `Enumerate`, `Zip`, `Take`, `Skip`, `StepBy`, `Rev`, `Map` 이 이 트레잇을 impl 합니다.

## 2.7.10a. `GenIter<T>` / `TryGenIter<T, E>` — 제너레이터의 값

`yield fn` 호출과 `yield { .. }` 블록의 값입니다 ([refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#코루틴-함수의-결과-타입)).

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

`TryGenIter` 는 `next` 가 던지므로 `Iterator` 를 구현하지 않습니다. `try for` 나 `try g.next()` 로 진행합니다. 비공개 필드는 사양으로 정하지 않습니다. 복사본은 같은 코루틴 frame 을 공유하므로 진행 위치도 공유합니다.

## 2.7.11. 모듈 매핑

각 트레잇과 lang slot, 표준 라이브러리 위치의 매핑은 다음과 같습니다.

| 항목                  | lang slot             | 표준 라이브러리 위치 |
|-----------------------|-----------------------|----------------------|
| `Iterator` 트레잇     | `iterator_trait`      | 표준 라이브러리      |
| `IntoIterator` 트레잇 | `into_iterator_trait` | 표준 라이브러리      |
| `Step` 트레잇         | (없음)                | 표준 라이브러리      |
| `FromIterator` 트레잇 | (없음)                | 표준 라이브러리      |
| `DoubleEndedIterator` | (없음)                | 표준 라이브러리      |
| `ExactSizeIterator`   | (없음)                | 표준 라이브러리      |
| `Range<T>` 등 5 종    | `range_type`          | `rok/range/`         |

primitive 자체에는 `Iterator` impl 이 없습니다 (i32 는 `for` 의 대상이 아닙니다). 컬렉션과 Range 만 해당됩니다.

## 2.7.12. async — `Stream` 트레잇

비동기 iterator 는 [2.8 stream](208-stream.md) 에서 다룹니다. `for await x in stream` syntax ([refs-lang 3.13](../../refs-lang/003-expressions/313-loop-expressions.md)) 가 `Stream::poll_next` 로 dispatch 됩니다.
