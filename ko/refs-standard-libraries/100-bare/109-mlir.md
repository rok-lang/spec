# 1.9. `rok::bare::mlir` — MLIR 다이얼렉트 직접 접근

`rok::bare::mlir` 모듈은 rok 컴파일러가 사용하는 MLIR 백엔드의 다이얼렉트 op 을 직접 호출할 수 있는 escape hatch 를 제공합니다. 외부 라이브러리 작성자가 텐서 / 자동 미분 / SIMD / GPU / 양자화 같은 가속 영역의 커널을 rok 안에서 표현하기 위한 자리입니다.

일반 rok 코드는 이 모듈을 보지 않습니다. rok 의 일상적인 추상은 [200-core/](../200-core/) 의 트레잇과 컬렉션으로 충분하며, MLIR 노출은 어디까지나 라이브러리 작성자의 영역에 한정됩니다.

## 1.9.1. 동기 — 왜 사용자 표면에 MLIR 이 필요한가

rok 의 일반 사용자는 MLIR 을 알 필요가 없습니다. 산술과 비교, 컬렉션, 동시성은 표준 라이브러리가 충분히 추상화해 두었기 때문입니다. 하지만 다음과 같은 라이브러리 작성자에게는 사정이 다릅니다.

- 텐서 라이브러리: `linalg.generic` / `tensor.*` / `vector.*` / `gpu.*` 다이얼렉트 위에서 커널을 합성
- 자동 미분 라이브러리: source-to-source 변환 결과를 MLIR op 그래프로 표현
- SIMD 라이브러리: `vector` 다이얼렉트로 폭 / 정렬 / shuffle 을 명시
- GPU 백엔드: `gpu.launch_func` 와 `gpu.module` 로 디바이스 코드 emit
- 양자화 / 희소 연산: 전용 다이얼렉트로 표현 후 표준 패스 활용

이런 라이브러리들이 rok 표준 라이브러리만으로 만들어질 수 없는 이유는, MLIR 의 multi-level 추상화가 곧 성능의 원천이기 때문입니다. `linalg.generic` 한 줄이 인접 연산 융합 / 임시 할당 제거 / SIMD / GPU lowering 까지 자동으로 풀어 주는데, 이것을 우회하면 PyTorch / JAX / Mojo 와의 성능 격차가 좁혀지지 않습니다.

## 1.9.2. 표면 — `__mlir_op` 호출

MLIR op 호출의 기본 형태는 다음과 같습니다.

```rok
use rok::bare::mlir::{__mlir_op, __mlir_attr, __mlir_type};

unsafe {
    let r: i32 = __mlir_op::<"arith.addi", i32>(a, b);
    let v: Tensor<f32, (3, 4)> = __mlir_op::<"linalg.generic", _>(...);
}
```

호출은 항상 `unsafe` 블록 안에서만 허용되며, 첫 번째 generic 인자가 다이얼렉트 op 이름, 그 다음이 결과 타입입니다. 인자 / 결과 타입 / 속성은 컴파일 타임에 검증됩니다.

### 속성 (`attributes`)

상당수 MLIR op 은 SSA operand 외에 컴파일 타임 속성 (predicate, axis index, alignment 등) 을 받습니다. `attributes` 키워드 인자로 전달합니다.

```rok
unsafe {
    // arith.cmpi 의 predicate ("eq" / "ne" / "slt" / ...)
    let eq: bool = __mlir_op::<"arith.cmpi", bool>(
        a, b,
        attributes = { "predicate": __mlir_attr::<"#arith<cmpi_predicate eq>"> },
    );

    // memref.load 의 정렬
    let v: i32 = __mlir_op::<"memref.load", i32>(
        buf, idx,
        attributes = { "alignment": __mlir_attr::<"4 : i64"> },
    );
}
```

`__mlir_attr::<"...">` 는 MLIR 텍스트 표현 그대로의 속성 리터럴이며, 컴파일러가 그 자리에서 파싱합니다.

