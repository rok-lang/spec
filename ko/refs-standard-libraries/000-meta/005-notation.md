# 0.5. 표기법

본 사양의 표기 규약은 [refs-lang 0.8 표기법](../../refs-lang/000-meta/008-notation.md) 을 그대로 따릅니다. 이 장에서는 그 위에 표준 라이브러리 사양에만 추가로 적용되는 규약을 정의합니다.

## 0.5.1. 트레잇 시그니처 표기

각 트레잇 spec 은 시그니처 블록으로 시작하며, 모든 메서드와 associated type, 그리고 supertrait 를 한 곳에 모아서 표기합니다.

```rok
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}
```

이 형태는 표준 라이브러리가 실제로 작성하는 형태와 동일하므로, 사용자도 본문에서 같은 형식으로 사용할 수 있습니다.

## 0.5.2. 메서드 시그니처 표

여러 메서드를 정의하는 트레잇은 다음과 같이 표 형식으로도 요약합니다.

| 메서드 | 시그니처                       | 의미      |
|--------|--------------------------------|-----------|
| `add`  | `fn(self, rhs: Rhs) -> Output` | 산술 덧셈 |
| `sub`  | `fn(self, rhs: Rhs) -> Output` | 산술 뺄셈 |

## 0.5.3. 트레잇 구현 매트릭스

각 트레잇 spec 의 끝에는 어떤 표준 라이브러리 타입이 해당 트레잇을 구현하는지 표로 명시합니다. 컬럼은 타입과 binding 메커니즘만 두며, 구현 파일 경로는 spec 외부의 영역으로 남깁니다.

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

## 0.5.4. 의미 정의 (semantics)

각 메서드의 의미는 산문으로 풀어 쓰며, 형식과 예시뿐 아니라 한계와 에러 케이스까지 함께 다룹니다.

```markdown
### `Add::add(self, rhs)`

산술 덧셈. 두 값을 더한 결과를 반환합니다.

- `i32 + i32`: 2's complement wrap (debug=panic, release=wrap)
- `f64 + f64`: IEEE 754 덧셈
- `int + Int`: 무한정 정밀도. 결과가 i128 범위 넘어가면 BigInt 로 promote

오버플로 핸들링은 [2.1 숫자 타입](../200-core/201-numbers.md) 의 fixed-width 산술
규칙을 따릅니다. saturate / checked / wrapping 변형은 별도 메서드 (`saturating_add` 등)
로 노출됩니다.
```

## 0.5.5. 예시

각 트레잇과 메서드는 하나 이상의 동작 예시를 함께 제시합니다. 예시는 묵시적인 `use rok::core::**;` 가 적용된 상태를 가정하므로, 별도의 import 없이도 그대로 실행할 수 있는 코드여야 합니다.

```rok
let a: i32 = 5;
let b: i32 = 3;
let c = a + b;        //→ 8 (i32)

let x: f64 = 1.5;
let y: f64 = 2.5;
let z = x + y;        //→ 4.0 (f64)
```

## 0.5.6. cross-reference

다른 문서를 참조할 때는 위치에 따라 두 가지 형식을 사용합니다. 같은 사양 안에서는 `[2.7 iter](../200-core/207-iter.md)` 처럼 표기하거나 약식으로 "refs-standard-libraries 2.7" 이라 부르며, refs-lang 을 참조할 때는 `[refs-lang 6.4](../../refs-lang/006-meta-system/604-lang-magic.md)` 형태를 따릅니다.

## 0.5.7. 사양 / 구현의 분리

본 사양이 정의하는 것은 **사양 (specification)** 뿐입니다. 컴파일러와 표준 라이브러리의 실제 구현은 별도의 저장소에서 관리되며, 본 사양과 충돌할 경우 본 사양이 우선합니다.
