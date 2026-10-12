# 9. 실행 / 툴 개요

본 블록에서는 rok 의 컴파일러 도구 (rok) 와 실행 환경 (REPL / Jupyter 커널) 사양을 정의하고 있습니다. [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 에서 정의한 두 엔진 ([JIT](../000-meta/007-glossary.md#jit-just-in-time) / [AOT](../000-meta/007-glossary.md#aot-ahead-of-time)) 과 세 사용자 표면 (REPL / Jupyter / AOT 산출물 직접 실행) 을 어떻게 구현하는지 다루고 있습니다.

## 목차

- [9.1. `rok` CLI 인터페이스](901-rok-cli.md)
- [9.2. AOT 빌드 파이프라인](902-aot-pipeline.md)
- [9.3. 패키지 아카이브 (.a) 포맷](903-package-archive.md)
- [9.4. RHIR bytecode 포맷](904-rhir-bytecode.md)
- [9.5. 심볼 맹글링 스킴](905-symbol-mangling.md)
- [9.6. JIT 엔진 (RokJitSession)](906-jit-engine.md)
- [9.7. JIT 셀 실행 흐름과 상태 관리](907-jit-cells.md)
- [9.8. Jupyter 커널 통합](908-jupyter-integration.md)
- [9.9. 디버그 / 진단 도구](909-debug-tools.md)

## 도구 표면

도구 표면은 다음과 같이 구성되어 있습니다.

```
rok  (single binary)                     rokls  (= rok lsp, for editors)    [planned]
  │
  ├── package
  │   ├── rok new / init  : create a package
  │   ├── rok add         : add a dependency (path / git)               [planned]
  │   ├── rok remove      : remove a dependency                         [planned]
  │   ├── rok update      : refresh rok.lock                            [planned]
  │   └── rok clean       : remove build artifacts
  │
  ├── build / run
  │   ├── rok build       : AOT compile — .rok source to .a / executable
  │   ├── rok run         : AOT compile then run immediately
  │   ├── rok test        : collect + run test functions (#[test])
  │   ├── rok bench       : collect + run benchmarks (#[bench])         [planned]
  │   └── rok check       : type / diagnostics only (no native build)
  │
  ├── tools
  │   ├── rok fmt         : formatter
  │   ├── rok lint        : linter                                      [planned]
  │   ├── rok doc         : API documentation                           [planned]
  │   └── rok lsp         : LSP server (IDE integration)                [planned]
  │
  ├── interactive
  │   ├── rok repl        : terminal REPL (JIT engine)                  [planned]
  │   └── rok jupyter     : Jupyter kernel (JIT engine + ZMQ wire)      [planned]
  │
  ├── debug / diagnostic
  │   ├── rok demangle    : symbol demangle
  │   ├── rok rhir-text   : RHIR bytecode text dump                     [planned]
  │   └── rok inspect     : package inspect                             [planned]
  │
  └── rok version / help
```

> single binary: 단일 바이너리. for editors: 편집기용. package: 패키지. build / run: 빌드 / 실행. tools: 도구. interactive: 인터랙티브. debug / diagnostic: 디버그 / 진단. create a package: 패키지 생성. add / remove a dependency: 의존성 추가 / 제거. refresh rok.lock: lockfile 갱신. remove build artifacts: 빌드 산출물 제거. AOT compile: AOT 컴파일. compile then run immediately: AOT 컴파일 후 즉시 실행. collect + run: 수집 + 실행. no native build: native 빌드 없이. formatter: 포매터. linter: 린터. API documentation: API 문서. LSP server: LSP 서버. terminal REPL: 터미널 REPL. Jupyter kernel: Jupyter 커널. symbol demangle: 심볼 demangle. RHIR bytecode text dump: RHIR bytecode 텍스트 출력. package inspect: 패키지 검사. planned: 계획 — 아직 제공하지 않음.

toolchain 은 `rok` 바이너리 하나이며, 컴파일러, 패키지 매니저, REPL, LSP, 포매터, 린터를 모두 담습니다. 별도 도구 (`rok-fmt` 등) 는 두지 않습니다. 편집기를 위한 `rokls` 만 `rok lsp` 와 같은 독립 바이너리로 둡니다.

## 두 엔진의 구현

[실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 에서 정한 두 엔진은 다음과 같이 구분되어 있습니다.

```
JIT engine  ──  used by rok repl / jupyter
AOT engine  ──  used by rok build / run / test
```

> JIT engine: JIT 엔진. AOT engine: AOT 엔진. used by: 사용.

두 엔진은 같은 컴파일 파이프라인 (rok-hir → rok-mir → 표준 MLIR) 을 거치고 있으며, 마지막 중간 표현 이후에만 갈라지고 있습니다. 자세한 내용은 [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 과 [컴파일 파이프라인](../000-meta/002-pipeline.md) 에서 다루고 있습니다.

## 세 사용자 표면의 구현

세 사용자 표면과 그 구현 명령은 다음과 같이 정리되어 있습니다.

```
surface              command
────────             ──────
REPL                 rok repl
Jupyter (local)      rok jupyter        (invoked by Jupyter, not directly)
Jupyter (WASM)       separate wasm32 build (runs inside browser)
AOT direct exec      rok build → run artifact
```

> surface: 표면. command: 구현 명령. local: 로컬. invoked by Jupyter, not directly: 사용자가 직접 실행하지 않고 Jupyter 가 호출. separate wasm32 build: 별도 wasm32 빌드. runs inside browser: 브라우저 안에서 작동. AOT direct exec: AOT 직접 실행. run artifact: 산출물 실행.

각 표면의 구현은 후속 문서에서 다루고 있습니다.

## 패키지 빌드 산출물

[소스 단위](../000-meta/003-source-units.md) 의 패키지가 `libfoo.a` 한 개로 컴파일되고 있습니다.

```
foo/
├── lib.rok                  <- input
└── ...
       v rok build
foo/target/
├── libfoo.a                 <- artifact (static archive)
├── foo.o
└── ...
```

> input: 입력. artifact (static archive): 산출물 (정적 archive).

[archive](../000-meta/007-glossary.md#archive) 안의 구체적 형식은 [패키지 아카이브](903-package-archive.md) 에서 정의하고 있습니다.

## RHIR: 제네릭 함수의 휴대용 IR

제네릭 함수는 호출 지점에서 [단형화](../000-meta/007-glossary.md#monomorphization-단형화) 해야 하므로 바이너리 코드만으로는 부족합니다. 따라서 직렬화된 IR 인 RHIR (rok [HIR](../000-meta/007-glossary.md#hir-high-level-ir--mir-mid-level-ir)) bytecode 를 archive 안에 함께 포함하고 있습니다.

자세한 형식은 [RHIR bytecode 포맷](904-rhir-bytecode.md) 에서 정의하고 있습니다.

## 단일 의존성: `librokr.a`

빌드 산출물은 사용자 패키지의 `libfoo.a`, 런타임 `librokr.a`, 표준 라이브러리 `librok_*.a` 로 구성되어 있습니다. 외부 OS 라이브러리 외에는 추가 의존성이 없습니다.

자세한 빌드 흐름은 [AOT 빌드 파이프라인](902-aot-pipeline.md) 에서 정의하고 있습니다.