### MLIR 타입 표현

rok 타입과 직접 매핑되지 않는 MLIR 타입 (`!llvm.ptr`, `!memref<?xi32>`, `!tensor<?x?xf32>` 등) 은 `__mlir_type::<"...">` 로 명명합니다.

```rok
unsafe {
    let p = __mlir_op::<"llvm.alloca", __mlir_type<"!llvm.ptr">>(
        size,
        attributes = { "alignment": __mlir_attr::<"8 : i64"> },
    );
}
```

### Region / block 가진 op

`scf.if`, `scf.for`, `linalg.generic`, `gpu.launch` 처럼 region 본체를 가진 op 은 rok 의 클로저로 region 을 제공합니다.

```rok
unsafe {
    // scf.if — then / else 두 region
    let r: i32 = __mlir_op::<"scf.if", i32>(
        cond,
        regions = [
            || -> i32 {
                __mlir_op::<"scf.yield", ()>(a);
                a
            },
            || -> i32 {
                __mlir_op::<"scf.yield", ()>(b);
                b
            },
        ],
    );

    // scf.for — 단일 region, 클로저가 induction var 와 iter args 를 받음
    let sum: i32 = __mlir_op::<"scf.for", i32>(
        lb, ub, step, init,
        regions = [
            |i: index, acc: i32| -> i32 {
                let next = __mlir_op::<"arith.addi", i32>(acc, x);
                __mlir_op::<"scf.yield", ()>(next);
                next
            },
        ],
    );
}
```

규칙:
- `regions = [c1, c2, ...]` 의 클로저 개수는 op 이 요구하는 region 개수와 정확히 일치해야 합니다.
- 각 클로저의 매개변수는 region 의 block argument 에 1:1 대응합니다.
- 클로저 본체는 마지막 op 이 region 의 terminator (보통 `scf.yield` / `linalg.yield` / `gpu.terminator`) 이어야 합니다.
- 클로저는 외부 변수를 캡처할 수 있으며, 캡처된 값은 region 안에서 SSA 값으로 사용됩니다.

### Variadic operand

`vector.shuffle` 처럼 가변 인자를 받는 op 은 rok 의 spread 문법으로 전달합니다.

```rok
unsafe {
    let masked: Vector<i32, 4> = __mlir_op::<"vector.shuffle", _>(
        v1, v2,
        attributes = { "mask": __mlir_attr::<"[0, 2, 4, 6]"> },
    );
}
```

## 1.9.3. 사용 가능 다이얼렉트

[refs-lang 8.7](../../refs-lang/008-abi/807-mlir-dialects.md) 에서 컴파일러가 이해하는 다이얼렉트 전체에 접근할 수 있습니다.

| 다이얼렉트 | 주요 사용처                         |
|------------|-------------------------------------|
| `arith`    | 산술 / 비교 (이미 stdlib 가 사용)   |
| `math`     | sqrt / sin / cos / exp / log        |
| `vector`   | SIMD 폭 / 정렬 / shuffle            |
| `linalg`   | element-wise / matmul / reduction   |
| `tensor`   | shape / extract / insert / reshape  |
| `memref`   | 버퍼 alloc / load / store           |
| `scf`      | structured 제어 흐름                |
| `cf`       | branch / switch                     |
| `gpu`      | GPU launch / kernel / shared memory |
| `affine`   | affine 분석 가능한 루프             |
| `async`    | async fork / await (런타임과 별개)  |
| `llvm`     | 저수준 intrinsic, raw pointer 연동  |

기본 stdlib 가 사용하는 8 다이얼렉트 (`arith` / `scf` / `cf` / `memref` / `index` / `math` / `func` / `llvm`) 외에도 `linalg` / `tensor` / `vector` / `gpu` / `affine` / `async` 가 라이브러리 작성자에게 열려 있습니다.

## 1.9.4. 안전 책임

`rok::bare::mlir` 사용자는 다음 책임을 집니다.

