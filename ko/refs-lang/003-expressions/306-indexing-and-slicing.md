# 3.6. 인덱싱과 슬라이싱

대괄호 (`[`, `]`) 로 둘러싼 표현식은 값의 부분에 접근하는 두 형태에 쓴다.

```
index_expr      ::= primary "[" expr ("," expr)* "]"
slice_expr      ::= primary "[" slice_arg ("," slice_arg)* "]"
slice_arg       ::= expr
                  | expr? ":" expr? (":" expr?)?
```

## 3.6.1. 인덱싱: `arr[i]`

```nok
let v = arr[0];
let last = arr[arr.len() - 1];
let m = matrix[i, j];        // 다차원, 인덱스를 콤마로 구분
```

- 좌변은 임의의 표현식.
- 대괄호 안에는 한 개 이상의 인덱스 표현식을 콤마로 구분.
- 결과는 인덱싱된 자리의 값.

### desugar

```
arr[i]                 →   <typeof(arr) as Index<typeof(i)>>::index(arr, i)
arr[i] = v;            →   <typeof(arr) as IndexSet<typeof(i)>>::index_set(ref arr, i, v)
matrix[i, j]           →   <typeof(matrix) as Index<(typeof(i), typeof(j))>>::index(matrix, (i, j))
```

읽기와 쓰기는 다른 트레잇으로 매핑된다. 읽기 전용 자료에 쓰기를 시도하면 컴파일 에러가 난다.

다차원 인덱싱은 인덱스들을 튜플로 묶어 단일 인덱스 트레잇 호출로 desugar 된다.

### 음수 인덱스

음수 인덱스의 의미 (예: 뒤에서부터 세는 Python 식) 는 표준 라이브러리의 `Index` 구현에 달려 있다. 언어는 `[-1]` 을 유효한 식 자리로 받아들이고 `Index<i32>` (또는 음수가 가능한 적절한 타입) 의 호출로 desugar 한다.

음수 인덱스 지원 여부와 동작은 컬렉션마다 다를 수 있다. `List` 는 지원, `Dict` 는 키 자리이므로 의미가 다르다.

### 경계 검사와 실패

대괄호 인덱싱은 유효 범위 밖에서 실패할 수 있다. 실패의 형태는 `Index` 구현이 결정한다 (보통 panic / 에러 발생).

panic 없이 Option 값을 받고 싶으면 라이브러리의 `.get(i)` 같은 메서드를 쓴다.

## 3.6.2. 슬라이싱: `arr[start:end]`

```nok
let part = arr[1:3];
let head = arr[:5];
let tail = arr[5:];
let all = arr[:];
let stepped = arr[::2];
let reverse = arr[::-1];
let neg = arr[-3:-1];
let full = arr[a:b:s];
```

대괄호 안에 슬라이스 형태가 들어가면 슬라이싱이다. 형태는 다음 BNF 를 따른다.

```
slice_expr  := expr? ':' expr? (':' expr?)?
              | expr ',' expr ',' ...           // 콤마면 인덱싱
```

- `[start : end]`. 시작 (포함) 부터 끝 (미포함).
- `[start : end : step]`. 단계 추가. 음수 step 으로 역순.
- 각 부분은 생략 가능. 생략하면 처음 / 끝 / 1.

### desugar

```
arr[a : b]            →   <typeof(arr) as Slice<typeof(a), typeof(b)>>::slice(arr, a, b, 1)
arr[a : b : s]        →   <typeof(arr) as Slice<typeof(a), typeof(b)>>::slice(arr, a, b, s)
arr[: b]              →   동일. 생략된 자리는 stdlib 가 정의한 기본값
```

생략된 자리의 기본값 (`start` 생략 시 0, `end` 생략 시 길이, `step` 생략 시 1, 음수 step 일 때 end 가 길이-1 등) 은 표준 라이브러리의 `Slice` 구현이 결정한다.

### 슬라이스 결과의 종류

슬라이스 결과는 보통 원본의 뷰 (zero-copy view) 다. 원소 복사 없이 영역을 가리킨다. 정확한 반환 타입은 표준 라이브러리에 정의된 `Slice` 의 연관 타입에 달려 있다.

### 슬라이스 대입

슬라이스에 대입할 수 있다 (지원 여부는 트레잇 구현에 따라 다르다).

```nok
arr[1:3] = other;
```

이는 영역의 원소를 한꺼번에 갱신한다. desugar 는 별개의 _SliceSet_ 트레잇으로 처리되며, `Slice` 와 `SliceSet` 두 트레잇이 읽기·쓰기를 분리한다.

## 3.6.3. 인덱싱과 슬라이싱의 토큰 분리

같은 `[ ... ]` 안에 콤마 (`,`) 가 등장하면 인덱싱 (다차원 / 키 튜플), 콜론 (`:`) 이 등장하면 슬라이싱이다. 한 괄호 안에 두 형태가 섞이는 것은 허용되지 않는다.

```
arr[1, 2]                  // OK — 인덱싱 (i, j)
arr[1:3]                   // OK — 슬라이싱
arr[1:3, 4:6]              // ERROR: 섞임
```

다차원 슬라이싱이 필요하면 표준 라이브러리의 다차원 슬라이스 `trait` (예: 행렬 라이브러리의 `MatrixSlice`) 가 별도 형태로 제공할 수 있다.

## 3.6.4. 빈 대괄호와 잘못된 형태

```
arr[]               // ERROR: 빈 대괄호
arr[a:b:c:d]        // ERROR: 콜론 셋 이상
arr[: : :]          // ERROR: 콜론 셋
```

위 형태들은 어휘·구문 단계에서 거부된다.

## 3.6.5. 인덱싱과 슬라이싱은 `trait` 의존

언어 자체는 어떤 타입에 인덱싱 / 슬라이싱이 가능한지를 결정하지 않는다. 표준 라이브러리가 `Index`, `IndexSet`, `Slice`, `SliceSet` 를 적절한 타입에 구현해 두면, `arr[i]` 와 `arr[1:3]` 이 그 타입에 대해 동작한다.

언어가 정의하는 것은 대괄호의 토큰 형태와 desugar 규칙까지다. 의미는 라이브러리.
