# 1.9. `rok::bare::mlir`: MLIR 다이얼렉트 직접 접근

`rok::bare::mlir` 모듈은 Rok 컴파일러의 MLIR 백엔드 다이얼렉트 op을 직접 호출하는 탈출구(escape hatch)입니다. 외부 라이브러리 작성자는 텐서 / 자동 미분 / SIMD / GPU / 양자화 같은 가속 영역의 커널을 Rok 안에서 표현할 때 이 모듈을 씁니다.

일반 Rok 코드는 이 모듈을 보지 않습니다. 일상적인 추상은 [200-core/](../200-core/)의 트레잇과 컬렉션으로 충분합니다. MLIR은 라이브러리 작성자에게만 노출합니다.

## 1.9.1. 동기: 사용자 표면에 MLIR이 필요한 이유

Rok의 일반 사용자는 MLIR을 몰라도 됩니다. 산술, 비교, 컬렉션, 동시성은 표준 라이브러리가 충분히 추상화해 두었습니다. 하지만 다음과 같은 라이브러리 작성자는 사정이 다릅니다.

- 텐서 라이브러리: `linalg.generic` / `tensor.*` / `vector.*` / `gpu.*` 다이얼렉트 위에서 커널을 합성합니다.
- 자동 미분 라이브러리: 소스 대 소스 변환 결과를 MLIR op 그래프로 표현합니다.
- SIMD 라이브러리: `vector` 다이얼렉트로 폭 / 정렬 / shuffle을 명시합니다.
- GPU 백엔드: `gpu.launch_func`와 `gpu.module`로 디바이스 코드를 내보냅니다.
- 양자화 / 희소 연산: 전용 다이얼렉트로 표현하고 표준 패스를 씁니다.

이런 라이브러리를 Rok 표준 라이브러리만으로 만들 수 없는 이유가 있습니다. MLIR의 다층 추상화가 곧 성능의 원천이기 때문입니다. `linalg.generic` 한 줄이 인접 연산 융합 / 임시 할당 제거 / SIMD / GPU lowering으로 자동으로 풀립니다. 이 경로를 우회하면 PyTorch / JAX / Mojo와 벌어진 성능 격차를 좁힐 수 없습니다.

## 1.9.2. 표면: `__mlir_op` 호출

MLIR op 호출의 기본 형태는 다음과 같습니다.

```rok
use rok::bare::mlir::{__mlir_op, __mlir_attr, __mlir_type};

unsafe {
    let r: i32 = __mlir_op::<"arith.addi", i32>(a, b);
    let v: Tensor<f32, (3, 4)> = __mlir_op::<"linalg.generic", _>(...);
}
```

호출은 `unsafe` 블록 안에서만 할 수 있습니다. 첫 번째 제네릭 인자는 다이얼렉트 op 이름(`"dialect.op"`)이고, 그다음은 결과 타입 `R`입니다.

