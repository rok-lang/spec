# 9. 실행 / 도구 개요

이 블록은 Rok의 컴파일러 `rokc`, 프런트엔드 `rok`, 실행 환경(REPL / Jupyter 커널)을 정의합니다. [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md)은 두 엔진([JIT](../000-meta/007-glossary.md#jit-just-in-time) / [AOT](../000-meta/007-glossary.md#aot-ahead-of-time))과 세 사용자 표면(REPL / Jupyter / AOT 산출물 직접 실행)을 정합니다. 이 블록은 그것을 어떻게 구현하는지 다룹니다.

## 목차

- [9.1. `rokc` CLI 인터페이스](901-rokc-cli.md)
- [9.2. AOT 빌드 파이프라인](902-aot-pipeline.md)
- [9.3. 패키지 아카이브(.a) 포맷](903-package-archive.md)
- [9.4. ROKHIR 포맷](904-rokhir-bytecode.md)
- [9.5. 심볼 맹글링 스킴](905-symbol-mangling.md)
- [9.6. JIT 엔진 (RokJitSession)](906-jit-engine.md)
- [9.7. JIT 셀 실행 흐름과 상태 관리](907-jit-cells.md)
- [9.8. Jupyter 커널 통합](908-jupyter-integration.md)
- [9.9. 디버그 / 진단 도구](909-debug-tools.md)
- [9.10. `rok`: 패키지 / 빌드 프런트엔드](910-rok-frontend.md)

## 도구 표면

도구 표면의 구성은 다음과 같습니다.

```
rok  (front-end, cargo-like)
  │   new / init, add / remove / update / deps (rok.rokon, rok.lock), env, toolchain
  │   build / check / run / test / bench / emit / fmt / lint / doc / clean / repl / lsp / demangle
  │        └── resolve deps, then call rokc ... --extern <dep roots>
  ▼
rokc  (compiler)
  │
  ├── build / run
  │   ├── rokc build      : AOT compile - .rok source to .a / executable
  │   ├── rokc run        : AOT compile then run immediately
  │   ├── rokc test       : collect + build the #[test] harness (rok test runs it)
  │   ├── rokc bench      : collect + build the #[bench] harness (rok bench runs it)
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
  │   └── rokc jupyter    : Jupyter kernel (REPL cell session + ZMQ)
  │
  ├── debug / diagnostic
  │   ├── rokc demangle   : symbol demangle
  │   ├── rokc rokhir-text  : ROKHIR interface text dump
  │   └── rokc inspect    : package inspect
  │
  └── rokc version / help

rokls  (= rok lsp, for editors)
librokr.a  (runtime, rokr)
```

toolchain은 컴파일러 `rokc`(rustc에 해당), cargo 식 프런트엔드 `rok`, 에디터용 LSP 실행 파일 `rokls`, 런타임 `librokr.a`로 이루어집니다. `rok`은 매니페스트 `rok.rokon` / `rok.lock`, 의존성, 환경, toolchain 고정을 맡습니다. 빌드와 도구 명령은 해소한 의존 패키지를 `--extern`으로 붙여 `rokc`에 넘깁니다([9.10](910-rok-frontend.md)). 별도 도구(`rok-fmt` 등)는 두지 않습니다.

## 두 엔진의 구현

[실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md)에서 정한 두 엔진은 다음과 같이 나뉩니다.

```
JIT engine  ──  used by rokc repl / jupyter
AOT engine  ──  used by rokc build / run / test
```

두 엔진은 같은 컴파일 파이프라인(rok-hir → rok-mir → 표준 MLIR)을 거치고, 마지막 중간 표현 이후에만 갈라집니다. 자세한 내용은 [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md)과 [컴파일 파이프라인](../000-meta/002-pipeline.md)에 있습니다.

## 세 사용자 표면의 구현

세 사용자 표면과 각 표면을 구현하는 명령은 다음과 같습니다.

```
surface              command
────────             ──────
REPL                 rokc repl
Jupyter              rokc jupyter        (invoked by Jupyter, not directly)
AOT direct exec      rokc build → run artifact
```

각 표면의 구현은 이어지는 절에서 다룹니다.

## 패키지 빌드 산출물

[소스 단위](../000-meta/003-source-units.md)의 라이브러리 패키지를 컴파일하면 `libfoo.a`와 인터페이스 `foo.rokhir`이 나옵니다.

```
foo/
├── src/
│   ├── lib.rok              <- input
│   └── ...
       v rokc build
foo/target/debug/
├── libfoo.a                 <- artifact (static archive)
├── foo.rokhir                 <- interface (ROKHIR)
├── build/                   <- intermediates (foo.o, ...)
└── ...
```

프로필별 레이아웃은 [9.2.15](902-aot-pipeline.md#9215-산출물)가 정의합니다. [아카이브](../000-meta/007-glossary.md#archive) 형식은 [패키지 아카이브](903-package-archive.md)가 정의합니다.

## ROKHIR: 제네릭 함수의 휴대용 IR

제네릭 함수는 호출 지점에서 [단형화](../000-meta/007-glossary.md#monomorphization-단형화)해야 하므로 바이너리 코드만으로는 부족합니다. 그래서 라이브러리 옆에 인터페이스 파일 ROKHIR을 둡니다. ROKHIR에는 구체 함수의 본문을 잘라 낸 패키지 소스가 들어갑니다. 단형화가 AST를 입력으로 쓰기 때문입니다.

자세한 형식은 [ROKHIR 포맷](904-rokhir-bytecode.md)에서 정의합니다.

## 단일 의존성: `librokr.a`

빌드 산출물은 사용자 패키지의 `libfoo.a`, 런타임 `librokr.a`, 표준 라이브러리 `librok_*.a`입니다. 외부 OS 라이브러리 외에는 추가 의존성이 없습니다.

자세한 빌드 흐름은 [AOT 빌드 파이프라인](902-aot-pipeline.md)에서 정의합니다.
