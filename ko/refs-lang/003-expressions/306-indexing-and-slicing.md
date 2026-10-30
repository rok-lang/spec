# 3.6. 인덱싱과 슬라이싱

대괄호(`[`, `]`)로 둘러싼 표현식은 값의 일부에 접근하는 두 가지 형태에 쓰입니다. 하나는 특정 위치의 원소를 꺼내는 인덱싱이고, 다른 하나는 영역을 잘라 내는 슬라이싱입니다.

```
index_expr      ::= primary "[" expr ("," expr)* "]"
slice_expr      ::= primary "[" slice_arg ("," slice_arg)* "]"
slice_arg       ::= expr
                  | expr? ":" expr? (":" expr?)?
```

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

읽기와 쓰기는 서로 다른 트레잇에 매핑되므로, 읽기 전용 자료에 쓰려고 하면 컴파일 에러입니다. 다차원 인덱싱은 인덱스를 튜플로 묶어 인덱스 트레잇 호출 하나로 풉니다. `g[i, j]`는 `Index<(I, J)>`를 거치는 `g[(i, j)]`입니다. 한 타입에 `Index<K>` impl이 여럿 있으면 인자 타입으로 그중 하나를 고릅니다.

### 음수 인덱스

위치로 인덱싱하는 타입(`List`, `Slice`, 고정 배열 `[T; N]`, 그리고 `str` / `StrBuf`의 슬라이스 경계)에서는 음수인 **부호 있는 정수** 인덱스를 끝에서부터 셉니다. 실제 위치는 `len + i`입니다.

```rok
let xs = [10, 20, 30];
xs[-1];                // 30
let i = -2;
xs[i];                 // 20 - 리터럴뿐 아니라 변수도
xs[-4];                // panic: index -4 out of bounds for length 3
```

- 부호 없는 정수 인덱스는 위치 그대로이며, 끝에서 세는 추가 비용이 없습니다.
- 타입 없는 음수 리터럴 인덱스는 `isize`입니다.
- `len + i < 0`이면 panic이 일어납니다(`index -4 out of bounds for length 3`).
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
let same = arr[1..3];        // 범위로도 - arr[1:3] 과 같음
let whole = arr[..];         // arr[:] 과 같음
```

대괄호 안에 슬라이스 형태가 오면 슬라이싱으로 처리하며, 그 형태는 다음 BNF를 따릅니다.

```
slice_expr  := expr? ':' expr? (':' expr?)?
              | range_expr                      // arr[a..b], arr[..]
              | expr ',' expr ',' ...           // 콤마면 인덱싱
