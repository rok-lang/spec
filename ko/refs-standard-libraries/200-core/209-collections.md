# 2.9. 컬렉션 — `List` / `Dict` / `Set` / `Tuple` / `Slice`

`rok::core::collections` 모듈은 범용 컨테이너를 제공합니다. 모든 컬렉션은 GC heap 에 할당되며, `#[literal(...)]` 로 바인딩된 리터럴 문법을 통해 직접 생성할 수 있습니다.

이 장에서는 다음 6 가지 컬렉션의 권위 시그니처를 정의합니다.

| 컬렉션              | 리터럴 문법                | 의미                          |
|---------------------|----------------------------|-------------------------------|
| `List<T>`           | `[1, 2, 3]`                | 동적 배열 (insertion order)   |
| `Dict<K, V>`        | `{"a": 1, "b": 2}`         | hash map (HashMap, 순서 없음) |
| `Set<T>`            | `{1, 2, 3}`                | hash set (순서 없음)          |
| `OrderedDict<K, V>` | `OrderedDict::from([...])` | insertion-order 보존 dict     |
| `OrderedSet<T>`     | `OrderedSet::from([...])`  | insertion-order 보존 set      |
| `Tuple<...>`        | `(1, "a", true)`           | 고정 크기 heterogeneous       |
| `Slice<T>`          | `xs[1:3]`                  | 컨테이너의 zero-copy view     |

## 2.9.1. 리터럴 문법 — `#[literal(query)]` 바인딩

리터럴 문법은 [refs-lang 6.5 `#[literal]`](../../refs-lang/006-meta-system/605-literal-magic.md) 의 **query DSL** 로 다음과 같이 정의됩니다.

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

리터럴 query 문법의 의미는 다음과 같이 정리할 수 있습니다. 먼저 `{...} % sep *` 는 0 회 이상 sep 으로 구분된 반복을 나타내고, `{...} % sep +` 는 1 회 이상의 반복을 의미하며, 원소가 하나인 Set 은 후행 콤마로 `{1,}` 라고 써서 block `{1}` 과 구분합니다. 토큰열 매칭의 경우 lexer 단계에서 직접 구분한 뒤 parser 가 query 매칭을 수행합니다.

### Dict / Set 모호성 해결

```rok
{"a": 1}      // Dict — `:` 패턴
{1, 2}        // Set — 값만
{1,}          // Set (단일 원소 — 후행 콤마)
{1}           // 항상 block — 슬롯이 Set 이어도 마찬가지
{ x = 1 }     // block expression — `{` 다음 토큰이 statement-like
```

parser 가 내용만 보고 결정합니다. `expr ":"` 형태이면 Dict, 콤마가 있으면 Set, 그 밖에는 block 입니다. 받는 슬롯의 타입은 판정에 쓰지 않습니다.

### `[...]` 의 타입

`[1, 2, 3]` 은 기대 타입이 없으면 `List<T>` 입니다. 고정 크기 배열 `[T; N]` 은 `let a: [i32; 3] = [1, 2, 3];` 처럼 기대 타입을 `[T; N]` 으로 적었을 때만 만들어집니다.

### 참조 핸들

`List`, `Dict`, `Set` 값은 GC heap 객체를 가리키는 참조 핸들입니다. `let ys = xs;` 나 인자 전달은 핸들을 복사하므로 같은 컬렉션을 공유합니다. 그래서 컬렉션을 바꾸는 메서드도 `ref self` 가 아니라 `self` 를 받습니다. 독립된 사본이 필요하면 `clone()` 을 부릅니다. `keys` / `values` / `entries` 는 컬렉션을 복사하지 않는 lazy iterator 를 돌려줍니다.

## 2.9.2. `List<T>` — 동적 배열

```rok
struct List<T> {
    // 내부: ptr + len + capacity
}
```

내부적으로는 연속 메모리 buffer 와 `len` / `capacity` 로 구성되며, amortized O(1) push/pop 와 O(1) random access 를 제공합니다.

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

    fn get(self, idx: usize) -> T | none;       // safe — out-of-bounds → none
    fn first(self) -> T | none;
    fn last(self) -> T | none;

    fn extend<I: IntoIterator<Item = T>>(self, iter: I);
    fn reserve(self, additional: usize);
    fn shrink_to_fit(self);

    fn iter(self) -> ListIter<T>;
    fn into_iter(self) -> ListIntoIter<T>;
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
| `IndexSet<usize, T>`  | 항상                | `xs[i] = v`                     |
| `Slice<Range<usize>>` | 항상                | `xs[1:3]`                       |
| `Display`             | **아님**            | **수동 출력 권장**              |
| `Debug`               | `T: Debug`          | `f"{xs:?}"` → `[1, 2, 3]`       |
| `Clone`               | `T: Clone`          | 깊은 복제                       |
| `PartialEq`           | `T: PartialEq`      | element-wise                    |
| `Eq`/`Ord`/`Hash`     | T 의 같은 트레잇 시 | lexicographic                   |

## 2.9.3. `Dict<K, V>` — hash map

```rok
struct Dict<K: Hash + Eq, V> {
    // 내부: HashMap (open addressing 또는 chain)
}
```

평균적으로 O(1) lookup / insert / remove 성능을 제공하며, 순서는 보장하지 않으므로 iteration 순서가 비결정적입니다.

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

    fn keys(self) -> KeysIter<K, V>;
    fn values(self) -> ValuesIter<K, V>;
    fn entries(self) -> EntriesIter<K, V>;            // (K, V) 쌍

    fn entry(self, k: K) -> Entry<K, V>;          // or_insert / and_modify
}
```

### 사용 예

```rok
let d: Dict<str, i32> = {"a": 1, "b": 2};
d["a"];                   //→ 1
d["c"];                   //→ panic (IndexError — 키 없음)
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

