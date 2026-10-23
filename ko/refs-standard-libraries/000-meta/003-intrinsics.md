# 0.3. intrinsic 메커니즘 - 컴파일러 ↔ 표준 라이브러리 경계

표준 라이브러리 트레잇 메서드, 가령 `i32::add` 의 본문은 결국 LLVM 의 `add` 명령어로 lowering 되어야 합니다. 이 장에서는 표준 라이브러리 메서드 본문이 어떤 경로를 통해 컴파일러 백엔드 (PrimOp / MIR / MLIR) 와 연결되는지 정의합니다.

## 0.3.1. 문제 - 부트스트랩 순환

문제의 핵심은 표준 라이브러리와 컴파일러 사이에서 발생하는 부트스트랩 순환입니다. 다음 흐름을 보면 어디에서 막히는지 분명해집니다.

```
사용자 코드:       let r = a + b;        (a, b: i32)
↓
컴파일러:         Add::add(a, b) 호출 dispatch     (refs-lang 3.1)
↓
표준 라이브러리:    impl Add for i32 { fn add(self, rhs) { ... } }
              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
              여기 "..." 자리에 무엇을 적나?
              "self + rhs" 라 쓰면 또 Add::add 디스패치 → 무한 재귀
              raw `arith.addi` 을 적을 수도 없음 (사용자 언어 표면 아님)
↓
LLVM:        add i32 %a, %b
```

따라서 표준 라이브러리 본문이 자기 자신을 호출하지 않으면서도 컴파일러 백엔드의 `add` 명령으로 곧장 이어지는 경로가 필요합니다.

## 0.3.2. 해결 - `#[intrinsic("rok.OP.TYPE")]` 속성

해결책은 표준 라이브러리 메서드 본문에 컴파일러가 인식하는 속성을 부착해 두는 것입니다. 그러면 컴파일러는 해당 호출을 만났을 때 본문을 lowering 하지 않고 곧바로 PrimOp / MIR 로 변환합니다.

```rok
impl Add for i32 {
    type Output = i32;

    #[intrinsic("rok.add.i32")]
    fn add(self, rhs: i32) -> i32 {
        // 본문이 비어 있어도 OK - 컴파일러가 intrinsic 경로 사용
        // 본문이 있어도 컴파일러가 우선
    }
}
```

컴파일러의 동작 순서는 다음과 같습니다. 먼저 `Add::add(a, b)` 호출을 dispatch 할 때 method_table 에서 `i32::add` 를 찾고, 이어서 그 메서드에 `#[intrinsic("rok.add.i32")]` 가 부착되어 있는지 확인합니다. 부착이 확인되면 본문 lowering 대신 `PrimOp::Add { ty: i32 }` 를 직접 emit 하며, 이후 MIR / MLIR / LLVM lowering 단계를 거쳐 결국 `add i32` 명령에 도달합니다.

이 메커니즘 덕분에 컴파일러의 가로채기 동작이 트레잇 dispatch 안으로 자연스럽게 흡수됩니다.

## 0.3.3. intrinsic 이름 컨벤션

intrinsic 이름은 `rok.<OP>.<TYPE>` 형태를 따릅니다.

```
rok.add.i32        i32 + i32
rok.add.i64
rok.add.f32
rok.add.f64
rok.sub.i32
rok.mul.i32
rok.div.i32
rok.rem.i32
rok.and.i32        bitwise AND
rok.or.i32
rok.xor.i32
rok.shl.i32
rok.shr.i32
rok.eq.i32
rok.lt.i32
...
rok.neg.i32        unary
rok.not.bool       logical not (bool)
rok.not.i32        bitwise not (정수)
rok.cast.i32.i64   type cast
```

각 OP 는 컴파일러의 `PrimOp` enum 과 1:1 로 대응됩니다. 이름 자체는 LLVM 명령어와 Rust 의 intrinsic 명명 규약에서 영향을 받았습니다.

## 0.3.4. 본문 fallback 정책

intrinsic 이 부착된 메서드라 하더라도 본문을 함께 가질 수 있습니다. 다음 예시가 그 양상을 잘 보여줍니다.

```rok
impl Add for BigInt {
    type Output = BigInt;

    // BigInt 는 fixed-width 가 아니라 intrinsic 없음 - 본문이 권위
    fn add(self, rhs: BigInt) -> BigInt {
        // 실제 BigInt 덧셈 구현
        ...
    }
}

impl Add for i32 {
    type Output = i32;

    #[intrinsic("rok.add.i32")]
    fn add(self, rhs: i32) -> i32 {
        // 본문 - fn 값으로 꺼내 부르거나 intrinsic 을 모르는 환경에서 실행
        self + rhs                    // primitive 연산자는 기계 연산으로 직접 lower
    }
}
```

규칙은 다음과 같이 요약됩니다. intrinsic 이 활성화된 정상 컴파일 경로에서는 본문이 무시되며, intrinsic 을 인식하지 못하는 실험 backend 나 interpret 환경에서는 본문이 실제로 사용됩니다. 만약 본문이 비어 있다면 stub `panic("intrinsic not available")` 로 처리됩니다.

메서드를 `fn` 값으로 꺼낸 경우 (`let f = i32::add;`) 그 값을 호출하면 본문이 실행됩니다. intrinsic 은 직접 호출 지점에만 적용되므로, intrinsic 메서드의 본문도 같은 의미의 올바른 구현이어야 합니다.

## 0.3.5. 사용자 코드의 intrinsic 사용

`#[intrinsic(...)]` 은 표준 라이브러리가 컴파일러와 결합되는 경계에서만 동작하는 메커니즘이므로, 사용자 코드는 이 속성을 직접 부착할 수 없습니다. 함수, 제네릭 함수, 메서드 모두 같습니다.

```rok
// 사용자 코드 - 거부.
#[intrinsic("rok.add.i32")]
fn my_add(a: i32, b: i32) -> i32 { ... }
// 컴파일 에러: "intrinsic 속성은 표준 라이브러리 한정"
```

이 제약은 표준 라이브러리 trait 외부에서 컴파일러 백엔드를 임의로 호출할 수 있는 escape hatch 가 열리는 것을 막아 줍니다.

## 0.3.6. 다른 메커니즘과의 비교

intrinsic 은 컴파일러와 표준 라이브러리를 잇는 여러 매직 가운데 하나일 뿐이며, 다음 표는 각각의 역할을 정리한 것입니다.

| 메커니즘              | 용도                                                        | spec                                                                   |
|-----------------------|-------------------------------------------------------------|------------------------------------------------------------------------|
| `#[intrinsic("...")]` | 표준 라이브러리 메서드 ↔ 컴파일러 백엔드 PrimOp             | 본 문서                                                                |
| `#[lang(...)]`        | 표준 라이브러리 항목 ↔ 컴파일러 슬롯 (none, iter, index 등) | [refs-lang 6.4](../../refs-lang/006-meta-system/604-lang-magic.md)     |
| `#[infix(...)]`       | 토큰 ↔ 표준 라이브러리 트레잇 메서드                        | [refs-lang 6.7](../../refs-lang/006-meta-system/607-operator-magic.md) |
| `#[literal(...)]`     | 리터럴 형태 ↔ 표준 라이브러리 핸들러                        | [refs-lang 6.5](../../refs-lang/006-meta-system/605-literal-magic.md)  |

네 매직은 서로 직교합니다. `#[intrinsic]` 이 컴파일 백엔드와의 결합을 담당한다면, 나머지는 모두 언어 syntax 와의 결합을 다룹니다.
