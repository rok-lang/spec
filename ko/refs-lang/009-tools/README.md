# 9. 실행 / 툴 개요

본 블록에서는 rok 의 컴파일러 `rokc`, front-end `rok`, 그리고 실행 환경 (REPL / Jupyter 커널) 사양을 정의하고 있습니다. [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 에서 정의한 두 엔진 ([JIT](../000-meta/007-glossary.md#jit-just-in-time) / [AOT](../000-meta/007-glossary.md#aot-ahead-of-time)) 과 세 사용자 표면 (REPL / Jupyter / AOT 산출물 직접 실행) 을 어떻게 구현하는지 다루고 있습니다.

## 목차

- [9.1. `rokc` CLI 인터페이스](901-rokc-cli.md)
- [9.2. AOT 빌드 파이프라인](902-aot-pipeline.md)
- [9.3. 패키지 아카이브 (.a) 포맷](903-package-archive.md)
- [9.4. RHIR bytecode 포맷](904-rhir-bytecode.md)
- [9.5. 심볼 맹글링 스킴](905-symbol-mangling.md)
- [9.6. JIT 엔진 (RokJitSession)](906-jit-engine.md)
- [9.7. JIT 셀 실행 흐름과 상태 관리](907-jit-cells.md)
- [9.8. Jupyter 커널 통합](908-jupyter-integration.md)
- [9.9. 디버그 / 진단 도구](909-debug-tools.md)
- [9.10. `rok` — 패키지 / 빌드 front-end](910-rok-frontend.md)

## 도구 표면

도구 표면은 다음과 같이 구성되어 있습니다.

```
rok  (front-end, cargo-like)
  │   new / init, add / remove / update / deps (Rok.toml, Rok.lock), env, toolchain
  │   build / check / run / test / bench / emit / fmt / lint / doc / clean / repl / lsp / demangle
  │        └── resolve deps, then call rokc ... --extern <dep roots>
  ▼
rokc  (compiler)
  │
  ├── build / run
  │   ├── rokc build      : AOT compile — .rok source to .a / executable
  │   ├── rokc run        : AOT compile then run immediately
  │   ├── rokc test       : collect + run test functions (#[test])
  │   ├── rokc bench      : collect + run benchmarks (#[bench])
  │   ├── rokc check      : type / diagnostics only (no native build)
  │   └── rokc clean      : remove build artifacts
  │
  ├── tools
  │   ├── rokc fmt        : formatter
  │   ├── rokc lint       : linter
  │   ├── rokc doc        : API documentation
  │   └── rokc lsp        : LSP server (IDE integration)
  │
  ├── interactive
  │   ├── rokc repl       : terminal REPL (each cell runs once)
  │   └── rokc jupyter    : Jupyter kernel (JIT engine + ZMQ wire)      [planned]
  │
  ├── debug / diagnostic
  │   ├── rokc demangle   : symbol demangle
  │   ├── rokc rhir-text  : RHIR bytecode text dump                     [planned]
  │   └── rokc inspect    : package inspect                             [planned]
  │
  └── rokc version / help

rokls  (= rok lsp, for editors)
librokr.a  (runtime, rokr)
```

> front-end, cargo-like: cargo 식 front-end. resolve deps, then call rokc: 의존성을 해소한 뒤 rokc 를 호출. dep roots: 의존 패키지 루트. compiler: 컴파일러. build / run: 빌드 / 실행. tools: 도구. interactive: 인터랙티브. debug / diagnostic: 디버그 / 진단. AOT compile: AOT 컴파일. compile then run immediately: AOT 컴파일 후 즉시 실행. collect + run: 수집 + 실행. no native build: native 빌드 없이. remove build artifacts: 빌드 산출물 제거. formatter: 포매터. linter: 린터. API documentation: API 문서. LSP server: LSP 서버. terminal REPL: 터미널 REPL. each cell runs once: 각 셀을 한 번만 실행. Jupyter kernel: Jupyter 커널. symbol demangle: 심볼 demangle. RHIR bytecode text dump: RHIR bytecode 텍스트 출력. package inspect: 패키지 검사. for editors: 편집기용. runtime: 런타임. planned: 계획 — 아직 제공하지 않음.

toolchain 은 컴파일러 `rokc` (rustc 에 해당), cargo 식 front-end `rok`, 편집기용 LSP 실행 파일 `rokls`, 런타임 `librokr.a` 로 이루어집니다. `rok` 는 매니페스트 `Rok.toml` / `Rok.lock`, 의존성, 환경, toolchain 고정을 맡고, 빌드와 도구 명령은 해소한 의존 패키지를 `--extern` 으로 붙여 `rokc` 에 위임합니다 ([9.10](910-rok-frontend.md)). 별도 도구 (`rok-fmt` 등) 는 두지 않습니다.

## 두 엔진의 구현

[실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 에서 정한 두 엔진은 다음과 같이 구분되어 있습니다.

```
JIT engine  ──  used by rokc repl / jupyter
AOT engine  ──  used by rokc build / run / test
```

> JIT engine: JIT 엔진. AOT engine: AOT 엔진. used by: 사용.

두 엔진은 같은 컴파일 파이프라인 (rok-hir → rok-mir → 표준 MLIR) 을 거치고 있으며, 마지막 중간 표현 이후에만 갈라지고 있습니다. 자세한 내용은 [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 과 [컴파일 파이프라인](../000-meta/002-pipeline.md) 에서 다루고 있습니다.

## 세 사용자 표면의 구현

세 사용자 표면과 그 구현 명령은 다음과 같이 정리되어 있습니다.

```
surface              command
────────             ──────
REPL                 rokc repl
Jupyter (local)      rokc jupyter        (invoked by Jupyter, not directly)
Jupyter (WASM)       separate wasm32 build (runs inside browser)
AOT direct exec      rokc build → run artifact
```

> surface: 표면. command: 구현 명령. local: 로컬. invoked by Jupyter, not directly: 사용자가 직접 실행하지 않고 Jupyter 가 호출. separate wasm32 build: 별도 wasm32 빌드. runs inside browser: 브라우저 안에서 작동. AOT direct exec: AOT 직접 실행. run artifact: 산출물 실행.

각 표면의 구현은 후속 문서에서 다루고 있습니다.

## 패키지 빌드 산출물

[소스 단위](../000-meta/003-source-units.md) 의 패키지가 `libfoo.a` 한 개로 컴파일되고 있습니다.

```
foo/
├── lib.rok                  <- input
└── ...
       v rokc build
foo/target/debug/
├── libfoo.a                 <- artifact (static archive)
├── build/                   <- intermediates (foo.o, ...)
└── ...
```

> input: 입력. artifact (static archive): 산출물 (정적 archive). intermediates: 중간 산출물. profile 별 레이아웃은 [9.2.15](902-aot-pipeline.md#9215-산출물) 를 참조합니다.

[archive](../000-meta/007-glossary.md#archive) 안의 구체적 형식은 [패키지 아카이브](903-package-archive.md) 에서 정의하고 있습니다.

## RHIR: 제네릭 함수의 휴대용 IR

제네릭 함수는 호출 지점에서 [단형화](../000-meta/007-glossary.md#monomorphization-단형화) 해야 하므로 바이너리 코드만으로는 부족합니다. 따라서 직렬화된 IR 인 RHIR (rok [HIR](../000-meta/007-glossary.md#hir-high-level-ir--mir-mid-level-ir)) bytecode 를 archive 안에 함께 포함하고 있습니다.

자세한 형식은 [RHIR bytecode 포맷](904-rhir-bytecode.md) 에서 정의하고 있습니다.

## 단일 의존성: `librokr.a`

빌드 산출물은 사용자 패키지의 `libfoo.a`, 런타임 `librokr.a`, 표준 라이브러리 `librok_*.a` 로 구성되어 있습니다. 외부 OS 라이브러리 외에는 추가 의존성이 없습니다.

자세한 빌드 흐름은 [AOT 빌드 파이프라인](902-aot-pipeline.md) 에서 정의하고 있습니다.
