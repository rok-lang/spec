# 2.9. Collections - `List` / `Dict` / `Set` / `Tuple` / `Slice`

The `rok::core::collections` module provides general-purpose containers. Every collection is allocated on the GC heap and can be created directly through literal syntax bound with `#[literal(...)]`.

This chapter defines the authoritative signatures of the following 6 collections.

| Collection          | Literal syntax             | Meaning                         |
|---------------------|----------------------------|---------------------------------|
| `List<T>`           | `[1, 2, 3]`                | Dynamic array (insertion order) |
| `Dict<K, V>`        | `{"a": 1, "b": 2}`         | Hash map (unordered)   |
| `Set<T>`            | `{1, 2, 3}`                | Hash set (unordered)            |
| `OrderedDict<K, V>` | `OrderedDict::from([...])` | Insertion-order-preserving dict |
| `OrderedSet<T>`     | `OrderedSet::from([...])`  | Insertion-order-preserving set  |
| `Tuple<...>`        | `(1, "a", true)`           | Fixed-size heterogeneous        |
| `Slice<T>`          | `xs[1:3]`                  | Zero-copy view of a container   |

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

A fixed array implements `Clone` / `Copy` / `Default` / `Debug` / `PartialEq` / `Eq` / `PartialOrd` / `Ord` / `Hash` when its element does, and has `len()` and `is_empty()`. Ordering is lexicographic, so comparison operators (`==`, `<`, etc.) can be used directly.

```rok
let a: [i32; 3] = [1, 2, 3];
a.len();                 //→ 3
a == [1, 2, 3];          //→ true
a < [1, 2, 4];           //→ true (lexicographic)
```

### Reference handles

