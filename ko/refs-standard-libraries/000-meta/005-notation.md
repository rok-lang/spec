# 0.5. 표기법

이 사양의 표기 규약은 [refs-lang 0.8 표기법](../../refs-lang/000-meta/008-notation.md)을 그대로 따릅니다. 이 장은 표준 라이브러리 사양에만 덧붙는 규약을 정의합니다.

## 0.5.1. 트레잇 시그니처 표기

각 트레잇 사양은 시그니처 블록으로 시작합니다. 이 블록은 모든 메서드, 연관 타입(associated type), 상위 트레잇(supertrait)을 한곳에 모아 보여 줍니다.

```rok
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}
```

이 형태는 표준 라이브러리를 실제로 작성하는 형태와 같습니다. 그래서 사용자도 자기 코드에서 같은 형식을 쓸 수 있습니다.

## 0.5.2. 메서드 시그니처 표

메서드가 여럿인 트레잇은 다음과 같이 표로도 요약합니다.

| 메서드 | 시그니처                       | 의미      |
|--------|--------------------------------|-----------|
| `add`  | `fn(self, rhs: Rhs) -> Output` | 산술 덧셈 |
| `sub`  | `fn(self, rhs: Rhs) -> Output` | 산술 뺄셈 |

## 0.5.3. 트레잇 구현 매트릭스

각 트레잇 사양의 끝에는 그 트레잇을 구현하는 표준 라이브러리 타입을 표로 적습니다. 열은 타입과 binding 메커니즘 두 가지뿐입니다. 구현 파일 경로는 사양에 적지 않습니다.

```markdown
## X.Y.Z. 트레잇 구현 매트릭스

| 타입  | binding                         | 비고                 |
|-------|---------------------------------|----------------------|
| `i32` | `#[intrinsic("rok.add.i32")]`   | 컴파일러 PrimOp 직결 |
| `i64` | `#[intrinsic("rok.add.i64")]`   |                      |
| `f32` | `#[intrinsic("rok.add.f32")]`   |                      |
| `f64` | `#[intrinsic("rok.add.f64")]`   |                      |
| `int` | (intrinsic 없음, auto-promote) | 표준 라이브러리 정의 |
```

## 0.5.4. 의미 정의(semantics)

각 메서드의 의미는 산문으로 풀어 씁니다. 형식과 예시뿐 아니라 한계와 에러 상황도 다룹니다.

```markdown
### `Add::add(self, rhs)`

산술 덧셈. 두 값을 더한 결과를 반환합니다.

- `i32 + i32`: 2's complement wrap (debug=panic, release=wrap)
- `f64 + f64`: IEEE 754 덧셈
- `int + Int`: 무한정 정밀도. 결과가 i64 범위 넘어가면 BigInt 로 promote

오버플로 핸들링은 [2.1 숫자 타입](../200-core/201-numbers.md) 의 fixed-width 산술
규칙을 따릅니다. saturate / checked / wrapping 변형은 별도 메서드 (`saturating_add` 등)
로 노출됩니다.
```

## 0.5.5. 예시

각 트레잇과 메서드에는 동작하는 예시를 하나 이상 붙입니다. 예시는 묵시적인 `use rok::core::**;`가 적용된 상태를 가정합니다. 그래서 아무것도 가져오지 않고 그대로 실행되는 코드여야 합니다.

```rok
let a: i32 = 5;
let b: i32 = 3;
let c = a + b;        //→ 8 (i32)

let x: f64 = 1.5;
let y: f64 = 2.5;
let z = x + y;        //→ 4.0 (f64)
```

## 0.5.6. 상호 참조

다른 문서를 참조할 때는 위치에 따라 두 형식 중 하나를 씁니다. 같은 사양 안에서는 `[2.7 iter](../200-core/207-iter.md)`처럼 쓰거나 약식으로 "refs-standard-libraries 2.7"이라고 부릅니다. refs-lang은 `[refs-lang 6.4](../../refs-lang/006-meta-system/604-lang-magic.md)` 형태로 참조합니다.

## 0.5.7. 사양과 구현의 분리

이 문서는 **사양**(specification)만 정의합니다. 컴파일러와 표준 라이브러리의 실제 구현은 별도 저장소에서 관리합니다. 구현이 이 사양과 충돌하면 이 사양이 우선합니다.

## 0.5.8. `try_` 접두사

`try_` 이름은 다음 세 형태 가운데 하나를 나타내며, 이 셋만 나타냅니다.

1. 막히지 않는 시도: 기본 형태가 기다리는 일을 지금 하고, 할 수 없었으면 그렇다고 알립니다(`Mutex::try_lock`, `Receiver::try_recv`, `Sender::try_send`).
2. 던지는 형태: 기본 형태가 panic하거나 결과를 낮추는 자리, 또는 기본 형태가 없는 자리에서 던집니다(`TryFrom::try_from`, `FormatSpec::try_parse`, `File::try_clone`).
3. 받은 클로저가 던진 것을 그대로 넘기는 형태(`Iterator::try_fold`, `Iterator::try_for_each`, `par::try_scope`).

기본 형태가 panic하는 자리에서 `none`을 돌려주는 형태는 `try_` 형태가 아닙니다. 그런 형태는 `task::current_id`(task 밖이면 `none`)나 `Float::parse_or_none`처럼 돌려주는 것을 이름에 담습니다. 옛 이름 `task::try_id`와 `Float::try_parse`는 0.1.6부터 deprecated입니다([refs-lang 6.2.7](../../refs-lang/006-meta-system/602-built-in-attributes.md)). 표준 라이브러리의 새 이름은 이 규칙을 따릅니다. 규칙에 어긋나는 예전 이름은 이름을 바꿀 수 있는 릴리스까지 그대로 둡니다.
