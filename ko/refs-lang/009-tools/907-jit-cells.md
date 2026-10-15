# 9.7. JIT 셀 실행 흐름과 상태 관리

이 절에서는 [JIT 엔진](906-jit-engine.md) 이 사용자 입력 (cell) 을 어떻게 처리하는지 자세히 정의합니다.

## 9.7.0. 셀 세션의 규칙

다음은 셀 세션이 지키는 규칙입니다. 구현은 [9.6 JIT 엔진](906-jit-engine.md) 입니다.

1. **한 번만 실행.** 각 셀의 코드는 정확히 한 번 실행됩니다. 앞 셀의 부수 효과 (I/O, 시각, 난수) 는 되풀이되지 않습니다.
2. **상태 유지.** 셀 최상위의 `let` 바인딩, `static` 의 상태, `any` 값 (`any Tr` 포함) 은 이후 셀로 이어집니다. 바인딩은 대입할 수 있고, 같은 이름의 재 `let` 은 새 바인딩으로 가립니다 (9.7.3).
3. **항목 유지와 재정의.** `fn` / `struct` / `impl` 등 항목은 유지됩니다. 같은 항목을 다시 정의하면 다음 셀부터 새 정의가 쓰입니다. 재정의 전에 컴파일된 코드도 이후 셀이 다시 닿으면 새 정의를 씁니다 (재정의된 `f` 를 부르는 `g` 는 새 `f` 를 부름). `static` 을 다시 정의하면 그 상태는 새로 시작합니다. 타입이 재정의된 바인딩은 더 이상 쓸 수 없으며, 쓰려 하면 그 사실을 note 로 알려 줍니다.
4. **`any` 의 정체성.** `any` 의 타입 tag 는 한 세션의 모든 셀에서 같습니다. 앞 셀에서 넣은 `any` 값을 뒤 셀에서 `is` / 패턴으로 꺼낼 수 있습니다.
5. **실패한 셀.** panic 하거나 프로세스를 끝내거나 죽은 셀은 바인딩에 부분 효과를 남기지 않습니다. 세션은 그 셀 이전의 상태로 계속됩니다. 이미 일어난 외부 효과 (I/O 등) 는 되돌리지 않습니다.
6. **표시.** 식 셀은 값을 `타입: 값` (Debug) 으로 출력하고, `Debug` 가 없으면 타입만 출력합니다 (9.7.7).

다음은 구현이 정합니다 (implementation-defined).

- 최상위에서 이름을 적을 수 없는 타입 (클로저, 비공개 타입 등) 의 바인딩. 구현은 그런 바인딩을 만들 때 거부할 수 있습니다.
- 프로세스 수준의 효과: 셀 안의 `exit()`, 실패한 셀 뒤에 남은 앞 셀의 스레드 등. 현재 구현에서 `exit()` 를 부른 셀은 실패한 셀로 다룹니다.

> [!NOTE]
> **현재 구현.** `rokc repl` (과 `rok`, `rok repl`, Jupyter 커널) 은 9.6 의 JIT host 를 씁니다. 환경 변수 `ROK_REPL_TIME=1` 은 셀마다 단계별 시간과 새로 컴파일한 / 재사용한 함수 수를 출력합니다. 이름을 적을 수 없는 타입의 바인딩은 거부하고 `--replay` 를 안내합니다.
>
> `--replay` 모드 (9.6.6) 는 셀마다 지금까지의 세션 전체를 하나의 익명 `main.rok` 로 다시 컴파일해 버리는 자식 프로세스에서 처음부터 실행하고, 새 셀의 출력만 보여 줍니다. 클로저 같은 바인딩도 유지할 수 있지만 규칙 1 을 지키지 않습니다 (앞 셀의 부수 효과가 되풀이됨).

## 9.7.1. Cell 의 종류

REPL 의 한 줄과 Jupyter 의 한 셀은 그 내용에 따라 처리 방식이 달라집니다.

```
1. expression                → evaluate + store last_value + print
2. statement (let / call)    → execute (return value is unit)
3. definition (fn / struct)  → add to the session's items
4. import (use / mod)        → update the session's visible names
5. meta / attribute          → handle as appropriate
```

> expression: 표현식. evaluate: 평가. store last_value: last_value 저장. print: 출력. statement: 문장. let / call: let / 함수 호출. execute: 실행. return value is unit: 반환값은 unit. definition: 정의. add to the session's items: 세션의 항목에 추가. update the session's visible names: 세션의 가시 이름 갱신. meta / attribute: 메타 / 속성. handle as appropriate: 적절한 처리.

다섯 가지 모두 같은 cell 처리 흐름을 따르며, 마지막 출력만 달라집니다.

