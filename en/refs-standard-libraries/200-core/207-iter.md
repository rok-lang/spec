# 2.7. iter - iteration traits

The `rok::core::iter` module is the **desugaring authority** for `for x in xs`. Every `for` loop dispatches to the traits of this chapter.

The `for` syntax of [refs-lang 3.13 Loop expressions](../../refs-lang/003-expressions/313-loop-expressions.md) desugars to a combination of `IntoIterator::into_iter` and `Iterator::next` calls.

## 2.7.1. `Iterator` - stepping

The first trait is `Iterator`, with the following signature.

```rok
#[lang("iterator_trait")]
trait Iterator {
    type Item;

    fn next(ref self) -> Item | none;
}
```

The core behavior is simple: each call to `next` returns the next element, and returns `none` when there are no more elements.

Note that the signature is `ref self`. The iterator itself holds **state** and updates it in place on every call. Iterators are values: a copied `ListIter` continues independently, and `next(ref self)` advances only that copy.

### Example (user code)

Calling it directly looks like this.

```rok
let xs: List<i32> = [10, 20, 30];
let it = xs.into_iter();         // iterator created by IntoIterator
let v1 = it.next();              //→ 10
let v2 = it.next();              //→ 20
let v3 = it.next();              //→ 30
let v4 = it.next();              //→ none
```

In real code, `for` is more common than direct calls.

```rok
for x in xs {
    use(x);
}
// → the compiler desugars this to:
//   let __it = xs.into_iter();
//   loop {
//       match __it.next() {
//           v: i32 => use(v),
//           none => break,
//       }
//   }
```

### Methods - rich default impls

The only method a user must implement for `Iterator` is `next(ref self)`. All other methods are provided as default impls on top of `next`.

> [!NOTE]
> The `fn(Item) -> U` in the signatures below is the unified callable type defined by [refs-lang 4.4 Function types](../../refs-lang/004-functions/404-function-type.md), and accepts **both functions and closures**. Both `it.map(some_fn)` and `it.map(|x| x * 2)` fit the same position. Note that this is a single form, unlike Rust's `fn`/`Fn`/`FnMut`/`FnOnce`.

```rok
impl Iterator {
    fn next(ref self) -> Item | none;       // user-defined

    // transforming adapters (lazy)
    fn map<U>(self, f: fn(Item) -> U) -> Map<Self, U>;
    fn filter(self, pred: fn(Item) -> bool) -> Filter<Self>;
    fn enumerate(self) -> Enumerate<Self>;        // (index, item) pairs
    fn zip<J: Iterator>(self, other: J) -> Zip<Self, J>;    // Item = (Item, J::Item)
    fn chain<I: Iterator<Item = Item>>(self, other: I) -> Chain<Self, I>;
    fn take(self, n: usize) -> Take<Self>;
    fn skip(self, n: usize) -> Skip<Self>;
    fn step_by(self, step: usize) -> StepBy<Self>;
    fn rev(self) -> Rev<Self>;                    // DoubleEndedIterator only
    fn flatten(self) -> Flatten<Self> where Item: IntoIterator;
    fn flat_map<U: IntoIterator>(self, f: fn(Item) -> U) -> FlatMap<Self, U>;
    fn peekable(self) -> Peekable<Self>;          // peek at the next element
    fn take_while(self, pred: fn(Item) -> bool) -> TakeWhile<Self>;
    fn skip_while(self, pred: fn(Item) -> bool) -> SkipWhile<Self>;
    fn scan<S>(self, init: S, f: fn(S, Item) -> S) -> Scan<Self, S>;   // Item = S
    fn inspect(self, f: fn(Item)) -> InspectIter<Self>;    // f sees each item
    fn cycle(self) -> Cycle<Self>;                // endless repeat
    fn chunks(self, size: usize) -> Chunks<Self>;     // Item = List<Item>
    fn windows(self, size: usize) -> Windows<Self>;   // Item = List<Item>

    fn exact_len(self) -> usize | none;           // length without stepping, else none

    // terminal methods (eager)
    fn count(self) -> usize;
    fn sum(self) -> Item where Item: Add<Output = Item> + Zero;     // 0 if empty
    fn product(self) -> Item where Item: Mul<Output = Item> + One;  // 1 if empty
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
    fn for_each(self, f: fn(Item));               // side effects only
    fn last(self) -> Item | none;
    fn nth(ref self, n: usize) -> Item | none;
}
```

