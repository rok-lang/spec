# 9.7. JIT 셀 실행 흐름과 상태 관리

[JIT 엔진](906-jit-engine.md) 의 NokJitSession 이 사용자 입력 (cell) 을 어떻게 처리하는지를 자세히 정의한다.

## 9.7.1. Cell 의 종류

REPL 의 한 줄과 Jupyter 의 한 셀은 내용에 따라 처리 방식이 달라진다.

```
1. 표현식 (식)              → 평가 + last_value 저장 + 출력
2. 문장 (let / 함수 호출)    → 실행 (반환값은 unit)
3. 정의 (fn / struct / ...)  → hir_env 에 추가
4. import (use / mod)        → hir_env 의 가시 표 갱신
5. 메타 / 속성              → 적절한 처리
```

다섯 모두 같은 cell 처리 흐름을 따른다 (마지막 출력만 다르다).

## 9.7.2. 실행 흐름: 자세히

```
nokc 입력 받음:
  ↓
1. parse  →  AST
  ↓ (parse error 면 즉시 표시, cell 추가하지 않음)
2. lower to HIR  →  HIR + delta
  ↓ (type error 면 표시, cell 추가하지 않음)
3. cells.push(new_cell)
4. hir_env.apply(delta)
  ↓
5. 새 항목 / 식에 specialize 가 필요한지 결정
  ↓
6. specialize → mir → 표준 MLIR 텍스트
  ↓
7. mlir_ee 의 transformer 훅으로 in-process lowering
  ↓
8. ORCv2 add_module(module, &cell.tracker)
  ↓ (link error 면 cell.tracker.remove(), 롤백)
9. top-level 식 / 함수의 심볼을 lookup
  ↓
10. 그 함수 호출 → 결과 + 타입
  ↓
11. last_value 갱신, 결과 출력
```

각 단계가 transactional 이다. 실패 시 직전 상태로 깨끗이 롤백된다.

## 9.7.3. Top-level 변수 저장

REPL 과 Jupyter 의 셀 사이 변수 공유. top-level 변수는 세션 내내 살아 있어야 한다.

```nok
[1] let x = 42i32;
[2] let y = x + 1;       // 이전 x 사용
```

### 저장 형태

```
untyped (let x = 42)        → {tag:u32, payload:i64} 구조 LLVM global
typed (let x: i32 = 42i32)  → 해당 타입 LLVM global
heap payload                → GC root 등록
                             (llvm.gcroot shadow stack,
                              nokr 의 shadow stack 체인 연결)
```

각 top-level 변수는 LLVM global 로 저장된다. 다음 cell 이 같은 변수를 참조하면 그 global 주소로 찾는다.

### GC root 등록