`d[k]` 표기는 키의 존재가 보장됨을 가정하므로, 안전한 접근이 필요할 때는 `.get(k)` 를 사용합니다.

## 2.9.4. `Set<T>` — hash set

```rok
struct Set<T: Hash + Eq> {
    // 내부: Dict<T, ()>
}
```

순서는 보장되지 않으며, 평균 O(1) 성능을 제공합니다.

```rok
impl<T: Hash + Eq> Set<T> {
    fn new() -> Set<T>;
    fn len(self) -> usize;
    fn is_empty(self) -> bool;

    fn insert(self, x: T) -> bool;        // true if 새로 들어감
    fn remove(self, x: T) -> bool;
    fn contains(self, x: T) -> bool;

    fn union(self, other: Set<T>) -> Set<T>;
    fn intersection(self, other: Set<T>) -> Set<T>;
    fn difference(self, other: Set<T>) -> Set<T>;
    fn symmetric_difference(self, other: Set<T>) -> Set<T>;

    fn is_subset(self, other: Set<T>) -> bool;
    fn is_superset(self, other: Set<T>) -> bool;
    fn is_disjoint(self, other: Set<T>) -> bool;
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

## 2.9.5. `OrderedDict<K, V>` / `OrderedSet<T>` — insertion-order

이들은 `Dict` / `Set` 와 동일한 API 를 제공하면서도 **insertion-order 를 보존**한다는 점이 특징이며, Python 3.7+ 의 `dict` 나 Java 의 LinkedHashMap 과 동등한 의미를 가집니다.

```rok
struct OrderedDict<K: Hash + Eq, V> { /* ... */ }
struct OrderedSet<T: Hash + Eq> { /* ... */ }
```

config, JSON-like 구조, 출력 reproducibility 처럼 순서 보존이 필요한 상황에서 사용하면 적합하며, linked list 또는 indices vector 사용으로 인해 약간의 오버헤드가 발생합니다.

### 생성

리터럴 문법은 별도로 두지 않으며, 생성자 호출 형태로만 만듭니다.

```rok
let d = OrderedDict::from([("a", 1), ("b", 2)]);
let s = OrderedSet::from([1, 2, 3]);
```

## 2.9.6. `Tuple<...>` — 고정 heterogeneous

```rok
#[literal( "(" expr "," {expr} % "," * ")" )]
struct Tuple { /* heterogeneous, 컴파일 타임 결정 */ }
```

각 위치별로 서로 다른 타입을 가질 수 있으며, 길이는 컴파일 타임에 고정됩니다.

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

`Display` 는 자동으로 도출되지 않으므로 필요할 때마다 직접 구현하는 편이 권장됩니다.

### 함수 다중 반환

[refs-lang 4.3.5 다중 반환](../../refs-lang/004-functions/403-return.md#435-다중-반환-튜플로-묶기) 의 multi-return 은 Tuple 반환으로 처리됩니다.

```rok
fn divmod(a: i32, b: i32) -> (i32, i32) {
    (a / b, a % b)
}