> [!IMPORTANT]
> 언어 의미론의 관점에서 셀의 연속은 암묵 `main.rok` 의 끝에 top-level 문과 항목을 덧붙이는 것과 같습니다 ([4.16.10](../004-functions/416-entry-point.md#41610-jit--repl-과의-관계)). 셀의 `let` 은 암묵 `main` 의 지역 변수이므로 이후 셀의 문에서는 보이지만, 셀에서 선언한 `fn` 본체에서는 보이지 않습니다. 아래 9.7.3 의 LLVM global 저장은 셀 경계를 넘어 지역 변수를 유지하기 위한 구현 방식이며, 이 가시성 규칙을 바꾸지 않습니다. 셀 최상위의 같은 이름 재 `let` (9.7.3), 셀 재실행 (9.7.4), 마지막 식 값의 자동 표시 (9.7.7) 는 세션 조작으로서 대화형 표면에만 있습니다.

## 9.7.2. 실행 흐름: 자세히

```
1. parse → AST                      (에러: 표시, 셀을 버림)
2. HIR 로 낮춤, 타입 검사           (에러: 표시, 셀을 버림)
3. MIR → MLIR → LLVM IR 텍스트      (재사용 / 재정의 처리, 9.6.3 / 9.6.4)
4. host: 스냅샷을 fork 한 프로세스에서 JIT 컴파일, 셀 진입 함수 실행
5. 성공: 새 스냅샷, 바인딩과 항목을 세션에 반영, 결과 출력
   실패: 이전 스냅샷으로 계속, 에러 표시
```

각 단계는 transactional 하게 동작하며, 실패하면 직전 상태로 롤백됩니다.

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

### Jupyter 의 셀 재실행

Jupyter 에서 셀을 고쳐 다시 실행하면 새 셀로 실행됩니다. 그 셀이 다시 정의한 항목과 바인딩은 이후 셀부터 새 것이 쓰입니다 (9.7.0 규칙 2, 3). 앞서 실행된 셀의 효과를 지우지는 않습니다.

### Undo 의 의미

REPL 에는 명시적인 undo 가 없으며, 마지막 cell 을 그대로 덮어쓰거나 새 cell 을 추가하면서 작업을 진행합니다.

Jupyter 에서도 셀을 지우거나 다시 실행해 앞 셀의 효과를 되돌리지는 않습니다. 처음 상태로 돌아가려면 세션을 다시 시작합니다 (REPL 의 `:reset`).

## 9.7.5. 에러 모드

### 셀 컴파일 에러

```
parse error / type error / 의미 에러.
  - cell 추가하지 않음.
  - 세션의 바인딩과 항목은 그대로.
  - 이미 컴파일된 함수의 재사용 정보는 유지.
  - 사용자에게 에러 표시.
```

이 단계는 완전한 transactional 동작을 보장하므로, 에러가 발생하기 직전의 상태가 그대로 유지됩니다.

### 셀 런타임 에러

```
실행 중 panic / 잡히지 않은 throw / 프로세스 종료 / 세그폴트.
  - cell 은 추가하지 않음 — 바인딩과 항목은 그 셀 이전 상태 그대로.
  - 사용자에게 에러 표시.
  - 다음 cell 진행 가능.
```

바인딩과 항목은 셀 이전 상태로 돌아가지만 (9.7.0 규칙 5), 이미 일어난 외부 효과 (출력, 파일 쓰기 등) 는 되돌리지 않습니다. 실패한 셀을 세션에 남기지 않으므로, 고친 셀을 다시 입력하면 됩니다.

### JIT 실패

host 가 셀의 IR 을 JIT 컴파일하거나 링크하지 못하면 (알 수 없는 `extern` 심볼 등) 셀 런타임 에러와 같이 다룹니다. 셀을 실행한 fork 프로세스가 끝나고 이전 스냅샷이 이어받습니다.

## 9.7.6. 캐시

한 세션 안에서는 host 가 이미 컴파일한 함수를 재사용합니다 (9.6.4). 세션을 넘는 디스크 캐시는 없습니다.

## 9.7.7. 시각화 / 출력

### REPL 출력

```
> 1 + 2
i32: 3
> [1, 2, 3]
List<i32>: [1, 2, 3]
```

마지막 식의 값과 타입을 `타입: 값` 으로 자동 출력하며, 값은 `Debug` 로 형식화합니다. `Debug` 를 구현하지 않은 타입이면 타입만 출력합니다. `let` 셀은 묶은 값을 같은 `타입: 값` 형태로 출력합니다 (값을 출력할 수 없으면 타입만, 단위 타입이면 출력 없음).

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