### `sum` / `product` and `Zero` / `One`

`sum` starts from `Zero::zero()` and accumulates with `+`; `product` starts from `One::one()` and accumulates with `*`. The sum of an empty iterator is therefore 0 and its product is 1. `Zero` / `One` are traits of `rok::core::num` ([2.1](201-numbers.md)).

### Stopping, running state, groups

- `take_while(pred)` yields items until `pred` first rejects one. That item is taken from the inner iterator and dropped; nothing after it is taken, so side effects past the stop do not run. `skip_while(pred)` drops items until `pred` first rejects one, then yields that one and every later item unchecked.
- `scan(init, f)` yields the running state: `f(init, x0)`, then `f` of that and `x1`, and so on. It is `fold` that yields every step, so it yields as many items as the inner iterator.
- `inspect(f)` passes each item to `f` and yields it unchanged.
- `cycle()` restarts from a copy of the iterator as `cycle` received it whenever it runs out, so it is endless unless the iterator is empty, in which case it yields nothing. A generator's copies share one position (2.7.10a), so cycling one does not repeat it.
- `chunks(size)` yields lists of `size` items in order, the last one shorter when the items run out. `windows(size)` yields lists of `size` consecutive items sliding by one, and nothing when there are fewer than `size` items. A `size` of 0 panics for both. `List` and `Slice` have their own `chunks` / `windows` that yield views ([2.9.7](209-collections.md#297-slicet---view)).
- `try_fold` and `try_for_each` take a throwing `f`: the first error stops the loop, no later item is visited, and the error is thrown to the caller.
- `partition(pred)` returns the items `pred` accepts and the rest, as two lists, each in order. `unzip()` splits an iterator of pairs `(A, B)` into a `List<A>` and a `List<B>`. Its items must be pairs: `A` and `B` are inferred from `Item = (A, B)`, and items of any other form are an error at the call.

```rok
let xs = [1, 2, 3, 4];
let small: List<i32> = xs.iter().take_while(|x| x < 3).collect();   //→ [1, 2]
let sums: List<i32> = xs.iter().scan(0, |acc, x| acc + x).collect(); //→ [1, 3, 6, 10]
let pairs: List<List<i32>> = xs.iter().windows(2).collect();        //→ [[1, 2], [2, 3], [3, 4]]
let (even, odd) = xs.iter().partition(|x| x % 2 == 0);              //→ [2, 4], [1, 3]
let (ns, ss) = [(1, "a"), (2, "b")].iter().unzip();                 //→ [1, 2], ["a", "b"]
```

### Lazy vs eager

These methods fall into two groups. Transforming adapters such as `map`, `filter` and `take` are **lazy**: they only return a new iterator and perform no computation. Stepping starts only when a terminal method such as `count`, `sum`, `collect` or `for_each` is called.

```rok
let xs = [1, 2, 3, 4, 5];
let r = xs.into_iter()
    .map(|x| x * 2)            // lazy: nothing computed yet
    .filter(|x| x > 4)         // lazy
    .sum();                    // eager: stepping happens here
//→ 6 + 8 + 10 = 24
```

### User-defined Iterator

An example of defining an Iterator directly:

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

## 2.7.2. `IntoIterator` - where `for` starts

Next is the `IntoIterator` trait, the starting point of a `for` loop.

```rok
#[lang("into_iterator_trait")]
trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Item>;

    fn into_iter(self) -> IntoIter;
}
```

`IntoIterator` is the adapter that lets a collection create an iterator. `for x in xs` does not treat **xs itself** as an iterator; it creates an iterator with `xs.into_iter()` and calls `next` on it.

### Why they are separate

The two traits are separate because the two viewpoints are fundamentally different.

- A collection itself (List, Dict, etc.) is a stateless container and does not know which position it points to.
- An iterator has progress state and tracks how far it has gone.

Iterating the same collection twice therefore needs a new iterator each time, and `into_iter` is where it is created.

### Automatic impl

`Iterator` itself gets an automatic impl of `IntoIterator`, whose `into_iter` returns self unchanged.

```rok
// every Iterator is automatically an IntoIterator
impl<I: Iterator> IntoIterator for I {
    type Item = I::Item;
    type IntoIter = I;
    fn into_iter(self) -> I { self }
}
```

So `for x in iter` works as is: if iter is already an Iterator, `into_iter` returns self.

### Exact form of the `for` desugaring

The desugaring of a `for` loop, stated more precisely:

```rok
for x in expr {
    body
}
```

desugars to:

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

See [refs-lang 3.13](../../refs-lang/003-expressions/313-loop-expressions.md) for the detailed desugaring rules.

## 2.7.3. `Step` - the one-step abstraction

Iteration over `Range<T>` rests on the simple operation "move to the next value". This differs in meaning from general arithmetic (`Add`): for example, the next value of a `char` is codepoint + 1, but `char + char` itself is undefined. Rok therefore puts stepping in a separate trait.

```rok
trait Step: PartialOrd + Clone {
    fn forward_checked(self, count: usize) -> Self | none;
    fn backward_checked(self, count: usize) -> Self | none;
    fn steps_between(start: Self, end: Self) -> usize | none;
}
```

The methods mean:

- `forward_checked(self, n)`: returns the value `n` steps forward from `self`, or `none` if it exceeds the type's range.
- `backward_checked(self, n)`: the value moved backward; likewise `none` if out of range.
- `steps_between(start, end)`: computes how many steps lie from `start` to `end`, returning `none` when that is not representable (e.g. negative, overflow).

### Why it is separate from the arithmetic traits

`Step` is separated from `Add` for three reasons.

- **Separation of meaning**: `+1` is a simple increment, not arbitrary arithmetic. `MyVec3 + MyVec3` is meaningful but `Range<MyVec3>` is not; without a separate `Step` impl it is rejected.
- **Excluding floating-point**: `f32` / `f64` have no `Step` impl. `Range<f64>` is therefore a compile error, and ambiguous notation such as `0.0..1.0` is removed at the specification level.
- **Efficient skipping**: `Iterator::nth(n)` and `step_by(n)` call `forward_checked(_, n)` once and run in O(1). With only `succ`, the same operation would be O(n).

### Step impl matrix

The standard library implements `Step` only for integer primitives and `char`.

| Type                 | impl     | Notes                                                |
|----------------------|----------|------------------------------------------------------|
| `i8`..`i64`, `isize` | **yes**  | `forward_checked` returns `none` on overflow         |
| `u8`..`u64`, `usize` | **yes**  | `backward_checked` returns `none` on underflow       |
| `int`                | **no**   | no standard library impl                             |
| `char`               | **yes**  | steps by codepoint, skipping surrogates (`0xD800..=0xDFFF`) |
| `bool`               | **no**   | no standard library impl                             |
| `f32`, `f64`         | **no**   | `Range<f64>` rejected: ambiguous meaning             |
| `float`              | **no**   | (same as above)                                      |

User-defined types may add `impl Step for MyType` freely, which immediately enables `Range<MyType>`.

```rok
enum Weekday { Mon, Tue, Wed, Thu, Fri, Sat, Sun }

impl Step for Weekday {
    fn forward_checked(self, n: usize) -> Weekday | none { ... }
    fn backward_checked(self, n: usize) -> Weekday | none { ... }
    fn steps_between(s: Weekday, e: Weekday) -> usize | none { ... }
}

for d in Weekday::Mon..=Weekday::Fri {
    schedule(d);     // the 5 weekdays
}
```

## 2.7.4. `Range<T>` / `RangeInclusive<T>` / `RangeFrom<T>` / `RangeTo<T>` / `RangeToInclusive<T>` / `RangeFull`

The `a..b` syntax of [refs-lang 3.7 Range expressions](../../refs-lang/003-expressions/307-range-expressions.md) desugars to these standard library types.

### `Range<T>` - `a..b` (exclusive)

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

`#[literal( expr ".." expr )]` registers the literal syntax, so that notation such as `0..5` desugars to the `Range` struct ([refs-lang 6.5](../../refs-lang/006-meta-system/605-literal-magic.md) is authoritative).

```rok
for i in 0..5 {
    use(i);          //→ 0, 1, 2, 3, 4
}

let v: List<i32> = (0..10).collect();   //→ [0, 1, ..., 9]
```

### `RangeInclusive<T>` - `a..=b`

```rok
#[literal( expr "..=" expr )]
struct RangeInclusive<T> {
    start: T;
    end: T;
}

impl<T: Step> Iterator for RangeInclusive<T> { ... }
```

Fields other than `start` and `end` (e.g. an internal flag marking that the end was reached) are implementation details and private. Even when `end` is the maximum of the type, `a..=b` yields exactly up to `b`.

```rok
for i in 0..=5 {
    use(i);          //→ 0, 1, 2, 3, 4, 5
}
```

### `RangeFrom<T>` - `a..` (infinite)

```rok
#[literal( expr ".." )]
struct RangeFrom<T> { start: T; }

impl<T: Step> Iterator for RangeFrom<T> {
    type Item = T;
    fn next(ref self) -> T | none {
        let cur = self.start.clone();
        self.start = self.start.forward_checked(1)?;  // overflow → none (e.g. reaching i32::MAX)
        cur
    }
}
```

```rok
let it = (10..).take(3);     //→ [10, 11, 12] (take is a lazy adapter)
```

`RangeFrom` does not yield the maximum of the type. Since the maximum has no next value, iteration ends on reaching the maximum, before yielding it (`(254u8..)` yields only `254` and ends).

### `RangeTo<T>` / `RangeToInclusive<T>` / `RangeFull`

```rok
#[literal( ".." expr )]
struct RangeTo<T> { end: T; }

#[literal( "..=" expr )]
struct RangeToInclusive<T> { end: T; }               // ..=b

#[literal( ".." )]
struct RangeFull;
```

`..b` / `..=b` (no start) and `..` (neither) are meaningful only in slice or index contexts and do not create iterators.

```rok
let xs = [1, 2, 3, 4, 5];
let s = xs[..3];          //→ [1, 2, 3]  (Slice: the Slice trait of 2.3 ops)
```

## 2.7.5. `FromIterator<T>` - dispatch of `collect`

The trait that handles dispatch of `collect` is `FromIterator<T>`.

```rok
trait FromIterator<T> {
    fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Self;
}
```

When `Iterator::collect` is called, the target collection's `from_iter` is called.

```rok
let v: List<i32> = (0..5).collect();
// = List::from_iter((0..5).into_iter())
```

Every collection (`List`, `Dict`, `Set`, `str`, etc.) implements `FromIterator`.

## 2.7.6. IntoIterator impls per collection

How each collection implements `IntoIterator`:

| Collection          | `Item` type | Behavior                                  |
|---------------------|-------------|-------------------------------------------|
| `List<T>`           | `T`         | In order from the first element to the last |
| `Dict<K, V>`        | `(K, V)`    | Iteration order **not guaranteed** (HashDict) |
| `OrderedDict<K, V>` | `(K, V)`    | Insertion order                           |
| `Set<T>`            | `T`         | Iteration order **not guaranteed**        |
| `OrderedSet<T>`     | `T`         | Insertion order                           |
| `Tuple<A, B, C>`    | (special)   | **Creates no iterator**; use pattern matching  |
| `Range<T>`          | `T`         | From start to end-1                       |
| `Slice<T>`          | `T`         | From the start of the view to its end     |

### Iteration order of Dict / Set

The default `Dict<K, V>` and `Set<T>` are **hash-based**, so iteration order depends on the hash distribution and may differ between runs or compilations.

When order must be guaranteed, use `OrderedDict<K, V>` or `OrderedSet<T>`, which preserve insertion order.

## 2.7.7. Infinite iterators and `take`

An infinite iterator, such as `RangeFrom` or a user-defined one, combined directly with an eager method loops forever.

```rok
(0..).count();         // infinite loop: count runs to the end
(0..).sum();           // infinite loop
```

Make it finite with `take(n)`.

```rok
(0..).take(100).count();     //→ 100
(0..).take(10).sum();        //→ 0+1+...+9 = 45
```

`Iterator` does not detect infinity. The user must deliberately combine lazy adapters.

## 2.7.8. Function broadcast - desugaring `f.(x)`

Function broadcast in [refs-lang](../../refs-lang/003-expressions/303-broadcast-operators.md) (`sin.(xs)`, `f.(a, b)`) desugars to `Iterator::map` with no trait of its own.

```rok
sin.(xs)   ≡   xs.into_iter().map(sin).collect()
g.(a, b)   ≡   a.into_iter().zip(b.into_iter()).map(|(x, y)| g(x, y)).collect()
```

## 2.7.9. `DoubleEndedIterator` - bidirectional stepping

Iterators that can step in both directions are modeled by the `DoubleEndedIterator` trait.

```rok
trait DoubleEndedIterator: Iterator {
    fn next_back(ref self) -> Item | none;
}
```

This trait represents iterators that can step from the back; the `rev()` adapter and some methods such as `rfold` and `rfind` require it.

The iterators of `List<T>` and `Slice<T>`, and the following types, must implement this trait.

- `Range<T>`, `RangeInclusive<T>`
- `Rev<I>` (when `I` is bidirectional), `Map<I, U>`, `Filter<I>`, `Chain<A, B>`, `InspectIter<I>` (when the component iterators are bidirectional)
- `Enumerate<I>`, `Zip<A, B>`, `Take<I>`, `Skip<I>`, `StepBy<I>`: when the component iterators are `DoubleEndedIterator + ExactSizeIterator`, because counting from the back needs the length.

`Dict<K, V>` and `Set<T>`, being unordered, do not implement it.

```rok
let r: List<i32> = (0..5).rev().collect();    //→ [4, 3, 2, 1, 0]
```

## 2.7.10. `ExactSizeIterator` - iterators with known length

An iterator whose element count is known before stepping is represented by `ExactSizeIterator`.

```rok
trait ExactSizeIterator: Iterator {
    fn len(self) -> usize;
}
```

`len()` is the number of remaining elements. `Iterator::exact_len()` returns that number when it is known without stepping and `none` otherwise (the default); the standard-library `ExactSizeIterator` types return their `len()`, as do the adapters listed below when built on them. The default `count()` returns `exact_len()` when it is not `none`, so `count()` does not call `map`'s function: `xs.iter().map(f).take(3).count()` calls `f` zero times. The same holds for `scan`'s and `inspect`'s functions. An iterator of unknown length, such as `Filter`, `TakeWhile`, `SkipWhile` or `(0..).take(100)`, is stepped. The iterators of `List`, `Slice` and `Range`, the `windows` / `chunks` views of `List` and `Slice`, and `Enumerate`, `Zip`, `Take`, `Skip`, `StepBy`, `Rev`, `Map`, `Scan`, `InspectIter`, `Chunks` when their component iterators are all `ExactSizeIterator`, implement this trait. `Windows` has an `exact_len` over an exact-size iterator, and `Cycle` has one (0) only when its iterator is empty.

## 2.7.10a. `GenIter<T>` / `TryGenIter<T, E>` - the value of a generator

These are the values of a `yield fn` call and of a `yield { .. }` block ([refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#result-types-of-coroutine-functions)).

```rok
struct GenIter<T> { /* private */ }
impl<T> Iterator for GenIter<T> {
    type Item = T;
    fn next(ref self) -> T | none;          // runs the body to the next yield
}

struct TryGenIter<T, E> { /* private */ }    // yield fn .. throws (E)
impl<T, E> TryGenIter<T, E> {
    fn next(ref self) throws (E) -> T | none;
}
```

`TryGenIter` does not implement `Iterator`, because its `next` throws. It is advanced with `try for` or `try g.next()`. The private fields are not specified. Copies share the same coroutine frame and therefore share the progress position.

## 2.7.11. Module mapping

The mapping of each trait to its lang slot and standard library location:

| Item                  | lang slot             | Standard library location |
|-----------------------|-----------------------|---------------------------|
| `Iterator` trait      | `iterator_trait`      | Standard library          |
| `IntoIterator` trait  | `into_iterator_trait` | Standard library          |
| `Step` trait          | (none)                | Standard library          |
| `FromIterator` trait  | (none)                | Standard library          |
| `DoubleEndedIterator` | (none)                | Standard library          |
| `ExactSizeIterator`   | (none)                | Standard library          |
| `Range<T>` and the other 5 types | `range_type` | `rok/range/`            |

Primitives themselves have no `Iterator` impl (i32 is not a `for` target). Only collections and Ranges are.

## 2.7.12. async - the `Stream` trait

Asynchronous iterators are covered in [2.8 stream](208-stream.md). The `for await x in stream` syntax ([refs-lang 3.13](../../refs-lang/003-expressions/313-loop-expressions.md)) dispatches to `Stream::poll_next`.