let (q, r) = divmod(17, 5);     //→ q = 3, r = 2
```

## 2.9.7. `Slice<T>` — zero-copy view

```rok
struct Slice<T> {
    ptr: ConstPtr<T>;
    len: usize;
    stride: isize;       // 1 = contiguous, n = step (`xs[::n]`), -1 = reverse
}
```

`xs[1:3]` / `xs[::2]` / `xs[1:]` 와 같은 표현이 desugar 된 결과이며, 별도의 할당 없이 원본의 view 로 동작합니다.

### 핵심 메서드

```rok
impl<T> Slice<T> {
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn iter(self) -> SliceIter<T>;
    fn to_list(self) -> List<T>;             // 새 List 할당
    fn first(self) -> T | none;
    fn last(self) -> T | none;
    fn get(self, i: usize) -> T | none;
}
```

### 사용 예

```rok
let xs: List<i32> = [1, 2, 3, 4, 5];
let s1: Slice<i32> = xs[1:4];           //→ view of [2, 3, 4]
let s2: Slice<i32> = xs[::2];           //→ view of [1, 3, 5]
let s3: Slice<i32> = xs[..];            //→ view of all

s1.len();                               //→ 3
s1.iter().sum::<i32>();                 //→ 9

let owned: List<i32> = s1.to_list();    //→ [2, 3, 4] (새 List)
```

### variadic args 와의 통합

`fn f(xs: ...T)` 의 `xs` 는 `Slice<T>` 형태로 zero-alloc 뷰를 제공합니다.

```rok
fn sum(xs: ...i32) -> i32 {
    let total = 0;
    for x in xs {            // xs: Slice<i32>
        total = total + x;   // 재할당은 `let` 없이
    }
    total
}

sum(1, 2, 3);                //→ 6 (스택 임시 배열 + Slice 전달)
```

## 2.9.8. `Index` / `IndexSet` / `Slice` 트레잇 매핑

[2.3 ops](203-ops.md) 의 트레잇이 컨테이너의 `[]`, `[i] = v`, `[a:b]` 의 의미를 정의합니다.

| 컨테이너     | `Index<I>`                     | `IndexSet<I, V>`          | `Slice<R>`                              |
|--------------|--------------------------------|---------------------------|-----------------------------------------|
| `List<T>`    | `Index<usize>` → T             | `IndexSet<usize, T>`      | `Slice<Range<usize>>`                   |
| `Dict<K, V>` | `Index<K>` → V                 | `IndexSet<K, V>`          | **없음**                                |
| `Set<T>`     | **없음**                       | **없음**                  | **없음**                                |
| `Tuple<...>` | **컴파일러 직접** (`.0`/.1)    | **컴파일러 직접**         | **없음**                                |
| `Slice<T>`   | `Index<usize>` → T             | **없음** (read-only view) | `Slice<Range<usize>>`                   |
| `str`        | **없음** (직접 byte 접근 거부) | **없음**                  | `Slice<Range<usize>>` (UTF-8 안전 검증) |

`str` 의 `s[10]` 은 UTF-8 멀티바이트 문자의 안전성을 위해 의도적으로 제공되지 않습니다. 따라서 문자 단위 접근에는 `s.chars()` 를, 부분 추출에는 `s[10..15]` (slice 로서 멀티바이트 경계가 검증됩니다) 를 사용합니다.

## 2.9.9. `IntoIterator` / `FromIterator`

모든 컨테이너는 `IntoIterator` 를 구현합니다.

| 컨테이너            | `Item`            | 순서                              |
|---------------------|-------------------|-----------------------------------|
| `List<T>`           | `T`               | 첫 → 마지막                       |
| `Dict<K, V>`        | `(K, V)`          | **비결정**                        |
| `Set<T>`            | `T`               | **비결정**                        |
| `OrderedDict<K, V>` | `(K, V)`          | insertion order                   |
| `OrderedSet<T>`     | `T`               | insertion order                   |
| `Tuple<T...>`       | **구현하지 않음** | (heterogeneous — Iterator 의미 X) |
| `Slice<T>`          | `T`               | view 순서 (stride 적용)           |

또한 `FromIterator<T>` 는 `collect::<C>()` 가 어떤 컨테이너를 만들지 결정합니다.

```rok
let v: List<i32> = (0..5).collect();
let s: Set<i32> = (0..5).collect();
let d: Dict<str, i32> = [("a", 1), ("b", 2)].into_iter().collect();
```

## 2.9.10. 모듈 매핑

| 컨테이너      | 모듈                                     | 내부 구조                           |
|---------------|------------------------------------------|-------------------------------------|
| `List<T>`     | `rok::core::collections::list`            | Rust `Vec` 영감                     |
| `Dict<K, V>`  | `rok::core::collections::dict`            | open addressing                     |
| `Set<T>`      | `rok::core::collections::set`             | `Dict<T, ()>` 위                    |
| `OrderedDict` | `rok::core::collections::ordered_dict`    | linked-hash 또는 indices            |
| `OrderedSet`  | `rok::core::collections::ordered_set`     | (위와 동일)                         |
| `Tuple`       | **컴파일러 내장** + 표준 라이브러리 hook | `#[literal(...)]` + 컴파일러 lookup |
| `Slice<T>`    | `rok::core::collections::slice`           | struct ptr + len + stride           |

