# 3.6. Indexing and slicing

An expression enclosed in square brackets (`[`, `]`) is used in two forms that access part of a value: indexing, which extracts the element at a given position, and slicing, which cuts out a region.

```
index_expr      ::= primary "[" expr ("," expr)* "]"
slice_expr      ::= primary "[" slice_arg "]"          // no "," (3.6.3)
slice_arg       ::= expr? ":" expr? (":" expr?)?
```

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

Reads and writes map to different traits, so attempting to write to read-only data is a compile error. Multidimensional indexing desugars to a single index trait call with the indices bundled into a tuple: `g[i, j]` is `g[(i, j)]` through `Index<(I, J)>`. When a type has several `Index<K>` impls, the argument type selects one.

### Negative indices

For types indexed by position (`List`, `Slice`, fixed arrays `[T; N]`, and the slice bounds of `str` / `StrBuf`), a negative **signed integer** index counts from the end. The actual position is `len + i`.

```rok
let xs = [10, 20, 30];
xs[-1];                // 30
let i = -2;
xs[i];                 // 20 (variables too, not just literals)
xs[-4];                // panic: index -4 out of bounds for length 3
```

- An unsigned integer index is the position as is, with no extra cost for counting from the end.
- An untyped negative literal index is `isize`.
- If `len + i < 0`, it panics (`index -4 out of bounds for length 3`).
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

- Bounds are clamped to the length. Slicing never panics because of range. `xs[5:100]` takes as much as possible; `start >= end` (positive step) gives an empty result.
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
xs[::-1];              // [5, 4, 3, 2, 1]
xs[-1:0:-1];           // [5, 4, 3, 2]
xs[3:1:-1];            // [4, 3] (positions 3, 2)
xs[1:4:-2];            // []
xs[::0];               // panic: step 0
```

Single-element indexing (3.6.1) differs: negatives count from the end, but going out of range panics.

Bounds of `str` / `StrBuf` are byte offsets; they are clamped first, and if the resulting position falls in the middle of a UTF-8 character, it panics. `str` slices have no step (`s[::2]` is a type error).

### Desugaring

Slicing desugars to a range and a call to the `Sliceable` trait ([stdlib 2.3.4](../../refs-standard-libraries/200-core/203-ops.md#234-indexing-traits)).

```
arr[a : b]            →   arr.slice(a..b)                       // unsigned bounds, no step
arr[a : b]            →   arr.slice(slice_bound(a, n)..slice_bound(b, n))   // signed bounds
arr[a : b : k]        →   arr.slice(slice_range(a, b, k, n)).step(k)
arr[: b] / arr[a :]   →   omitted bounds use the defaults above (none for slice_range when a step is present)
arr[a..b] / arr[..]   →   arr.slice(a..b) / arr.slice(0..n)
```

`n` is `arr.len()`, and `arr` is evaluated only once. Every `Sliceable<Range<usize>>` implementation clamps the range with `clamp_range`, so calling `arr.slice(9..2)` directly does not panic either. The helpers used live in `rok::core::index`.

| helper                                                         | Meaning                                                            |
|----------------------------------------------------------------|--------------------------------------------------------------------|
| `clamp_range(r: Range<usize>, n: usize) -> Range<usize>`       | clamps the end to `n` and the start to the end                     |
| `slice_bound(i: isize, len: usize) -> usize`                   | `len + i` if negative; below 0 becomes 0                           |
| `slice_range(start: isize \| none, stop: isize \| none, step: isize, len: usize) -> Range<usize>` | forward range equivalent to Python's `slice.indices`. With `.step(step)` appended, it gives `arr[start:stop:step]` |
| `index_from_end(i: isize, len: usize) -> usize`                | for a single index. `len + i` if negative; panics on overflow      |

### Kinds of `Slice` results

The result type is determined by an associated type of `Sliceable`. In the standard library it is as follows ([stdlib 2.9.7](../../refs-standard-libraries/200-core/209-collections.md#297-slicet---view)).

| Target               | Result                                                               |
|----------------------|----------------------------------------------------------------------|
| `List<T>`            | `Slice<T>`: a view of the list buffer (element writes are visible)   |
| `Slice<T>`           | `Slice<T>`: a view of a view                                         |
| `[T; N]`             | `Slice<T>`: a view of a copy of the array (arrays are values)        |
| `str`, `StrBuf`      | new `str`: a byte copy (because `str` must remain a C string), with UTF-8 boundary checks |

### Slice assignment

Slice assignment (`arr[1:3] = other;`) is not supported. Assign elements one at a time or use collection methods.

## 3.6.3. Token separation of indexing and slicing

Within the same `[ ... ]`, a comma (`,`) means indexing (multidimensional or key tuple) and a colon (`:`) means slicing. Mixing the two forms in one bracket is not allowed.

```
arr[1, 2]                  // OK: indexing (i, j)
arr[1:3]                   // OK: slicing
arr[1:3, 4:6]              // ERROR: mixed
```

If multidimensional slicing is needed, a standard library multidimensional slice `trait` (for example something like `MatrixSlice` in a matrix library) may provide it in a separate form.

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
