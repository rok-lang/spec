# 2.9. 컬렉션: `List` / `Dict` / `Set` / `Tuple` / `Slice`

`rok::core::collections` 모듈은 범용 컨테이너를 정의합니다. 모든 컬렉션은 GC 힙에 할당됩니다. `#[literal(...)]`로 묶은 리터럴 문법으로 바로 만들 수 있습니다.

이 장은 다음 컬렉션 6가지의 시그니처를 정의합니다.

| 컬렉션              | 리터럴 문법                | 의미                          |
|---------------------|----------------------------|-------------------------------|
| `List<T>`           | `[1, 2, 3]`                | 동적 배열(삽입 순서)          |
| `Dict<K, V>`        | `{"a": 1, "b": 2}`         | 해시 맵(순서 없음)            |
| `Set<T>`            | `{1, 2, 3}`                | 해시 집합(순서 없음)          |
| `OrderedDict<K, V>` | `OrderedDict::from([...])` | 삽입 순서를 지키는 dict       |
| `OrderedSet<T>`     | `OrderedSet::from([...])`  | 삽입 순서를 지키는 set        |
| `Tuple<...>`        | `(1, "a", true)`           | 고정 크기, 원소 타입이 제각각 |
| `Slice<T>`          | `xs[1:3]`                  | 복사 없는 컨테이너 뷰         |

## 2.9.1. 리터럴 문법: `#[literal(query)]` 바인딩

리터럴 문법은 [refs-lang 6.5 `#[literal]`](../../refs-lang/006-meta-system/605-literal-magic.md)의 **query DSL**로 다음과 같이 정의합니다.

```rok
#[literal( "[" {expr} % "," * "]" )]
struct List<T> { /* ... */ }

#[literal( "{" {expr ":" expr} % "," * "}" )]
struct Dict<K, V> { /* ... */ }

#[literal( "{" {expr} % "," + "}" )]
struct Set<T> { /* ... */ }

#[literal( "(" expr "," {expr} % "," * ")" )]
struct Tuple { /* ... */ }
```

리터럴 query 문법의 뜻은 다음과 같습니다. `{...} % sep *`는 sep으로 구분한 0번 이상의 반복입니다. `{...} % sep +`는 1번 이상의 반복입니다. 원소가 하나인 Set은 뒤에 콤마를 붙여 `{1,}`로 씁니다. 그래야 블록 `{1}`과 구분됩니다. 토큰열을 매칭할 때는 렉서가 먼저 토큰을 직접 구분하고, 그다음 파서가 query를 매칭합니다.

### Dict / Set 모호성 해결