- `R`은 Rok 타입입니다. `_`이면 기대 타입을, `()`이면 결과 없음을 뜻합니다.
- 나머지 인자가 operand입니다. 속성은 `attributes = { "k": __mlir_attr::<"text"> }`로 줍니다.
- 컴파일러는 op을 MLIR generic form(`"dialect.op"(%a, %b) {k = text} : (..) -> R`)으로 내보냅니다. op 이름, operand, 속성은 `rokc`에 내장된 MLIR이 검증합니다([0.2.4](../../refs-lang/000-meta/002-pipeline.md#024-백엔드)). `__mlir_op`을 쓰는 모듈은 어느 최적화 수준에서나 MLIR → LLVM 백엔드로 컴파일합니다([refs-lang 9.2.10](../../refs-lang/009-tools/902-aot-pipeline.md#9210-백엔드-선택)).

### 속성(`attributes`)

많은 MLIR op은 SSA operand 말고도 컴파일 타임 속성(predicate, 축 인덱스, 정렬 등)을 받습니다. 속성은 `attributes` 키워드 인자로 넘깁니다.

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

`__mlir_attr::<"...">`는 MLIR 텍스트 표현 그대로인 속성 리터럴입니다. 컴파일러는 텍스트를 그대로 MLIR에 넘깁니다. `__mlir_attr::<"...">()`처럼 빈 호출로 적어도 같습니다.

### MLIR 타입 표현

Rok 타입에 직접 매핑되지 않는 MLIR 타입(`index`, `!llvm.ptr`, `memref<?xi32>` 등)은 타입 위치에 `__mlir_type<"...">`로 적습니다(`__mlir_type::<"...">`도 됩니다).

```rok
unsafe {
    let p = __mlir_op::<"llvm.alloca", __mlir_type<"!llvm.ptr">>(
        size,
        attributes = { "alignment": __mlir_attr::<"8 : i64"> },
    );
}
```

`__mlir_type<"..">`는 `__mlir_op`의 결과 타입 `R`, operand, 함수 매개변수와 반환, `let`의 타입으로 쓸 수 있습니다. 그 값은 집합체(struct, 배열, GC 객체) 안에 저장할 수 없습니다.

### region / block을 가진 op

`scf.if`, `scf.for`, `linalg.generic`처럼 region 본체가 있는 op은 `regions = [..]`의 클로저로 region을 줍니다.

```rok
unsafe {
    // scf.if - then / else 두 region
    let r: i32 = __mlir_op::<"scf.if", i32>(
        cond,
        regions = [
            || { __mlir_op::<"scf.yield", ()>(a); },
            || { __mlir_op::<"scf.yield", ()>(b); },
        ],
    );

    // scf.for - lb / ub / step 은 index 값
    let lb = __mlir_op::<"arith.index_cast", __mlir_type<"index">>(0i64);
    let ub = __mlir_op::<"arith.index_cast", __mlir_type<"index">>(n);
    let step = __mlir_op::<"arith.index_cast", __mlir_type<"index">>(1i64);
    let sum: i32 = __mlir_op::<"scf.for", i32>(
        lb, ub, step, init,
        regions = [
            |i: __mlir_type<"index">, acc: i32| {
                let next = __mlir_op::<"arith.addi", i32>(acc, x);
                __mlir_op::<"scf.yield", ()>(next);
            },
        ],
    );
}
```

규칙은 다음과 같습니다.

- 클로저 하나가 region 하나입니다. 클로저 개수는 op이 요구하는 region 개수와 같아야 합니다.
- 클로저의 매개변수는 entry block의 인자입니다. 매개변수에는 타입을 적어야 합니다.
- region의 결과는 terminator(`scf.yield` 등)의 operand입니다. 클로저의 마지막 값과 `-> R`은 의미가 없습니다.
- 본체는 op을 차례로 늘어놓은 직선 코드입니다. 제어 흐름은 `scf` op으로 씁니다. `if` / `match` / 반복문 / `return` / `break` / `continue` / panic은 에러입니다.
- region은 generic form으로 나가고, 바깥 값을 그대로 쓸 수 있습니다. 그래서 region이 바깥 값을 볼 수 없는 op(`gpu.func` 등)은 지원하지 않습니다.
- 현재 한계: op의 결과는 하나, region 본체의 block은 하나입니다.

### 가변 개수 operand

`vector.shuffle`처럼 가변 인자를 받는 op에는 Rok의 펼치기(spread) 문법으로 인자를 넘깁니다.

```rok
unsafe {
    let masked: Vector<i32, 4> = __mlir_op::<"vector.shuffle", _>(
        v1, v2,
        attributes = { "mask": __mlir_attr::<"[0, 2, 4, 6]"> },
    );
}
```

## 1.9.3. 사용 가능 다이얼렉트

[refs-lang 8.7](../../refs-lang/008-abi/807-mlir-dialects.md)에 나온, 컴파일러가 이해하는 다이얼렉트 전체에 접근할 수 있습니다.

| 다이얼렉트 | 주요 사용처                         |
|------------|-------------------------------------|
| `arith`    | 산술 / 비교 (이미 stdlib이 씀)     |
| `math`     | sqrt / sin / cos / exp / log        |
| `vector`   | SIMD 폭 / 정렬 / shuffle            |
| `linalg`   | element-wise / matmul / reduction   |
| `tensor`   | shape / extract / insert / reshape  |
| `memref`   | 버퍼 할당 / load / store            |
| `scf`      | 구조적 제어 흐름 (기본으로 `--convert-scf-to-cf`로 낮춤) |
| `cf`       | branch / switch                     |
| `gpu`      | GPU launch / kernel / shared memory |
| `affine`   | affine 분석 가능한 루프             |
| `async`    | async fork / await (런타임과 별개)  |
| `llvm`     | 저수준 intrinsic, 원시 포인터 연동  |

기본 stdlib이 쓰는 다이얼렉트 8개(`arith` / `scf` / `cf` / `memref` / `index` / `math` / `func` / `llvm`) 외에도 `linalg` / `tensor` / `vector` / `gpu` / `affine` / `async`가 라이브러리 작성자에게 열려 있습니다.

등록되는 것은 이 다이얼렉트와, lowering에 쓰는 `bufferization` / `ub` / `dlti`뿐입니다. 그 밖의 upstream 다이얼렉트의 op은 로드하지 않은 플러그인의 op처럼 `rokc`가 알지 못합니다([1.10.7](110-mlir-plugin.md#1107-빌드-격리)).

## 1.9.4. 안전 책임

`rok::bare::mlir` 사용자는 다음 책임을 집니다.

- **타입 일치**: MLIR op의 operand / result 타입과 Rok 타입이 정확히 대응하도록 낮출 수 있어야 합니다.
- **다이얼렉트 가용성**: 일부 다이얼렉트(특히 `gpu`)는 해당 백엔드를 켠 빌드에서만 낮출 수 있습니다.
- **안정성**: MLIR 다이얼렉트는 LLVM 버전마다 op 이름 / 시그니처가 바뀔 수 있습니다. 이 모듈을 직접 쓰는 라이브러리는 LLVM 버전을 고정하기를 권장합니다.
- **GC 루트**: `memref` / `llvm.ptr` 위에 GC 객체 데이터를 노출할 때는 [1.6 `Pinned<T>`](106-gc-pin.md)로 고정해 두어야 합니다.

이 책임이 부담스러운 라이브러리는 stdlib의 안전한 추상만으로 작성하기를 권장합니다. `rok::bare::mlir`은 가속이 꼭 필요한 영역에서만 꺼내는 도구입니다.

## 1.9.5. 일반 사용자 코드와 관계

```rok
// 일반 사용자 - MLIR 보지 않음
let xs: List<f32> = [1.0, 2.0, 3.0];
let sum = xs.iter().sum();

// 라이브러리 내부 - rok::bare::mlir 사용
mod tensor::ops {
    pub fn matmul<...>(a: Tensor<f32, (M, K)>, b: Tensor<f32, (K, N)>) -> Tensor<f32, (M, N)> {
        unsafe {
            __mlir_op::<"linalg.matmul", Tensor<f32, (M, N)>>(a, b)
        }
    }
}
```

권장 패턴은 라이브러리가 `rok::bare::mlir` 위에 안전한 표면을 얹어 일반 사용자에게 노출하는 방식입니다. 사용자는 `tensor::matmul(a, b)`만 호출합니다. 내부 MLIR 호출은 라이브러리가 책임지고 캡슐화합니다.

[1.7 FFI](107-ffi.md)의 `extern "C"`를 라이브러리 안에 캡슐화하는 패턴과 같은 모델입니다. `unsafe` tier 전체에서 일관된 사용 방식입니다.

## 1.9.6. 다른 언어와 비교

| 언어    | 사용자 표면 MLIR / IR 노출         | 위치                                                 |
|---------|------------------------------------|------------------------------------------------------|
| Mojo    | `__mlir_op` 직접 (기본)            | 일반 사용자도 접근                                   |
| Swift   | `Builtin.X` (stdlib 내부)          | stdlib 작성자 한정                                   |
| Julia   | `llvmcall` (사용자 가능)           | 권장 안 함, 외부 라이브러리 한정                     |
| Rust    | LLVM intrinsic (사용자 불가)       | nightly + intrinsics crate                           |
| **Rok** | **`rok::bare::mlir` (명시 unsafe)** | **라이브러리 작성자 한정, 일반 사용자는 보지 않음** |

Rok의 모델은 Swift와 Julia에 가깝습니다. Mojo와 달리 일반 사용자에게 MLIR을 노출하지 않습니다. `unsafe`라는 명시적 진입을 강제해서, 사용자가 어떤 영역에 들어가는지 코드에 드러냅니다.

## 1.9.7. 권한 / escape hatch 정책

`rok::bare::mlir`도 [0.3 intrinsic 메커니즘](../000-meta/003-intrinsics.md)의 사용자 코드 제약과 같은 정책을 따릅니다.

- 사용자 코드는 `__mlir_op`을 호출할 수 있지만, `unsafe { ... }` 블록 안에서만 호출할 수 있습니다.
- `#[intrinsic("...")]` 속성은 여전히 표준 라이브러리만 붙일 수 있습니다.
- 두 메커니즘의 관계: intrinsic은 '컴파일러가 stdlib 메서드 본문을 가로채는 것'이고, `__mlir_op`은 '사용자가 MLIR op을 직접 표현하는 것'입니다.
