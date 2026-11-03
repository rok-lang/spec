# 3.6. 인덱싱과 슬라이싱

대괄호(`[`, `]`)로 둘러싼 표현식은 값의 일부에 접근하는 두 가지 형태에 쓰입니다. 하나는 특정 위치의 원소를 꺼내는 인덱싱이고, 다른 하나는 영역을 잘라 내는 슬라이싱입니다.

```
index_expr      ::= primary "[" part ("," part)* ","? "]"
part            ::= expr | slice_arg
slice_arg       ::= expr? ":" expr? (":" expr?)?
```

`slice_arg` 하나만 든 괄호는 슬라이싱입니다(3.6.2). 그 밖의 괄호는 모두 인덱싱이며, 콤마로 나뉜 부분 하나가 인자 하나입니다(3.6.1, 3.6.3).

## 3.6.1. 인덱싱: `arr[i]`

인덱싱은 대괄호 안에 인덱스 표현식을 써서 그 자리의 값을 얻습니다.

```rok
let v = arr[0];
let last = arr[arr.len() - 1];
let m = matrix[i, j];        // 다차원, 인덱스를 콤마로 구분
```

좌변에는 임의의 표현식이 올 수 있고, 대괄호 안에는 인덱스 표현식을 하나 이상 콤마로 구분해 씁니다. 결과는 인덱싱된 자리의 값입니다.

### 풀어 쓰기

인덱싱 표현식은 다음과 같이 트레잇 호출로 풀립니다.

```
arr[i]                 →   <typeof(arr) as Index<typeof(i)>>::index(arr, i)
arr[i] = v;            →   <typeof(arr) as IndexSet<typeof(i)>>::index_set(ref arr, i, v)
matrix[i, j]           →   <typeof(matrix) as Index<(typeof(i), typeof(j))>>::index(matrix, (i, j))
```

이름 붙은 마지막 부분 `default:`는 기본값이 있는 인덱스 `d[k, default: v]`를 만듭니다.

```
d[k, default: v]                  →   d.get_or(k, v)                      // 읽기, d는 그대로
d[k, default: v] = e;             →   { let t = d; let i = k; t.get_or_insert(i, v); t[i] = e; }
d[k, default: v] op= e;           →   위와 같고 t[i] op= e
d[k, default: v].f op= e;         →   위와 같고 t[i].f op= e
d[k, default: v].m(args)          →   d.get_or_insert(k, v).m(args)
```