`List`, `Dict` and `Set` values are reference handles to GC heap objects. `let ys = xs;` and argument passing copy the handle, so the same collection is shared. Methods that modify a collection therefore take `self`, not `ref self`. Call `clone()` for an independent copy. A struct element is a value: `xs[i]` and `d[k]` return a copy, and a change is stored with `xs[i] = v` ([refs-lang 2.3.8](../../refs-lang/002-types/203-struct.md#238-value-semantics)). `keys` / `values` / `entries` return lazy iterators that do not copy the collection.

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
    fn set(self, idx: usize, x: T) throws (IndexError);   // xs[idx] = x panics instead
    fn first(self) -> T | none;
    fn last(self) -> T | none;

    fn extend<I: IntoIterator<Item = T>>(self, iter: I);
    fn sort(self) where T: Ord;                  // ascending, stable
    fn sort_by(self, cmp: fn(T, T) -> Ordering); // stable
    fn reserve(self, additional: usize);
    fn shrink_to_fit(self);
    fn reverse(self);                            // in place

    fn iter(self) -> ListIter<T>;
    fn into_iter(self) -> ListIntoIter<T>;
}
```

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
ys[100];                  //→ panic (IndexError)
ys.get(100);              //→ none

for x in ys {
    use(x);               //→ 10, 20, 30
}

let total: i32 = ys.into_iter().sum();   //→ 60
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
- `Dict<K, V>` and `Set<T>` compare and hash independently of order. They are equal if they hold the same elements (keys and values).
- All three collections implement `Debug` when their elements implement it, printing in iteration order.
- A collection whose elements are `Eq + Hash` can therefore be a `Set` element or a `Dict` key (`Set<List<i32>>`).

### Capacity and removal - common rules

- A size that overflows in `reserve`, `with_capacity`, `repeat` or growth panics with "capacity overflow", at every optimization level.
- Removing an element (`pop`, `remove`, `clear`, `truncate`, `drain`, ..) releases the reference: the GC no longer sees it through the collection.

## 2.9.3. `Dict<K, V>` - hash map

```rok
struct Dict<K: Hash + Eq, V> {
    // internal: HashMap (open addressing or chaining)
}
```

It provides average O(1) lookup / insert / remove. Order is not guaranteed, so iteration order is nondeterministic.

### Core methods

```rok
impl<K: Hash + Eq, V> Dict<K, V> {
    fn new() -> Dict<K, V>;
    fn with_capacity(cap: usize) -> Dict<K, V>;

    fn len(self) -> usize;
    fn is_empty(self) -> bool;

    fn insert(self, k: K, v: V) -> V | none;       // returns the previous V / none if absent
    fn remove(self, k: K) -> V | none;
    fn get(self, k: K) -> V | none;
    fn contains_key(self, k: K) -> bool;
    fn clear(self);

    fn keys(self) -> KeysIter<K, V>;
    fn values(self) -> ValuesIter<K, V>;
    fn entries(self) -> EntriesIter<K, V>;            // (K, V) pairs

    fn entry(self, k: K) -> Entry<K, V>;          // the slot for that key (present or not)
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

```rok
let counts: Dict<str, i32> = {};
for w in words {
    counts.entry(w).and_modify(|n| n + 1).or_insert(1);
}
```

### Examples

```rok
let d: Dict<str, i32> = {"a": 1, "b": 2};
d["a"];                   //→ 1
d["c"];                   //→ panic (IndexError: key missing)
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

The `d[k]` notation assumes the key is present; use `.get(k)` when safe access is needed.

`d += e` (`AddAssign<Dict<K, V>>`) inserts every entry of `e` into `d` in place; on a shared key the value from `e` wins.

`Dict::from(pairs)` (`From<List<(K, V)>>`) builds a dict from a list of pairs; on a repeated key the later value wins.

### Lazy views

`keys` / `values` / `entries` are lazy views onto the current table, not copies. Modifying the dict during iteration gives unspecified results (elements may be missed or seen twice), but memory safety is preserved.

## 2.9.4. `Set<T>` - hash set

```rok
struct Set<T: Hash + Eq> {
    // internal: Dict<T, bool>
}
```

Order is not guaranteed; average performance is O(1).

```rok
impl<T: Hash + Eq> Set<T> {
    fn new() -> Set<T>;
    fn len(self) -> usize;
    fn is_empty(self) -> bool;

    fn insert(self, x: T) -> bool;        // true if newly inserted
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

`Set::from(xs)` (`From<List<T>>`) builds a set from a list, dropping duplicates. `s += x` (`AddAssign<T>`) inserts `x`.

## 2.9.5. `OrderedDict<K, V>` / `OrderedSet<T>` - insertion order

These provide the same API as `Dict` / `Set` but **preserve insertion order**, equivalent in meaning to Python 3.7+ `dict` or Java's LinkedHashMap.

```rok
struct OrderedDict<K: Hash + Eq, V> { /* ... */ }
struct OrderedSet<T: Hash + Eq> { /* ... */ }
```

They suit situations that need order preservation, such as config, JSON-like structures and reproducible output; the indices vector holding the order adds slight overhead.

- `OrderedDict` has `new` / `with_capacity` / `from` / `len` / `is_empty` / `insert` / `get` / `contains_key` / `remove` / `clear` / `keys` / `values` / `entries`, plus `d[k]` and `d[k] = v`. `keys` / `values` / `entries` iterate in insertion order. `insert` on an existing key changes only the value; the order is unchanged.
- `OrderedSet` has `new` / `from` / `len` / `is_empty` / `insert` / `remove` / `contains` / `clear` / `iter` / `to_list` and the set operations `union` / `intersection` / `difference` / `symmetric_difference` / `is_subset` / `is_superset` / `is_disjoint`. The result of a set operation follows the order of the left operand first.
- Both implement `Default`, `Clone`, `IntoIterator`, `FromIterator`, `PartialEq` / `Eq` and `Hash` (order-independent, as for `Dict` / `Set`) and `Debug`, so either can be a `Set` element or a `Dict` key.
- `d += e` on an `OrderedDict` merges `e` in place (its values win; new keys go last); `s += x` on an `OrderedSet` inserts `x`.

### Construction

There is no literal syntax; they are created only through constructor calls.

```rok
let d = OrderedDict::from([("a", 1), ("b", 2)]);
let s = OrderedSet::from([1, 2, 3]);
```

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
    fn set(self, i: usize, x: T) throws (IndexError);
    fn front(self) -> T | none;
    fn back(self) -> T | none;
    fn clear(self);
    fn iter(self) -> DequeIter<T>;             // front to back
}
```

`set` throws `IndexError` when `i` is out of range. `capacity` is the number of elements the ring buffer holds before it grows. It implements `d[i]` / `d[i] = v` (panics when out of range), `IntoIterator`, `FromIterator` (front to back), `Default`, `Clone` (`T: Clone`), `PartialEq` / `Eq` (element-wise, front to back) and `Debug`. `d += x` (`AddAssign<T>`) pushes `x` at the back.

## 2.9.5b. `BinaryHeap<T>` - priority queue

```rok
impl<T: Ord> BinaryHeap<T> {
    fn new() -> BinaryHeap<T>;                 // max-heap: largest first
}
impl<T> BinaryHeap<T> {
    fn by(before: fn(T, T) -> bool) -> BinaryHeap<T>;   // a comes first if before(a, b)
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn peek(self) -> T | none;
    fn push(self, x: T);
    fn pop(self) -> T | none;
    fn clear(self);
    fn retain(self, keep: fn(T) -> bool);
    fn iter(self) -> ListIter<T>;              // unordered
}
```

`retain` filters in place and restores the heap order, O(n). It implements `IntoIterator` (unordered, as `iter`), `Debug` (storage order, the top first) and `Clone` (`T: Clone`; the copy keeps the order of `by`); with `T: Ord`, `Default` and `FromIterator` build a max-heap, as `new`.

A heap is not `Send`: its order is a `fn` without `+ Send`. To hand the items to another thread, send them as a `List` and `collect()` a `BinaryHeap` on the receiving side.

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
    fn iter(self) -> SliceIter<T>;
    fn to_list(self) -> List<T>;             // allocates a new List
}

impl Slice<u8> {
    fn find<B: AsBytes>(self, needle: B) -> usize | none;   // offset of the first occurrence; an empty needle is at 0
}

impl<T> Index<usize> for Slice<T> { ... }               // s[i]: panics when out of range
impl<T> Sliceable<Range<usize>> for Slice<T> { ... }    // s[a:b]: a view of a view; the range is clamped to the length
impl<T> IntoIterator for Slice<T> { ... }
impl<T: PartialEq> PartialEq for Slice<T> { ... }
impl<T: Debug> Debug for Slice<T> { ... }
```

The needle of `find` is any `AsBytes` type, defined in `rok::core::collections::slice`:

```rok
trait AsBytes {
    fn byte_slice(self) -> Slice<u8>;      // the bytes as a view, no copy
}
```

It is implemented for `str` (its UTF-8 bytes), `List<u8>`, `Slice<u8>` and `[u8; N]`, so a byte string passes as the needle (its `Slice<u8>` rule, [refs-lang 1.7.5](../../refs-lang/001-lexical/107-string-literals.md#175-byte-strings)). `std::io` re-exports `AsBytes`.

`Slice` is read-only (no `IndexSet`). Negative indices count from the end ([refs-lang 3.6.1](../../refs-lang/003-expressions/306-indexing-and-slicing.md#361-indexing-arri)).

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
| `str`        | **none** (direct byte access rejected) | **none**          | `Sliceable<Range<usize>>` → `str` (copy, UTF-8 boundary check) |

`s[10]` on a `str` is deliberately not provided, for the safety of UTF-8 multibyte characters. Use `s.chars()` for character-wise access and `s[10..15]` for extraction (as a slice, multibyte boundaries are validated).

## 2.9.9. `IntoIterator` / `FromIterator`

Every container implements `IntoIterator`.

| Container           | `Item`              | Order                             |
|---------------------|---------------------|-----------------------------------|
| `List<T>`           | `T`                 | first → last                      |
| `Dict<K, V>`        | `(K, V)`            | **nondeterministic**              |
| `Set<T>`            | `T`                 | **nondeterministic**              |
| `OrderedDict<K, V>` | `(K, V)`            | insertion order                   |
| `OrderedSet<T>`     | `T`                 | insertion order                   |
| `Deque<T>`          | `T`                 | front → back                      |
| `BinaryHeap<T>`     | `T`                 | **unordered** (not pop order)     |
| `Tuple<T...>`       | **not implemented** | (heterogeneous, no Iterator meaning)  |
| `Slice<T>`          | `T`                 | view order (stride applied)       |

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
| `Dict<K, V>`  | `rok::core::collections::dict`            | open addressing                     |
| `Set<T>`      | `rok::core::collections::set`             | on top of `Dict<T, bool>`           |
| `OrderedDict` | `rok::core::collections::ordered_dict`    | linked-hash or indices              |
| `OrderedSet`  | `rok::core::collections::ordered_set`     | (same as above)                     |
| `Deque<T>`    | `rok::core::collections::deque`           | ring buffer                         |
| `BinaryHeap<T>` | `rok::core::collections::heap`          | implicit binary tree on a `List`    |
| `Tuple`       | **compiler built-in** + standard library hook | `#[literal(...)]` + compiler lookup |
| `Slice<T>`    | `rok::core::collections::slice`           | struct ptr + len + stride           |

## 2.9.11. Comparison with other languages

| Aspect          | Rust                 | Python                     | Rok                                    |
|-----------------|----------------------|----------------------------|----------------------------------------|
| Dynamic array   | `Vec<T>`             | `list`                     | `List<T>`                              |
| Hash map        | `HashMap<K, V>`      | `dict` (insertion-ordered) | `Dict<K, V>` + `OrderedDict<K, V>`     |
| Hash set        | `HashSet<T>`         | `set`                      | `Set<T>` + `OrderedSet<T>`             |
| Fixed hetero    | `(T1, T2)` tuple     | `tuple`                    | `Tuple<T1, T2>`                        |
| View            | `&[T]` / `&str`      | **slice object**           | `Slice<T>`                             |
| Literals        | `vec![...]` (macro)  | `[...]` `{...}` `(...)`    | `[...]` `{...}` `(...)` (`#[literal]`) |
| Ordered map     | `BTreeMap` (sorted map) | **default dict**        | separate `OrderedDict`                 |

Rok differs most in these ways. First, all 6 collections have first-class literal syntax, so no macro such as `vec![]` is used. Second, insertion order of `Dict` / `Set` is not guaranteed (unlike Python 3.7+), so `OrderedDict` must be used explicitly when order matters. Third, `Slice<T>` is a view with a stride, so even an expression like `xs[::2]` is zero-copy.

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
