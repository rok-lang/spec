# 9.6. JIT 엔진 (RokJitSession)

이 절에서는 REPL 과 Jupyter 가 사용하는 JIT 실행 엔진의 구조를 정의합니다. [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 에서 정한 JIT 엔진의 구체적인 구현에 해당합니다.

> [!NOTE]
> 이 장의 ORCv2 기반 엔진은 아직 구현되지 않았습니다. 현재 `rokc repl` 은 셀마다 AOT 파이프라인으로 동적 라이브러리를 만들어 오래 사는 host 프로세스에 올리는 방식으로 셀 세션의 규칙 ([9.7.0](907-jit-cells.md#970-셀-세션의-규칙)) 을 구현합니다. 셀은 한 번만 실행되고 바인딩과 항목은 host 안에 유지됩니다.

## 9.6.1. MLIR ExecutionEngine

JIT 의 핵심은 MLIR 의 `MlirExecutionEngine` (ORCv2 기반) 입니다.

- 표준 MLIR 도구 체인의 일부입니다.
- in-process 로 동작합니다. 별도의 외부 프로세스를 호출하지 않고 현재 rokc 프로세스 안에서 컴파일과 실행을 모두 수행합니다.
- 증분 처리를 지원합니다. 한 번에 한 모듈씩 추가할 수 있습니다.

## 9.6.2. 영감

- clang-repl / Cling 의 incremental TU 모델에서 영향을 받았습니다.
- Julia 의 MethodInstance 캐시 모델에서 영향을 받았습니다.

이 둘을 결합한 결과가 바로 증분 컴파일과 specialization 캐시입니다.

## 9.6.3. RokJitSession 구조

```
RokJitSession {
    mlir_ee     : MlirExecutionEngine  // ORCv2 래퍼
    main_dylib  : JITDylib              // top-level 심볼 누적
    cells       : List<Cell>            // 입력 단위 이력
    hir_env     : HirEnv                // 이름→item, scope, ops 등록
    spec_cache  : Dict<SpecKey, ModuleHandle>  // specialize 캐시
    last_value  : Value | none           // 마지막 식의 값
    options     : JitOptions             // O 단계, 디스크 캐시 등
}
```

각 필드의 역할은 아래와 같습니다.

### `mlir_ee`: ORCv2 ExecutionEngine

- MLIR 모듈을 기계어로 컴파일하고 메모리에 로드합니다.
- 심볼 lookup 을 수행합니다. 즉, 이름으로 함수 주소를 가져옵니다.
- 심볼 등록을 수행합니다. 새 모듈을 dylib 에 추가합니다.
- 심볼 제거를 수행합니다. `tracker.remove()` 로 모듈 단위로 제거할 수 있습니다.

### `main_dylib`: 메인 JITDylib

ORCv2 의 심볼 namespace 입니다. 사용자 정의, 표준 라이브러리, 런타임이 모두 이 dylib 에 누적됩니다.

### `cells`: 입력 단위 이력

REPL 과 Jupyter 의 각 입력 (한 줄 또는 한 셀) 을 Cell 객체로 보존합니다.

```
Cell {
    id        : CellId
    source    : str             // 원본 입력
    delta     : HirDelta         // hir_env 에 적용된 변경
    tracker   : ResourceTracker  // ORCv2 의 심볼 추적자 (제거 가능)
    timestamp : Time
}
```

각 Cell 은 ORCv2 의 ResourceTracker 를 가지며, 이를 통해 해당 cell 의 심볼을 한꺼번에 제거할 수 있습니다.

### `hir_env`: [HIR](../000-meta/007-glossary.md#hir-high-level-ir--mir-mid-level-ir) 환경

이름, 항목, 스코프, 연산자 등록 정보를 포함합니다.

- 각 cell 이 delta 로 변경합니다. `apply` 와 `revert` 를 모두 지원합니다.
- transactional 하게 동작합니다. 컴파일 에러가 발생하면 변경되지 않습니다.

### `spec_cache`: specialization 캐시

[제네릭](../002-types/210-generics.md) 의 [단형화](../000-meta/007-glossary.md#monomorphization-단형화) 결과를 캐시합니다.

```
SpecKey = (item_id, type_tuple_hash)
```

같은 `fn` 의 같은 타입 조합은 한 번만 단형화됩니다.

캐시는 세션 전체에서 공유됩니다.

### `last_value`: 마지막 식의 값

REPL 의 마지막 식의 값과 타입을 포함합니다.

```
> 1 + 2
i32: 3
> _last + 10
i32: 13
```

## 9.6.4. 두 엔진의 공통 파이프라인

[실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 및 [컴파일 파이프라인](../000-meta/002-pipeline.md) 에 정의된 공통 파이프라인입니다. [JIT](../000-meta/007-glossary.md#jit-just-in-time) 와 [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 두 엔진 모두 같은 단계를 통과하게 됩니다.

```
code → parse → hir → specialize(hir, types) → mir → 표준 MLIR 텍스트
                                                          ↓
                                                    mlir_ee 로 in-process 로
                                                    JIT 컴파일 + 실행
```

AOT 와 완전히 동일한 파이프라인이며, JIT 는 마지막 단계에서 외부 도구를 호출하는 대신 `mlir_ee` 를 사용한다는 점만 다릅니다.

## 9.6.5. 실행 흐름 (REPL 한 줄 / Jupyter 한 셀)

```
1. 사용자 입력 → parse → hir
2. cells.push(Cell { delta, hir_changes })
3. hir_env.apply(delta)
4. specialize(hir, top_level types) → mir
5. mir → 표준 MLIR 텍스트
6. mlir_ee 로 모듈 추가 + materialize
7. ORCv2 가 top-level expr 의 함수 주소 lookup
8. 그 함수를 호출 → 결과 + 타입을 last_value 에 저장
9. 결과 출력 (REPL) / 셀 결과 표시 (Jupyter)
```

## 9.6.6. SpecKey: specialization 캐시 키

```
SpecKey = (
    item_id    : u64,           // 정의된 항목의 id
    type_hash  : u128            // 인자 타입 튜플의 SHA-256 앞 부분
)
```

같은 (item_id, type_hash) 의 specialization 은 한 번만 컴파일됩니다.

세션 안에서 이전 cell 의 specialization 을 다음 cell 이 재사용합니다. 이는 AOT 의 specialize 캐시와 동일한 메커니즘입니다.

## 9.6.7. ORCv2 의 ResourceTracker

ORCv2 의 심볼 추적자는 모듈 단위로 심볼을 묶어 줍니다. 그 결과 그룹 단위로 추가하거나 제거할 수 있습니다.

```
let tracker = ee.create_tracker();
ee.add_module(module, &tracker);
// ... 심볼 사용 ...
tracker.remove();   // 그 모듈의 모든 심볼 제거
```

각 cell 의 cell.tracker 가 그 cell 이 추가한 심볼을 추적하게 됩니다.

cell 을 삭제하거나 재실행할 때 `tracker.remove()` 를 호출하면 그 cell 의 영향만 깔끔하게 롤백할 수 있습니다.

## 9.6.8. hir_env 의 transactional 동작

```
hir_env.apply(delta):
    // 변경 적용

hir_env.revert(delta):
    // 변경 되돌리기 (정확히 반대)
```

각 delta 는 역연산이 가능하므로 트랜잭션처럼 동작합니다.

cell 의 컴파일 에러가 발생할 때의 처리는 다음과 같습니다.

```
1. parse 까지 OK.
2. hir 변환 시 type error 발견.
3. cells 에 추가하지 않고 hir_env 도 변경하지 않음.
4. 사용자에게 에러 표시.
```

이러한 transactional semantics 덕분에 사용자가 부분 상태로 빠지는 함정을 차단할 수 있습니다.

## 9.6.9. materialize 실패

ORCv2 가 심볼 link 에 실패하는 경우 (예: 알 수 없는 `extern` 심볼) 의 처리는 아래와 같습니다.

```
1. tracker.remove() 로 그 cell 의 심볼 제거.
2. cells 에서 그 cell pop.
3. hir_env.revert(delta).
4. 에러 표시.
```

이처럼 깨끗한 롤백이 보장되므로 다음 cell 이 영향을 받지 않습니다.

## 9.6.10. MLIR ExecutionEngine 옵션

JIT 세션을 시작할 때 사용할 수 있는 옵션은 다음과 같습니다.

| 옵션                 | 의미                                                                |
|----------------------|---------------------------------------------------------------------|
| `enableObjectCache`  | specialize 결과 .o 덤프 가능 — 디스크 캐시와 연결                   |
| `jitCodeGenOptLevel` | 최적화 단계. REPL 은 O0/O1 (빠른 컴파일), Jupyter 는 O2 (실행 효율) |
| `transformer` 훅     | mlir-opt 외부 호출 없이 in-process 에서 직접 lowering               |

`transformer` 훅은 외부 mlir-opt 프로세스를 띄우지 않고 현재 프로세스 안에서 모든 패스를 적용합니다. 이를 통해 JIT 의 지연 시간을 최소화할 수 있습니다.

## 9.6.11. JIT 와 AOT 의 차이

| 측면        | JIT                      | AOT                               |
|-------------|--------------------------|-----------------------------------|
| 단계        | 같은 파이프라인          | 같은 파이프라인                   |
| 마지막 단계 | mlir_ee 로 in-process    | mlir-opt → mlir-translate → clang |
| specialize  | 호출 지점이 보일 때 즉시 | 모든 호출 지점 수집 후 일괄       |
| 결과        | 메모리 + 디스크 캐시     | .a 파일                           |
| 실행        | 즉시                     | 산출물 실행                       |

두 엔진은 같은 단형화 기계와 같은 MLIR lowering 을 공유합니다.

## 9.6.12. REPL 과 Jupyter 의 미세한 차이

REPL 은 빠른 입력과 출력을 우선합니다.

- O0/O1 을 사용하여 빠른 컴파일을 지향합니다.
- 작은 cell 을 가정하므로 캐시가 작아도 충분합니다.

Jupyter 는 셀 결과와 시각화를 우선합니다.

- O2 를 사용하여 실행 효율을 높입니다.
- 큰 셀이나 데이터 분석을 가정합니다.

자세한 차이는 `rokc repl` 과 `rokc jupyter` 의 옵션이 결정합니다.

## 9.6.13. 다음: 셀 실행 / 상태 / 에러

[JIT 셀 실행 흐름과 상태 관리](907-jit-cells.md) 에서 Cell 별 상세 동작 (top-level 변수, undo, 셀 재실행, 디스크 캐시 등) 을 정의합니다.
