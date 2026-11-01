# 2.9. Collections - `List` / `Dict` / `Set` / `Tuple` / `Slice`

The `rok::core::collections` module provides general-purpose containers. Every collection is allocated on the GC heap and can be created directly through literal syntax bound with `#[literal(...)]`.

This chapter defines the authoritative signatures of the following collections, with `Deque<T>` and `BinaryHeap<T>`.

| Collection          | Literal syntax             | Meaning                         |
|---------------------|----------------------------|---------------------------------|
| `List<T>`           | `[1, 2, 3]`                | Dynamic array (insertion order) |
| `Dict<K, V, S>`     | `{"a": 1, "b": 2}`         | Hash map (insertion order) |
| `Set<T, S>`         | `{1, 2, 3}`                | Hash set (insertion order)      |
| `Tuple<...>`        | `(1, "a", true)`           | Fixed-size heterogeneous        |
| `Slice<T>`          | `xs[1:3]`                  | Zero-copy view of a container   |
| `SliceMut<T>`       | `xs[1:3]` where a `SliceMut` is expected | Writable view of a list |

## 2.9.1. Literal syntax - `#[literal(query)]` binding

Literal syntax is defined with the **query DSL** of [refs-lang 6.5 `#[literal]`](../../refs-lang/006-meta-system/605-literal-magic.md) as follows.

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

The literal query syntax means: `{...} % sep *` is zero or more repetitions separated by sep, and `{...} % sep +` is one or more. A single-element Set is written `{1,}` with a trailing comma to distinguish it from the block `{1}`. For token-sequence matching, the lexer distinguishes tokens directly and the parser then performs query matching.

### Resolving the Dict / Set ambiguity

