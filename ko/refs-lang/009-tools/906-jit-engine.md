# 9.6. JIT 엔진 (NokJitSession)

REPL / Jupyter 가 사용하는 JIT 실행 엔진의 구조를 정의한다. [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 에서 정한 JIT 엔진의 구체적 구현이다.

## 9.6.1. MLIR ExecutionEngine

JIT 의 핵심은 MLIR 의 `MlirExecutionEngine` (ORCv2 기반) 이다.

- 표준 MLIR 도구체인의 일부이다.
- in-process. 별도의 외부 프로세스 호출 없이 현재 nokc 프로세스 안에서 컴파일과 실행을 한다.
- 증분 처리. 한 번에 한 모듈씩 추가할 수 있다.

## 9.6.2. 영감

- clang-repl / Cling: incremental TU 모델.
- Julia: MethodInstance 캐시 모델.

이 둘을 결합한 결과가 증분 컴파일과 specialization 캐시이다.

## 9.6.3. NokJitSession 구조

```
NokJitSession {
    mlir_ee     : MlirExecutionEngine  // ORCv2 래퍼
    main_dylib  : JITDylib              // top-level 심볼 누적
    cells       : List<Cell>            // 입력 단위 이력
    hir_env     : HirEnv                // 이름→item, scope, ops 등록
    spec_cache  : Dict<SpecKey, ModuleHandle>  // specialize 캐시
    last_value  : Value | none           // 마지막 식의 값
    options     : JitOptions             // O 단계, 디스크 캐시 등
}
```

각 필드의 역할:

### `mlir_ee`: ORCv2 ExecutionEngine

- MLIR 모듈을 기계어로 컴파일하고 메모리에 로드한다.
- 심볼 lookup. 이름으로 함수 주소를 가져온다.
- 심볼 등록. 새 모듈을 dylib 에 추가한다.
- 심볼 제거. `tracker.remove()` 로 모듈 단위로 제거한다.

### `main_dylib`: 메인 JITDylib

ORCv2 의 심볼 namespace 이다. 사용자 정의, 표준 라이브러리, 런타임 모두 이 dylib 에 누적된다.

### `cells`: 입력 단위 이력

REPL / Jupyter 의 각 입력 (한 줄 또는 한 셀) 을 Cell 객체로 보존한다.

```
Cell {
    id        : CellId
    source    : str             // 원본 입력
    delta     : HirDelta         // hir_env 에 적용된 변경
    tracker   : ResourceTracker  // ORCv2 의 심볼 추적자 (제거 가능)
    timestamp : Time
}
```

각 Cell 은 ORCv2 의 ResourceTracker 를 가지며 해당 cell 의 심볼을 한꺼번에 제거한다.

### `hir_env`: [HIR](../000-meta/007-glossary.md#hir-high-level-ir-/-mir-mid-level-ir) 환경

이름, 항목, 스코프, 연산자 등록 정보를 담는다.

- 각 cell 이 delta 로 변경한다. `apply` 와 `revert` 를 지원한다.
- transactional. 컴파일 에러 시 변경되지 않는다.

### `spec_cache`: specialization 캐시

[제네릭](../002-types/210-generics.md) 의 [specialize](../000-meta/007-glossary.md#specialize) 결과를 캐시한다.

```
SpecKey = (item_id, type_tuple_hash)
```

같은 `fn` 의 같은 타입 조합은 한 번만 specialize 한다.

캐시는 세션 전체에서 공유된다.

### `last_value`: 마지막 식의 값

REPL 의 마지막 식의 값과 타입을 담는다.

```
> 1 + 2
i32: 3
> _last + 10
i32: 13
```

## 9.6.4. 두 엔진 공통 파이프라인

[실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 과 [컴파일 파이프라인](../000-meta/002-pipeline.md) 의 공통 파이프라인이다. [JIT](../000-meta/007-glossary.md#jit-just-in-time) / [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 두 엔진 모두 같은 단계를 거친다.

```
code → parse → hir → specialize(hir, types) → mir → 표준 MLIR 텍스트
                                                          ↓
                                                    mlir_ee 로 in-process 로
                                                    JIT 컴파일 + 실행
```

AOT 와 완전히 같은 파이프라인이다. JIT 는 마지막 단계에서 외부 도구 호출 대신 `mlir_ee` 를 쓴다.

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

같은 (item_id, type_hash) 의 specialization 은 한 번만 컴파일된다.

세션 안에서 이전 cell 의 specialization 을 다음 cell 이 재사용한다. AOT 의 specialize 캐시와 같은 메커니즘이다.

## 9.6.7. ORCv2 의 ResourceTracker

ORCv2 의 심볼 추적자는 모듈 단위로 심볼을 묶는다. 그룹 단위로 추가하거나 제거할 수 있다.

```
let tracker = ee.create_tracker();
ee.add_module(module, &tracker);
// ... 심볼 사용 ...
tracker.remove();   // 그 모듈의 모든 심볼 제거
```

각 cell 의 cell.tracker 가 그 cell 이 추가한 심볼을 추적한다.

cell 삭제 / 재실행 시 `tracker.remove()` 로 그 cell 의 영향만 롤백한다.

## 9.6.8. hir_env 의 transactional 동작

```
hir_env.apply(delta):
    // 변경 적용

hir_env.revert(delta):
    // 변경 되돌리기 (정확히 반대)
```

각 delta 는 역연산이 가능하다. 트랜잭션처럼 동작한다.

cell 의 컴파일 에러 시:

```
1. parse 까지 OK.
2. hir 변환 시 type error 발견.
3. cells 에 추가하지 않고 hir_env 도 변경하지 않음.
4. 사용자에게 에러 표시.
```

이 transactional semantics 덕분에 사용자가 부분 상태로 빠지는 함정이 차단된다.

## 9.6.9. materialize 실패

ORCv2 가 심볼 link 에 실패하는 경우 (예: 알 수 없는 `extern` 심볼).

```
1. tracker.remove() 로 그 cell 의 심볼 제거.
2. cells 에서 그 cell pop.
3. hir_env.revert(delta).
4. 에러 표시.
```

깨끗한 롤백 덕분에 다음 cell 이 영향받지 않는다.

## 9.6.10. MLIR ExecutionEngine 옵션

JIT 세션 시작 시 옵션:

| 옵션                 | 의미                                                                |
| -------------------- | ------------------------------------------------------------------- |
| `enableObjectCache`  | specialize 결과 .o 덤프 가능 — 디스크 캐시와 연결                   |
| `jitCodeGenOptLevel` | 최적화 단계. REPL 은 O0/O1 (빠른 컴파일), Jupyter 는 O2 (실행 효율) |
| `transformer` 훅     | mlir-opt 외부 호출 없이 in-process 에서 직접 lowering               |

`transformer` 훅은 외부 mlir-opt 프로세스를 띄우지 않고 현재 프로세스 안에서 모든 패스를 적용한다. JIT 의 지연 시간을 최소화한다.

## 9.6.11. JIT 와 AOT 의 차이

| 측면        | JIT                      | AOT                               |
| ----------- | ------------------------ | --------------------------------- |
| 단계        | 같은 파이프라인          | 같은 파이프라인                   |
| 마지막 단계 | mlir_ee 로 in-process    | mlir-opt → mlir-translate → clang |
| specialize  | 호출 지점이 보일 때 즉시 | 모든 호출 지점 수집 후 일괄       |
| 결과        | 메모리 + 디스크 캐시     | .a 파일                           |
| 실행        | 즉시                     | 산출물 실행                       |

두 엔진이 같은 specialize 기계와 같은 MLIR lowering 을 공유한다.

## 9.6.12. REPL vs Jupyter 의 미세 차이

REPL 은 빠른 입력 / 출력 우선:

- O0/O1 (빠른 컴파일).
- 작은 cell 을 가정한다. 캐시가 작아도 된다.

Jupyter 는 셀 결과 / 시각화 우선:

- O2 (실행 효율).
- 큰 셀 / 데이터 분석을 가정한다.

자세한 차이는 `nokc repl` / `nokc jupyter` 의 옵션이 결정한다.

## 9.6.13. 다음: 셀 실행 / 상태 / 에러

[JIT 셀 실행 흐름과 상태 관리](907-jit-cells.md) 에서 Cell 별 상세 동작 (top-level 변수, undo, 셀 재실행, 디스크 캐시 등) 을 정의한다.
