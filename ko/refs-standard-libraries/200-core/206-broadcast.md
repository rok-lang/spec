# 2.6. broadcast: 원소별 연산자

`rok::core::ops`의 브로드캐스트 트레잇이 Julia식 점 연산자의 트레잇을 정하는 권위입니다. 점 연산자(`.+` / `.*` / `.==` / `f.(x)`)는 원소별 의미를 문법에서 눈에 띄게 구분합니다. 그래서 사용자 정의 `*`(예: 행렬 곱)와 헷갈릴 일이 없습니다.

이 장은 브로드캐스트 트레잇 20개와 함수 브로드캐스트가 풀리는 규칙을 함께 정의합니다.

트레잇은 `rok::core::ops`에 선언되고, 그 경로 하나만 있습니다([0.1.6](../000-meta/001-tier-model.md#016-항목마다-경로-하나)). `broadcast` 하위 모듈은 없습니다. [2.3.10](203-ops.md#2310-unicode-수학-연산자-트레잇)의 `⋅` 내적 트레잇 `Dot`도 같은 곳에 있습니다. `Dot`은 브로드캐스트 트레잇 20개에 속하지 않습니다. `a ⋅ b`는 원소별로 동작하지 않고 두 피연산자를 값 하나로 줄입니다(`fn dot(self, rhs: Rhs) -> Output`). 표준 라이브러리는 어떤 타입에도 이 트레잇을 구현하지 않습니다.

## 2.6.1. 동기: 별도 dot 연산자를 두는 이유

NumPy와 MATLAB에서 `*`는 배열에서는 원소별 곱, 행렬에서는 행렬 곱을 뜻합니다. 이 **드러나지 않는 모호성(silent ambiguity)**은 오래전부터 버그의 원인이었습니다. PEP 465가 `@`를 도입한 동기도 같습니다.

Rok의 정책은 다음과 같습니다.

```rok
m * n       // 사용자 정의 Mul: matrix multiplication, dot product, etc.
m .* n      // element-wise (Hadamard product), 항상 broadcast
```

점 접두사(`.`)가 원소별 의도를 눈에 보이게 드러내고, 컴파일러가 이 구분을 강제합니다. 그래서 List/Vec/Matrix 등 어떤 컨테이너에서든 `.+`는 늘 `DotAdd::dot_add`로 디스패치됩니다.

## 2.6.2. 트레잇 20개: 연산자 매핑

트레잇은 모두 산술(7) + 비교(6) + 논리(2) + 비트(3) + 시프트(2) = 20개입니다.

> [!NOTE]
> 이 장은 트레잇 20개를 모두 정의합니다. 산술 7종, 비교 6종, 논리 2종, 비트 3종, 시프트 2종입니다. 점 비교(`.==`/`.!=`/`.<`/`.>`/`.<=`/`.>=`)는 [2.5.8 broadcast 비교](205-cmp.md#258-broadcast-비교-트레잇)
> 에 **간단한 시그니처**만 있습니다. **전체 권위**는 이 장에 있습니다.

### 산술(7)

| 트레잇      | 연산자   | 우선순위   | 의미                          |
|-------------|----------|------------|-------------------------------|
| `DotAdd`    | `.+`     | 9 (좌)     | 원소별 덧셈             |
| `DotSub`    | `.-`     | 9 (좌)     | 원소별 뺄셈             |
| `DotMul`    | `.*`     | 10 (좌)    | 원소별 곱 (Hadamard)    |
| `DotDiv`    | `./`     | 10 (좌)    | 원소별 나눗셈           |
| `DotRem`    | `.%`     | 10 (좌)    | 원소별 나머지 (Rem)     |
| `DotModulo` | `.%%`    | 10 (좌)    | 원소별 floored modulo   |
| `DotPow`    | `.**`    | 13 (우)    | 원소별 거듭제곱 (`**`, 2.3.1) |

### 비교(6)

| 트레잇  | 연산자   | 우선순위   | 의미                                |
|---------|----------|------------|-------------------------------------|
| `DotEq` | `.==`    | 4 (비결합) | 원소별 == → `Container<bool>` |
| `DotNe` | `.!=`    | 4 (비결합) | 원소별 !=                     |
| `DotLt` | `.<`     | 4 (비결합) | 원소별 <                      |
| `DotGt` | `.>`     | 4 (비결합) | 원소별 >                      |
| `DotLe` | `.<=`    | 4 (비결합) | 원소별 <=                     |
| `DotGe` | `.>=`    | 4 (비결합) | 원소별 >=                     |

### 논리(2)

| 트레잇          | 연산자   | 우선순위   | 의미                                     |
|-----------------|----------|------------|------------------------------------------|
| `DotLogicalAnd` | `.&&`    | 3 (좌)     | 원소별 논리 AND (단락 평가 없음) |
| `DotLogicalOr`  | `.\|\|`    | 2 (좌)     | 원소별 논리 OR (단락 평가 없음)  |

> [!IMPORTANT]
> **`.&&` / `.||`는 단락 평가(short-circuit)를 하지 않습니다**. 일반 `&&` / `||`와 달리 브로드캐스트는 합친 루프 하나로 낮추므로 양쪽 피연산자를 늘 평가합니다. 의미는 bool 컨테이너를 원소별로 논리 결합한 것과 같습니다. 부수 효과가 있는 함수 호출을 어느 쪽 피연산자에 두든 그 호출은 늘 실행되니 주의하세요.

### 비트(3)

| 트레잇      | 연산자   | 우선순위   | 의미           |
|-------------|----------|------------|----------------|
| `DotBitAnd` | `.&`     | 7 (좌)     | 원소별 & |
| `DotBitOr`  | `.\|`     | 5 (좌)     | 원소별 \| |
| `DotBitXor` | `.^`     | 6 (좌)     | 원소별 ^ |

### 시프트(2)

| 트레잇   | 연산자   | 우선순위   | 의미            |
|----------|----------|------------|-----------------|
| `DotShl` | `.<<`    | 8 (좌)     | 원소별 << |
| `DotShr` | `.>>`    | 8 (좌)     | 원소별 >> |

우선순위와 결합성은 일반 트레잇([2.3 ops](203-ops.md), [2.5 cmp](205-cmp.md))과 같습니다.

## 2.6.3. 트레잇 시그니처: 통일 형태

모든 점 트레잇은 같은 형태를 따릅니다.

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

산술 점 트레잇은 `Add`/`Sub`/`Mul`/`Div`/`Rem`/`Modulo`와 완전히 대칭입니다. 연관 타입 `Output`에 중위 속성을 붙인 형태입니다. 비교 점 트레잇 여섯 개(`DotEq`, `DotNe`, `DotLt`, `DotGt`, `DotLe`, `DotGe`)도 같은 형태입니다. 트레잇마다 `Output`이 있고, 그 타입은 보통 `Container<bool>`입니다. 논리 점 트레잇(`DotLogicalAnd` / `DotLogicalOr`)은 보통 양쪽에 `Container<bool>`을 받아 `Container<bool>`을 돌려줍니다. 단락 평가가 없으므로 양쪽 컨테이너를 모두 끝까지 평가합니다.

### 함수 broadcast: 트레잇 없음

함수 브로드캐스트는 다음처럼 씁니다.

```rok
f.(x)              // 함수 broadcast
f.(a, b)           // 여러 인자
```

함수 브로드캐스트에는 별도의 트레잇이 없고, 다음과 같이 풀립니다.

```rok
f.(x)        →  x.into_iter().map(f).collect()
g.(a, b)     →  a.into_iter().zip(b.into_iter()).map(|(x, y)| g(x, y)).collect()
```

이 식은 [2.7 iter](207-iter.md)의 `map`/`zip`으로 바로 디스패치됩니다. broadcast 모듈은 별도의 트레잇을 두지 않습니다. 함수 자체는 디스패치 대상이 아니라 호출할 수 있는 값(callable)이기 때문입니다.

## 2.6.4. 의미 정의: 원소별

### 산술 dot: 같은 모양 두 컨테이너

모양이 같은 두 컨테이너의 산술 점 연산은 다음과 같이 동작합니다.

```rok
let a: List<i32> = [1, 2, 3];
let b: List<i32> = [10, 20, 30];

let c = a .+ b;          //→ [11, 22, 33]
let d = a .* b;          //→ [10, 40, 90]
let e = b .- a;          //→ [9, 18, 27]
let f = b ./ a;          //→ [10, 10, 10]
```

길이가 다르면 메서드 이름을 담아 실행 중에 panic이 일어납니다(`List::dot_add: lengths 3 and 2 differ; make both sides the same length`). 한 루프로 합쳐진 연산자 식(2.6.6)은 연산자가 여럿일 수 있어 `broadcast:`로 시작합니다. 다만 컴파일 시간에 크기를 알 수 있으면 컴파일 에러로 잡힙니다.

```rok
let x: [i32; 3] = [1, 2, 3];
let y: [i32; 4] = [1, 2, 3, 4];
let z = x .+ y;          // ERROR (컴파일: 고정 size 다름)

let p: List<i32> = [1, 2, 3];
let q: List<i32> = [1, 2];
let r = p .+ q;          // panic at runtime: length mismatch
```

### 스칼라 broadcast

스칼라와 컨테이너를 함께 쓰면 다음과 같이 동작합니다.

```rok
let a: List<i32> = [1, 2, 3];
let s = a .+ 10;         //→ [11, 12, 13]  (scalar broadcast)
let m = 2 .* a;          //→ [2, 4, 6]
let t = a ./ 2;          //→ [0, 1, 1]    (i32 truncate)
```

이 경우 별도의 구현(`DotAdd<T> for List<T>`, 그리고 스칼라가 왼쪽인 형태의 `DotAdd<List<T>> for T`)이 스칼라 쪽을 처리합니다. `Slice<T>`에도 같은 한 쌍이 있습니다. 고정 배열 `[T; N]`에는 스칼라가 왼쪽인 구현이 없습니다. 라이브러리 컨테이너도 자기 타입에 대해 같은 방식으로 스칼라가 왼쪽인 형태를 얻습니다. `Matrix`를 정의한 패키지에서는 로컬 타입 `Matrix`가 어떤 타입 매개변수보다 앞에 오므로 orphan 규칙이 `impl DotMul<Matrix> for f64`를 허용하고([2.9.5](../../refs-lang/002-types/209-impl.md#295-orphan-규칙)), `2.0 .* m`과 `m .* 2.0`이 모두 됩니다.

### 비교 dot: `Container<bool>` 반환

비교 점 연산자는 결과로 `Container<bool>`을 반환합니다.

```rok
let a: List<i32> = [1, 5, 3];
let b: List<i32> = [2, 5, 1];

a .== b;                 //→ [false, true, false]
a .< b;                  //→ [true, false, false]
a .>= b;                 //→ [false, true, true]

a .< 4;                  //→ [true, false, true]   (scalar broadcast)
```

### 나머지 dot: `.%`와 `.%%`

나머지 계열 점 연산자는 일반 `%` / `%%`와 같은 의미를 원소별로 적용합니다.

```rok
let a: List<i32> = [-7, 5, -3, 8];
let b: List<i32> = [3, 3, 3, 3];

a .% b;                  //→ [-1, 2, 0, 2]   (Rem: 부호는 좌측 따름)
a .%% b;                 //→ [2, 2, 0, 2]    (Modulo: 항상 [0, 3))
```

### 논리 dot: `.&&`와 `.||`

논리 점 연산자는 두 `Container<bool>`을 원소별로 논리 결합합니다. 점 없는 형태와 달리 단락 평가가 없으므로 양쪽 컨테이너를 늘 끝까지 평가합니다.

```rok
let p: List<bool> = [true, false, true, false];
let q: List<bool> = [true, true, false, false];

p .&& q;                 //→ [true, false, false, false]
p .|| q;                 //→ [true, true, true, false]

// 비교 결과를 결합하는 패턴: 가장 흔한 사용
let xs: List<i32> = [1, 5, 3, 8];
(xs .>= 2) .&& (xs .< 6);  //→ [false, true, true, false]
```

### 비트 / 시프트 dot

비트와 시프트 계열 점 연산자도 같은 방식으로 동작합니다.

```rok
let xs: List<u8> = [0b1100, 0b1010, 0b0011];
let ys: List<u8> = [0b1010, 0b1010, 0b1111];

xs .& ys;                //→ [0b1000, 0b1010, 0b0011]
xs .| ys;                //→ [0b1110, 0b1010, 0b1111]
xs .^ ys;                //→ [0b0110, 0b0000, 0b1100]

let ns: List<u32> = [1, 2, 4];
ns .<< 2;                //→ [4, 8, 16]
```

## 2.6.5. 함수 broadcast: `f.(x)`

함수 브로드캐스트는 일반 함수 호출에 점을 붙여 씁니다.

```rok
let xs: List<f64> = [0.0, 0.5, 1.0];

let s = f64::sin.(xs);               //→ [0.0, 0.479..., 0.841...]
let r = f64::sqrt.(xs);              //→ [0.0, 0.707..., 1.0]
let g = (|x: f64| x * 2.0).(xs);     //→ [0.0, 1.0, 2.0]
```

인자 여러 개를 한꺼번에 브로드캐스트해도 됩니다.

```rok
let a: List<f64> = [1.0, 2.0, 3.0];
let b: List<f64> = [10.0, 20.0, 30.0];

let r = f64::pow.(a, b);             //→ [1.0, 4_194_304.0, ...]
let m = f64::max.(a, b);             //→ [10.0, 20.0, 30.0]
```

`List`, `Slice`, 배열인 인자는 함께 순회하고, 그 밖의 인자는 호출마다 그대로 넘깁니다(`f64::pow.(xs, 2.0)`). 순회하는 인자는 길이가 같아야 하고, 실행 중에 검사합니다. 인자 개수에는 **제한이 없으므로** 인자가 몇 개인 함수에든 적용할 수 있습니다.

## 2.6.6. 루프 융합

컴파일러는 이어진 점 연산을 루프 하나로 합칩니다.

```rok
let result = f64::sin.(a) .+ f64::cos.(b) .* 2.0;
```

단순하게 낮추면 임시 List 할당 3번과 루프 3개가 필요합니다. Rok은 `List`, `Slice`, 배열과 모든 `Broadcastable` 타입에 대해 중간 컬렉션 없는 루프 하나를 보장합니다(정확한 조건은 [3.3.4](../../refs-lang/003-expressions/303-broadcast-operators.md#334-자동-루프-융합)). 제자리 형태(`y .= ..`, `y .+= ..`, [3.3.7](../../refs-lang/003-expressions/303-broadcast-operators.md#337-제자리-브로드캐스트))는 할당 없이 대상에 씁니다.

```rok
// 개념적으로는 다음과 같은 단일 elementwise 호출과 같습니다.
let result = elementwise(a, b, |x, y| x.sin() + y.cos() * 2.0);
```

### `Broadcastable`과 `Materialize`

융합 프로토콜은 `rok::core::ops`의 트레잇 두 개입니다.

```rok
pub trait Broadcastable<const N: usize> {
    type Elem;
    /// 원소를 배열로: 저장소를 공유하는 뷰.
    fn broadcast_view(self) -> Array<Elem, N>;
}

pub trait Materialize<U, const N: usize> {
    type Output;
    /// 새 행 우선 배열 `out`을 감싼 결과.
    fn materialize(self, out: Array<U, N>) -> Output;
}
```

`Array`([2.9.14](209-collections.md#2914-arrayt-const-n-usize-n차원-배열))는 버퍼, 오프셋, 모양, 보폭이므로 보폭이 있는 어떤 저장소든 나타냅니다. 직접 만든 타입(스톤의 `Tensor`, 데이터프레임 열)은 가진 `Array`를 돌려주어 `Broadcastable`을 구현하고, 앞선 피연산자일 때 결과가 되려면 `Materialize`를 구현합니다. `Array`는 둘 다 구현합니다. 뷰는 자기 자신이고, `Array`가 앞서는 식은 `Array<U, N>`을 줍니다.

```rok
struct Signal { data: Array<f64, 1>; }
impl Broadcastable<1> for Signal {
    type Elem = f64;
    fn broadcast_view(self) -> Array<f64, 1> { self.data }
}
impl Materialize<f64, 1> for Signal {
    type Output = Signal;
    fn materialize(self, out: Array<f64, 1>) -> Signal { Signal { data: out } }
}
let t = s .* 2.0 .+ k;      // 루프 하나; Signal
```

융합된 식은 피연산자마다 뷰를 얻고(`broadcast_view`), 뷰를 한 모양으로 늘리고(2.6.8), 앞선 피연산자(랭크가 가장 높은 것 중 첫째)를 위해 결과를 할당해 그 피연산자의 `materialize`에 넘깁니다. 루프는 버퍼를 직접 돕니다. 결과와 모든 뷰가 연속이면 위치 한 줄을, 아니면 뷰마다 보폭을 써서 마지막 축의 행을 돕니다. `Materialize`가 없는 앞선 피연산자는 그 트레잇을 밝히는 컴파일 에러입니다.

## 2.6.7. 트레잇 구현 매트릭스

컨테이너 타입별 점 트레잇 구현은 다음과 같습니다.

| 컨테이너         | DotAdd   | DotMul   | DotEq       | 비고                                    |
|------------------|----------|----------|-------------|-----------------------------------------|
| `List<T>`        | T:Add    | T:Mul    | T:PartialEq | 원소별, 길이가 같아야 함                |
| `[T; N]` (고정)  | T:Add    | T:Mul    | T:PartialEq | 고정 N, 컴파일 시간                     |
| `Slice<T>`       | T:Add    | T:Mul    | T:PartialEq | 뷰, 원소별                              |
| `i32`/`f64` 등   | 스칼라 왼쪽 | 스칼라 왼쪽 | 스칼라 왼쪽 | 스칼라끼리의 impl 은 없음(일반 `+`/`*`/`==`); 스칼라 왼쪽 형태(2.6.4)를 위해 `T`가 `DotAdd<List<T>>` / `DotAdd<Slice<T>>`를 구현 |
| `T \| none`       | (연산자) | (연산자) | (연산자)    | 옵션 전파 (`none .+ x = none`)          |

`T | none`에서 점 연산의 의미는 옵션 전파(option propagation)입니다. 어느 한쪽이 `none`이면 결과도 `none`입니다. 이 처리는 라이브러리 구현 없이 연산자 수준에서 합니다. 점 식 트리의 피연산자를 모두 소스 순서대로 한 번씩 평가한 다음, `T | none` 피연산자를 검사합니다. 처음 만난 `none`이 있으면 트리를 실행하지 않고 값이 `none`이 됩니다. 아니면 `T` 값으로 트리를 실행합니다(2.6.6대로 융합하거나 점 트레잇을 호출). 그래서 점 트레잇은 `none`을 받지 않습니다. 값의 타입은 트리의 타입에 `| none`을 더한 것입니다.

core의 n차원 `Array<T, N>`([2.9.14](209-collections.md#2914-arrayt-const-n-usize-n차원-배열))은 모든 점 트레잇을 구현합니다. 같은 랭크의 배열과 배열, 배열과 스칼라, 스칼라와 배열을 2.6.8 규칙으로 받습니다. 라이브러리의 수치 컨테이너(`Tensor<T>`, 데이터프레임 열)도 같은 트레잇을 직접 구현하며 왼쪽 스칼라도 받을 수 있습니다. orphan 규칙이 `Tensor`를 정의한 패키지의 `impl DotMul<Tensor> for f64`를 허용하기 때문입니다([refs-lang 2.9.5](../../refs-lang/002-types/209-impl.md#295-orphan-규칙)). 이런 타입의 점 식은 그 타입이 `Broadcastable`이 아니면 연산자마다 트레잇을 호출합니다(2.6.6). `Array`는 `Broadcastable`이므로 점 식이 융합되고, 트레잇 구현은 바운드와 이름으로 부르는 호출에 쓰입니다.

## 2.6.8. NumPy 식 broadcasting 규칙

규칙은 하나이며, 모든 배열 타입이 같도록 core에 있습니다. `rok::core::array::broadcast_shape(a, b)`는 랭크가 같은 두 모양을 받아 축마다 같은 길이는 그대로 두고 길이 1은 다른 쪽 길이로 늘립니다. 그 밖의 쌍은 두 모양과 축을 밝혀 panic합니다(`broadcast: shapes [2, 3] and [2, 2] differ on axis 1; each axis must match or have length 1`). 스칼라는 모든 원소에 적용합니다. 점 식에서 랭크가 다른 피연산자는 마지막 축끼리 맞춥니다. 짧은 쪽은 앞에 길이 1인 축이 있는 것으로 봅니다(`[3]` 벡터와 `[2, 3]` 행렬은 `[2, 3]`). 결과의 랭크는 가장 높은 랭크이며 타입에 드러납니다. 점 트레잇 구현은 한 랭크만 받으므로 제네릭 코드와 이름으로 부르는 호출은 랭크를 명시적으로 맞춥니다(`v[none, :]`은 `[3]` 벡터를 `[1, 3]` 행으로 바꿉니다). `Array::broadcast_to(shape)`는 늘린 뷰 자체를 줍니다(늘린 축은 보폭 0, 복사 없음).

1차원 컨테이너(`List<T>`, `[T; N]`, `Slice<T>`)는 축이 하나이고 길이 1 늘리기가 없습니다. 길이가 같아야 합니다(2.6.4).

## 2.6.9. 다른 언어와 비교

다른 주요 언어와 비교하면 다음과 같습니다.

| 측면             | NumPy                | Julia                      | MATLAB           | Rok                           |
|------------------|----------------------|----------------------------|------------------|-------------------------------|
| 원소별 연산      | `*` (암묵)           | `.*` (명시)                | `.*` (명시)      | `.*` (명시)                   |
| 행렬 곱          | `@` (PEP 465)        | `*`                        | `*`              | `*` (사용자 정의 Mul)         |
| 원소별 덧셈      | `+` (암묵)           | `.+`                       | `.+`             | `.+`                          |
| 함수 브로드캐스트 | `np.vectorize(f)(x)` | `f.(x)`                    | `arrayfun(f, x)` | `f.(x)`                       |
| 루프 합치기      | **없음**             | `@.` 매크로 (자동)         | **없음**         | 점 연산 연쇄 자동 (MLIR)      |
| 비교             | `==`가 배열 반환     | `.==`                      | `==` (혼란)      | `.==`                         |
| 점 전용 트레잇   | **없음**             | **없음** (`*` overload만) | **없음**         | **DotAdd/DotMul/...** (20개) |

Rok이 다른 언어와 가장 크게 다른 점은 두 가지입니다. 첫째, 점 연산자마다 별도의 트레잇이 있습니다. Julia는 오버로드한 `+` 위에 점 문법만 얹습니다. Rok에서는 `Add`와 `DotAdd`가 서로 다른 트레잇이고, `Matrix * Matrix`와 `Matrix .* Matrix`는 전혀 다른 디스패치를 따릅니다. 둘째, 컴파일러가 점 사용을 강제합니다. 일반 `*`로는 브로드캐스트할 수 **없습니다**. 그래서 사용자 코드에 기대지 않는 결정론적 분리가 보장됩니다.
