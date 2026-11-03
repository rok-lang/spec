# 3.6. Indexing and slicing

An expression enclosed in square brackets (`[`, `]`) is used in two forms that access part of a value: indexing, which extracts the element at a given position, and slicing, which cuts out a region.

```
index_expr      ::= primary "[" part ("," part)* ","? "]"
part            ::= expr | slice_arg
slice_arg       ::= expr? ":" expr? (":" expr?)?
```

A bracket holding one `slice_arg` alone is slicing (3.6.2); every other bracket is indexing, each comma-separated part one argument (3.6.1, 3.6.3).

## 3.6.1. Indexing: `arr[i]`

Indexing writes an index expression inside the brackets to obtain the value at that place.

```rok
let v = arr[0];
let last = arr[arr.len() - 1];
let m = matrix[i, j];        // multidimensional, indices separated by commas
```

The left-hand side may be any expression; inside the brackets go one or more index expressions separated by commas. The result is the value at the indexed place.

### Desugaring

Indexing expressions desugar to trait calls as follows.

```
arr[i]                 →   <typeof(arr) as Index<typeof(i)>>::index(arr, i)
arr[i] = v;            →   <typeof(arr) as IndexSet<typeof(i)>>::index_set(ref arr, i, v)
matrix[i, j]           →   <typeof(matrix) as Index<(typeof(i), typeof(j))>>::index(matrix, (i, j))
```

A named last part `default:` gives an index with a default, `d[k, default: v]`:

```
d[k, default: v]                  →   d.get_or(k, v)                      // a read; d unchanged
d[k, default: v] = e;             →   { let t = d; let i = k; t.get_or_insert(i, v); t[i] = e; }
d[k, default: v] op= e;           →   the same, with t[i] op= e
d[k, default: v].f op= e;         →   the same, with t[i].f op= e
d[k, default: v].m(args)          →   d.get_or_insert(k, v).m(args)
```