```

기본 형태 `[start : end]`는 시작(포함)부터 끝(미포함)까지의 영역입니다. 보폭(step)을 더한 `[start : end : step]`도 쓸 수 있습니다. 의미는 **Python 슬라이스와 같습니다**.

- 경계는 길이에 맞춰 잘라 맞춥니다(clamp). 슬라이싱은 범위를 벗어나도 panic이 일어나지 않습니다. `xs[5:100]`은 가능한 만큼 가져오고, `start >= end`(양수 step)이면 빈 결과입니다.
- 부호 있는 정수 경계가 음수이면 끝에서부터 센 뒤(`len + i`) 잘라 맞춥니다. `xs[-10:]`은 전체이고 `xs[:-10]`은 빈 결과입니다.
- step이 양수이면 생략한 `start`는 0, 생략한 `end`는 길이입니다. step을 생략하면 1입니다.
- step이 음수이면 경계를 Python처럼 읽습니다. `start`에서 시작해 `end` 바로 앞까지 거꾸로 갑니다. 생략한 `start`는 마지막 원소이고, 생략한 `end`는 첫 원소 앞입니다.
- step이 0이면 panic이 일어납니다(`slice step must not be zero`).

```rok
let xs = [1, 2, 3, 4, 5];
xs[-2:];               // [4, 5]
xs[1:-1];              // [2, 3, 4]
xs[5:100];             // [] - clamp
xs[-10:];              // [1, 2, 3, 4, 5]
xs[:-10];              // []
xs[3:1];               // [] - start >= end
xs[::-1];              // [5, 4, 3, 2, 1]
xs[-1:0:-1];           // [5, 4, 3, 2]
xs[3:1:-1];            // [4, 3] - 위치 3, 2
xs[1:4:-2];            // []
xs[::0];               // panic - step 0
```

단일 원소 인덱싱(3.6.1)은 다릅니다. 음수는 끝에서부터 세지만, 범위를 벗어나면 panic이 일어납니다.

`str` / `StrBuf`의 경계는 바이트 오프셋입니다. 먼저 잘라 맞추고, 그 위치가 UTF-8 문자 중간이면 panic이 일어납니다. `str` 슬라이스에는 step이 없습니다(`s[::2]`는 타입 에러).

### 풀어 쓰기

슬라이싱은 범위와 `Sliceable` 트레잇([stdlib 2.3.4](../../refs-standard-libraries/200-core/203-ops.md#234-인덱싱-트레잇)) 호출로 풀립니다.

```
arr[a : b]            →   arr.slice(a..b)                       // 부호 없는 경계, step 없음
arr[a : b]            →   arr.slice(slice_bound(a, n)..slice_bound(b, n))   // 부호 있는 경계
arr[a : b : k]        →   arr.slice(slice_range(a, b, k, n)).step(k)
arr[: b] / arr[a :]   →   생략한 경계는 위의 기본값 (step 이 있으면 slice_range 의 none)
arr[a..b] / arr[..]   →   arr.slice(a..b) / arr.slice(0..n)
```

`n`은 `arr.len()`이고 `arr`은 한 번만 평가합니다. 모든 `Sliceable<Range<usize>>` 구현은 범위를 `clamp_range`로 잘라 맞추므로, `arr.slice(9..2)`를 직접 호출해도 panic이 일어나지 않습니다. 여기에 쓰는 도우미 함수는 `rok::core::index`에 있습니다.

| 도우미 함수                                                    | 의미                                                               |
|----------------------------------------------------------------|--------------------------------------------------------------------|
| `clamp_range(r: Range<usize>, n: usize) -> Range<usize>`       | 끝을 `n`으로, 시작을 끝으로 잘라 맞춤                               |
| `slice_bound(i: isize, len: usize) -> usize`                   | 음수면 `len + i`, 0 아래는 0                                       |
| `slice_range(start: isize \| none, stop: isize \| none, step: isize, len: usize) -> Range<usize>` | Python `slice.indices`와 같은 정방향 범위. `.step(step)`을 붙이면 `arr[start:stop:step]`   |
| `index_from_end(i: isize, len: usize) -> usize`                | 단일 인덱스용. 음수면 `len + i`, 넘치면 panic                      |

### 슬라이스 결과의 종류

결과 타입은 `Sliceable`의 연관 타입이 정합니다. 표준 라이브러리에서는 다음과 같습니다([stdlib 2.9.7](../../refs-standard-libraries/200-core/209-collections.md#297-slicet-뷰)).

| 대상                 | 결과                                                                 |
|----------------------|----------------------------------------------------------------------|
| `List<T>`            | `Slice<T>`: 리스트 버퍼의 뷰 (원소 쓰기가 보임)                      |
| `Slice<T>`           | `Slice<T>`: 뷰의 뷰                                                  |
| `[T; N]`             | `Slice<T>`: 배열 사본의 뷰 (배열은 값)                               |
| `str`, `StrBuf`      | 새 `str`: 바이트 사본 (`str`은 C 문자열로 남아야 하므로), UTF-8 경계 검사  |

### 슬라이스 대입

슬라이스 대입(`arr[1:3] = other;`)은 지원하지 않습니다. 원소를 하나씩 대입하거나 컬렉션 메서드를 쓰세요.

## 3.6.3. 인덱싱과 슬라이싱의 토큰 분리

같은 `[ ... ]` 안이라도 콤마(`,`)가 있으면 인덱싱(다차원 또는 키 튜플)이고, 콜론(`:`)이 있으면 슬라이싱입니다. 한 괄호 안에서 두 형태를 섞을 수 없습니다.

```
arr[1, 2]                  // OK - 인덱싱 (i, j)
arr[1:3]                   // OK - 슬라이싱
arr[1:3, 4:6]              // ERROR: 섞임
```

다차원 슬라이싱은 표준 라이브러리의 다차원 슬라이스 `trait`이 별도 형태로 제공할 수 있습니다(예: 행렬 라이브러리의 `MatrixSlice`).

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
