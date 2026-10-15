# 3.6. 인덱싱과 슬라이싱

대괄호 (`[`, `]`) 로 둘러싼 표현식은 값의 부분에 접근하는 두 가지 형태에 쓰입니다. 하나는 특정 위치의 원소를 꺼내는 인덱싱이고, 다른 하나는 일정 영역을 잘라내는 슬라이싱입니다.

```
index_expr      ::= primary "[" expr ("," expr)* "]"
slice_expr      ::= primary "[" slice_arg ("," slice_arg)* "]"
slice_arg       ::= expr
                  | expr? ":" expr? (":" expr?)?
```

## 3.6.1. 인덱싱: `arr[i]`

인덱싱은 대괄호 안에 인덱스 표현식을 적어 해당 자리의 값을 얻는 방식입니다.

```rok
let v = arr[0];
let last = arr[arr.len() - 1];
let m = matrix[i, j];        // 다차원, 인덱스를 콤마로 구분
```

좌변에는 임의의 표현식이 올 수 있으며, 대괄호 안에는 한 개 이상의 인덱스 표현식을 콤마로 구분해 적습니다. 결과로는 인덱싱된 자리의 값을 돌려받게 됩니다.

### desugar

인덱싱 표현식은 다음과 같이 트레잇 호출로 desugar 됩니다.

```
arr[i]                 →   <typeof(arr) as Index<typeof(i)>>::index(arr, i)
arr[i] = v;            →   <typeof(arr) as IndexSet<typeof(i)>>::index_set(ref arr, i, v)
matrix[i, j]           →   <typeof(matrix) as Index<(typeof(i), typeof(j))>>::index(matrix, (i, j))
```

읽기와 쓰기는 서로 다른 트레잇으로 매핑되기 때문에, 읽기 전용 자료에 쓰기를 시도하면 컴파일 에러가 발생합니다. 또한 다차원 인덱싱은 인덱스들을 튜플로 묶어 단일 인덱스 트레잇 호출로 desugar 된다는 점도 알아두시면 좋겠습니다.

### 음수 인덱스

위치로 인덱싱하는 타입 (`List`, `Slice`, 고정 배열 `[T; N]`, 그리고 `str` / `StrBuf` 의 슬라이스 경계) 에서 **부호 있는 정수** 인덱스가 음수이면 끝에서부터 셉니다. 실제 위치는 `len + i` 입니다.

```rok
let xs = [10, 20, 30];
xs[-1];                // 30
let i = -2;
xs[i];                 // 20 — 리터럴뿐 아니라 변수도
xs[-4];                // panic: index -4 out of bounds for length 3
```

- 부호 없는 정수 인덱스는 그대로의 위치이며, 끝에서 세는 추가 비용이 없습니다.
- 타입 없는 음수 리터럴 인덱스는 `isize` 입니다.
- `len + i < 0` 이면 panic 입니다 (`index -4 out of bounds for length 3`).
- 키로 인덱싱하는 타입 (`Dict`) 은 영향이 없습니다. 음수 키는 그대로 키입니다.
- 변환에는 `rok::core::index::index_from_end(i: isize, len: usize) -> usize` 가 쓰입니다.

평가 순서는 대상, 인덱스, (대입이면) 값 순이며 대상은 한 번만 평가됩니다.

### 경계 검사와 실패

대괄호 인덱싱은 유효 범위 밖에서 실패할 수 있는데, 실패가 어떤 형태로 드러나는지는 `Index` 구현이 결정합니다. 보통은 panic 이나 에러 발생 형태로 처리됩니다.

만약 panic 없이 `T | none` 값을 받고 싶다면, 라이브러리에서 제공하는 `.get(i)` 같은 메서드를 사용하면 됩니다.

## 3.6.2. 슬라이싱: `arr[start:end]`

슬라이싱은 대괄호 안에 콜론을 포함한 슬라이스 형태가 들어갈 때 사용되는 형식입니다.

