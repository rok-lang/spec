# 9.7. JIT 셀 실행 흐름과 상태 관리

이 절에서는 [JIT 엔진](906-jit-engine.md) 의 RokJitSession 이 사용자 입력 (cell) 을 어떻게 처리하는지 자세히 정의합니다.

> [!NOTE]
> **재실행 (replay) 구현.** 규범은 셀 세션의 의미 ([4.16.10](../004-functions/416-entry-point.md#41610-jit--repl-과의-관계): 셀을 차례로 이어 붙인 암묵 `main.rok`) 이며, JIT 는 그것을 구현하는 한 방법입니다. 적합한 구현은 JIT 없이, 셀마다 지금까지의 세션 전체를 하나의 익명 `main.rok` 로 다시 컴파일해 처음부터 다시 실행하고 새 셀의 출력만 보여 줄 수도 있습니다. 현재 `rokc repl` 이 이 방식입니다. 이 경우 앞 셀의 부수 효과 (I/O, 시각, 난수) 가 매번 되풀이되고, 아래 9.7.6 캐시와 materialize 단계는 없습니다.

## 9.7.1. Cell 의 종류

REPL 의 한 줄과 Jupyter 의 한 셀은 그 내용에 따라 처리 방식이 달라집니다.

```
1. expression                → evaluate + store last_value + print
2. statement (let / call)    → execute (return value is unit)
3. definition (fn / struct)  → add to hir_env
4. import (use / mod)        → update visibility table of hir_env
5. meta / attribute          → handle as appropriate
```

> expression: 표현식. evaluate: 평가. store last_value: last_value 저장. print: 출력. statement: 문장. let / call: let / 함수 호출. execute: 실행. return value is unit: 반환값은 unit. definition: 정의. add to hir_env: hir_env 에 추가. update visibility table of hir_env: hir_env 의 가시 표 갱신. meta / attribute: 메타 / 속성. handle as appropriate: 적절한 처리.

다섯 가지 모두 같은 cell 처리 흐름을 따르며, 마지막 출력만 달라집니다.

> [!IMPORTANT]
> 언어 의미론의 관점에서 셀의 연속은 암묵 `main.rok` 의 끝에 top-level 문과 항목을 덧붙이는 것과 같습니다 ([4.16.10](../004-functions/416-entry-point.md#41610-jit--repl-과의-관계)). 셀의 `let` 은 암묵 `main` 의 지역 변수이므로 이후 셀의 문에서는 보이지만, 셀에서 선언한 `fn` 본체에서는 보이지 않습니다. 아래 9.7.3 의 LLVM global 저장은 셀 경계를 넘어 지역 변수를 유지하기 위한 구현 방식이며, 이 가시성 규칙을 바꾸지 않습니다. 셀 최상위의 같은 이름 재 `let` (9.7.3), 셀 삭제와 재실행 (9.7.4), 마지막 식 값의 자동 표시 (9.7.7) 는 세션 조작으로서 대화형 표면에만 있습니다.

## 9.7.2. 실행 흐름: 자세히

```
rokc receives input:
  ↓
1. parse  →  AST
  ↓ (on parse error: show immediately, do not add cell)
2. lower to HIR  →  HIR + delta
  ↓ (on type error: show, do not add cell)
3. cells.push(new_cell)
4. hir_env.apply(delta)
  ↓
5. decide if new item / expr needs specialize
  ↓
6. specialize → mir → standard MLIR text
  ↓
7. in-process lowering via mlir_ee transformer hook
  ↓
8. ORCv2 add_module(module, &cell.tracker)
  ↓ (on link error: cell.tracker.remove(), rollback)
9. lookup symbol of top-level expr / function
  ↓
10. call that function → result + type
  ↓
11. update last_value, print result
```

> rokc receives input: rokc 입력 받음. on parse error: show immediately, do not add cell: parse error 면 즉시 표시, cell 추가하지 않습니다. on type error: show, do not add cell: type error 면 표시, cell 추가하지 않습니다. decide if new item / expr needs specialize: 새 항목 / 식에 specialize 가 필요한지 결정. standard MLIR text: 표준 MLIR 텍스트. in-process lowering via mlir_ee transformer hook: mlir_ee 의 transformer 훅으로 in-process lowering. on link error: rollback: link error 면 롤백. lookup symbol of top-level expr / function: top-level 식 / 함수의 심볼을 lookup. call that function: 그 함수 호출. result + type: 결과 + 타입. update last_value, print result: last_value 갱신, 결과 출력.

각 단계는 transactional 하게 동작하며, 실패하면 직전 상태로 깨끗하게 롤백됩니다.

## 9.7.3. Top-level 변수 저장

REPL 과 Jupyter 의 셀 사이에서 변수가 공유되어야 하므로, top-level 변수는 세션이 유지되는 동안 계속 살아 있어야 합니다.

```rok
[1] let x = 42i32;
[2] let y = x + 1;       // 이전 x 사용
```

### 저장 형태

```
untyped (let x = 42)        → LLVM global of {tag:u32, payload:i64}
typed (let x: i32 = 42i32)  → LLVM global of that type
heap payload                → register GC root
                             (compiler-emitted shadow stack frame,
                              linked into rokr shadow stack chain)
```

> LLVM global of struct: 구조 LLVM global. LLVM global of that type: 해당 타입 LLVM global. register GC root: GC root 등록. linked into rokr shadow stack chain: rokr 의 shadow stack 체인 연결.

각 top-level 변수는 LLVM global 로 저장되며, 다음 cell 이 같은 변수를 참조할 때 그 global 주소를 통해 찾아갑니다.

### GC root 등록

heap 객체를 가리키는 top-level 변수는 GC 가 root 로 추적해야 하므로, [shadow stack](../000-meta/007-glossary.md#shadow-stack) 체인에 이어 둡니다.

자세한 메커니즘은 [배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md) 의 shadow stack 부분을 참조합니다.

### 같은 이름의 재선언

```rok
[1] let x = 1i32;            // x: i32
[2] let x = "hi";            // 셀 최상위의 재선언 — 새 바인딩 (x: str)
```

[변수 선언](../005-modules/501-variable-declaration.md#514-같은-스코프-내-재선언-금지) 에서 같은 스코프 안의 재선언은 ERROR 로 처리됩니다. 그러나 셀을 고쳐 다시 실행하는 흐름을 지원하기 위해, REPL 과 Jupyter 셀의 최상위에서는 같은 이름을 다시 `let` 으로 선언할 수 있습니다. 재선언은 새 바인딩을 만들며, 새 바인딩의 타입은 이전 바인딩과 달라도 됩니다. 이후 셀은 새 바인딩을 봅니다. 이 예외는 셀 최상위에만 적용되며, 셀 안의 블록이나 일반 소스 파일에는 적용되지 않습니다.

이때 기존 global 은 덮어쓰며, 메모리는 그대로 둔 채 이름 자리만 새 정의를 가리키도록 바꿉니다.

이러한 동작 덕분에 "아, 이 변수를 다시 정의해야겠다" 와 같은 REPL 의 자연스러운 사용 패턴이 매끄럽게 지원됩니다.

## 9.7.4. Undo / 셀 재실행

### Jupyter 의 셀 삭제

```
사용자가 cell N 삭제.
  cell = cells[N]
  cell.tracker.remove()       // ORCv2 가 그 cell 의 심볼 제거
  hir_env.revert(cell.delta)   // HIR 환경 되돌림
  cells.remove(N)
  spec_cache 는 유지            // 다른 cell 이 쓸 수 있음
                                // (수명은 pkg_hash 에 종속, 다음 cell 에서 재사용)
```

### Jupyter 의 같은 id 셀 재실행

```
같은 id 의 cell 을 다시 실행.
  old_cell = cells.find(id)
  old_cell.tracker.remove()
  hir_env.revert(old_cell.delta)
  cells.remove(old_cell)

  // 새 입력으로 일반 실행 흐름
```

이러한 모델 덕분에 Jupyter 가 사용자에게 일관된 셀 단위 의미를 제공할 수 있습니다.

### Undo 의 의미

REPL 에는 명시적인 undo 가 없으며, 마지막 cell 을 그대로 덮어쓰거나 새 cell 을 추가하면서 작업을 진행합니다.

Jupyter 에서는 셀 삭제와 재실행이 자연스럽게 undo 의 역할을 대신해 줍니다.

## 9.7.5. 에러 모드

### 셀 컴파일 에러

```
parse error / type error / 의미 에러.
  - cell 추가하지 않음.
  - hir_env 변경하지 않음.
  - spec_cache 유지.
  - 사용자에게 에러 표시.
```

이 단계는 완전한 transactional 동작을 보장하므로, 에러가 발생하기 직전의 상태가 그대로 유지됩니다.

### 셀 런타임 에러

```
실행 중 panic / throw / 세그폴트.
  - cell 은 추가됨 (정의는 이미 이루어졌으므로).
  - last_value 에 에러 값 저장.
  - 사용자에게 stack trace.
  - 다음 cell 진행 가능.
```

런타임 에러는 부분적인 효과가 이미 남아 있을 수 있어 완전한 롤백이 불가능하므로, 사용자가 직접 선택해 undo (Jupyter 셀 삭제) 를 수행하게 됩니다.

재실행 구현에서는 실패한 셀을 남겨 두면 이후 모든 재실행이 같은 지점에서 실패하므로, 런타임 에러가 난 셀도 **버립니다** (컴파일 에러와 같이 세션에 추가하지 않음). 이미 출력된 부분 효과는 되돌리지 않습니다.

### JIT materialize 실패

```
ORCv2 의 link 단계에서 실패 (alien 심볼 등):
  - cell.tracker.remove() — 그 cell 이 추가한 심볼 모두 제거.
  - cells 에서 그 cell pop.
  - hir_env.revert.
  - 에러 표시.
```

이 경우에도 깨끗하게 롤백되며, 컴파일 에러와 동일한 transactional 동작을 따릅니다.

## 9.7.6. 디스크 캐시

### 두 단계 캐시

```
1. session cache (in-memory):
   - keep JIT results (MLIR module + LLVM module) in memory.
   - discard on exit.

2. disk cache (optional):
   - ~/.cache/rok/<pkg_hash>/<typehash>.o
   - ORCv2 auto-dumps via enableObjectCache hook.
   - next session reuses same (pkg_hash, typehash).
```

> session cache: 세션 캐시. keep JIT results in memory: JIT 결과 메모리 보존. MLIR module + LLVM module: MLIR module + LLVM 모듈. discard on exit: 종료 시 폐기. disk cache: 디스크 캐시. optional: 선택. ORCv2 auto-dumps via enableObjectCache hook: enableObjectCache 훅으로 ORCv2 가 자동 dump. next session reuses: 다음 세션이 재사용.

세션 안에서는 메모리 캐시가 빠르게 동작하며, 세션을 종료한 뒤 다시 시작할 때에는 디스크 캐시가 재컴파일을 차단해 줍니다.

### 캐시 무효화

다음 중 어느 하나에라도 해당하면 캐시는 무효로 처리됩니다.

- `pkg_hash` 변경 (소스 변경).
- `rokc` 버전 변경.
- `abi_version_hash` 변경.

캐시 항목의 메타가 위 세 가지 hash 를 모두 담고 있으므로, mismatch 가 발생하면 자동으로 폐기됩니다.

### 캐시 디렉토리 구조

```
~/.cache/rok/
├── <pkg_hash_1>/
│   ├── meta.json                 # rokc version, abi version, etc.
│   ├── <typehash_a>.o            # specialize result
│   ├── <typehash_b>.o
│   └── ...
├── <pkg_hash_2>/
│   └── ...
└── ...
```

> rokc version, abi version, etc.: rokc 버전, abi 버전 등. specialize result: specialize 결과.

`rokc clean --jit-cache` 명령이나 `rm -rf ~/.cache/rok/` 를 사용해 사용자가 수동으로 정리할 수 있습니다.

## 9.7.7. 시각화 / 출력

### REPL 출력

```
> 1 + 2
i32: 3
> [1, 2, 3]
List<i32>: [1, 2, 3]
```

마지막 식의 값과 타입을 자동으로 출력하며, 표준 라이브러리의 `Display` 또는 `Debug` 트레잇을 사용합니다. `let` 셀은 묶은 값을 같은 `타입: 값` 형태로 출력합니다 (값을 출력할 수 없으면 타입만, 단위 타입이면 출력 없음).

```
> let xs = [1, 2, 3];
List<i32>: [1, 2, 3]
```

### Jupyter 출력

각 cell 의 결과를 Jupyter 의 display protocol 을 통해 노출합니다.

- 단순 값은 문자열로 표현됩니다.
- 컬렉션은 표 또는 JSON 으로 표현됩니다.
- 이미지나 그래프는 라이브러리가 `Display::display_jupyter` 와 같은 hook 을 통해 노출합니다.

자세한 Jupyter 통합 내용은 [Jupyter 통합](908-jupyter-integration.md) 에서 정의합니다.

## 9.7.8. 입력 단위: REPL 식별

REPL 은 완성된 식이나 문장 단위로 입력을 처리합니다.

```
> 1 + 2
i32: 3
> let x =
... 42i32;          (continued input)
i32: 42
```

> continued input: 계속 입력.

식이나 블록이 닫히지 않았다면 `...` 프롬프트를 표시하며 입력을 계속 받습니다.

## 9.7.9. 입력 단위: Jupyter 셀

Jupyter 의 한 셀은 여러 줄로 구성된 입력이며, 한 번에 모두 처리됩니다.

```
[Cell 1]:
let x = 42i32;
let y = x * 2i32;
y                       # 마지막 식 = cell 결과
```

cell 의 마지막 식이 곧 cell 의 결과가 되며, 그 외의 부분은 사이드 이펙트 (정의나 호출) 로 처리됩니다.

## 9.7.10. 다음

[Jupyter 통합](908-jupyter-integration.md) 에서는 Jupyter 커널 통합 (로컬 ZMQ 와 브라우저 WASM) 을 정의합니다.