```rok
{"a": 1}      // Dict: `:` pattern
{1, 2}        // Set: values only
{1,}          // Set (single element, trailing comma)
{1}           // always a block, even when the slot is a Set
{ x = 1 }     // block expression: the token after `{` is statement-like
```

The parser decides from the contents alone: the `expr ":"` form is a Dict, a comma makes it a Set, and anything else is a block. The type of the receiving slot is not used in the decision.

### The type of `[...]`

`[1, 2, 3]` is `List<T>` when there is no expected type. A fixed-size array `[T; N]` is created only when the expected type is written as `[T; N]`, as in `let a: [i32; 3] = [1, 2, 3];`. Fixed arrays are built directly by the compiler without going through a handler.

A fixed array implements `Clone` / `Copy` / `Default` / `Debug` / `PartialEq` / `Eq` / `PartialOrd` / `Ord` / `Hash` when its element does. Ordering is lexicographic, so comparison operators (`==`, `<`, etc.) can be used directly. Its methods:

```rok
impl<T, const N: usize> [T; N] {
    fn len(self) -> usize;                       // N
    fn is_empty(self) -> bool;
    fn iter(self) -> SliceIter<T>;               // over a copy (arrays are values)
    fn first(self) -> T | none;
    fn last(self) -> T | none;
    fn map<U>(self, f: fn(T) -> U) -> [U; N];    // a new array
    fn fill(ref self, x: T);                     // in place
}

impl<T: PartialEq, const N: usize> [T; N] {
    fn contains(self, x: T) -> bool;
}
```

```rok
let a: [i32; 3] = [1, 2, 3];
a.len();                 //→ 3
a == [1, 2, 3];          //→ true
a < [1, 2, 4];           //→ true (lexicographic)
```

### Reference handles

`List`, `Dict` and `Set` values are reference handles to GC heap objects. `let ys = xs;` and argument passing copy the handle, so the same collection is shared. Methods that modify a collection therefore take `self`, not `ref self`. Call `clone()` for an independent copy. A struct element is a value: `xs[i]` and `d[k]` return a copy, and a change is stored with `xs[i] = v` ([refs-lang 2.3.8](../../refs-lang/002-types/203-struct.md#238-value-semantics)). `iter` / `keys` / `values` return lazy iterators that do not copy the collection.

## 2.9.2. `List<T>` - dynamic array

```rok
struct List<T> {
    // internal: ptr + len + capacity
}
```

Internally it consists of a contiguous memory buffer plus `len` / `capacity`, and provides amortized O(1) push/pop and O(1) random access.

### Core methods

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
    fn truncate(self, len: usize);              // keeps the first len elements; no change if fewer
    fn drain(self, r: Range<usize>) -> List<T>; // removes and returns r's elements in order; panics if r is reversed or ends past len()

    fn get(self, idx: usize) -> T | none;       // safe: out-of-bounds → none
    fn first(self) -> T | none;
    fn last(self) -> T | none;
    fn swap(self, i: usize, j: usize);          // panics when either is out of bounds
    fn fill(self, x: T);                         // every element becomes x

    fn extend<I: IntoIterator<Item = T>>(self, iter: I);
    fn retain(self, keep: fn(T) -> bool);        // in place, order kept
    fn sort(self) where T: Ord;                  // ascending, stable
    fn sort_by(self, cmp: fn(T, T) -> Ordering); // stable
    fn sort_by_key<K: Ord>(self, key: fn(T) -> K); // ascending by key, stable
    fn reserve(self, additional: usize);
    fn shrink_to_fit(self);
    fn reverse(self);                            // in place

    fn split_at(self, mid: usize) -> (Slice<T>, Slice<T>);   // views of [0, mid) and [mid, len); mid > len panics
    fn windows(self, size: usize) -> slice::Windows<T>;   // self[:].windows(size)
    fn chunks(self, size: usize) -> slice::Chunks<T>;     // self[:].chunks(size)
    fn partition_point(self, pred: fn(T) -> bool) -> usize;   // self[:].partition_point(pred)

    fn iter(self) -> ListIter<T>;
    fn into_iter(self) -> ListIntoIter<T>;
}

impl<T: PartialEq> List<T> {
    fn contains(self, x: T) -> bool;
    fn dedup(self);                              // drops each element equal to the one before it
}

impl<T: Ord> List<T> {
    fn binary_search(self, x: T) -> usize | none;   // ascending list: the first index of x, O(log n)
}
```

A positional read has a `T | none` form (`get`, `first`, `last`, `pop`); every other positional operation panics out of range (`xs[i]`, `xs[i] = v`, `insert`, `remove`, `swap`, `drain`, `split_at`). There is no `set`: the write is `xs[i] = v`, and a write that skips an out-of-range index checks first (`if i < xs.len() { xs[i] = v }`).

`partition_point(pred)` is the first index whose element fails `pred`, which must hold for a prefix of the list and fail for the rest; O(log n). The insertion point of `x` in an ascending list is `xs.partition_point(|e| e < x)`, and by a key `xs.partition_point(|e| key(e) < k)`. `binary_search(x)` returns the first index of an element equal to `x` (the lower bound), or `none`.

```rok
impl List<str> {
    fn join(self, sep: str) -> str;              // joins the elements with sep
}

impl List<u8> {
    fn find<B: AsBytes>(self, needle: B) -> usize | none;   // self[:].find(needle)
}
```

### Examples

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
zs.partition_point(|x| x < 3);               //→ 2 (where 3 would go)
let (a, b) = zs.split_at(1);                 //→ views of [1] and [2, 4]
```

### Trait implementations

| Trait                 | Condition           | Meaning                         |
|-----------------------|---------------------|---------------------------------|
| `IntoIterator`        | Always              | `for x in xs`                   |
| `FromIterator<T>`     | Always              | `(0..5).collect::<List<i32>>()` |
| `Index<usize>`        | Always              | `xs[i]`                         |
| `IndexSet<usize>`     | Always (`Value = T`) | `xs[i] = v`                    |
| `AddAssign<T>`        | Always              | `xs += x` pushes `x`            |
| `AddAssign<List<T>>`  | Always              | `xs += ys` appends every element of `ys` |
| `Sliceable<Range<usize>>` | Always          | `xs[1:3]`                       |
| `Display`             | **No**              | **Manual output recommended**   |
| `Debug`               | `T: Debug`          | `f"{xs:?}"` → `[1, 2, 3]`       |
| `Clone`               | `T: Clone`          | Deep copy                       |
| `PartialEq`           | `T: PartialEq`      | element-wise                    |
| `Eq`/`Ord`/`Hash`     | When T has the same trait | lexicographic             |

### Comparison / hashing / Debug - common rules

- `List<T>` implements `PartialEq` / `Eq` / `Hash` element by element, in order.
- `Dict<K, V, S>` and `Set<T, S>` compare and hash independently of order. They are equal if they hold the same elements (keys and values). Each entry is hashed alone with the process key and the results are summed ([2.5.6](205-cmp.md#256-hash---hashing-eq-consistency-promise)), so the hash does not depend on `S` either.
- All three collections implement `Debug` when their elements implement it, printing in iteration order.
- A collection whose elements are `Eq + Hash` can therefore be a `Set` element or a `Dict` key (`Set<List<i32>>`).

### Capacity and removal - common rules

- An index out of range panics naming the index and the length: `index 100 out of bounds for length 3`. A method names itself first (`List::remove: index 5 out of bounds for length 3`, `List::drain: range 2..4 out of bounds for length 3`), and the panic points at the caller.
- A size that overflows in `reserve`, `with_capacity`, `repeat` or growth panics with "capacity overflow", at every optimization level.
- Removing an element (`pop`, `remove`, `clear`, `truncate`, `drain`, ..) releases the reference: the GC no longer sees it through the collection.

## 2.9.3. `Dict<K, V, S>` - hash map

```rok
struct Dict<K: Hash + Eq, V, S = RandomState> {
    // internal: compact entries in insertion order, plus an index of
    // linear-probed slots (none for a small table)
}
```

It provides average O(1) lookup / insert / remove. Iteration follows insertion order: a key inserted again while present keeps its place, and one removed and inserted again goes last.

`S` builds the hashers ([2.5.11](205-cmp.md#2511-rokcorehash---hashers)). The default, `RandomState`, uses a key drawn once per process, so where a key sits in the table changes between runs; the iteration order does not. A dict built during constant evaluation is rehashed with the process key on its first use at run time. `Dict<K, V>` is `Dict<K, V, RandomState>`, and so is every `{k: v}` literal.

### Core methods

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

    fn insert(self, k: K, v: V) -> V | none;       // returns the previous V / none if absent
    fn remove(self, k: K) -> V | none;
    fn get(self, k: K) -> V | none;
    fn contains_key(self, k: K) -> bool;
    fn clear(self);
    fn retain(self, keep: fn(K, V) -> bool);      // in place
    fn extend<I: IntoIterator<Item = (K, V)>>(self, pairs: I);   // a later pair's value wins

    fn iter(self) -> DictIter<K, V, S>;              // (K, V) pairs
    fn keys(self) -> Keys<K, V, S>;
    fn values(self) -> Values<K, V, S>;

    fn entry(self, k: K) -> Entry<K, V, S>;       // the slot for that key (present or not)
}

struct Entry<K, V> { /* dict + key */ }

impl<K: Hash + Eq, V> Entry<K, V> {
    fn key(self) -> K;
    fn or_insert(self, v: V) -> V;                   // inserts v if absent; returns the value
    fn or_insert_with(self, f: fn() -> V) -> V;      // inserts f() if absent; returns the value
    fn and_modify(self, f: fn(V) -> V) -> Entry<K, V>;   // if present, replaces it with f(value)
}

impl<K: Hash + Eq, V: Default> Entry<K, V> {
    fn or_default(self) -> V;                         // inserts V::default() if absent; returns the value
}
```

`Default`, `FromIterator` and `From<List<(K, V)>>` need `S: BuildHasher + Default`; the other impls take any `S: BuildHasher`.

```rok
use rok::core::hash::BuildHasherDefault;

let fixed: Dict<str, i32, BuildHasherDefault<Fnv>> = Dict::with_hasher(BuildHasherDefault::new());
```

```rok
let counts: Dict<str, i32> = Dict::new();   // `{}` is an empty block, not an empty Dict
for w in words {
    counts.entry(w).and_modify(|n| n + 1).or_insert(1);
}
```

### Examples

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

The `d[k]` notation assumes the key is present; use `.get(k)` when safe access is needed. The panic shows the key as `{k:?}` does, or as `<K>` (the type name) when `K` has no `Debug`, and `V` as the value type's name.

`d += e` (`AddAssign<Dict<K, V>>`) inserts every entry of `e` into `d` in place; on a shared key the value from `e` wins.

`Dict::from(pairs)` (`From<List<(K, V)>>`) builds a dict from a list of pairs; on a repeated key the later value wins.

### Lazy views

`iter` / `keys` / `values` are lazy views onto the current table, not copies. Modifying the dict during iteration gives unspecified results (elements may be missed or seen twice), but memory safety is preserved.

## 2.9.4. `Set<T, S>` - hash set

```rok
struct Set<T: Hash + Eq, S = RandomState> {
    // internal: Dict<T, bool, S>
}
```

It iterates in insertion order, as `Dict` does; average performance is O(1). `S` builds the hashers, as for `Dict`.

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

    fn insert(self, x: T) -> bool;        // true if newly inserted
    fn remove(self, x: T) -> bool;
    fn contains(self, x: T) -> bool;
    fn clear(self);
    fn retain(self, keep: fn(T) -> bool);         // in place
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

`Set::from(xs)` (`From<List<T>>`) builds a set from a list, dropping duplicates. `s += x` (`AddAssign<T>`) inserts `x`.

## 2.9.5. Insertion order

`Dict` and `Set` iterate in insertion order, like Python 3.7+ `dict` or Java's `LinkedHashMap`, so there is no separate ordered type. `iter`, `keys`, `values`, `to_list`, `Debug` and `for` all follow it:

- a key inserted again while present keeps its place (only the value changes);
- a key removed and inserted again goes last;
- `retain` keeps the survivors' order, and `extend` and `d += e` insert in order (new keys go last);
- the result of a set operation follows the order of the left operand first.

Equality and hashing ignore the order ([2.9.2](#292-listt---dynamic-array)), so `{"a": 1, "b": 2} == {"b": 2, "a": 1}`.

## 2.9.5a. `Deque<T>` - double-ended queue

A double-ended queue over a ring buffer.

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
    fn iter(self) -> DequeIter<T>;             // front to back
}
```

There is no `set`: the write is `d[i] = v`. `capacity` is the number of elements the ring buffer holds before it grows. It implements `d[i]` / `d[i] = v` (panics when out of range, as `List` does: `index 5 out of bounds for length 3`), `IntoIterator`, `FromIterator` (front to back), `Default`, `Clone` (`T: Clone`), `PartialEq` / `Eq` (element-wise, front to back) and `Debug`. `d += x` (`AddAssign<T>`) pushes `x` at the back.

## 2.9.5b. `BinaryHeap<T>` - priority queue

```rok
impl<T: Ord> BinaryHeap<T> {
    fn new() -> BinaryHeap<T>;                 // max-heap: largest first
}
impl<T> BinaryHeap<T> {
    fn by_ordering(cmp: fn(T, T) -> Ordering + Send) -> BinaryHeap<T>; // the greatest by cmp first, as new() by Ord::cmp
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn peek(self) -> T | none;
    fn push(self, x: T);
    fn pop(self) -> T | none;
    fn clear(self);
    fn retain(self, keep: fn(T) -> bool);
    fn iter(self) -> HeapIter<T>;              // unordered
}
```

`retain` filters in place and restores the heap order, O(n). It implements `IntoIterator` (unordered, as `iter`), `Debug` (storage order, the top first) and `Clone` (`T: Clone`; the copy keeps the heap's order); with `T: Ord`, `Default` and `FromIterator` build a max-heap, as `new`.

A comparator returns `Ordering`, as `List::sort_by`'s does: `by_ordering(|a, b| b.cmp(a))` pops the least first.

The comparator is a `fn .. + Send`, so a heap is `Send` when `T` is and can be sent through a channel or put in a `Mutex`. A comparator that captures a value that is not `Send + Sync` (a collection, for one) is a compile error whose help says to drop the capture: compute the value inside the closure, or capture an array.

## 2.9.6. `Tuple<...>` - fixed heterogeneous

```rok
#[literal( "(" expr "," {expr} % "," * ")" )]
struct Tuple { /* heterogeneous, determined at compile time */ }
```

Tuples are intrinsic to the language and cannot be registered as a literal handler ([refs-lang 6.5.7](../../refs-lang/006-meta-system/605-literal-magic.md#657-user-defined-literals)). A tuple expression must contain at least one comma. `(e)` is a grouping expression; a one-element tuple is `(e,)`.

Each position may have a different type, and the length is fixed at compile time.

```rok
let t: Tuple<i32, str, bool> = (42, "hello", true);
let n = t.0;              //→ 42
let s = t.1;              //→ "hello"
let b = t.2;              //→ true

// pattern destructuring
let (n, s, b) = t;
```

### Trait implementations

| Trait        | Condition                     |
|--------------|-------------------------------|
| `Clone`      | Every component `Clone`       |
| `Copy`       | Every component `Copy`        |
| `Debug`      | Every component `Debug`       |
| `PartialEq`  | Every component `PartialEq`   |
| `Eq`         | Every component `Eq`          |
| `PartialOrd` | Every component `PartialOrd`  |
| `Ord`        | Every component `Ord`         |
| `Hash`       | Every component `Hash`        |

The traits in the table are provided by `rok::core::tuple` for tuples of 1 to 12 elements. `PartialOrd` / `Ord` are lexicographic (from the first element; the first differing element decides), so `<` / `<=` / `>` / `>=` can be used on tuples directly (`(1, "b") < (1, "c")`).

`Display` is not derived automatically; implementing it directly where needed is recommended.

### Multiple return values

The multi-return of [refs-lang 4.3.5 Multiple return](../../refs-lang/004-functions/403-return.md#435-multiple-returns-tuples) is handled as a Tuple return.

```rok
fn divmod(a: i32, b: i32) -> (i32, i32) {
    (a / b, a % b)
}

let (q, r) = divmod(17, 5);     //→ q = 3, r = 2
```

## 2.9.7. `Slice<T>` - view

```rok
struct Slice<T> {
    // GC buffer + start position + length + stride (1 = contiguous, k = step, negative = reversed)
}
```

It is the result of expressions such as `xs[1:3]` / `xs[::2]` / `xs[1:]` ([2.3.4 `Sliceable`](203-ops.md#234-indexing-traits)). A view does not copy elements; it points into the original's GC buffer, and that buffer stays alive while the view is alive.

| Source     | What the view sees                                                              |
|------------|---------------------------------------------------------------------------------|
| `List<T>`  | The list's current buffer. Element writes are visible; changes after the list grows and moves its buffer are not |
| `Slice<T>` | The same buffer (a view of a view)                                              |
| `[T; N]`   | A copy of the array (arrays are values)                                         |

### Core methods

```rok
impl<T> Slice<T> {
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn get(self, i: usize) -> T | none;
    fn first(self) -> T | none;
    fn last(self) -> T | none;
    fn step(self, k: isize) -> Slice<T>;     // every k-th; from the end if k < 0. k == 0 panics
    fn split_at(self, mid: usize) -> (Slice<T>, Slice<T>);   // [0, mid) and [mid, len); mid > len panics
    fn windows(self, size: usize) -> slice::Windows<T>;   // overlapping views of size elements; size == 0 panics
    fn chunks(self, size: usize) -> slice::Chunks<T>;     // views of size elements, the last shorter; size == 0 panics
    fn iter(self) -> SliceIter<T>;
    fn to_list(self) -> List<T>;             // allocates a new List
}

impl<T> Slice<T> {
    fn partition_point(self, pred: fn(T) -> bool) -> usize;   // the first index whose element fails pred, O(log n)
}

impl<T: PartialEq> Slice<T> {
    fn contains(self, x: T) -> bool;
}

impl<T: Ord> Slice<T> {
    fn binary_search(self, x: T) -> usize | none;   // ascending view: the first index of x, O(log n)
}

impl Slice<u8> {
    fn find<B: AsBytes>(self, needle: B) -> usize | none;   // offset of the first occurrence; an empty needle is at 0
}

impl<T> Index<usize> for Slice<T> { ... }               // s[i]: panics when out of range (`slice index 5 out of bounds for length 3`)
impl<T> Sliceable<Range<usize>> for Slice<T> { ... }    // s[a:b]: a view of a view; the range is clamped to the length
impl<T> IntoIterator for Slice<T> { ... }
impl<T: PartialEq> PartialEq for Slice<T> { ... }
impl<T: Debug> Debug for Slice<T> { ... }
```

`slice::Windows<T>` and `slice::Chunks<T>` are `ExactSizeIterator`s with `Item = Slice<T>`: each window or chunk is a view of the same buffer, not a copy. `Iterator::windows` / `chunks` ([2.7.1](207-iter.md#271-iterator---stepping)) collect lists instead, for iterators that have no buffer.

The needle of `find` is any `AsBytes` type ([2.11.13](211-conversion.md#21113-asbytes-trait---bytes-as-a-view)):

```rok
trait AsBytes {
    fn byte_slice(self) -> Slice<u8>;      // the bytes as a view, no copy
}
```

It is implemented for `str` (its UTF-8 bytes), `List<u8>`, `Slice<u8>` and `[u8; N]`, so a byte string passes as the needle (its `Slice<u8>` rule, [refs-lang 1.7.5](../../refs-lang/001-lexical/107-string-literals.md#175-byte-strings)).

`Slice` is read-only (no `IndexSet`); the view that writes a list is `SliceMut<T>` ([2.9.7a](#297a-slicemutt---writable-view)). Negative indices count from the end ([refs-lang 3.6.1](../../refs-lang/003-expressions/306-indexing-and-slicing.md#361-indexing-arri)).

### Examples

```rok
let xs: List<i32> = [1, 2, 3, 4, 5];
let s1: Slice<i32> = xs[1:4];           //→ view of [2, 3, 4]
let s2: Slice<i32> = xs[::2];           //→ view of [1, 3, 5]
let s3: Slice<i32> = xs[..];            //→ view of all

s1.len();                               //→ 3
let t: i32 = s1.iter().sum();           //→ 9
xs[2] = 30;
s1[1];                                  //→ 30 (the list's element write is visible)

let owned: List<i32> = s1.to_list();    //→ [2, 30, 4] (new List)
```

### Integration with variadic args

The `xs` of `fn f(xs: ...T)` is a `Slice<T>` view ([refs-lang 4.2.5](../../refs-lang/004-functions/402-parameters.md#425-variadic-t)). Individual arguments and a list spread (`f(..ys)`) are received in the same shape.

```rok
fn sum(xs: ...i32) -> i32 {
    let total = 0;
    for x in xs {            // xs: Slice<i32>
        total = total + x;   // reassignment without `let`
    }
    total
}

sum(1, 2, 3);                //→ 6
sum(..[4, 5]);               //→ 9 (list spread)
```

## 2.9.7a. `SliceMut<T>` - writable view

```rok
struct SliceMut<T> {
    // the list's buffer + start + length, and the list with its version when the view was made
}
```

`SliceMut<T>` is a contiguous view that writes the list it views, like Rust's `&mut [T]`. A function that fills or reorders part of a list takes one (`fn zero(buf: SliceMut<u8>)`). It is made where a `SliceMut<T>` is expected: an argument, an annotated `let`, a return value.

| Written there                   | The view                                                                 |
|---------------------------------|--------------------------------------------------------------------------|
| `xs` (a `List<T>`)              | The whole list ([refs-lang 2.14.6](../../refs-lang/002-types/214-cast-and-conversion.md#2146-context-based-automatic-conversion)) |
| `xs[a:b]`, `xs[a..b]`, `xs[..]` | That range of the list, the bounds clamped as for `Slice` ([refs-lang 3.6.2](../../refs-lang/003-expressions/306-indexing-and-slicing.md#362-slicing-arrstartend)) |
| `s[a:b]` (a `SliceMut<T>`)      | A writable view of the view                                              |

Anywhere else `xs[a:b]` is still a read-only `Slice<T>`. A stepped range (`xs[a:b:k]`) is never a `SliceMut`: it is not contiguous.

### Core methods

```rok
impl<T> SliceMut<T> {
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn get(self, i: usize) -> T | none;
    fn first(self) -> T | none;
    fn last(self) -> T | none;
    fn fill(self, x: T);
    fn swap(self, i: usize, j: usize);                 // out of range panics
    fn reverse(self);
    fn copy_from_slice(self, src: Slice<T>);           // src.len() != len panics; src may overlap the view
    fn split_at(self, mid: usize) -> (SliceMut<T>, SliceMut<T>);   // [0, mid) and [mid, len), both writable; mid > len panics
    fn sort_by(self, cmp: fn(T, T) -> Ordering);       // stable
    fn sort_by_key<K: Ord>(self, key: fn(T) -> K);     // stable
    fn as_slice(self) -> Slice<T>;                     // the same elements, read-only
    fn iter(self) -> SliceMutIter<T>;
    fn to_list(self) -> List<T>;                       // allocates a new List
    unsafe fn raw_parts(self) -> (GcBuf<T>, usize, usize);   // (buffer, start, len), for runtime calls that fill a region
}

impl<T: Ord> SliceMut<T> {
    fn sort(self);                                     // ascending, stable
}

impl<T: PartialEq> SliceMut<T> {
    fn contains(self, x: T) -> bool;
}

impl<T> Index<usize> for SliceMut<T> { ... }              // s[i]: panics when out of range
impl<T> IndexSet<usize> for SliceMut<T> { ... }           // s[i] = x writes element i of the list
impl<T> Sliceable<Range<usize>> for SliceMut<T> { ... }   // s[a:b]: a writable view of the view; the range is clamped
impl<T> IntoIterator for SliceMut<T> { ... }
impl<T: PartialEq> PartialEq for SliceMut<T> { ... }
impl<T: Debug> Debug for SliceMut<T> { ... }
```

`SliceMutIter<T>` is a `DoubleEndedIterator` and `ExactSizeIterator` with `Item = T`. Like `List`, a `SliceMut` is not `Sync` ([2.13.9](213-marker.md#2139-send--sync---thread-boundary-markers)); a hand-off copies its list and keeps the view on the copy. Negative indices count from the end, as on a list.

### Resizing the list while a view lives

A resize may move the list's elements to a new buffer, so a view is not used after its list is resized. A resize is any `List` method that changes the length or the buffer: `push`, `pop`, `insert`, `remove`, `clear`, `truncate`, `drain`, `set_len`, `reserve`, `shrink_to_fit`, `extend`, `retain`, `dedup` and `+=`. Element writes (`xs[i] = x`, `fill`, `swap`, `sort`, `reverse`) are not resizes.

- Where one function shows both, using a view after a resize of its list is a compile error, `E0138` ([refs-lang 9.9.4](../../refs-lang/009-tools/909-debug-tools.md#994-diagnostics--error-messages)). The view is followed through copies of the list (`let ys = xs`), views of the view (`s[1:]`, `split_at`), a `for` loop over it, branches and the next pass of a loop. Assigning the view again (`s = xs[..]`) makes it usable again, and a resize on a path that leaves (`return`, `break`) does not reach the code after it.
- Every other case is checked at run time: a resize inside a called function, a view kept in a struct, an `async fn` that waits while its caller resizes. A list carries a version that each resize bumps, and a view made before the bump panics on its next use (`SliceMut used after its list was resized (push, insert, clear, ...): make the view again after the resize`).

```rok
let xs: List<i32> = [1, 2, 3];
let s: SliceMut<i32> = xs[1:];
xs.push(4);
s[0] = 9;          // error[E0138]: `s` views `xs`, which `xs.push(..)` resized at line 3
```

The run-time check is what lets a `SliceMut` be an `async fn`'s argument and live across an `await`.

### Examples

```rok
fn zero(buf: SliceMut<u8>) {
    buf.fill(0);
}

let xs: List<i32> = [5, 4, 3, 2, 1];
let mid: SliceMut<i32> = xs[1:4];
mid.sort();                             // xs is [5, 2, 3, 4, 1]
mid[0] = 20;                            // xs is [5, 20, 3, 4, 1]
let (a, b) = mid.split_at(1);
b.fill(0);                              // xs is [5, 20, 0, 0, 1]

let bytes: List<u8> = [1, 2, 3];
zero(bytes[1:]);                        // [1, 0, 0]
zero(bytes);                            // the whole list: [0, 0, 0]
```

## 2.9.8. `Index` / `IndexSet` / `Sliceable` trait mapping

The traits of [2.3 ops](203-ops.md) define the meaning of `[]`, `[i] = v` and `[a:b]` on containers.

| Container    | `Index<I>`                     | `IndexSet<I>`             | `Sliceable<R>`                          |
|--------------|--------------------------------|---------------------------|-----------------------------------------|
| `List<T>`    | `Index<usize>` → T             | `IndexSet<usize>` (Value = T) | `Sliceable<Range<usize>>` → `Slice<T>`  |
| `Dict<K, V>` | `Index<K>` → V                 | `IndexSet<K>` (Value = V) | **none**                                |
| `Set<T>`     | **none**                       | **none**                  | **none**                                |
| `Tuple<...>` | **compiler-direct** (`.0`/.1)  | **compiler-direct**       | **none**                                |
| `[T; N]`     | `Index<usize>` → T             | `IndexSet<usize>` (Value = T) | `Sliceable<Range<usize>>` → `Slice<T>` (copy) |
| `Slice<T>`   | `Index<usize>` → T             | **none** (read-only view) | `Sliceable<Range<usize>>` → `Slice<T>`  |
| `SliceMut<T>` | `Index<usize>` → T            | `IndexSet<usize>` (Value = T) | `Sliceable<Range<usize>>` → `SliceMut<T>` |
| `str`        | **none** (direct byte access rejected) | **none**          | `Sliceable<Range<usize>>` → `str` (copy, UTF-8 boundary check) |

`s[10]` on a `str` is deliberately not provided, for the safety of UTF-8 multibyte characters. Use `s.chars()` for character-wise access and `s[10..15]` for extraction (as a slice, multibyte boundaries are validated).

## 2.9.9. `IntoIterator` / `FromIterator`

Every container implements `IntoIterator`.

| Container           | `Item`              | Order                             |
|---------------------|---------------------|-----------------------------------|
| `List<T>`           | `T`                 | first → last                      |
| `Dict<K, V>`        | `(K, V)`            | insertion order                   |
| `Set<T>`            | `T`                 | insertion order                   |
| `Deque<T>`          | `T`                 | front → back                      |
| `BinaryHeap<T>`     | `T`                 | **unordered** (not pop order)     |
| `Tuple<T...>`       | **not implemented** | (heterogeneous, no Iterator meaning)  |
| `Slice<T>`          | `T`                 | view order (stride applied)       |
| `SliceMut<T>`       | `T`                 | view order                        |

`FromIterator<T>` determines which container `collect::<C>()` builds.

```rok
let v: List<i32> = (0..5).collect();
let s: Set<i32> = (0..5).collect();
let d: Dict<str, i32> = [("a", 1), ("b", 2)].into_iter().collect();
```

## 2.9.10. Module mapping

| Container     | Module                                   | Internal structure                  |
|---------------|------------------------------------------|-------------------------------------|
| `List<T>`     | `rok::core::collections::list`            | inspired by Rust `Vec`              |
| `Dict<K, V, S>` | `rok::core::collections::dict`          | compact entries + linear probing    |
| `Set<T, S>`   | `rok::core::collections::set`             | on top of `Dict<T, bool, S>`        |
| `Deque<T>`    | `rok::core::collections::deque`           | ring buffer                         |
| `BinaryHeap<T>` | `rok::core::collections::heap`          | implicit binary tree on a `List`    |
| `Tuple`       | **compiler built-in** + standard library hook | `#[literal(...)]` + compiler lookup |
| `Slice<T>`    | `rok::core::collections::slice`           | struct ptr + len + stride           |
| `SliceMut<T>` | `rok::core::collections::slice`           | buffer + start + len, list + version |

## 2.9.11. Comparison with other languages

| Aspect          | Rust                 | Python                     | Rok                                    |
|-----------------|----------------------|----------------------------|----------------------------------------|
| Dynamic array   | `Vec<T>`             | `list`                     | `List<T>`                              |
| Hash map        | `HashMap<K, V>`      | `dict` (insertion-ordered) | `Dict<K, V>` (insertion-ordered)       |
| Hash set        | `HashSet<T>`         | `set`                      | `Set<T>` (insertion-ordered)           |
| Fixed hetero    | `(T1, T2)` tuple     | `tuple`                    | `Tuple<T1, T2>`                        |
| View            | `&[T]` / `&str`      | **slice object**           | `Slice<T>`                             |
| Writable view   | `&mut [T]`           | **none** (slice assignment) | `SliceMut<T>`                         |
| Literals        | `vec![...]` (macro)  | `[...]` `{...}` `(...)`    | `[...]` `{...}` `(...)` (`#[literal]`) |
| Ordered map     | `BTreeMap` (sorted map) | **default dict**        | **default `Dict`**                     |

Rok differs most in these ways. First, `List`, `Dict`, `Set`, `Tuple` and `Slice` have first-class literal syntax, so no macro such as `vec![]` is used. Second, `Dict` / `Set` iterate in insertion order, as Python 3.7+ `dict` does, so no separate ordered type is needed. Third, `Slice<T>` is a view with a stride, so even an expression like `xs[::2]` is zero-copy.

## 2.9.12. `GcBuf<T>` - raw GC buffer

`GcBuf<T>` in `rok::core::collections::raw` is the fixed-length GC buffer used by collection implementations such as `List` / `Dict`, and is a type known to the compiler. `GcBuf` is defined by `rok::bare` (`rok::bare::mem`) and re-exported here. It sits on top of the runtime's word-traced buffer (type id `2`, [refs-lang 8.2.12](../../refs-lang/008-abi/802-scan-table.md#8212-runtime-reserved-type-ids)).

```rok
struct GcBuf<T> { /* representation known to the compiler */ }

impl<T> GcBuf<T> {
    #[intrinsic("rok.buf_new")]
    unsafe fn new(len: usize) -> GcBuf<T>;           // every word set to 0 (null)
    #[intrinsic("rok.buf_get")]
    unsafe fn get(self, i: usize) -> T;        // no bounds check (the owner tracks the length)
    #[intrinsic("rok.buf_set")]
    unsafe fn set(self, i: usize, v: T);       // includes the write barrier
}
```

Each element occupies one word. Ordinary user code uses `List<T>`; `GcBuf<T>` is for implementing collections.

`GcBuf<T>` and its primitives (`buf_new` / `buf_get` / `buf_set`, `new` / `get` / `set`, indexing a `GcBuf`) are `unsafe`: they skip the element-type and initialization checks the collections keep.

```rok
#[intrinsic("rok.buf_same")]
fn buf_same<T>(a: GcBuf<T>, b: GcBuf<T>) -> bool;   // whether they are the same buffer (identity, not contents). Pins nothing
#[intrinsic("rok.buf_erase")]
pub(pkg) fn buf_erase<T>(buf: GcBuf<T>) -> GcBuf<u8>;   // typed as GcBuf<u8> for runtime parameters taking any GC object
```
