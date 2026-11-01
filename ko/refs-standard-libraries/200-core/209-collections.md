# 2.9. 컬렉션: `List` / `Dict` / `Set` / `Tuple` / `Slice`

`rok::core::collections` 모듈은 범용 컨테이너를 정의합니다. 모든 컬렉션은 GC 힙에 할당됩니다. `#[literal(...)]`로 묶은 리터럴 문법으로 바로 만들 수 있습니다.

이 장은 다음 컬렉션과 `Deque<T>`, `BinaryHeap<T>`의 시그니처를 정의합니다.

| 컬렉션              | 리터럴 문법                | 의미                          |
|---------------------|----------------------------|-------------------------------|
| `List<T>`           | `[1, 2, 3]`                | 동적 배열(삽입 순서)          |
| `Dict<K, V, S>`     | `{"a": 1, "b": 2}`         | 해시 맵(삽입 순서)            |
| `Set<T, S>`         | `{1, 2, 3}`                | 해시 집합(삽입 순서)          |
| `Tuple<...>`        | `(1, "a", true)`           | 고정 크기, 원소 타입이 제각각 |
| `Slice<T>`          | `xs[1:3]`                  | 복사 없는 컨테이너 뷰         |
| `SliceMut<T>`       | `SliceMut`을 기대하는 자리의 `xs[1:3]` | 리스트에 쓰는 뷰 |

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

원소가 `Clone` / `Copy` / `Default` / `Debug` / `PartialEq` / `Eq` / `PartialOrd` / `Ord` / `Hash`를 구현하면 고정 배열도 그 트레잇을 구현합니다. 순서 비교는 사전식이므로 비교 연산자(`==`, `<` 등)를 바로 쓸 수 있습니다. 메서드는 다음과 같습니다.

```rok
impl<T, const N: usize> [T; N] {
    fn len(self) -> usize;                       // N
    fn is_empty(self) -> bool;
    fn iter(self) -> SliceIter<T>;               // 복사본 위에서 (배열은 값)
    fn first(self) -> T | none;
    fn last(self) -> T | none;
    fn map<U>(self, f: fn(T) -> U) -> [U; N];    // 새 배열
    fn fill(ref self, x: T);                     // 제자리
}

impl<T: PartialEq, const N: usize> [T; N] {
    fn contains(self, x: T) -> bool;
}
```

```rok
let a: [i32; 3] = [1, 2, 3];
a.len();                 //→ 3
a == [1, 2, 3];          //→ true
a < [1, 2, 4];           //→ true (사전식)
```

### 참조 핸들