`d`와 `k`는 한 번 평가하고 `v`는 식이 실행될 때마다 평가합니다. `get_or`와 `get_or_insert`가 있는 타입이 이 형태를 받습니다(`Dict`, [stdlib 2.9.13](../../refs-standard-libraries/200-core/209-collections.md#2913-기본값-연산자-생성자)).

읽기와 쓰기는 서로 다른 트레잇에 매핑되므로, 읽기 전용 자료에 쓰려고 하면 컴파일 에러입니다. 다차원 인덱싱은 인덱스를 튜플로 묶어 인덱스 트레잇 호출 하나로 풉니다. `g[i, j]`는 `Index<(I, J)>`를 거치는 `g[(i, j)]`입니다. 한 타입에 `Index<K>` impl이 여럿 있으면 인자 타입으로 그중 하나를 고릅니다.

### 음수 인덱스

위치로 인덱싱하는 타입(`List`, `Slice`, 고정 배열 `[T; N]`, 그리고 `str` / `StrBuf`의 슬라이스 경계)에서는 음수인 **부호 있는 정수** 인덱스를 끝에서부터 셉니다. 실제 위치는 `len + i`입니다.

이 규칙은 인덱스를 어떻게 썼는지가 아니라 그 값에 따릅니다. 실행 중에 계산한 음수도 리터럴과 똑같이 끝에서부터 세며, 끝에서 세는 형식이 따로 있지 않습니다. `i = 0`이면 `xs[i - 1]`은 panic이 아니라 마지막 원소입니다.

```rok
let xs = [10, 20, 30];
xs[-1];                // 30
let i = -2;
xs[i];                 // 20 (리터럴뿐 아니라 변수도)
let j = 0;
xs[j - 1];             // 30 (계산한 -1도 마지막 원소)
xs[-4];                // panic: index -4 out of bounds for length 3
```

- 부호 없는 정수 인덱스는 위치 그대로이며, 끝에서 세는 추가 비용이 없습니다.
- 타입 없는 음수 리터럴 인덱스는 `isize`입니다.
- `len + i < 0`이면 panic이 일어납니다(`index -4 out of bounds for length 3`). 끝을 넘는 음수 아닌 인덱스도 panic입니다(`xs[3]`).
- 끝에서부터 세면 안 되는 인덱스는 부호 없는 정수(`usize`)로 쓰거나, 쓰기 전에 검사합니다.
- 키로 인덱싱하는 타입(`Dict`)은 영향을 받지 않습니다. 음수 키는 그냥 키입니다.
- 변환에는 `rok::core::index::index_from_end(i: isize, len: usize) -> usize`를 씁니다.

평가 순서는 대상, 인덱스, (대입이면) 값 순이고, 대상은 한 번만 평가합니다.

### 경계 검사와 실패

대괄호 인덱싱은 유효 범위 밖에서 실패할 수 있습니다. 실패가 어떻게 드러나는지는 `Index` 구현이 정합니다. 보통은 panic이나 에러입니다.

panic 없이 `T | none` 값을 받으려면 라이브러리의 `.get(i)` 같은 메서드를 쓰면 됩니다.

## 3.6.2. 슬라이싱: `arr[start:end]`

슬라이싱은 대괄호 안에 콜론이 들어간 슬라이스 형태가 올 때 쓰는 형식입니다.

```rok
let part = arr[1:3];
let head = arr[:5];
let tail = arr[5:];
let all = arr[:];
let stepped = arr[::2];
let reverse = arr[::-1];
let full = arr[a:b:s];
let same = arr[1..3];        // 범위로도 (arr[1:3] 과 같음)
let whole = arr[..];         // arr[:] 과 같음
```

대괄호 안에 슬라이스 형태가 오면 슬라이싱으로 처리하며, 그 형태는 다음 BNF를 따릅니다.

```
slice_expr  := expr? ':' expr? (':' expr?)?
              | range_expr                      // arr[a..b], arr[..]
              | expr ',' expr ',' ...           // 콤마면 인덱싱
```

기본 형태 `[start : end]`는 시작(포함)부터 끝(미포함)까지의 영역입니다. 보폭(step)을 더한 `[start : end : step]`도 쓸 수 있습니다. 의미는 **Python 슬라이스와 같습니다**.

- 경계는 길이에 맞춰 잘라 맞춥니다(clamp). 경계를 `a:b`로 쓰든 범위 `a..b`로 쓰든, 슬라이싱은 범위를 벗어나도 panic이 일어나지 않습니다. `xs[5:100]`은 가능한 만큼 가져오고, `start >= end`(양수 step)이면 빈 결과입니다.
- 부호 있는 정수 경계가 음수이면 끝에서부터 센 뒤(`len + i`) 잘라 맞춥니다. `xs[-10:]`은 전체이고 `xs[:-10]`은 빈 결과입니다.
- step이 양수이면 생략한 `start`는 0, 생략한 `end`는 길이입니다. step을 생략하면 1입니다.
- step이 음수이면 경계를 Python처럼 읽습니다. `start`에서 시작해 `end` 바로 앞까지 거꾸로 갑니다. 생략한 `start`는 마지막 원소이고, 생략한 `end`는 첫 원소 앞입니다.
- step이 0이면 panic이 일어납니다(`slice step must not be zero`).

```rok
let xs = [1, 2, 3, 4, 5];
xs[-2:];               // [4, 5]
xs[1:-1];              // [2, 3, 4]
xs[5:100];             // [] (clamp)
xs[-10:];              // [1, 2, 3, 4, 5]
xs[:-10];              // []
xs[3:1];               // [] (start >= end)
xs[1..10];             // [2, 3, 4, 5] (범위도 잘라 맞춤)
xs[3..1];              // []
"abc"[1..10];          // "bc"
xs[::-1];              // [5, 4, 3, 2, 1]
xs[-1:0:-1];           // [5, 4, 3, 2]
xs[3:1:-1];            // [4, 3] (위치 3, 2)
xs[1:4:-2];            // []
xs[::0];               // panic: step 0
```

단일 원소 인덱싱(3.6.1)은 다릅니다. 음수는 끝에서부터 세지만, 범위를 벗어나면 panic이 일어납니다. 두 규칙 모두 의도한 설계입니다. 인덱스는 반드시 있어야 하는 원소 하나를 가리키고, 슬라이스는 비어 있어도 되는 영역을 가리킵니다.

`str` / `StrBuf`의 경계는 바이트 오프셋입니다. 먼저 잘라 맞추고, 그 위치가 UTF-8 문자 중간이면 panic이 일어납니다. `str` 슬라이스에는 step이 없습니다(`s[::2]`는 타입 에러).

### 풀어 쓰기

슬라이싱은 범위와 `Sliceable` 트레잇([stdlib 2.3.4](../../refs-standard-libraries/200-core/203-ops.md#234-인덱싱-트레잇)) 호출로 풀립니다.

```
arr[a : b]            →   arr.slice(a..b)                       // 부호 없는 경계, step 없음
arr[a : b]            →   arr.slice(slice_bound(a, n)..slice_bound(b, n))   // 부호 있는 경계
arr[a : b : k]        →   arr.slice(slice_range(a, b, k, n)).step(k)
arr[: b] / arr[a :]   →   생략한 경계는 위의 기본값 (step 이 있으면 slice_range 의 none)
arr[a..b] / arr[..]   →   arr.slice(a..b) / arr.slice(0..n)
arr[a : b]            →   slice_mut(arr, a..b)                  // SliceMut 을 기대하는 자리의 List
```

`n`은 `arr.len()`이고 `arr`은 한 번만 평가합니다. 모든 `Sliceable<Range<usize>>` 구현은 범위를 `clamp_range`로 잘라 맞추므로, `arr.slice(9..2)`를 직접 호출해도 panic이 일어나지 않습니다. 여기에 쓰는 도우미 함수는 `rok::core::index`에 있습니다.

| 도우미 함수                                                    | 의미                                                               |
|----------------------------------------------------------------|--------------------------------------------------------------------|
| `clamp_range(r: Range<usize>, n: usize) -> Range<usize>`       | 끝을 `n`으로, 시작을 끝으로 잘라 맞춤                               |
| `slice_bound(i: isize, len: usize) -> usize`                   | 음수면 `len + i`, 0 아래는 0                                       |
| `slice_range(start: isize \| none, stop: isize \| none, step: isize, len: usize) -> Range<usize>` | Python `slice.indices`와 같은 정방향 범위. `.step(step)`을 붙이면 `arr[start:stop:step]`   |
| `index_from_end(i: isize, len: usize) -> usize`                | 단일 인덱스용. 음수면 `len + i`, 넘치면 panic                      |
| `slice_mut(xs: List<T>, r: Range<usize> \| none) -> SliceMut<T>` | `xs[r]`의 쓰기 가능한 뷰(`none`이면 `xs` 전체), `r`은 잘라 맞춤. 낮춘 코드 전용 |

### 슬라이스 결과의 종류

결과 타입은 `Sliceable`의 연관 타입이 정합니다. 표준 라이브러리에서는 다음과 같습니다([stdlib 2.9.7](../../refs-standard-libraries/200-core/209-collections.md#297-slicet-뷰)).

| 대상                 | 결과                                                                 |
|----------------------|----------------------------------------------------------------------|
| `List<T>`            | `Slice<T>`: 리스트 버퍼의 뷰 (원소 쓰기가 보임)                      |
| `Slice<T>`           | `Slice<T>`: 뷰의 뷰                                                  |
| `[T; N]`             | `Slice<T>`: 배열 사본의 뷰 (배열은 값)                               |
| `SliceMut<T>`을 기대하는 자리의 `List<T>` | `SliceMut<T>`: 리스트에 쓰는 뷰([stdlib 2.9.7a](../../refs-standard-libraries/200-core/209-collections.md#297a-slicemutt-쓰기-가능한-뷰)), step 없음 |
| `SliceMut<T>`        | `SliceMut<T>`: 뷰의 쓰기 가능한 뷰                                   |
| `str`, `StrBuf`      | 새 `str`: 바이트 사본 (`str`은 C 문자열로 남아야 하므로), UTF-8 경계 검사  |

### 슬라이스 대입

슬라이스는 자리(place)입니다. `arr[a:b:k] = v;`는 슬라이스를 거쳐 저장합니다.

```
xs[a : b] = ys;       →   { let t = xs; slice_mut(t, a..b).copy_from_slice(ys[:]) }   // List
a[a : b : k] = v;     →   IndexSet::index_set(a, SliceArg::new(a, b, k), v)           // AxisIndexed 타입 (3.6.3)
```

- `List`에서는 정확히 같은 길이의 `List` / `Slice`에서 값을 가져옵니다(다르면 `copy_from_slice`가 panic합니다. 길이를 바꾸려면 `splice`). 스텝이 있는 리스트 슬라이스에는 대입할 수 없습니다(에러, 원소를 하나씩 저장하라는 도움말).
- `Array` 같은 축 인덱싱 타입에서는 선택이 뷰이고, 값은 그 뷰를 채우는 스칼라(`a[1:3] = 0.0`)이거나 그 모양으로 늘어나는 배열(`a[1:3] = b`)입니다. 값의 타입을 받는 `IndexSet` impl이 고릅니다.
- `str`, `StrBuf`, `Slice`, 고정 배열은 슬라이스 대입을 받지 않습니다(에러). `SliceMut` 뷰는 원소 단위로 쓰거나 `copy_from_slice`로 씁니다.

## 3.6.3. 다축 인덱싱

괄호 안의 콤마로 나뉜 부분 하나하나가 인자 하나이며, 부분은 슬라이스일 수 있습니다: `m[1:, ::2]`, `m[i, :]`. 부분들은 튜플 하나로 묶여 `Index` / `IndexSet`(3.6.1)을 거치고, 슬라이스 부분은 값 `rok::core::index::SliceArg { start, stop, step }`(각각 `isize | none`, 빠진 부분은 `none`)로 넘어갑니다.

```
m[1:, ::2]          →   Index::index(m, (SliceArg::new(1, none, none), SliceArg::new(none, none, 2)))
m[i, :]             →   Index::index(m, (i, SliceArg::FULL))
m[1:3, :] = x       →   IndexSet::index_set(m, (SliceArg::new(1, 3, none), SliceArg::FULL), x)
```

`SliceArg::indices(len)`는 3.6.2의 규칙을 한 축에 적용해 `(first, count, step)`을 돌려줍니다.

표시 trait `rok::core::index::AxisIndexed`를 구현한 타입(`Array`가 구현합니다, [stdlib 2.9.14](../../refs-standard-libraries/200-core/209-collections.md#2914-arrayt-const-n-usize-n차원-배열))은 모든 부분을 *축 인자*로 받습니다. 그래서 `isize`에 대한 impl 한 벌이 인덱스를 쓰는 모든 방식을 받습니다.

| 부분                         | 축 인자                           | 의미 (`Array`)                                |
|-----------------------------|----------------------------------|-----------------------------------------------|
| 아무 타입의 정수              | `isize` (리터럴은 `isize`, `usize`·`u64`·`i64`·`u32`는 검사, 그 밖은 넓힘) | 위치 하나. 그 축은 빠짐. 음수는 끝에서 셈(3.6.1) |
| `a:b:k`                     | `SliceArg`                       | 축의 보폭 있는 범위(3.6.2 규칙)                  |
| 다른 부분 옆의 `..`          | `rok::core::index::Rest`         | 다른 부분이 남긴 모든 축, 통째로                  |
| `none`                      | `rok::core::index::NewAxis`      | 길이 1인 새 축                                  |
| 그 밖(마스크)                 | 쓴 그대로                         | impl이 정함 (`Array<bool, N>`: 마스크)          |

```rok
use rok::core::array::Array;

let a: Array<f64, 2> = [[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]];
a[1, :]          // [4.0, 5.0, 6.0]: 1행, 뷰
a[:, -1]         // [3.0, 6.0]: 마지막 열
a[::-1, 1:]      // 행 뒤집기, 열 1..
a[.., 0]         // [1.0, 4.0]
a[:, none]       // 모양 [2, 1, 3]
a[a .> 2.0]      // [3.0, 4.0, 5.0, 6.0]: 마스크는 복사본을 고름
a[:, 0] = 0.0;   // 열 하나를 채움
```

`..` 하나만(`a[..]`)은 여전히 전체 범위입니다(3.6.2). `AxisIndexed`가 아닌 타입은 부분을 그 자신의 타입으로 받고(정수 리터럴은 기본 타입), 무엇을 받을지는 그 타입의 `Index` impl이 정합니다.

한 축으로 인덱싱되는 타입(`List`, `Slice`, 고정 배열, `str`)에 슬라이스가 섞인 여러 부분의 인덱스를 쓰면 에러입니다(E0144). 도움말은 슬라이스와 인덱스를 다른 괄호에 쓰거나(`m[1:][0]`) 데이터를 `Array`에 두라고 안내합니다.

## 3.6.4. 빈 대괄호와 잘못된 형태

다음은 모두 잘못된 형태입니다.

```
arr[]               // ERROR: 빈 대괄호
arr[a:b:c:d]        // ERROR: 콜론 셋 이상
arr[: : :]          // ERROR: 콜론 셋
```

컴파일러는 이 형태를 어휘 단계와 구문 단계에서 거부합니다.

## 3.6.5. 인덱싱과 슬라이싱은 `trait` 의존

어떤 타입이 인덱싱이나 슬라이싱을 지원하는지는 언어가 직접 정하지 않습니다. 표준 라이브러리가 그 타입에 `Index`, `IndexSet`, `Sliceable`을 구현해야 비로소 `arr[i]`와 `arr[1:3]`이 그 타입에서 동작합니다.

언어는 대괄호의 토큰 형태와 풀어 쓰는 규칙을 정의하고, 의미는 라이브러리가 정합니다.