- **타입 일치**: MLIR op 의 operand / result 타입과 rok 타입이 정확히 lower 가능해야 합니다.
- **다이얼렉트 가용성**: 일부 다이얼렉트 (특히 `gpu`) 는 backend 가 활성화된 빌드에서만 lower 가능합니다.
- **안정성**: MLIR 다이얼렉트는 LLVM 버전마다 op 이름 / 시그니처가 변할 수 있으며, 이 모듈을 직접 쓰는 라이브러리는 LLVM 버전을 pin 하는 것이 권장됩니다.
- **GC root**: `memref` / `llvm.ptr` 위에 GC 객체 데이터를 노출할 때는 [1.6 `Pinned<T>`](106-gc-pin.md) 로 pin 을 잡아 두어야 합니다.

이 책임이 부담스러운 라이브러리는 stdlib 의 안전 추상만으로 작성하면 됩니다. `rok::bare::mlir` 은 어디까지나 가속이 필수인 영역에서만 꺼내는 도구입니다.

## 1.9.5. 일반 사용자 코드와의 관계

```rok
// 일반 사용자 — MLIR 보지 않음
let xs: List<f32> = [1.0, 2.0, 3.0];
let sum = xs.iter().sum();

// 라이브러리 내부 — rok::bare::mlir 사용
mod tensor::ops {
    pub fn matmul<...>(a: Tensor<f32, (M, K)>, b: Tensor<f32, (K, N)>) -> Tensor<f32, (M, N)> {
        unsafe {
            __mlir_op::<"linalg.matmul", Tensor<f32, (M, N)>>(a, b)
        }
    }
}
```

라이브러리가 `rok::bare::mlir` 위에 안전한 표면을 얹어 일반 사용자에게 노출하는 패턴이 권장됩니다. 사용자는 `tensor::matmul(a, b)` 만 호출하고, 내부의 MLIR 호출은 라이브러리 책임으로 캡슐화됩니다.

이는 [1.7 FFI](107-ffi.md) 의 `extern "C"` 가 라이브러리 안에 캡슐화되는 패턴과 동형이며, `unsafe` tier 의 일관된 사용 모델입니다.

## 1.9.6. 다른 언어와의 비교

| 언어    | 사용자 표면 MLIR / IR 노출         | 위치                                                 |
|---------|------------------------------------|------------------------------------------------------|
| Mojo    | `__mlir_op` 직접 (기본)            | 일반 사용자도 접근                                   |
| Swift   | `Builtin.X` (stdlib 내부)          | stdlib 작성자 한정                                   |
| Julia   | `llvmcall` (사용자 가능)           | 권장 안 함, 외부 라이브러리 한정                     |
| Rust    | LLVM intrinsic (사용자 불가)       | nightly + intrinsics crate                           |
| **rok** | **`rok::bare::mlir` (명시 unsafe)** | **라이브러리 작성자 한정 — 일반 사용자는 보지 않음** |

rok 의 모델은 Swift 와 Julia 에 가깝습니다. Mojo 처럼 일반 사용자에게도 MLIR 을 노출하지 않으며, `unsafe` 라는 명시적 진입을 강제해 사용자가 어떤 영역에 발을 들이는지 코드 위에 드러냅니다.

## 1.9.7. 권한 / escape hatch 정책

`rok::bare::mlir` 도 [0.3 intrinsic 메커니즘](../000-meta/003-intrinsics.md) 의 사용자 코드 제약과 같은 정책 하에 동작합니다.

- 사용자 코드는 `__mlir_op` 호출 가능, 다만 `unsafe { ... }` 블록 안에서만
- `#[intrinsic("...")]` attribute 부착은 여전히 표준 라이브러리 한정
- 두 메커니즘의 관계: intrinsic 은 *컴파일러가 stdlib 메서드 본문을 가로챔*, `__mlir_op` 은 *사용자가 직접 MLIR op 을 표현*
