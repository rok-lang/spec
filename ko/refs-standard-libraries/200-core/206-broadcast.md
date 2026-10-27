# 2.6. broadcast - element-wise 연산자

`rok::core::ops::broadcast` 모듈은 Julia 식 dot 연산자에 대한 트레잇 권위를 제공합니다. dot 연산자(`.+` / `.*` / `.==` / `f.(x)`) 는 element-wise 의미를 문법 자체에 시각적으로 분리하여 드러내므로, 사용자 정의 `*`(예: matrix multiply 등) 과 혼동될 여지를 미리 차단합니다.

이 장에서는 19 개의 broadcast 트레잇과 함수 broadcast desugar 규칙을 함께 정의합니다.

트레잇 자체는 `rok::core::ops` 에 선언되며, `rok::core::ops::broadcast` 가 이들을 한 경로로 re-export 합니다. 여기에 [2.3.10](203-ops.md#2310-unicode-수학-연산자-트레잇) 의 `⋅` 내적 트레잇 `Dot` 도 함께 re-export 됩니다. `Dot` 은 19 개 broadcast 트레잇에 속하지 않습니다. `a ⋅ b` 는 원소별로 동작하지 않고 두 피연산자를 하나의 값으로 줄이며 (`fn dot(self, rhs: Rhs) -> Output`), 표준 라이브러리는 어떤 타입에도 이를 구현하지 않습니다.

## 2.6.1. 동기 - 왜 별도 dot 연산자인가요

NumPy 나 MATLAB 에서 `*` 가 배열에서는 element-wise 로, 행렬에서는 곱셈으로 의미가 바뀌는 **silent ambiguity** 는 흔히 버그의 원인이 되어 왔습니다(PEP 465 의 `@` 도입 동기 역시 같은 맥락입니다).

이에 비해 Rok 의 정책은 다음과 같이 명확합니다.

```rok
m * n       // 사용자 정의 Mul - matrix multiplication, dot product, etc.
m .* n      // element-wise (Hadamard product) - 항상 broadcast
```

dot 접두사(`.`) 가 시각적인 신호 역할을 하여 element-wise 의도를 명시하며, 컴파일러가 이를 강제합니다. 즉 List/Vec/Matrix 등 어떤 컨테이너에서든 `.+` 는 일관되게 `DotAdd::dot_add` 로 dispatch 됩니다.

## 2.6.2. 19 트레잇 - operator 매핑

전체 트레잇은 산술(6) + 비교(6) + 논리(2) + 비트(3) + shift(2) = **19**개로 구성됩니다.

> [!NOTE]
> 이 장은 19 트레잇을 기준으로 정의하며, 산술 6종, 비교 6종, 논리 2종, 비트 3종, shift 2종을 모두 포함합니다. dot 비교(`.==`/`.!=`/`.<`/`.>`/`.<=`/`.>=`) 는 [2.5.8 broadcast 비교](205-cmp.md#258-broadcast-비교-트레잇)
> 에서 **간단 시그니처** 만 제시했고, 이 장이 **전체 권위** 를 갖습니다.

### 산술 (6)

| 트레잇      | operator | precedence | 의미                          |
|-------------|----------|------------|-------------------------------|
| `DotAdd`    | `.+`     | 7 (left)   | element-wise 덧셈             |
| `DotSub`    | `.-`     | 7 (left)   | element-wise 뺄셈             |
| `DotMul`    | `.*`     | 8 (left)   | element-wise 곱 (Hadamard)    |
| `DotDiv`    | `./`     | 8 (left)   | element-wise 나눗셈           |
| `DotRem`    | `.%`     | 8 (left)   | element-wise 나머지 (Rem)     |
| `DotModulo` | `.%%`    | 8 (left)   | element-wise floored modulo   |

### 비교 (6)

| 트레잇  | operator | precedence | 의미                                |
|---------|----------|------------|-------------------------------------|
| `DotEq` | `.==`    | 4 (none)   | element-wise == → `Container<bool>` |
| `DotNe` | `.!=`    | 4 (none)   | element-wise !=                     |
| `DotLt` | `.<`     | 4 (none)   | element-wise <                      |
| `DotGt` | `.>`     | 4 (none)   | element-wise >                      |
| `DotLe` | `.<=`    | 4 (none)   | element-wise <=                     |
| `DotGe` | `.>=`    | 4 (none)   | element-wise >=                     |

### 논리 (2)

| 트레잇          | operator | precedence | 의미                                     |
|-----------------|----------|------------|------------------------------------------|
| `DotLogicalAnd` | `.&&`    | 3 (left)   | element-wise 논리 AND (no short-circuit) |
| `DotLogicalOr`  | `.\|\|`    | 2 (left)   | element-wise 논리 OR (no short-circuit)  |

> [!IMPORTANT]
> **`.&&` / `.||` 는 short-circuit 평가를 하지 않습니다**. 일반 `&&` / `||` 와 달리, broadcast 가 fused loop 으로 lower 되는 특성상 양쪽 인자가 항상 평가됩니다. 의미적으로는 bool 컨테이너에 대한 element-wise 논리 결합과 동일합니다. 부수효과가 있는 함수 호출을 한쪽 인자에 두면 항상 실행되니 주의가 필요합니다.

### 비트 (3)

| 트레잇      | operator | precedence | 의미           |
|-------------|----------|------------|----------------|
| `DotBitAnd` | `.&`     | 6 (left)   | element-wise & |
| `DotBitOr`  | `.\|`     | 5 (left)   | element-wise \| |
| `DotBitXor` | `.^`     | 6 (left)   | element-wise ^ |

### shift (2)

| 트레잇   | operator | precedence | 의미            |
|----------|----------|------------|-----------------|
| `DotShl` | `.<<`    | 9 (left)   | element-wise << |
| `DotShr` | `.>>`    | 9 (left)   | element-wise >> |

precedence 와 associativity 는 일반 트레잇([2.3 ops](203-ops.md), [2.5 cmp](205-cmp.md)) 과 동형으로 맞추어 두었습니다.

## 2.6.3. 트레잇 시그니처 - 통일 형태

모든 dot 트레잇은 다음과 같이 통일된 형태를 따릅니다.

```rok
trait DotAdd<Rhs = Self> {
    type Output;
    #[infix(".+")]
    fn dot_add(self, rhs: Rhs) -> Output;
}

trait DotMul<Rhs = Self> {
    type Output;
    #[infix(".*")]
    fn dot_mul(self, rhs: Rhs) -> Output;
}

trait DotModulo<Rhs = Self> {
    type Output;
    #[infix(".%%")]
    fn dot_modulo(self, rhs: Rhs) -> Output;
}

trait DotEq<Rhs = Self> {
    type Output;
    #[infix(".==")]
    fn dot_eq(self, rhs: Rhs) -> Output;
}

trait DotLogicalAnd<Rhs = Self> {
    type Output;
    #[infix(".&&")]
    fn dot_logical_and(self, rhs: Rhs) -> Output;
}
```

산술 dot 트레잇은 `Add`/`Sub`/`Mul`/`Div`/`Rem`/`Modulo` 와 완전히 대칭을 이루며, `Output` associated type 에 infix 어트리뷰트를 부착하는 형태입니다. 여섯 비교 dot 트레잇 (`DotEq`, `DotNe`, `DotLt`, `DotGt`, `DotLe`, `DotGe`) 도 같은 형태이며, 각자 `Output` 을 가지고 그것은 보통 `Container<bool>` 입니다. 논리 dot 트레잇 (`DotLogicalAnd` / `DotLogicalOr`) 은 `Container<bool>` 양쪽을 받아 `Container<bool>` 을 돌려주는 형태가 일반적이며, short-circuit 이 없으므로 양쪽 컨테이너가 모두 끝까지 평가됩니다.

### 함수 broadcast - 트레잇 X

함수 broadcast 는 다음과 같은 형태로 사용됩니다.

```rok
f.(x)              // 함수 broadcast
f.(a, b)           // 여러 인자
```

함수 broadcast 는 별도의 트레잇을 두지 않고 다음과 같이 desugar 됩니다.

```rok
f.(x)        →  x.into_iter().map(f).collect()
g.(a, b)     →  a.into_iter().zip(b.into_iter()).map(|(x, y)| g(x, y)).collect()
```

[2.7 iter](207-iter.md) 의 `map`/`zip` 으로 직접 dispatch 되며, broadcast 모듈이 별도의 트레잇을 두지 않습니다. 이는 함수 자체가 dispatch 대상이 아니라 callable 이라는 차이에서 비롯됩니다.

## 2.6.4. 의미 정의 - element-wise

### 산술 dot - 같은 모양 두 컨테이너

같은 모양을 가진 두 컨테이너에 대한 산술 dot 연산은 다음과 같이 동작합니다.

```rok
let a: List<i32> = [1, 2, 3];
let b: List<i32> = [10, 20, 30];

let c = a .+ b;          //→ [11, 22, 33]
let d = a .* b;          //→ [10, 40, 90]
let e = b .- a;          //→ [9, 18, 27]
let f = b ./ a;          //→ [10, 10, 10]
```

길이가 일치하지 않으면 런타임 panic 이 발생합니다. 다만 컴파일 타임에 size 를 알 수 있다면 컴파일 에러로 잡힙니다.

```rok
let x: [i32; 3] = [1, 2, 3];
let y: [i32; 4] = [1, 2, 3, 4];
let z = x .+ y;          // ERROR (컴파일 - 고정 size 다름)

let p: List<i32> = [1, 2, 3];
let q: List<i32> = [1, 2];
let r = p .+ q;          // panic at runtime - length mismatch
```

### scalar broadcast

스칼라와 컨테이너를 함께 사용하면 다음과 같이 동작합니다.

```rok
let a: List<i32> = [1, 2, 3];
let s = a .+ 10;         //→ [11, 12, 13]  (scalar broadcast)
let m = 2 .* a;          //→ [2, 4, 6]
let t = a ./ 2;          //→ [0, 1, 1]    (i32 truncate)
```

이 경우 별도의 impl (`DotAdd<T> for List<T>`, 그리고 scalar 가 왼쪽인 형태의 `DotAdd<List<T>> for T`) 이 scalar 쪽을 처리합니다. 고정 배열 `[T; N]` 에는 scalar 가 왼쪽인 impl 이 없습니다.

### 비교 dot - `Container<bool>` 반환

비교 dot 연산자는 결과로 `Container<bool>` 을 반환합니다.

```rok
let a: List<i32> = [1, 5, 3];
let b: List<i32> = [2, 5, 1];

a .== b;                 //→ [false, true, false]
a .< b;                  //→ [true, false, false]
a .>= b;                 //→ [false, true, true]

a .< 4;                  //→ [true, false, true]   (scalar broadcast)
```

### 나머지 dot - `.%` 와 `.%%`

나머지 계열 dot 연산자는 일반 `%` / `%%` 와 동일한 의미를 element-wise 로 적용합니다.

```rok
let a: List<i32> = [-7, 5, -3, 8];
let b: List<i32> = [3, 3, 3, 3];

a .% b;                  //→ [-1, 2, 0, 2]   (Rem - 부호는 좌측 따름)
a .%% b;                 //→ [2, 2, 0, 2]    (Modulo - 항상 [0, 3))
```

### 논리 dot - `.&&` 와 `.||`

논리 dot 연산자는 두 `Container<bool>` 의 element-wise 논리 결합을 산출합니다. 점 없는 형태와 달리 short-circuit 이 없으므로, 양쪽 컨테이너가 항상 끝까지 평가됩니다.

```rok
let p: List<bool> = [true, false, true, false];
let q: List<bool> = [true, true, false, false];

p .&& q;                 //→ [true, false, false, false]
p .|| q;                 //→ [true, true, true, false]

// 비교 결과를 결합하는 패턴 - 가장 흔한 사용
let xs: List<i32> = [1, 5, 3, 8];
(xs .>= 2) .&& (xs .< 6);  //→ [false, true, true, false]
```

### 비트 / shift dot

비트 및 shift 계열 dot 연산자도 같은 방식으로 동작합니다.

```rok
let xs: List<u8> = [0b1100, 0b1010, 0b0011];
let ys: List<u8> = [0b1010, 0b1010, 0b1111];

xs .& ys;                //→ [0b1000, 0b1010, 0b0011]
xs .| ys;                //→ [0b1110, 0b1010, 0b1111]
xs .^ ys;                //→ [0b0110, 0b0000, 0b1100]

let ns: List<u32> = [1, 2, 4];
ns .<< 2;                //→ [4, 8, 16]
```

## 2.6.5. 함수 broadcast - `f.(x)`

함수 broadcast 는 일반 함수 호출 자리에 dot 을 붙여 사용할 수 있습니다.

```rok
let xs: List<f64> = [0.0, 0.5, 1.0];

let s = sin.(xs);                    //→ [0.0, 0.479..., 0.841...]
let r = f64::sqrt.(xs);              //→ [0.0, 0.707..., 1.0]
let g = (|x: f64| x * 2.0).(xs);     //→ [0.0, 1.0, 2.0]
```

여러 인자를 동시에 broadcast 하는 것도 가능합니다.

```rok
let a: List<f64> = [1.0, 2.0, 3.0];
let b: List<f64> = [10.0, 20.0, 30.0];

let r = f64::pow.(a, b);             //→ [1.0, 4_194_304.0, ...]
let m = max.(a, b);                  //→ [10.0, 20.0, 30.0]
```

`List`, `Slice`, 배열인 인자는 함께 순회하고, 그 밖의 인자는 매 호출에 그대로 넘깁니다 (`pow.(xs, 2.0)`). 순회하는 인자는 길이가 같아야 하며 런타임에 검사합니다. 인자 개수에는 **제한이 없어** 임의의 N-ary 함수까지 적용할 수 있습니다.

## 2.6.6. loop fusion

연속해서 등장하는 dot 연산은 컴파일러가 단일 loop 로 합쳐 줍니다.

```rok
let result = sin.(a) .+ cos.(b) .* 2.0;
```

일반적인 lowering 이라면 3 개의 임시 List 할당과 3회의 loop 가 필요합니다. Rok 은 중간 컬렉션 없는 단일 loop 를 보장하며, 그 방법 (예: `linalg.generic` op) 은 컴파일러가 고릅니다.

```rok
// 개념적으로는 다음과 같은 단일 elementwise 호출과 같습니다.
let result = elementwise(a, b, |x, y| sin(x) + cos(y) * 2.0);
```

## 2.6.7. 트레잇 구현 매트릭스

각 컨테이너 타입에 대한 dot 트레잇 impl 은 다음과 같이 매핑됩니다.

| 컨테이너         | DotAdd   | DotMul   | DotEq       | 비고                                    |
|------------------|----------|----------|-------------|-----------------------------------------|
| `List<T>`        | T:Add    | T:Mul    | T:PartialEq | element-wise, 길이 일치                 |
| `[T; N]` (고정)  | T:Add    | T:Mul    | T:PartialEq | 고정 N - 컴파일 타임                    |
| `Slice<T>`       | T:Add    | T:Mul    | T:PartialEq | view - element-wise                     |
| `i32`/`f64`/etc. | **없음** | **없음** | **없음**    | 스칼라끼리는 일반 `+`/`*`/`==` 사용     |
| `T \| none`       | (연산자) | (연산자) | (연산자)    | option propagation (`none .+ x = none`) |

`T | none` 의 dot 의미는 option propagation 입니다. 즉 어느 한쪽이 `none` 이면 결과도 `none` 으로 전파됩니다. 이것은 라이브러리 impl 없이 연산자 수준에서 처리합니다.

`Matrix<T>` / `Tensor<T>` 같은 수치 컨테이너는 외부 라이브러리에서 정의되며, 그곳에서 동일한 dot 트레잇을 직접 구현하여 NumPy 식 broadcasting 규칙을 따르도록 되어 있습니다.

## 2.6.8. NumPy 식 broadcasting 규칙

차원이 다른 배열 간의 broadcast 는 NumPy 방식을 따릅니다. 1차원과 0차원(스칼라) 이 만나면 scalar 가 모든 원소에 적용되고, 같은 차원에 같은 shape 이라면 pointwise 로 동작하며, 다른 shape 일 경우에는 trailing dim 부터 일치시키되 1 이면 stretch 됩니다.

본 절은 dot 트레잇의 일반 의미만 정의하며, 다차원 broadcasting 규칙의 전체 권위는 외부 텐서 라이브러리에 위임됩니다. Rok 표준 라이브러리는 1 차원 컨테이너 (`List<T>`, `[T; N]`, `Slice<T>`) 의 element-wise 의미만 책임집니다.

## 2.6.9. 다른 언어와의 비교

다른 주요 언어와 비교하면 다음과 같습니다.

| 측면             | NumPy                | Julia                      | MATLAB           | Rok                           |
|------------------|----------------------|----------------------------|------------------|-------------------------------|
| element-wise     | `*` (silent)         | `.*` (명시)                | `.*` (명시)      | `.*` (명시)                   |
| matrix mul       | `@` (PEP 465)        | `*`                        | `*`              | `*` (사용자 정의 Mul)         |
| element-wise add | `+` (silent)         | `.+`                       | `.+`             | `.+`                          |
| 함수 broadcast   | `np.vectorize(f)(x)` | `f.(x)`                    | `arrayfun(f, x)` | `f.(x)`                       |
| loop fusion      | **없음**             | `@.` 매크로 (자동)         | **없음**         | dot chain 자동 (MLIR)         |
| comparison       | `==` returns array   | `.==`                      | `==` (혼란)      | `.==`                         |
| dot 별도 트레잇  | **없음**             | **없음** (`*` overload 만) | **없음**         | **DotAdd/DotMul/...** (19 개) |

Rok 이 다른 언어와 가장 크게 다른 지점은 두 가지로 요약할 수 있습니다. 첫째, dot 연산자가 모두 별도의 트레잇을 갖습니다. Julia 가 `+` 를 overload 한 위에 dot syntax 만 얹은 형태인 데 비해, Rok 에서는 `Add` 와 `DotAdd` 가 서로 다른 트레잇이며 `Matrix * Matrix` 와 `Matrix .* Matrix` 가 완전히 다른 dispatch 를 따릅니다. 둘째, 컴파일러가 dot 사용을 강제합니다. 일반 `*` 로는 broadcast 가 **불가능** 하므로, 사용자 코드에 의존하지 않는 결정론적 분리가 보장됩니다.