```rok
let part = arr[1:3];
let head = arr[:5];
let tail = arr[5:];
let all = arr[:];
let stepped = arr[::2];
let reverse = arr[::-1];
let full = arr[a:b:s];
let same = arr[1..3];        // 범위로도 — arr[1:3] 과 같음
let whole = arr[..];         // arr[:] 과 같음
```

대괄호 안에 슬라이스 형태가 들어가면 슬라이싱으로 처리되며, 그 형태는 다음 BNF 를 따릅니다.

```
slice_expr  := expr? ':' expr? (':' expr?)?
              | range_expr                      // arr[a..b], arr[..]
              | expr ',' expr ',' ...           // 콤마면 인덱싱
```

기본 형태인 `[start : end]` 는 시작 (포함) 부터 끝 (미포함) 까지의 영역을 의미합니다. 여기에 단계를 더한 `[start : end : step]` 도 쓸 수 있습니다. 의미는 **Python 의 슬라이스와 같습니다**.

- 경계는 길이에 맞춰 잘라 맞춥니다 (clamp). 슬라이싱은 범위 때문에 panic 하지 않습니다. `xs[5:100]` 은 가능한 만큼, `start >= end` (양수 step) 이면 빈 결과입니다.
- 부호 있는 정수 경계가 음수이면 끝에서부터 센 뒤 (`len + i`) 잘라 맞춥니다. `xs[-10:]` 은 전체, `xs[:-10]` 은 빈 결과입니다.
- step 이 양수이면 생략한 `start` 는 0, 생략한 `end` 는 길이입니다. step 을 생략하면 1 입니다.
- step 이 음수이면 경계를 Python 처럼 읽습니다. `start` 에서 시작해 `end` 바로 앞까지 거꾸로 걷습니다. 생략한 `start` 는 마지막 원소, 생략한 `end` 는 첫 원소 앞입니다.
- step 이 0 이면 panic 입니다 (`slice step must not be zero`).

```rok
let xs = [1, 2, 3, 4, 5];
xs[-2:];               // [4, 5]
xs[1:-1];              // [2, 3, 4]
xs[5:100];             // [] — clamp
xs[-10:];              // [1, 2, 3, 4, 5]
xs[:-10];              // []
xs[3:1];               // [] — start >= end
xs[::-1];              // [5, 4, 3, 2, 1]
xs[-1:0:-1];           // [5, 4, 3, 2]
xs[3:1:-1];            // [4, 3] — 위치 3, 2
xs[1:4:-2];            // []
xs[::0];               // panic — step 0
```

단일 원소 인덱싱 (3.6.1) 은 다릅니다. 음수는 끝에서부터 세지만, 범위를 벗어나면 panic 입니다.

`str` / `StrBuf` 의 경계는 바이트 offset 이며, 먼저 잘라 맞춘 뒤 그 위치가 UTF-8 문자 중간이면 panic 입니다. `str` 슬라이스에는 step 이 없습니다 (`s[::2]` 는 타입 에러).

### desugar