heap 객체를 가리키는 top-level 변수는 GC 가 root 로 추적해야 한다. [shadow stack](../000-meta/007-glossary.md#shadow-stack) 체인에 이어 둔다.

자세한 메커니즘은 [배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md) 의 shadow stack 을 참조한다.

### 같은 이름의 재할당

```nok
[1] let x = 1;
[2] let x: i32 = 5i32;       // 같은 이름 재할당
```

[`any` 슬롯](../002-types/202-any-slot.md) 에서 같은 스코프 안의 재선언은 ERROR 이다. 하지만 REPL 과 Jupyter 의 셀 사이는 다른 cell 이므로 덮어쓸 수 있다.

기존 global 은 덮어쓴다. 메모리는 그대로 두고 이름 자리만 새 정의를 가리킨다.

이 동작 덕분에 REPL 의 자연스러운 사용 패턴 ("아, 이 변수를 다시 정의하자") 이 지원된다.

## 9.7.4. Undo / 셀 재실행

### Jupyter 의 셀 삭제

```
사용자가 cell N 삭제:
  cell = cells[N]
  cell.tracker.remove()       // ORCv2 가 그 cell 의 심볼 제거
  hir_env.revert(cell.delta)   // HIR 환경 되돌림
  cells.remove(N)
  spec_cache 는 유지            // 다른 cell 이 쓸 수 있음
                                // (수명은 pkg_hash 에 종속, 다음 cell 에서 재사용)
```

### Jupyter 의 같은 id 셀 재실행

```
같은 id 의 cell 을 다시 실행:
  old_cell = cells.find(id)
  old_cell.tracker.remove()
  hir_env.revert(old_cell.delta)
  cells.remove(old_cell)

  // 새 입력으로 일반 실행 흐름
```

이 모델 덕분에 Jupyter 가 사용자에게 일관된 셀 단위 의미를 제공한다.

### Undo 의 의미

REPL 에는 명시적 undo 가 없다. 마지막 cell 을 그냥 덮어쓰거나 새 cell 을 추가해 진행한다.

Jupyter 에서는 셀 삭제와 재실행이 자연스러운 undo 역할을 한다.

## 9.7.5. 에러 모드

### 셀 컴파일 에러

```
parse error / type error / 의미 에러:
  - cell 추가하지 않음.
  - hir_env 변경하지 않음.
  - spec_cache 유지.
  - 사용자에게 에러 표시.
```

완전한 transactional 이다. 에러 이전 상태가 그대로 유지된다.

### 셀 런타임 에러

```
실행 중 panic / throw / 세그폴트:
  - cell 은 추가됨 (정의는 이미 이루어졌으므로).
  - last_value 에 에러 값 저장.
  - 사용자에게 stack trace.
  - 다음 cell 진행 가능.
```

런타임 에러는 부분 효과가 남을 수 있어 완전 롤백이 불가능하다. 사용자가 선택해서 undo 한다 (Jupyter 셀 삭제).

### JIT materialize 실패

```
ORCv2 의 link 단계에서 실패 (alien 심볼 등):
  - cell.tracker.remove() — 그 cell 이 추가한 심볼 모두 제거.
  - cells 에서 그 cell pop.
  - hir_env.revert.
  - 에러 표시.
```

깨끗한 롤백. 컴파일 에러와 같은 transactional 동작이다.

## 9.7.6. 디스크 캐시

### 두 단계 캐시

```
1. 세션 캐시 (in-memory):
   - JIT 결과 (MLIR module + LLVM 모듈) 메모리 보존.
   - 종료 시 폐기.

2. 디스크 캐시 (선택):
   - ~/.cache/nok/<pkg_hash>/<typehash>.o
   - enableObjectCache 훅으로 ORCv2 가 자동 dump.
   - 다음 세션이 같은 (pkg_hash, typehash) 재사용.
```

세션 안에서는 메모리 캐시가 빠르다. 세션 종료 후 재시작 시에는 디스크 캐시가 재컴파일을 막아준다.

### 캐시 무효화

다음 중 하나에 해당하면 캐시가 무효이다.

- `pkg_hash` 변경 (소스 변경).
- `nokc` 버전 변경.
- `abi_version_hash` 변경.

캐시 항목의 메타가 위 세 hash 를 모두 담고 있어 mismatch 시 자동으로 폐기된다.

### 캐시 디렉토리 구조

```
~/.cache/nok/
├── <pkg_hash_1>/
│   ├── meta.json                 # nokc 버전, abi 버전 등
│   ├── <typehash_a>.o            # specialize 결과
│   ├── <typehash_b>.o
│   └── ...
├── <pkg_hash_2>/
│   └── ...
└── ...
```

`nokc clean --jit-cache` 또는 `rm -rf ~/.cache/nok/` 로 사용자가 수동으로 청소할 수 있다.

## 9.7.7. 시각화 / 출력

### REPL 출력

```
> 1 + 2
i32: 3
> [1, 2, 3]
List<i32>: [1, 2, 3]
```

마지막 식의 값과 타입을 자동 출력한다. 표준 라이브러리의 `Display` 또는 `Debug` 트레잇을 쓴다.

### Jupyter 출력

각 cell 의 결과를 Jupyter 의 display protocol 로 노출한다.

- 단순 값: 문자열로.
- 컬렉션: 표 / JSON.
- 이미지 / 그래프: 라이브러리가 `Display::display_jupyter` 같은 hook 을 통해.

자세한 Jupyter 통합은 [Jupyter 통합](908-jupyter-integration.md) 에서 정의한다.

## 9.7.8. 입력 단위: REPL 식별

REPL 은 완성된 식 / 문장 단위로 처리한다.

```
> 1 + 2
i32: 3
> let x =
... 42i32;          (계속 입력)
i32: 42
```

닫히지 않은 식 / 블록이면 `...` 프롬프트로 계속 입력한다.

## 9.7.9. 입력 단위: Jupyter 셀

Jupyter 의 한 셀은 여러 줄 입력이며 한 번에 모두 처리한다.

```
[Cell 1]:
let x = 42i32;
let y = x * 2i32;
y                       # 마지막 식 = cell 결과
```

cell 의 마지막 식이 cell 의 결과이다. 그 외는 사이드 이펙트 (정의나 호출) 로 처리한다.

## 9.7.10. 다음

[Jupyter 통합](908-jupyter-integration.md) 에서 Jupyter 커널 통합 (로컬 ZMQ + 브라우저 WASM) 을 정의한다.