`d` and `k` are evaluated once; `v` each time the expression runs. A type takes the form by having `get_or` and `get_or_insert` (`Dict`, [stdlib 2.9.13](../../refs-standard-libraries/200-core/209-collections.md#2913-defaults-operators-and-constructors)).

Reads and writes map to different traits, so attempting to write to read-only data is a compile error. Multidimensional indexing desugars to a single index trait call with the indices bundled into a tuple: `g[i, j]` is `g[(i, j)]` through `Index<(I, J)>`. When a type has several `Index<K>` impls, the argument type selects one.

### Negative indices

For types indexed by position (`List`, `Slice`, fixed arrays `[T; N]`, and the slice bounds of `str` / `StrBuf`), a negative **signed integer** index counts from the end. The actual position is `len + i`.

This is a rule of the index's value, not of how it is written: a negative value computed at run time counts from the end exactly as a literal does, and there is no separate from-the-end form. With `i = 0`, `xs[i - 1]` is the last element, not a panic.

```rok
let xs = [10, 20, 30];
xs[-1];                // 30
let i = -2;
xs[i];                 // 20 (variables too, not just literals)
let j = 0;
xs[j - 1];             // 30 (a computed -1 is the last element)
xs[-4];                // panic: index -4 out of bounds for length 3
```

- An unsigned integer index is the position as is, with no extra cost for counting from the end.
- An untyped negative literal index is `isize`.
- If `len + i < 0`, it panics (`index -4 out of bounds for length 3`). A non-negative index past the end panics too (`xs[3]`).
- An index that must not count from the end is unsigned (`usize`), or is checked before use.
- Types indexed by key (`Dict`) are unaffected. A negative key is just a key.
- The conversion uses `rok::core::index::index_from_end(i: isize, len: usize) -> usize`.

Evaluation order is target, index, then (for assignment) value; the target is evaluated only once.

### Bounds checking and failure

Bracket indexing can fail outside the valid range; how the failure manifests is decided by the `Index` implementation. Usually it is a panic or an error.

To get a `T | none` value without panicking, use a library-provided method such as `.get(i)`.

## 3.6.2. Slicing: `arr[start:end]`

Slicing is the form used when the brackets contain a slice form including colons.

```rok
let part = arr[1:3];
let head = arr[:5];
let tail = arr[5:];
let all = arr[:];
let stepped = arr[::2];
let reverse = arr[::-1];
let full = arr[a:b:s];
let same = arr[1..3];        // a range also works (same as arr[1:3])
let whole = arr[..];         // same as arr[:]
```

A slice form inside the brackets is processed as slicing, following this BNF.

```
slice_expr  := expr? ':' expr? (':' expr?)?
              | range_expr                      // arr[a..b], arr[..]
              | expr ',' expr ',' ...           // a comma means indexing
```

The basic form `[start : end]` denotes the region from start (inclusive) to end (exclusive). A step may be added: `[start : end : step]`. The semantics are **the same as Python slices**.

- Bounds are clamped to the length. Slicing never panics because of range, whichever form writes the bounds (`a:b` or a range `a..b`). `xs[5:100]` takes as much as possible; `start >= end` (positive step) gives an empty result.
- A negative signed integer bound counts from the end (`len + i`) and is then clamped. `xs[-10:]` is everything; `xs[:-10]` is empty.
- With a positive step, an omitted `start` is 0 and an omitted `end` is the length. An omitted step is 1.
- With a negative step, bounds are read as in Python: walk backward from `start` up to just before `end`. An omitted `start` is the last element; an omitted `end` is before the first element.
- A step of 0 panics (`slice step must not be zero`).

```rok
let xs = [1, 2, 3, 4, 5];
xs[-2:];               // [4, 5]
xs[1:-1];              // [2, 3, 4]
xs[5:100];             // [] (clamp)
xs[-10:];              // [1, 2, 3, 4, 5]
xs[:-10];              // []
xs[3:1];               // [] (start >= end)
xs[1..10];             // [2, 3, 4, 5] (a range clamps too)
xs[3..1];              // []
"abc"[1..10];          // "bc"
xs[::-1];              // [5, 4, 3, 2, 1]
xs[-1:0:-1];           // [5, 4, 3, 2]
xs[3:1:-1];            // [4, 3] (positions 3, 2)
xs[1:4:-2];            // []
xs[::0];               // panic: step 0
```

Single-element indexing (3.6.1) differs: negatives count from the end, but going out of range panics. Both rules are by design: an index names one element, which must exist; a slice names a region, which may be empty.

Bounds of `str` / `StrBuf` are byte offsets; they are clamped first, and if the resulting position falls in the middle of a UTF-8 character, it panics. `str` slices have no step (`s[::2]` is a type error).

### Desugaring

Slicing desugars to a range and a call to the `Sliceable` trait ([stdlib 2.3.4](../../refs-standard-libraries/200-core/203-ops.md#234-indexing-traits)).

```
arr[a : b]            →   arr.slice(a..b)                       // unsigned bounds, no step
arr[a : b]            →   arr.slice(slice_bound(a, n)..slice_bound(b, n))   // signed bounds
arr[a : b : k]        →   arr.slice(slice_range(a, b, k, n)).step(k)
arr[: b] / arr[a :]   →   omitted bounds use the defaults above (none for slice_range when a step is present)
arr[a..b] / arr[..]   →   arr.slice(a..b) / arr.slice(0..n)
arr[a : b]            →   slice_mut(arr, a..b)                  // a List where a SliceMut is expected
```

`n` is `arr.len()`, and `arr` is evaluated only once. Every `Sliceable<Range<usize>>` implementation clamps the range with `clamp_range`, so calling `arr.slice(9..2)` directly does not panic either. The helpers used live in `rok::core::index`.

| helper                                                         | Meaning                                                            |
|----------------------------------------------------------------|--------------------------------------------------------------------|
| `clamp_range(r: Range<usize>, n: usize) -> Range<usize>`       | clamps the end to `n` and the start to the end                     |
| `slice_bound(i: isize, len: usize) -> usize`                   | `len + i` if negative; below 0 becomes 0                           |
| `slice_range(start: isize \| none, stop: isize \| none, step: isize, len: usize) -> Range<usize>` | forward range equivalent to Python's `slice.indices`. With `.step(step)` appended, it gives `arr[start:stop:step]` |
| `index_from_end(i: isize, len: usize) -> usize`                | for a single index. `len + i` if negative; panics on overflow      |
| `slice_mut(xs: List<T>, r: Range<usize> \| none) -> SliceMut<T>` | the writable view of `xs[r]` (`none`: all of `xs`), `r` clamped; lowered code only |

### Kinds of `Slice` results

The result type is determined by an associated type of `Sliceable`. In the standard library it is as follows ([stdlib 2.9.7](../../refs-standard-libraries/200-core/209-collections.md#297-slicet---view)).

| Target               | Result                                                               |
|----------------------|----------------------------------------------------------------------|
| `List<T>`            | `Slice<T>`: a view of the list buffer (element writes are visible)   |
| `Slice<T>`           | `Slice<T>`: a view of a view                                         |
| `[T; N]`             | `Slice<T>`: a view of a copy of the array (arrays are values)        |
| `List<T>` where a `SliceMut<T>` is expected | `SliceMut<T>`: a writable view of the list ([stdlib 2.9.7a](../../refs-standard-libraries/200-core/209-collections.md#297a-slicemutt---writable-view)); no step |
| `SliceMut<T>`        | `SliceMut<T>`: a writable view of the view                           |
| `str`, `StrBuf`      | new `str`: a byte copy (because `str` must remain a C string), with UTF-8 boundary checks |

### Slice assignment

A slice is a place: `arr[a:b:k] = v;` stores through it.

```
xs[a : b] = ys;       →   { let t = xs; slice_mut(t, a..b).copy_from_slice(ys[:]) }   // a List
a[a : b : k] = v;     →   IndexSet::index_set(a, SliceArg::new(a, b, k), v)           // an AxisIndexed type (3.6.3)
```

- On a `List`, the values come from a `List` / `Slice` of exactly the same length (`copy_from_slice` panics otherwise; a change of length is `splice`). A stepped slice of a list cannot be assigned (error, with the help to store elements one at a time).
- On an axis-indexed type such as `Array` the selection is a view, and the value is a scalar that fills it (`a[1:3] = 0.0`) or an array that stretches to its shape (`a[1:3] = b`), whichever `IndexSet` impl takes the value's type.
- `str`, `StrBuf`, `Slice` and fixed arrays take no slice assignment (error); a `SliceMut` view is written element by element or with `copy_from_slice`.

## 3.6.3. Multi-axis indexing

Each comma-separated part of a bracket is one argument, and a part may be a slice: `m[1:, ::2]`, `m[i, :]`. The parts are passed as one tuple through `Index` / `IndexSet` (3.6.1), a slice part as the value `rok::core::index::SliceArg { start, stop, step }` (each `isize | none`, a missing part `none`):

```
m[1:, ::2]          →   Index::index(m, (SliceArg::new(1, none, none), SliceArg::new(none, none, 2)))
m[i, :]             →   Index::index(m, (i, SliceArg::FULL))
m[1:3, :] = x       →   IndexSet::index_set(m, (SliceArg::new(1, 3, none), SliceArg::FULL), x)
```

`SliceArg::indices(len)` applies the rules of 3.6.2 to one axis and returns `(first, count, step)`.

A type that implements the marker trait `rok::core::index::AxisIndexed` (`Array` does, [stdlib 2.9.14](../../refs-standard-libraries/200-core/209-collections.md#2914-arrayt-const-n-usize---n-dimensional-array)) receives every part as an *axis argument*, so one set of impls over `isize` serves every way of writing the index:

| Part                        | Axis argument                    | Meaning (for `Array`)                         |
|-----------------------------|----------------------------------|-----------------------------------------------|
| an integer of any type      | `isize` (a literal is one; `usize`, `u64`, `i64` and `u32` are checked, other types widen) | one position; the axis is dropped; negative counts from the end (3.6.1) |
| `a:b:k`                     | `SliceArg`                       | a strided range of the axis (3.6.2 rules)      |
| `..` beside other parts     | `rok::core::index::Rest`         | every axis the other parts leave, whole        |
| `none`                      | `rok::core::index::NewAxis`      | a new axis of length 1                         |
| anything else (a mask)      | as written                       | the impl decides (`Array<bool, N>`: a mask)    |

```rok
use rok::core::array::Array;

let a: Array<f64, 2> = [[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]];
a[1, :]          // [4.0, 5.0, 6.0]: row 1, a view
a[:, -1]         // [3.0, 6.0]: the last column
a[::-1, 1:]      // rows reversed, columns 1..
a[.., 0]         // [1.0, 4.0]
a[:, none]       // shape [2, 1, 3]
a[a .> 2.0]      // [3.0, 4.0, 5.0, 6.0]: a mask selects a copy
a[:, 0] = 0.0;   // fills a column
```

`..` alone (`a[..]`) stays the whole range (3.6.2). A type that is not `AxisIndexed` gets the parts with their own types (an integer literal takes its default type), and its `Index` impls decide what they accept.

A multi-part index with a slice among the parts on a type indexed along one axis (`List`, `Slice`, fixed arrays, `str`) is an error (E0144), with the help to slice and index in separate brackets (`m[1:][0]`) or to keep the data in an `Array`.

## 3.6.4. Empty brackets and malformed forms

The following are all malformed.

```
arr[]               // ERROR: empty brackets
arr[a:b:c:d]        // ERROR: three or more colons
arr[: : :]          // ERROR: three colons
```

These forms are rejected in the lexical and syntactic phases.

## 3.6.5. Indexing and slicing depend on `trait`s

The language itself does not decide which types support indexing or slicing. Only once the standard library implements `Index`, `IndexSet`, and `Sliceable` for a type do `arr[i]` and `arr[1:3]` work for that type.

The language defines the bracket token forms and the desugaring rules; the meaning belongs to the library.