슬라이싱은 범위와 `Sliceable` 트레잇 ([stdlib 2.3.4](../../refs-standard-libraries/200-core/203-ops.md#234-인덱싱-트레잇)) 호출로 desugar 됩니다.

```
arr[a : b]            →   arr.slice(a..b)                       // 부호 없는 경계, step 없음
arr[a : b]            →   arr.slice(slice_bound(a, n)..slice_bound(b, n))   // 부호 있는 경계
arr[a : b : k]        →   arr.slice(slice_range(a, b, k, n)).step(k)
arr[: b] / arr[a :]   →   생략한 경계는 위의 기본값 (step 이 있으면 slice_range 의 none)
arr[a..b] / arr[..]   →   arr.slice(a..b) / arr.slice(0..n)
```

`n` 은 `arr.len()` 이며 `arr` 은 한 번만 평가됩니다. 모든 `Sliceable<Range<usize>>` 구현은 범위를 `clamp_range` 로 잘라 맞추므로 `arr.slice(9..2)` 를 직접 불러도 panic 하지 않습니다. 쓰이는 helper 는 `rok::core::index` 에 있습니다.

| helper                                                         | 의미                                                               |
|----------------------------------------------------------------|--------------------------------------------------------------------|
| `clamp_range(r: Range<usize>, n: usize) -> Range<usize>`       | 끝을 `n` 으로, 시작을 끝으로 잘라 맞춤                              |
| `slice_bound(i: isize, len: usize) -> usize`                   | 음수면 `len + i`, 0 아래는 0                                       |
| `slice_range(start: isize \| none, stop: isize \| none, step: isize, len: usize) -> Range<usize>` | Python `slice.indices` 와 같은 정방향 범위. `.step(step)` 을 붙이면 `arr[start:stop:step]` |
| `index_from_end(i: isize, len: usize) -> usize`                | 단일 인덱스용. 음수면 `len + i`, 넘치면 panic                      |

### 슬라이스 결과의 종류

결과 타입은 `Sliceable` 의 연관 타입이 정합니다. 표준 라이브러리에서는 다음과 같습니다 ([stdlib 2.9.7](../../refs-standard-libraries/200-core/209-collections.md#297-slicet--view)).

| 대상                 | 결과                                                                 |
|----------------------|----------------------------------------------------------------------|
| `List<T>`            | `Slice<T>` — 리스트 버퍼의 view (원소 쓰기가 보임)                   |
| `Slice<T>`           | `Slice<T>` — view 의 view                                            |
| `[T; N]`             | `Slice<T>` — 배열 사본의 view (배열은 값)                            |
| `str`, `StrBuf`      | 새 `str` — 바이트 사본 (`str` 은 C 문자열로 남아야 하므로), UTF-8 경계 검사 |

### 슬라이스 대입

슬라이스 대입 (`arr[1:3] = other;`) 은 지원하지 않습니다. 원소를 하나씩 대입하거나 컬렉션 메서드를 씁니다.

## 3.6.3. 인덱싱과 슬라이싱의 토큰 분리

같은 `[ ... ]` 안이라도, 콤마 (`,`) 가 등장하면 인덱싱 (다차원 또는 키 튜플) 으로, 콜론 (`:`) 이 등장하면 슬라이싱으로 해석됩니다. 한 괄호 안에서 두 형태를 섞는 것은 허용되지 않으니 주의해야 합니다.

```
arr[1, 2]                  // OK — 인덱싱 (i, j)
arr[1:3]                   // OK — 슬라이싱
arr[1:3, 4:6]              // ERROR: 섞임
```

다차원 슬라이싱이 필요하다면, 표준 라이브러리의 다차원 슬라이스 `trait` (예컨대 행렬 라이브러리의 `MatrixSlice` 같은 것) 가 별도 형태로 제공할 수 있습니다.

## 3.6.4. 빈 대괄호와 잘못된 형태

다음과 같은 형태들은 모두 잘못된 사용 예에 해당합니다.

```
arr[]               // ERROR: 빈 대괄호
arr[a:b:c:d]        // ERROR: 콜론 셋 이상
arr[: : :]          // ERROR: 콜론 셋
```

이러한 형태들은 어휘와 구문 단계에서 거부됩니다.

## 3.6.5. 인덱싱과 슬라이싱은 `trait` 의존

언어 자체는 어떤 타입에 인덱싱이나 슬라이싱이 가능한지를 직접 결정하지 않습니다. 표준 라이브러리가 `Index`, `IndexSet`, `Sliceable` 을 적절한 타입에 구현해 두면, 그제서야 `arr[i]` 와 `arr[1:3]` 이 그 타입에 대해 동작합니다.

요컨대 언어가 정의하는 것은 대괄호의 토큰 형태와 desugar 규칙까지이고, 그 의미는 라이브러리의 몫이라고 보면 됩니다.