## 2.9.11. 다른 언어와의 비교

| 측면        | Rust                 | Python                     | rok                                    |
|-------------|----------------------|----------------------------|----------------------------------------|
| 동적 배열   | `Vec<T>`             | `list`                     | `List<T>`                              |
| hash map    | `HashMap<K, V>`      | `dict` (insertion-ordered) | `Dict<K, V>` + `OrderedDict<K, V>`     |
| hash set    | `HashSet<T>`         | `set`                      | `Set<T>` + `OrderedSet<T>`             |
| 고정 hetero | `(T1, T2)` tuple     | `tuple`                    | `Tuple<T1, T2>`                        |
| view        | `&[T]` / `&str`      | **slice 객체**             | `Slice<T>`                             |
| 리터럴      | `vec![...]` (매크로) | `[...]` `{...}` `(...)`    | `[...]` `{...}` `(...)` (`#[literal]`) |
| ordered map | `BTreeMap` (정렬 맵) | **기본 dict**              | 별도 `OrderedDict`                     |

rok 의 가장 큰 차이점은 다음과 같이 정리할 수 있습니다. 첫째로 6 가지 컬렉션이 모두 1급 리터럴 문법을 가지므로 `vec![]` 같은 매크로를 사용하지 않습니다. 둘째로 `Dict` / `Set` 의 insertion order 는 보장되지 않으며, 이는 Python 3.7+ 와 다른 점이므로 순서가 필요한 경우에는 `OrderedDict` 를 명시적으로 사용해야 합니다. 셋째로 `Slice<T>` 가 stride 를 가지는 view 이기 때문에 `xs[::2]` 같은 표현조차 zero-copy 로 처리됩니다.

## 2.9.12. `GcBuf<T>` — raw GC buffer

`rok::core::collections::raw` 의 `GcBuf<T>` 는 `List` / `Dict` 같은 컬렉션 구현이 쓰는 고정 길이 GC buffer 이며, 컴파일러가 아는 타입입니다. 런타임의 word-traced buffer (type id `2`, [refs-lang 8.2.12](../../refs-lang/008-abi/802-scan-table.md#8212-런타임-예약-type-id)) 위에 놓입니다.

```rok
struct GcBuf<T> { /* 컴파일러가 아는 표현 */ }

impl<T> GcBuf<T> {
    #[intrinsic("rok.buf_new")]
    fn new(len: usize) -> GcBuf<T>;           // 모든 word 를 0 (null) 으로
    #[intrinsic("rok.buf_get")]
    fn get(self, i: usize) -> T;               // 범위 밖이면 panic
    #[intrinsic("rok.buf_set")]
    fn set(self, i: usize, v: T);              // write barrier 포함
}
```

한 원소는 한 word 를 차지합니다. 일반 사용자 코드는 `List<T>` 를 쓰며, `GcBuf<T>` 는 컬렉션 구현을 위한 것입니다.