```rok
{"a": 1}      // Dict: `:` 패턴
{1, 2}        // Set: 값만
{1,}          // Set (단일 원소, 후행 콤마)
{1}           // 항상 block. 슬롯이 Set 이어도 마찬가지
{ x = 1 }     // block expression: `{` 다음 토큰이 statement-like
```

파서가 내용만 보고 정합니다. `expr ":"` 형태이면 Dict, 콤마가 있으면 Set, 그 밖에는 블록입니다. 받는 슬롯의 타입은 판정에 쓰지 않습니다.

### `[...]`의 타입

`[1, 2, 3]`은 기대 타입이 없으면 `List<T>`입니다. 고정 크기 배열 `[T; N]`은 기대 타입을 `[T; N]`으로 적었을 때만 만들어집니다. 예를 들면 `let a: [i32; 3] = [1, 2, 3];`입니다. 고정 배열은 컴파일러가 핸들러를 거치지 않고 직접 만듭니다.

원소가 `Clone` / `Copy` / `Default` / `Debug` / `PartialEq` / `Eq` / `PartialOrd` / `Ord` / `Hash`를 구현하면 고정 배열도 그 트레잇을 구현합니다. 고정 배열에는 `len()`과 `is_empty()`가 있습니다. 순서 비교는 사전식이므로 비교 연산자(`==`, `<` 등)를 바로 쓸 수 있습니다.

```rok
let a: [i32; 3] = [1, 2, 3];
a.len();                 //→ 3
a == [1, 2, 3];          //→ true
a < [1, 2, 4];           //→ true (사전식)
```

### 참조 핸들

`List`, `Dict`, `Set` 값은 GC 힙 객체를 가리키는 참조 핸들입니다. `let ys = xs;`나 인자 전달은 핸들을 복사하므로 같은 컬렉션을 공유합니다. 그래서 컬렉션을 바꾸는 메서드도 `ref self`가 아니라 `self`를 받습니다. 독립된 사본이 필요하면 `clone()`을 부릅니다. struct 원소는 값입니다. `xs[i]`와 `d[k]`는 사본을 돌려주므로 바꾼 값은 `xs[i] = v`로 저장합니다([refs-lang 2.3.8](../../refs-lang/002-types/203-struct.md#238-값-의미)). `keys` / `values` / `entries`는 컬렉션을 복사하지 않는 지연 반복자를 돌려줍니다.

## 2.9.2. `List<T>`: 동적 배열

```rok
struct List<T> {
    // 내부: ptr + len + capacity
}
```

내부는 연속된 메모리 버퍼와 `len` / `capacity`로 이루어집니다. push와 pop은 분할 상환 O(1), 임의 접근은 O(1)입니다.

### 핵심 메서드

```rok
impl<T> List<T> {
    fn new() -> List<T>;
    fn with_capacity(cap: usize) -> List<T>;

    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn capacity(self) -> usize;

    fn push(self, x: T);
    fn pop(self) -> T | none;
    fn insert(self, idx: usize, x: T);
    fn remove(self, idx: usize) -> T;
    fn clear(self);
    fn truncate(self, len: usize);              // 앞의 len 개만 남김; 더 적으면 그대로
    fn drain(self, r: Range<usize>) -> List<T>; // r 의 원소를 순서대로 빼서 돌려줌; r 이 거꾸로이거나 len() 을 넘어 끝나면 panic

    fn get(self, idx: usize) -> T | none;       // safe: out-of-bounds → none
    fn set(self, idx: usize, x: T) throws (IndexError);   // xs[idx] = x panics instead
    fn first(self) -> T | none;
    fn last(self) -> T | none;

    fn extend<I: IntoIterator<Item = T>>(self, iter: I);
    fn sort(self) where T: Ord;                  // 오름차순, stable
    fn sort_by(self, cmp: fn(T, T) -> Ordering); // stable
    fn reserve(self, additional: usize);
    fn shrink_to_fit(self);
    fn reverse(self);                            // 제자리

    fn iter(self) -> ListIter<T>;
    fn into_iter(self) -> ListIntoIter<T>;
}
```

```rok
impl List<str> {
    fn join(self, sep: str) -> str;              // 원소를 sep 로 이어 붙임
}

impl List<u8> {
    fn find<B: AsBytes>(self, needle: B) -> usize | none;   // self[:].find(needle)
}
```

### 사용 예

```rok
let xs: List<i32> = [];
xs.push(1);
xs.push(2);
xs.push(3);

let ys: List<i32> = [10, 20, 30];
ys[0];                    //→ 10  (Index trait)
ys[100];                  //→ panic (IndexError)
ys.get(100);              //→ none

for x in ys {
    use(x);               //→ 10, 20, 30
}

let total: i32 = ys.into_iter().sum();   //→ 60
```

### 트레잇 구현

| 트레잇                | 구현 조건           | 의미                            |
|-----------------------|---------------------|---------------------------------|
| `IntoIterator`        | 항상                | `for x in xs`                   |
| `FromIterator<T>`     | 항상                | `(0..5).collect::<List<i32>>()` |
| `Index<usize>`        | 항상                | `xs[i]`                         |
| `IndexSet<usize>`     | 항상 (`Value = T`)  | `xs[i] = v`                     |
| `AddAssign<T>`        | 항상                | `xs += x`는 `x`를 뒤에 넣음     |
| `AddAssign<List<T>>`  | 항상                | `xs += ys`는 `ys`의 모든 원소를 덧붙임 |
| `Sliceable<Range<usize>>` | 항상            | `xs[1:3]`                       |
| `Display`             | **아님**            | **수동 출력 권장**              |
| `Debug`               | `T: Debug`          | `f"{xs:?}"` → `[1, 2, 3]`       |
| `Clone`               | `T: Clone`          | 깊은 복제                       |
| `PartialEq`           | `T: PartialEq`      | 원소별                          |
| `Eq`/`Ord`/`Hash`     | T가 같은 트레잇을 구현할 때 | 사전식                  |

### 비교 / 해시 / Debug: 공통 규칙

- `List<T>`는 `PartialEq` / `Eq` / `Hash`를 순서대로 원소별로 구현합니다.
- `Dict<K, V>`와 `Set<T>`는 순서와 무관하게 비교하고 해시합니다. 든 원소(키와 값)가 같으면 같습니다.
- 세 컬렉션 모두 원소가 그 트레잇을 구현하면 `Debug`를 구현합니다. 출력은 순회 순서를 따릅니다.
- 따라서 원소가 `Eq + Hash`인 컬렉션은 `Set` 원소나 `Dict` 키가 될 수 있습니다(`Set<List<i32>>`).

### 용량과 제거: 공통 규칙

- `reserve`, `with_capacity`, `repeat`, 확장에서 크기가 넘치면 "capacity overflow" panic이 일어납니다. 최적화 수준과 관계없이 같습니다.
- 원소를 빼면(`pop`, `remove`, `clear`, `truncate`, `drain` 등) 그 참조도 놓습니다. 그 뒤로 GC는 그 컬렉션을 거쳐 그 원소에 닿지 않습니다.

## 2.9.3. `Dict<K, V>`: 해시 맵

```rok
struct Dict<K: Hash + Eq, V> {
    // 내부: HashMap (open addressing 또는 chain)
}
```

조회, 삽입, 제거는 평균 O(1)입니다. 순서를 보장하지 않으므로 순회 순서는 비결정적입니다.

### 핵심 메서드

```rok
impl<K: Hash + Eq, V> Dict<K, V> {
    fn new() -> Dict<K, V>;
    fn with_capacity(cap: usize) -> Dict<K, V>;

    fn len(self) -> usize;
    fn is_empty(self) -> bool;

    fn insert(self, k: K, v: V) -> V | none;       // 기존 V 반환 / 없으면 none
    fn remove(self, k: K) -> V | none;
    fn get(self, k: K) -> V | none;
    fn contains_key(self, k: K) -> bool;
    fn clear(self);

    fn keys(self) -> KeysIter<K, V>;
    fn values(self) -> ValuesIter<K, V>;
    fn entries(self) -> EntriesIter<K, V>;            // (K, V) 쌍

    fn entry(self, k: K) -> Entry<K, V>;          // 그 키의 자리 (있든 없든)
}

struct Entry<K, V> { /* dict + key */ }

impl<K: Hash + Eq, V> Entry<K, V> {
    fn key(self) -> K;
    fn or_insert(self, v: V) -> V;                   // 없으면 v 를 넣고, 그 값
    fn or_insert_with(self, f: fn() -> V) -> V;      // 없으면 f() 를 넣고, 그 값
    fn and_modify(self, f: fn(V) -> V) -> Entry<K, V>;   // 있으면 f(값) 으로 바꿈
}

impl<K: Hash + Eq, V: Default> Entry<K, V> {
    fn or_default(self) -> V;                         // 없으면 V::default() 를 넣고, 그 값
}
```

```rok
let counts: Dict<str, i32> = {};
for w in words {
    counts.entry(w).and_modify(|n| n + 1).or_insert(1);
}
```

### 사용 예

```rok
let d: Dict<str, i32> = {"a": 1, "b": 2};
d["a"];                   //→ 1
d["c"];                   //→ panic (IndexError: 키 없음)
d.get("c");               //→ none

d.insert("c", 3);
d.contains_key("c");      //→ true

for (k, v) in d.entries() {
    log("{k} = {v}");
}
```

### `Index<K>`

```rok
impl<K: Hash + Eq, V> Index<K> for Dict<K, V> {
    type Output = V;
    fn index(self, k: K) -> V {
        self.get(k) ?? panic("key not found")
    }
}
```

`d[k]` 표기는 키가 있다고 가정합니다. 안전하게 접근하려면 `.get(k)`를 쓰세요.

`d += e`(`AddAssign<Dict<K, V>>`)는 `e`의 모든 항목을 `d`에 제자리에서 삽입합니다. 키가 겹치면 `e`의 값이 남습니다.

`Dict::from(pairs)`(`From<List<(K, V)>>`)는 쌍의 리스트로 dict를 만듭니다. 키가 겹치면 뒤의 값이 남습니다.

### lazy view

`keys` / `values` / `entries`는 사본이 아닙니다. 현재 테이블을 그대로 보는 지연 뷰입니다. 순회 도중 dict를 바꾸면 결과는 사양이 정하지 않습니다. 원소를 빠뜨리거나 두 번 볼 수 있습니다. 그래도 메모리 안전성은 지켜집니다.

## 2.9.4. `Set<T>`: 해시 집합

```rok
struct Set<T: Hash + Eq> {
    // 내부: Dict<T, bool>
}
```

순서는 보장하지 않습니다. 연산은 평균 O(1)입니다.

```rok
impl<T: Hash + Eq> Set<T> {
    fn new() -> Set<T>;
    fn len(self) -> usize;
    fn is_empty(self) -> bool;

    fn insert(self, x: T) -> bool;        // true if 새로 들어감
    fn remove(self, x: T) -> bool;
    fn contains(self, x: T) -> bool;
    fn clear(self);

    fn union(self, other: Set<T>) -> Set<T>;
    fn intersection(self, other: Set<T>) -> Set<T>;
    fn difference(self, other: Set<T>) -> Set<T>;
    fn symmetric_difference(self, other: Set<T>) -> Set<T>;

    fn is_subset(self, other: Set<T>) -> bool;
    fn is_superset(self, other: Set<T>) -> bool;
    fn is_disjoint(self, other: Set<T>) -> bool;

    fn iter(self) -> KeysIter<T, bool>;
    fn to_list(self) -> List<T>;
}
```

```rok
let s: Set<i32> = {1, 2, 3};
s.contains(2);            //→ true
s.insert(4);

let a: Set<i32> = {1, 2, 3};
let b: Set<i32> = {2, 3, 4};
let u = a.union(b);       //→ {1, 2, 3, 4}
let i = a.intersection(b);//→ {2, 3}
```

`Set::from(xs)`(`From<List<T>>`)는 리스트로 set을 만들고 중복은 버립니다. `s += x`(`AddAssign<T>`)는 `x`를 삽입합니다.

## 2.9.5. `OrderedDict<K, V>` / `OrderedSet<T>`: 삽입 순서

두 타입은 `Dict` / `Set`과 API가 같지만 **삽입 순서를 지킵니다**. 의미는 Python 3.7 이상의 `dict`나 Java의 LinkedHashMap과 같습니다.

```rok
struct OrderedDict<K: Hash + Eq, V> { /* ... */ }
struct OrderedSet<T: Hash + Eq> { /* ... */ }
```

설정, JSON 같은 구조, 재현 가능한 출력처럼 순서를 지켜야 할 때 알맞습니다. 순서를 담는 인덱스 벡터 때문에 오버헤드가 약간 있습니다.

- `OrderedDict`는 `new` / `with_capacity` / `from` / `len` / `is_empty` / `insert` / `get` / `contains_key` / `remove` / `clear` / `keys` / `values` / `entries`와 `d[k]`, `d[k] = v`가 있습니다. `keys` / `values` / `entries`는 삽입 순서로 돕니다. 이미 있는 키에 `insert`하면 값만 바뀌고 순서는 그대로입니다.
- `OrderedSet`은 `new` / `from` / `len` / `is_empty` / `insert` / `remove` / `contains` / `clear` / `iter` / `to_list`와 집합 연산 `union` / `intersection` / `difference` / `symmetric_difference` / `is_subset` / `is_superset` / `is_disjoint`가 있습니다. 집합 연산의 결과는 왼쪽 피연산자의 순서를 먼저 따릅니다.
- 둘 다 `Default`, `Clone`, `IntoIterator`, `FromIterator`, `PartialEq` / `Eq`와 `Hash`(`Dict` / `Set`처럼 순서 무관), `Debug`를 구현합니다. 그래서 `Set` 원소나 `Dict` 키가 될 수 있습니다.
- `OrderedDict`의 `d += e`는 `e`를 제자리에서 합칩니다. `e`의 값이 남고 새 키는 맨 뒤에 붙습니다. `OrderedSet`의 `s += x`는 `x`를 삽입합니다.

### 생성

리터럴 문법은 따로 없습니다. 생성자를 호출해서만 만듭니다.

```rok
let d = OrderedDict::from([("a", 1), ("b", 2)]);
let s = OrderedSet::from([1, 2, 3]);
```

## 2.9.5a. `Deque<T>`: 양끝 큐

링 버퍼 위에 만든 양끝 큐입니다.

```rok
impl<T> Deque<T> {
    fn new() -> Deque<T>;
    fn with_capacity(cap: usize) -> Deque<T>;
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn capacity(self) -> usize;
    fn push_back(self, x: T);
    fn push_front(self, x: T);
    fn pop_front(self) -> T | none;
    fn pop_back(self) -> T | none;
    fn get(self, i: usize) -> T | none;
    fn set(self, i: usize, x: T) throws (IndexError);
    fn front(self) -> T | none;
    fn back(self) -> T | none;
    fn clear(self);
    fn iter(self) -> DequeIter<T>;             // 앞에서 뒤로
}
```

`set`은 `i`가 범위 밖이면 `IndexError`를 던집니다. `capacity`는 링 버퍼가 커지기 전까지 담을 수 있는 원소 수입니다. 이 타입은 `d[i]` / `d[i] = v`를 구현합니다. 범위 밖이면 panic이 일어납니다. 그 밖에 `IntoIterator`, `FromIterator`(앞에서 뒤로), `Default`, `Clone`(`T: Clone`), `PartialEq` / `Eq`(앞에서 뒤로 원소별), `Debug`를 구현합니다. `d += x`(`AddAssign<T>`)는 `x`를 뒤에 넣습니다.

## 2.9.5b. `BinaryHeap<T>`: 우선순위 큐

```rok
impl<T: Ord> BinaryHeap<T> {
    fn new() -> BinaryHeap<T>;                 // max-heap: 가장 큰 것이 먼저
}
impl<T> BinaryHeap<T> {
    fn by(before: fn(T, T) -> bool) -> BinaryHeap<T>;   // before(a, b) 이면 a 가 먼저
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn peek(self) -> T | none;
    fn push(self, x: T);
    fn pop(self) -> T | none;
    fn clear(self);
    fn retain(self, keep: fn(T) -> bool);
    fn iter(self) -> ListIter<T>;              // 순서 없음
}
```

`retain`은 제자리에서 걸러 낸 뒤 힙 순서를 되살리며 O(n)입니다. `IntoIterator`(`iter`처럼 순서 없음), `Debug`(저장 순서, 꼭대기가 먼저), `Clone`(`T: Clone`, 사본도 `by`의 순서를 유지)을 구현합니다. `T: Ord`이면 `Default`와 `FromIterator`가 `new`처럼 최대 힙을 만듭니다.

힙은 `Send`가 아닙니다. 순서가 `+ Send` 없는 `fn`이기 때문입니다. 원소를 다른 스레드로 넘기려면 `List`로 보내고, 받는 쪽에서 `collect()`로 `BinaryHeap`을 만듭니다.

## 2.9.6. `Tuple<...>`: 고정 길이 이종 묶음

```rok
#[literal( "(" expr "," {expr} % "," * ")" )]
struct Tuple { /* heterogeneous, 컴파일 타임 결정 */ }
```

튜플은 언어에 내장되어 있어 리터럴 핸들러로 등록할 수 없습니다([refs-lang 6.5.7](../../refs-lang/006-meta-system/605-literal-magic.md#657-사용자-정의-리터럴)). 튜플 식에는 콤마가 하나 이상 있어야 합니다. `(e)`는 그룹 식이고, 항이 하나인 튜플은 `(e,)`입니다.

위치마다 타입이 다를 수 있습니다. 길이는 컴파일 타임에 고정됩니다.

```rok
let t: Tuple<i32, str, bool> = (42, "hello", true);
let n = t.0;              //→ 42
let s = t.1;              //→ "hello"
let b = t.2;              //→ true

// 패턴 분해
let (n, s, b) = t;
```

### 트레잇 구현

| 트레잇       | 조건                       |
|--------------|----------------------------|
| `Clone`      | 모든 컴포넌트 `Clone`      |
| `Copy`       | 모든 컴포넌트 `Copy`       |
| `Debug`      | 모든 컴포넌트 `Debug`      |
| `PartialEq`  | 모든 컴포넌트 `PartialEq`  |
| `Eq`         | 모든 컴포넌트 `Eq`         |
| `PartialOrd` | 모든 컴포넌트 `PartialOrd` |
| `Ord`        | 모든 컴포넌트 `Ord`        |
| `Hash`       | 모든 컴포넌트 `Hash`       |

항이 1~12개인 튜플에는 `rok::core::tuple`이 표의 트레잇을 구현합니다. `PartialOrd` / `Ord`는 사전식입니다. 첫 항부터 비교하고 처음으로 다른 항이 결과를 정합니다. 그래서 `<` / `<=` / `>` / `>=`를 튜플에 바로 쓸 수 있습니다(`(1, "b") < (1, "c")`).

`Display`는 자동으로 파생되지 않습니다. 필요한 곳에서 직접 구현하기를 권장합니다.

### 함수 다중 반환

[refs-lang 4.3.5 다중 반환](../../refs-lang/004-functions/403-return.md#435-다중-반환-튜플로-묶기)의 다중 반환은 Tuple을 반환하는 것으로 처리합니다.

```rok
fn divmod(a: i32, b: i32) -> (i32, i32) {
    (a / b, a % b)
}

let (q, r) = divmod(17, 5);     //→ q = 3, r = 2
```

## 2.9.7. `Slice<T>`: 뷰

```rok
struct Slice<T> {
    // GC 버퍼 + 시작 위치 + 길이 + stride (1 = 연속, k = step, 음수 = 역순)
}
```

`xs[1:3]` / `xs[::2]` / `xs[1:]` 같은 식의 결과입니다([2.3.4 `Sliceable`](203-ops.md#234-인덱싱-트레잇)). 뷰는 원소를 복사하지 않고 원본의 GC 버퍼를 가리킵니다. 뷰가 살아 있는 동안 그 버퍼도 살아 있습니다.

| 원본       | 뷰가 보는 것                                                                    |
|------------|---------------------------------------------------------------------------------|
| `List<T>`  | 리스트의 현재 버퍼. 원소 쓰기는 보이고, 리스트가 자라 버퍼를 옮긴 뒤의 변경은 보이지 않음 |
| `Slice<T>` | 같은 버퍼(뷰의 뷰)                                                              |
| `[T; N]`   | 배열의 사본(배열은 값)                                                          |

### 핵심 메서드

```rok
impl<T> Slice<T> {
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn get(self, i: usize) -> T | none;
    fn first(self) -> T | none;
    fn last(self) -> T | none;
    fn step(self, k: isize) -> Slice<T>;     // k 번째마다, k < 0 이면 끝에서부터. k == 0 은 panic
    fn iter(self) -> SliceIter<T>;
    fn to_list(self) -> List<T>;             // 새 List 할당
}

impl Slice<u8> {
    fn find<B: AsBytes>(self, needle: B) -> usize | none;   // 처음 나타나는 offset; 빈 needle 은 0
}

impl<T> Index<usize> for Slice<T> { ... }               // s[i]: 범위 밖이면 panic
impl<T> Sliceable<Range<usize>> for Slice<T> { ... }    // s[a:b]: view 의 view, 범위는 길이에 맞춰 잘라 맞춤
impl<T> IntoIterator for Slice<T> { ... }
impl<T: PartialEq> PartialEq for Slice<T> { ... }
impl<T: Debug> Debug for Slice<T> { ... }
```

`find`의 needle은 `rok::core::collections::slice`에 정의된 `AsBytes` 타입이면 무엇이든 됩니다.

```rok
trait AsBytes {
    fn byte_slice(self) -> Slice<u8>;      // 바이트를 복사 없이 view 로
}
```

`str`(UTF-8 바이트), `List<u8>`, `Slice<u8>`, `[u8; N]`이 이 트레잇을 구현합니다. 그래서 바이트 문자열도 needle로 넘길 수 있습니다(바이트 문자열의 `Slice<u8>` 규칙은 [refs-lang 1.7.5](../../refs-lang/001-lexical/107-string-literals.md#175-바이트-문자열)). `std::io`는 `AsBytes`를 다시 내보냅니다.

`Slice`는 읽기 전용입니다(`IndexSet` 없음). 음수 인덱스는 끝에서부터 셉니다([refs-lang 3.6.1](../../refs-lang/003-expressions/306-indexing-and-slicing.md#361-인덱싱-arri)).

### 사용 예

```rok
let xs: List<i32> = [1, 2, 3, 4, 5];
let s1: Slice<i32> = xs[1:4];           //→ view of [2, 3, 4]
let s2: Slice<i32> = xs[::2];           //→ view of [1, 3, 5]
let s3: Slice<i32> = xs[..];            //→ view of all

s1.len();                               //→ 3
let t: i32 = s1.iter().sum();           //→ 9
xs[2] = 30;
s1[1];                                  //→ 30 (리스트의 원소 쓰기가 보임)

let owned: List<i32> = s1.to_list();    //→ [2, 30, 4] (새 List)
```

### 가변 인자와 통합

`fn f(xs: ...T)`의 `xs`는 `Slice<T>` 뷰입니다([refs-lang 4.2.5](../../refs-lang/004-functions/402-parameters.md#425-variadic-t)). 인자를 하나씩 넘겨도, 리스트를 펼쳐(`f(..ys)`) 넘겨도 같은 모양으로 받습니다.

```rok
fn sum(xs: ...i32) -> i32 {
    let total = 0;
    for x in xs {            // xs: Slice<i32>
        total = total + x;   // 재할당은 `let` 없이
    }
    total
}

sum(1, 2, 3);                //→ 6
sum(..[4, 5]);               //→ 9 (리스트 spread)
```

## 2.9.8. `Index` / `IndexSet` / `Sliceable` 트레잇 매핑

[2.3 ops](203-ops.md)의 트레잇이 컨테이너의 `[]`, `[i] = v`, `[a:b]`의 의미를 정의합니다.

| 컨테이너     | `Index<I>`                     | `IndexSet<I>`             | `Sliceable<R>`                          |
|--------------|--------------------------------|---------------------------|-----------------------------------------|
| `List<T>`    | `Index<usize>` → T             | `IndexSet<usize>` (Value = T) | `Sliceable<Range<usize>>` → `Slice<T>`  |
| `Dict<K, V>` | `Index<K>` → V                 | `IndexSet<K>` (Value = V) | **없음**                                |
| `Set<T>`     | **없음**                       | **없음**                  | **없음**                                |
| `Tuple<...>` | **컴파일러 직접** (`.0`/.1)    | **컴파일러 직접**         | **없음**                                |
| `[T; N]`     | `Index<usize>` → T             | `IndexSet<usize>` (Value = T) | `Sliceable<Range<usize>>` → `Slice<T>` (사본) |
| `Slice<T>`   | `Index<usize>` → T             | **없음**(읽기 전용 뷰)    | `Sliceable<Range<usize>>` → `Slice<T>`  |
| `str`        | **없음**(바이트 직접 접근 거부) | **없음**                 | `Sliceable<Range<usize>>` → `str` (사본, UTF-8 경계 검사) |

`str`의 `s[10]`은 UTF-8 멀티바이트 문자를 지키려고 일부러 두지 않았습니다. 문자 단위로 접근하려면 `s.chars()`를 쓰세요. 일부를 잘라 내려면 `s[10..15]`를 쓰세요. 슬라이스이므로 멀티바이트 경계를 검사합니다.

## 2.9.9. `IntoIterator` / `FromIterator`

모든 컨테이너는 `IntoIterator`를 구현합니다.

| 컨테이너            | `Item`            | 순서                              |
|---------------------|-------------------|-----------------------------------|
| `List<T>`           | `T`               | 첫 → 마지막                       |
| `Dict<K, V>`        | `(K, V)`          | **비결정**                        |
| `Set<T>`            | `T`               | **비결정**                        |
| `OrderedDict<K, V>` | `(K, V)`          | 삽입 순서                         |
| `OrderedSet<T>`     | `T`               | 삽입 순서                         |
| `Deque<T>`          | `T`               | 앞 → 뒤                           |
| `BinaryHeap<T>`     | `T`               | **순서 없음**(pop 순서 아님)      |
| `Tuple<T...>`       | **구현하지 않음** | (원소 타입이 제각각이라 반복자로 볼 수 없음) |
| `Slice<T>`          | `T`               | 뷰 순서(stride 적용)              |

`collect::<C>()`가 어떤 컨테이너를 만들지는 `FromIterator<T>`가 정합니다.

```rok
let v: List<i32> = (0..5).collect();
let s: Set<i32> = (0..5).collect();
let d: Dict<str, i32> = [("a", 1), ("b", 2)].into_iter().collect();
```

## 2.9.10. 모듈 매핑

| 컨테이너      | 모듈                                     | 내부 구조                           |
|---------------|------------------------------------------|-------------------------------------|
| `List<T>`     | `rok::core::collections::list`            | Rust `Vec`을 본뜸                   |
| `Dict<K, V>`  | `rok::core::collections::dict`            | 개방 주소법                         |
| `Set<T>`      | `rok::core::collections::set`             | `Dict<T, bool>` 위                  |
| `OrderedDict` | `rok::core::collections::ordered_dict`    | 연결 해시 또는 인덱스 벡터          |
| `OrderedSet`  | `rok::core::collections::ordered_set`     | (위와 같음)                         |
| `Deque<T>`    | `rok::core::collections::deque`           | 링 버퍼                             |
| `BinaryHeap<T>` | `rok::core::collections::heap`          | `List` 위의 암묵적 이진 트리        |
| `Tuple`       | **컴파일러 내장** + 표준 라이브러리 훅  | `#[literal(...)]` + 컴파일러 조회   |
| `Slice<T>`    | `rok::core::collections::slice`           | struct ptr + len + stride           |

## 2.9.11. 다른 언어와 비교

| 측면        | Rust                 | Python                     | Rok                                    |
|-------------|----------------------|----------------------------|----------------------------------------|
| 동적 배열   | `Vec<T>`             | `list`                     | `List<T>`                              |
| 해시 맵     | `HashMap<K, V>`      | `dict`(삽입 순서 유지)     | `Dict<K, V>` + `OrderedDict<K, V>`     |
| 해시 집합   | `HashSet<T>`         | `set`                      | `Set<T>` + `OrderedSet<T>`             |
| 고정 이종   | `(T1, T2)` 튜플      | `tuple`                    | `Tuple<T1, T2>`                        |
| 뷰          | `&[T]` / `&str`      | **slice 객체**             | `Slice<T>`                             |
| 리터럴      | `vec![...]` (매크로) | `[...]` `{...}` `(...)`    | `[...]` `{...}` `(...)` (`#[literal]`) |
| 순서 있는 맵 | `BTreeMap`(정렬 맵) | **기본 dict**              | 별도 `OrderedDict`                     |

Rok이 가장 크게 다른 점은 다음과 같습니다. 첫째, 컬렉션 6가지 모두에 1급 리터럴 문법이 있으므로 `vec![]` 같은 매크로를 쓰지 않습니다. 둘째, `Dict` / `Set`은 삽입 순서를 보장하지 않습니다(Python 3.7 이상과 다릅니다). 그래서 순서가 중요하면 `OrderedDict`를 직접 골라 써야 합니다. 셋째, `Slice<T>`는 stride가 있는 뷰이므로 `xs[::2]` 같은 식도 복사 없이 동작합니다.

## 2.9.12. `GcBuf<T>`: 원시 GC 버퍼

`rok::core::collections::raw`의 `GcBuf<T>`는 `List` / `Dict` 같은 컬렉션 구현이 쓰는 고정 길이 GC 버퍼입니다. 컴파일러가 아는 타입입니다. `GcBuf`는 `rok::bare`(`rok::bare::mem`)가 정의하고 여기서 다시 내보냅니다. 이 버퍼는 런타임의 워드 단위 추적 버퍼(type id `2`, [refs-lang 8.2.12](../../refs-lang/008-abi/802-scan-table.md#8212-런타임-예약-type-id)) 위에 놓입니다.

```rok
struct GcBuf<T> { /* 컴파일러가 아는 표현 */ }

impl<T> GcBuf<T> {
    #[intrinsic("rok.buf_new")]
    unsafe fn new(len: usize) -> GcBuf<T>;           // 모든 word 를 0 (null) 으로
    #[intrinsic("rok.buf_get")]
    unsafe fn get(self, i: usize) -> T;        // 경계 검사 없음 (길이는 소유자가 관리)
    #[intrinsic("rok.buf_set")]
    unsafe fn set(self, i: usize, v: T);       // write barrier 포함
}
```

원소 하나는 워드 하나를 차지합니다. 일반 코드는 `List<T>`를 씁니다. `GcBuf<T>`는 컬렉션을 구현할 때 씁니다.

`GcBuf<T>`와 그 기본 연산(`buf_new` / `buf_get` / `buf_set`, `new` / `get` / `set`, `GcBuf` 인덱싱)은 `unsafe`입니다. 컬렉션이 지키는 원소 타입 검사와 초기화 검사를 건너뛰기 때문입니다.

```rok
#[intrinsic("rok.buf_same")]
fn buf_same<T>(a: GcBuf<T>, b: GcBuf<T>) -> bool;   // 같은 buffer 인가 (내용이 아니라 정체). 아무것도 고정하지 않음
#[intrinsic("rok.buf_erase")]
pub(pkg) fn buf_erase<T>(buf: GcBuf<T>) -> GcBuf<u8>;   // 어떤 GC 객체든 받는 런타임 매개변수를 위해 GcBuf<u8> 로 타입을 바꿈
```