`List`, `Dict`, `Set` 값은 GC 힙 객체를 가리키는 참조 핸들입니다. `let ys = xs;`나 인자 전달은 핸들을 복사하므로 같은 컬렉션을 공유합니다. 그래서 컬렉션을 바꾸는 메서드도 `ref self`가 아니라 `self`를 받습니다. 독립된 사본이 필요하면 `clone()`을 부릅니다. struct 원소는 값입니다. `xs[i]`와 `d[k]`는 사본을 돌려주므로 바꾼 값은 `xs[i] = v`로 저장합니다([refs-lang 2.3.8](../../refs-lang/002-types/203-struct.md#238-값-의미)). `iter` / `keys` / `values`는 컬렉션을 복사하지 않는 지연 반복자를 돌려줍니다.

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
    fn first(self) -> T | none;
    fn last(self) -> T | none;
    fn swap(self, i: usize, j: usize);          // 둘 중 하나라도 범위 밖이면 panic
    fn fill(self, x: T);                         // 모든 원소가 x 가 됨

    fn extend<I: IntoIterator<Item = T>>(self, iter: I);
    fn retain(self, keep: fn(T) -> bool);        // 제자리, 순서 유지
    fn sort(self) where T: Ord;                  // 오름차순, stable
    fn sort_by(self, cmp: fn(T, T) -> Ordering); // stable
    fn sort_by_key<K: Ord>(self, key: fn(T) -> K); // 키 기준 오름차순, stable
    fn reserve(self, additional: usize);
    fn shrink_to_fit(self);
    fn reverse(self);                            // 제자리

    fn split_at(self, mid: usize) -> (Slice<T>, Slice<T>);   // [0, mid) 와 [mid, len) 의 view; mid > len 이면 panic
    fn windows(self, size: usize) -> slice::Windows<T>;   // self[:].windows(size)
    fn chunks(self, size: usize) -> slice::Chunks<T>;     // self[:].chunks(size)
    fn partition_point(self, pred: fn(T) -> bool) -> usize;   // self[:].partition_point(pred)

    fn iter(self) -> ListIter<T>;
    fn into_iter(self) -> ListIntoIter<T>;
}

impl<T: PartialEq> List<T> {
    fn contains(self, x: T) -> bool;
    fn dedup(self);                              // 바로 앞 원소와 같은 원소를 버림
}

impl<T: Ord> List<T> {
    fn binary_search(self, x: T) -> usize | none;   // 오름차순 리스트에서 x 의 첫 인덱스, O(log n)
}
```

위치로 읽는 연산에는 `T | none` 형태가 있습니다(`get`, `first`, `last`, `pop`). 그 밖의 위치 연산은 범위를 벗어나면 panic이 일어납니다(`xs[i]`, `xs[i] = v`, `insert`, `remove`, `swap`, `drain`, `split_at`). `set`은 없습니다. 쓰기는 `xs[i] = v`이고, 범위 밖 인덱스를 건너뛰려면 먼저 검사합니다(`if i < xs.len() { xs[i] = v }`).

`partition_point(pred)`는 원소가 `pred`를 처음으로 만족하지 않는 인덱스입니다. `pred`는 리스트의 앞부분에서 참이고 나머지에서 거짓이어야 하며, O(log n)입니다. 오름차순 리스트에서 `x`가 들어갈 자리는 `xs.partition_point(|e| e < x)`이고, 키 기준이면 `xs.partition_point(|e| key(e) < k)`입니다. `binary_search(x)`는 `x`와 같은 원소 가운데 첫 인덱스(하한)를 돌려주고, 없으면 `none`입니다.

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
ys[100];                  //→ panic: index 100 out of bounds for length 3
ys.get(100);              //→ none

for x in ys {
    use(x);               //→ 10, 20, 30
}

let total: i32 = ys.into_iter().sum();   //→ 60

let zs: List<i32> = [3, 1, 1, 2, 2, 4];
zs.dedup();                                  //→ [3, 1, 2, 4]
zs.retain(|x| x != 3);                       //→ [1, 2, 4]
zs.binary_search(4);                         //→ 2
zs.partition_point(|x| x < 3);               //→ 2 (3이 들어갈 자리)
let (a, b) = zs.split_at(1);                 //→ [1] 과 [2, 4] 의 view
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
- `Dict<K, V, S>`와 `Set<T, S>`는 순서와 무관하게 비교하고 해시합니다. 든 원소(키와 값)가 같으면 같습니다. 원소마다 프로세스 키로 따로 해시해 결과를 더하므로([2.5.6](205-cmp.md#256-hash-해시eq-일관성-약속)) 해시도 `S`와 무관합니다.
- 세 컬렉션 모두 원소가 그 트레잇을 구현하면 `Debug`를 구현합니다. 출력은 순회 순서를 따릅니다.
- 따라서 원소가 `Eq + Hash`인 컬렉션은 `Set` 원소나 `Dict` 키가 될 수 있습니다(`Set<List<i32>>`).

### 용량과 제거: 공통 규칙

- 범위 밖 인덱스는 panic을 일으킵니다. 메시지에는 인덱스와 길이가 들어갑니다(`index 100 out of bounds for length 3`). 메서드는 메시지 앞에 자기 이름을 붙이고(`List::remove: index 5 out of bounds for length 3`, `List::drain: range 2..4 out of bounds for length 3`), panic 위치는 호출한 곳을 가리킵니다.
- `reserve`, `with_capacity`, `repeat`, 확장에서 크기가 넘치면 "capacity overflow" panic이 일어납니다. 최적화 수준과 관계없이 같습니다.
- 원소를 빼면(`pop`, `remove`, `clear`, `truncate`, `drain` 등) 그 참조도 놓습니다. 그 뒤로 GC는 그 컬렉션을 거쳐 그 원소에 닿지 않습니다.

## 2.9.3. `Dict<K, V, S>`: 해시 맵

```rok
struct Dict<K: Hash + Eq, V, S = RandomState> {
    // 내부: 삽입 순서로 놓인 원소 배열과 선형 탐사 슬롯의 인덱스
    // (작은 표는 인덱스 없음)
}
```

조회, 삽입, 제거는 평균 O(1)입니다. 순회는 삽입 순서를 따릅니다. 이미 있는 키를 다시 삽입하면 자리를 지키고, 제거한 뒤 다시 삽입한 키는 맨 뒤로 갑니다.

`S`가 해셔를 만듭니다([2.5.11](205-cmp.md#2511-rokcorehash-해셔)). 기본값 `RandomState`는 프로세스마다 한 번 뽑은 키를 쓰므로 키가 표의 어느 칸에 놓이는지는 실행마다 바뀝니다. 순회 순서는 바뀌지 않습니다. 상수 평가 중에 만든 dict는 런타임에 처음 쓸 때 프로세스 키로 다시 해시합니다. `Dict<K, V>`는 `Dict<K, V, RandomState>`이고, `{k: v}` 리터럴도 모두 그렇습니다.

### 핵심 메서드

```rok
impl<K: Hash + Eq, V> Dict<K, V> {
    fn new() -> Dict<K, V>;
    fn with_capacity(cap: usize) -> Dict<K, V>;
}

impl<K: Hash + Eq, V, S: BuildHasher> Dict<K, V, S> {
    fn with_hasher(s: S) -> Dict<K, V, S>;
    fn with_capacity_and_hasher(cap: usize, s: S) -> Dict<K, V, S>;
    fn hasher(self) -> S;

    fn len(self) -> usize;
    fn is_empty(self) -> bool;

    fn insert(self, k: K, v: V) -> V | none;       // 기존 V 반환 / 없으면 none
    fn remove(self, k: K) -> V | none;
    fn get(self, k: K) -> V | none;
    fn contains_key(self, k: K) -> bool;
    fn clear(self);
    fn retain(self, keep: fn(K, V) -> bool);      // 제자리
    fn extend<I: IntoIterator<Item = (K, V)>>(self, pairs: I);   // 뒤의 쌍의 값이 이김

    fn iter(self) -> DictIter<K, V, S>;              // (K, V) 쌍
    fn keys(self) -> Keys<K, V, S>;
    fn values(self) -> Values<K, V, S>;

    fn entry(self, k: K) -> Entry<K, V, S>;       // 그 키의 자리 (있든 없든)
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

`Default`, `FromIterator`, `From<List<(K, V)>>`는 `S: BuildHasher + Default`가 필요합니다. 나머지 구현은 모든 `S: BuildHasher`를 받습니다.

```rok
use rok::core::hash::BuildHasherDefault;

let fixed: Dict<str, i32, BuildHasherDefault<Fnv>> = Dict::with_hasher(BuildHasherDefault::new());
```

```rok
let counts: Dict<str, i32> = Dict::new();   // `{}`는 빈 Dict가 아니라 빈 블록
for w in words {
    counts.entry(w).and_modify(|n| n + 1).or_insert(1);
}
```

### 사용 예

```rok
let d: Dict<str, i32> = {"a": 1, "b": 2};
d["a"];                   //→ 1
d["c"];                   //→ panic: key "c" not found; use `d.get(k)` to get `i32 | none`
d.get("c");               //→ none

d.insert("c", 3);
d.contains_key("c");      //→ true

for (k, v) in d.iter() {
    log("{k} = {v}");
}
```

### `Index<K>`

```rok
impl<K: Hash + Eq, V> Index<K> for Dict<K, V> {
    type Output = V;
    fn index(self, k: K) -> V {
        self.get(k) ?? panic(f"key {k:?} not found; use `d.get(k)` to get `V | none`")
    }
}
```

`d[k]` 표기는 키가 있다고 가정합니다. 안전하게 접근하려면 `.get(k)`를 쓰세요. panic 메시지는 키를 `{k:?}`처럼 보여 줍니다. `K`에 `Debug`가 없으면 타입 이름 `<K>`로 보여 줍니다. `V` 자리에는 값 타입의 이름이 들어갑니다.

`d += e`(`AddAssign<Dict<K, V>>`)는 `e`의 모든 항목을 `d`에 제자리에서 삽입합니다. 키가 겹치면 `e`의 값이 남습니다.

`Dict::from(pairs)`(`From<List<(K, V)>>`)는 쌍의 리스트로 dict를 만듭니다. 키가 겹치면 뒤의 값이 남습니다.

### lazy view

`iter` / `keys` / `values`는 사본이 아닙니다. 현재 테이블을 그대로 보는 지연 뷰입니다. 순회 도중 dict를 바꾸면 결과는 사양이 정하지 않습니다. 원소를 빠뜨리거나 두 번 볼 수 있습니다. 그래도 메모리 안전성은 지켜집니다.

## 2.9.4. `Set<T, S>`: 해시 집합

```rok
struct Set<T: Hash + Eq, S = RandomState> {
    // 내부: Dict<T, bool, S>
}
```

`Dict`처럼 삽입 순서로 순회합니다. 연산은 평균 O(1)입니다. `Dict`처럼 `S`가 해셔를 만듭니다.

```rok
impl<T: Hash + Eq> Set<T> {
    fn new() -> Set<T>;
}

impl<T: Hash + Eq, S: BuildHasher> Set<T, S> {
    fn with_hasher(s: S) -> Set<T, S>;
    fn with_capacity_and_hasher(cap: usize, s: S) -> Set<T, S>;
    fn hasher(self) -> S;
    fn len(self) -> usize;
    fn is_empty(self) -> bool;

    fn insert(self, x: T) -> bool;        // true if 새로 들어감
    fn remove(self, x: T) -> bool;
    fn contains(self, x: T) -> bool;
    fn clear(self);
    fn retain(self, keep: fn(T) -> bool);         // 제자리
    fn extend<I: IntoIterator<Item = T>>(self, items: I);

    fn union(self, other: Set<T, S>) -> Set<T, S>;
    fn intersection(self, other: Set<T, S>) -> Set<T, S>;
    fn difference(self, other: Set<T, S>) -> Set<T, S>;
    fn symmetric_difference(self, other: Set<T, S>) -> Set<T, S>;

    fn is_subset(self, other: Set<T, S>) -> bool;
    fn is_superset(self, other: Set<T, S>) -> bool;
    fn is_disjoint(self, other: Set<T, S>) -> bool;

    fn iter(self) -> SetIter<T, S>;
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

## 2.9.5. 삽입 순서

`Dict`와 `Set`은 Python 3.7 이상의 `dict`나 Java의 `LinkedHashMap`처럼 삽입 순서로 순회합니다. 그래서 순서를 지키는 타입을 따로 두지 않습니다. `iter`, `keys`, `values`, `to_list`, `Debug`, `for`가 모두 이 순서를 따릅니다.

- 이미 있는 키를 다시 넣으면 자리는 그대로이고 값만 바뀝니다.
- 지웠다가 다시 넣은 키는 맨 뒤로 갑니다.
- `retain`은 남는 항목의 순서를 지키고, `extend`와 `d += e`는 순서대로 삽입합니다(새 키는 맨 뒤로).
- 집합 연산의 결과는 왼쪽 피연산자의 순서를 먼저 따릅니다.

같음 비교와 해시는 순서를 보지 않습니다([2.9.2](#292-listt-동적-배열)). 그래서 `{"a": 1, "b": 2} == {"b": 2, "a": 1}`입니다.

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
    fn front(self) -> T | none;
    fn back(self) -> T | none;
    fn clear(self);
    fn iter(self) -> DequeIter<T>;             // 앞에서 뒤로
}
```

`set`은 없습니다. 쓰기는 `d[i] = v`입니다. `capacity`는 링 버퍼가 커지기 전까지 담을 수 있는 원소 수입니다. 이 타입은 `d[i]` / `d[i] = v`를 구현합니다. 범위 밖이면 `List`처럼 panic이 일어납니다(`index 5 out of bounds for length 3`). 그 밖에 `IntoIterator`, `FromIterator`(앞에서 뒤로), `Default`, `Clone`(`T: Clone`), `PartialEq` / `Eq`(앞에서 뒤로 원소별), `Debug`를 구현합니다. `d += x`(`AddAssign<T>`)는 `x`를 뒤에 넣습니다.

## 2.9.5b. `BinaryHeap<T>`: 우선순위 큐

```rok
impl<T: Ord> BinaryHeap<T> {
    fn new() -> BinaryHeap<T>;                 // max-heap: 가장 큰 것이 먼저
}
impl<T> BinaryHeap<T> {
    fn by_ordering(cmp: fn(T, T) -> Ordering + Send) -> BinaryHeap<T>; // cmp 기준 가장 큰 것이 먼저, new()가 Ord::cmp 기준인 것처럼
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn peek(self) -> T | none;
    fn push(self, x: T);
    fn pop(self) -> T | none;
    fn clear(self);
    fn retain(self, keep: fn(T) -> bool);
    fn iter(self) -> HeapIter<T>;              // 순서 없음
}
```

`retain`은 제자리에서 걸러 낸 뒤 힙 순서를 되살리며 O(n)입니다. `IntoIterator`(`iter`처럼 순서 없음), `Debug`(저장 순서, 꼭대기가 먼저), `Clone`(`T: Clone`, 사본도 힙의 순서를 유지)을 구현합니다. `T: Ord`이면 `Default`와 `FromIterator`가 `new`처럼 최대 힙을 만듭니다.

비교 함수는 `List::sort_by`의 것처럼 `Ordering`을 반환합니다. `by_ordering(|a, b| b.cmp(a))`는 가장 작은 것을 먼저 꺼냅니다.

비교 함수는 `fn .. + Send`이므로, `T`가 `Send`이면 힙도 `Send`입니다. 채널로 보내거나 `Mutex`에 넣을 수 있습니다. 비교 함수가 `Send + Sync`가 아닌 값(컬렉션 등)을 캡처하면 컴파일 에러이고, help는 캡처를 없애라고 안내합니다. 값을 클로저 안에서 계산하거나 배열을 캡처합니다.

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
    fn split_at(self, mid: usize) -> (Slice<T>, Slice<T>);   // [0, mid) 와 [mid, len); mid > len 이면 panic
    fn windows(self, size: usize) -> slice::Windows<T>;   // size 개씩 겹치는 view; size == 0 은 panic
    fn chunks(self, size: usize) -> slice::Chunks<T>;     // size 개씩의 view, 마지막은 더 짧음; size == 0 은 panic
    fn iter(self) -> SliceIter<T>;
    fn to_list(self) -> List<T>;             // 새 List 할당
}

impl<T> Slice<T> {
    fn partition_point(self, pred: fn(T) -> bool) -> usize;   // 원소가 pred 를 처음 만족하지 않는 인덱스, O(log n)
}

impl<T: PartialEq> Slice<T> {
    fn contains(self, x: T) -> bool;
}

impl<T: Ord> Slice<T> {
    fn binary_search(self, x: T) -> usize | none;   // 오름차순 뷰에서 x 의 첫 인덱스, O(log n)
}

impl Slice<u8> {
    fn find<B: AsBytes>(self, needle: B) -> usize | none;   // 처음 나타나는 offset; 빈 needle 은 0
}

impl<T> Index<usize> for Slice<T> { ... }               // s[i]: 범위 밖이면 panic (`slice index 5 out of bounds for length 3`)
impl<T> Sliceable<Range<usize>> for Slice<T> { ... }    // s[a:b]: view 의 view, 범위는 길이에 맞춰 잘라 맞춤
impl<T> IntoIterator for Slice<T> { ... }
impl<T: PartialEq> PartialEq for Slice<T> { ... }
impl<T: Debug> Debug for Slice<T> { ... }
```

`slice::Windows<T>`와 `slice::Chunks<T>`는 `Item = Slice<T>`인 `ExactSizeIterator`입니다. 각 창(window)과 덩어리(chunk)는 복사본이 아니라 같은 버퍼의 뷰입니다. `Iterator::windows` / `chunks`([2.7.1](207-iter.md#271-iterator-단계-진행))는 버퍼가 없는 반복자에서 동작하므로 대신 리스트로 모읍니다.

`find`의 needle은 `AsBytes`([2.11.13](211-conversion.md#21113-asbytes-트레잇-바이트-뷰)) 타입이면 무엇이든 됩니다.

```rok
trait AsBytes {
    fn byte_slice(self) -> Slice<u8>;      // 바이트를 복사 없이 view 로
}
```

`str`(UTF-8 바이트), `List<u8>`, `Slice<u8>`, `[u8; N]`이 이 트레잇을 구현합니다. 그래서 바이트 문자열도 needle로 넘길 수 있습니다(바이트 문자열의 `Slice<u8>` 규칙은 [refs-lang 1.7.5](../../refs-lang/001-lexical/107-string-literals.md#175-바이트-문자열)).

`Slice`는 읽기 전용입니다(`IndexSet` 없음). 리스트에 쓰는 뷰는 `SliceMut<T>`입니다([2.9.7a](#297a-slicemutt-쓰기-가능한-뷰)). 음수 인덱스는 끝에서부터 셉니다([refs-lang 3.6.1](../../refs-lang/003-expressions/306-indexing-and-slicing.md#361-인덱싱-arri)).

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

## 2.9.7a. `SliceMut<T>`: 쓰기 가능한 뷰

```rok
struct SliceMut<T> {
    // 리스트의 버퍼 + 시작 위치 + 길이, 그리고 뷰를 만들 때의 리스트와 그 버전
}
```

`SliceMut<T>`는 보고 있는 리스트에 쓰는 연속 뷰입니다. Rust의 `&mut [T]`에 해당합니다. 리스트의 일부를 채우거나 순서를 바꾸는 함수가 이 타입을 받습니다(`fn zero(buf: SliceMut<u8>)`). `SliceMut<T>`를 기대하는 자리에서 만들어집니다. 인자, 타입을 적은 `let`, 반환값이 그런 자리입니다.

| 그 자리에 적은 것               | 뷰                                                                       |
|---------------------------------|--------------------------------------------------------------------------|
| `xs`(`List<T>`)                 | 리스트 전체([refs-lang 2.14.6](../../refs-lang/002-types/214-cast-and-conversion.md#2146-문맥-기반-자동-변환)) |
| `xs[a:b]`, `xs[a..b]`, `xs[..]` | 리스트의 그 범위. 경계는 `Slice`처럼 잘라 맞춤([refs-lang 3.6.2](../../refs-lang/003-expressions/306-indexing-and-slicing.md#362-슬라이싱-arrstartend)) |
| `s[a:b]`(`SliceMut<T>`)         | 뷰의 쓰기 가능한 뷰                                                      |

다른 자리에서 `xs[a:b]`는 그대로 읽기 전용 `Slice<T>`입니다. 간격 있는 범위(`xs[a:b:k]`)는 연속이 아니므로 `SliceMut`이 되지 않습니다.

### 핵심 메서드

```rok
impl<T> SliceMut<T> {
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn get(self, i: usize) -> T | none;
    fn first(self) -> T | none;
    fn last(self) -> T | none;
    fn fill(self, x: T);
    fn swap(self, i: usize, j: usize);                 // 범위 밖이면 panic
    fn reverse(self);
    fn copy_from_slice(self, src: Slice<T>);           // src.len() != len 이면 panic; src 가 뷰와 겹쳐도 됨
    fn split_at(self, mid: usize) -> (SliceMut<T>, SliceMut<T>);   // [0, mid) 와 [mid, len), 둘 다 쓰기 가능; mid > len 이면 panic
    fn sort_by(self, cmp: fn(T, T) -> Ordering);       // 안정 정렬
    fn sort_by_key<K: Ord>(self, key: fn(T) -> K);     // 안정 정렬
    fn as_slice(self) -> Slice<T>;                     // 같은 원소의 읽기 전용 뷰
    fn iter(self) -> SliceMutIter<T>;
    fn to_list(self) -> List<T>;                       // 새 List 할당
    unsafe fn raw_parts(self) -> (GcBuf<T>, usize, usize);   // (버퍼, 시작, 길이), 영역을 채우는 런타임 호출용
}

impl<T: Ord> SliceMut<T> {
    fn sort(self);                                     // 오름차순, 안정 정렬
}

impl<T: PartialEq> SliceMut<T> {
    fn contains(self, x: T) -> bool;
}

impl<T> Index<usize> for SliceMut<T> { ... }              // s[i]: 범위 밖이면 panic
impl<T> IndexSet<usize> for SliceMut<T> { ... }           // s[i] = x 는 리스트의 i 번째 원소에 씀
impl<T> Sliceable<Range<usize>> for SliceMut<T> { ... }   // s[a:b]: 뷰의 쓰기 가능한 뷰, 범위는 잘라 맞춤
impl<T> IntoIterator for SliceMut<T> { ... }
impl<T: PartialEq> PartialEq for SliceMut<T> { ... }
impl<T: Debug> Debug for SliceMut<T> { ... }
```

`SliceMutIter<T>`는 `Item = T`인 `DoubleEndedIterator`이자 `ExactSizeIterator`입니다. `SliceMut`은 `List`처럼 `Sync`가 아닙니다([2.13.9](213-marker.md#2139-send--sync-스레드-경계-마커)). 넘기기(hand-off)는 리스트를 복사하고, 뷰는 그 사본을 봅니다. 음수 인덱스는 리스트와 같이 끝에서부터 셉니다.

### 뷰가 살아 있는 동안 리스트 크기 바꾸기

크기를 바꾸면 리스트의 원소가 새 버퍼로 옮겨질 수 있습니다. 그래서 리스트의 크기를 바꾼 뒤에는 그 전에 만든 뷰를 쓰지 않습니다. 길이나 버퍼를 바꾸는 `List` 메서드는 모두 크기 바꾸기입니다. `push`, `pop`, `insert`, `remove`, `clear`, `truncate`, `drain`, `set_len`, `reserve`, `shrink_to_fit`, `extend`, `retain`, `dedup`, `+=`가 그렇습니다. 원소 쓰기(`xs[i] = x`, `fill`, `swap`, `sort`, `reverse`)는 크기 바꾸기가 아닙니다.

- 한 함수 안에서 둘 다 보이면, 리스트의 크기를 바꾼 뒤 뷰를 쓰는 것은 컴파일 에러 `E202`입니다([refs-lang 9.9.4](../../refs-lang/009-tools/909-debug-tools.md#994-진단--에러-메시지)). 컴파일러는 리스트의 복사(`let ys = xs`), 뷰의 뷰(`s[1:]`, `split_at`), 뷰를 도는 `for` 루프, 분기, 루프의 다음 회차까지 뷰를 따라갑니다. 뷰를 다시 대입하면(`s = xs[..]`) 다시 쓸 수 있고, 빠져나가는 경로(`return`, `break`)의 크기 바꾸기는 그 뒤의 코드에 닿지 않습니다.
- 나머지는 실행 시간에 검사합니다. 호출한 함수 안의 크기 바꾸기, 구조체에 넣어 둔 뷰, 호출자가 크기를 바꾸는 동안 기다리는 `async fn`이 그런 경우입니다. 리스트는 크기를 바꿀 때마다 올라가는 버전을 들고 있고, 그 전에 만든 뷰는 다음에 쓸 때 panic합니다(`SliceMut used after its list was resized (push, insert, clear, ...): make the view again after the resize`).

```rok
let xs: List<i32> = [1, 2, 3];
let s: SliceMut<i32> = xs[1:];
xs.push(4);
s[0] = 9;          // error[E202]: `s` views `xs`, which `xs.push(..)` resized at line 3
```

실행 시간 검사가 있으므로 `SliceMut`은 `async fn`의 인자가 될 수 있고 `await`을 넘어 살아 있을 수 있습니다.

### 사용 예

```rok
fn zero(buf: SliceMut<u8>) {
    buf.fill(0);
}

let xs: List<i32> = [5, 4, 3, 2, 1];
let mid: SliceMut<i32> = xs[1:4];
mid.sort();                             // xs 는 [5, 2, 3, 4, 1]
mid[0] = 20;                            // xs 는 [5, 20, 3, 4, 1]
let (a, b) = mid.split_at(1);
b.fill(0);                              // xs 는 [5, 20, 0, 0, 1]

let bytes: List<u8> = [1, 2, 3];
zero(bytes[1:]);                        // [1, 0, 0]
zero(bytes);                            // 리스트 전체: [0, 0, 0]
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
| `SliceMut<T>` | `Index<usize>` → T            | `IndexSet<usize>`(Value = T) | `Sliceable<Range<usize>>` → `SliceMut<T>` |
| `str`        | **없음**(바이트 직접 접근 거부) | **없음**                 | `Sliceable<Range<usize>>` → `str` (사본, UTF-8 경계 검사) |

`str`의 `s[10]`은 UTF-8 멀티바이트 문자를 지키려고 일부러 두지 않았습니다. 문자 단위로 접근하려면 `s.chars()`를 쓰세요. 일부를 잘라 내려면 `s[10..15]`를 쓰세요. 슬라이스이므로 멀티바이트 경계를 검사합니다.

## 2.9.9. `IntoIterator` / `FromIterator`

모든 컨테이너는 `IntoIterator`를 구현합니다.

| 컨테이너            | `Item`            | 순서                              |
|---------------------|-------------------|-----------------------------------|
| `List<T>`           | `T`               | 첫 → 마지막                       |
| `Dict<K, V>`        | `(K, V)`          | 삽입 순서                         |
| `Set<T>`            | `T`               | 삽입 순서                         |
| `Deque<T>`          | `T`               | 앞 → 뒤                           |
| `BinaryHeap<T>`     | `T`               | **순서 없음**(pop 순서 아님)      |
| `Tuple<T...>`       | **구현하지 않음** | (원소 타입이 제각각이라 반복자로 볼 수 없음) |
| `Slice<T>`          | `T`               | 뷰 순서(stride 적용)              |
| `SliceMut<T>`       | `T`               | 뷰 순서                           |

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
| `Dict<K, V, S>` | `rok::core::collections::dict`          | 원소 배열 + 선형 탐사               |
| `Set<T, S>`   | `rok::core::collections::set`             | `Dict<T, bool, S>` 위               |
| `Deque<T>`    | `rok::core::collections::deque`           | 링 버퍼                             |
| `BinaryHeap<T>` | `rok::core::collections::heap`          | `List` 위의 암묵적 이진 트리        |
| `Tuple`       | **컴파일러 내장** + 표준 라이브러리 훅  | `#[literal(...)]` + 컴파일러 조회   |
| `Slice<T>`    | `rok::core::collections::slice`           | struct ptr + len + stride           |
| `SliceMut<T>` | `rok::core::collections::slice`           | 버퍼 + 시작 + 길이, 리스트 + 버전   |

## 2.9.11. 다른 언어와 비교

| 측면        | Rust                 | Python                     | Rok                                    |
|-------------|----------------------|----------------------------|----------------------------------------|
| 동적 배열   | `Vec<T>`             | `list`                     | `List<T>`                              |
| 해시 맵     | `HashMap<K, V>`      | `dict`(삽입 순서 유지)     | `Dict<K, V>`(삽입 순서 유지)           |
| 해시 집합   | `HashSet<T>`         | `set`                      | `Set<T>`(삽입 순서 유지)               |
| 고정 이종   | `(T1, T2)` 튜플      | `tuple`                    | `Tuple<T1, T2>`                        |
| 뷰          | `&[T]` / `&str`      | **slice 객체**             | `Slice<T>`                             |
| 쓰기 뷰     | `&mut [T]`           | **없음**(슬라이스 대입)    | `SliceMut<T>`                          |
| 리터럴      | `vec![...]` (매크로) | `[...]` `{...}` `(...)`    | `[...]` `{...}` `(...)` (`#[literal]`) |
| 순서 있는 맵 | `BTreeMap`(정렬 맵) | **기본 dict**              | **기본 `Dict`**                        |

Rok이 가장 크게 다른 점은 다음과 같습니다. 첫째, `List`, `Dict`, `Set`, `Tuple`, `Slice`에 1급 리터럴 문법이 있으므로 `vec![]` 같은 매크로를 쓰지 않습니다. 둘째, `Dict` / `Set`은 Python 3.7 이상의 `dict`처럼 삽입 순서로 순회합니다. 그래서 순서를 지키는 타입을 따로 둘 필요가 없습니다. 셋째, `Slice<T>`는 stride가 있는 뷰이므로 `xs[::2]` 같은 식도 복사 없이 동작합니다.

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
