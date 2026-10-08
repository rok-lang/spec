# 9. 실행 / 툴 개요

본 블록은 nok 의 컴파일러 도구 (nokc) 와 실행 환경 (REPL / Jupyter 커널) 사양을 정의한다. [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 에서 정의한 두 엔진 ([JIT](../000-meta/007-glossary.md#jit-just-in-time) / [AOT](../000-meta/007-glossary.md#aot-ahead-of-time)) 과 세 사용자 표면 (REPL / Jupyter / AOT 산출물 직접 실행) 을 어떻게 구현하는지 다룬다.

## 목차

- [9.1. `nokc` CLI 인터페이스](901-nokc-cli.md)
- [9.2. AOT 빌드 파이프라인](902-aot-pipeline.md)
- [9.3. 패키지 아카이브 (.a) 포맷](903-package-archive.md)
- [9.4. NHIR bytecode 포맷](904-nhir-bytecode.md)
- [9.5. 심볼 맹글링 스킴](905-symbol-mangling.md)
- [9.6. JIT 엔진 (NokJitSession)](906-jit-engine.md)
- [9.7. JIT 셀 실행 흐름과 상태 관리](907-jit-cells.md)
- [9.8. Jupyter 커널 통합](908-jupyter-integration.md)
- [9.9. 디버그 / 진단 도구](909-debug-tools.md)

## 도구 표면

```
nokc  (단일 바이너리)
  │
  ├── 빌드 / 실행
  │   ├── nokc build      : AOT 컴파일 — .nok 소스를 .a / 실행 파일로
  │   ├── nokc run        : AOT 컴파일 후 즉시 실행
  │   └── nokc test       : 테스트 함수 (#[test]) 수집 + 실행
  │
  ├── 인터랙티브
  │   ├── nokc repl       : 터미널 REPL (JIT 엔진)
  │   └── nokc jupyter    : Jupyter 커널 (JIT 엔진 + ZMQ wire)
  │
  ├── 도구
  │   ├── nokc fmt        : 포매터
  │   ├── nokc check      : 타입 / 진단만 (코드 생성 없이)
  │   └── nokc lsp        : LSP 서버 (IDE 통합)
  │
  ├── 디버그 / 진단
  │   ├── nokc demangle   : 심볼 demangle
  │   ├── nokc nhir-text  : NHIR bytecode 텍스트 출력
  │   └── nokc inspect    : 패키지 검사
  │
  └── 보조
      ├── nokc new        : 새 패키지 생성
      ├── nokc clean      : 빌드 산출물 제거
      └── nokc version    : 버전
```

모든 명령어가 `nokc` 한 바이너리에 통합된다. 별도 도구 (`nok-fmt` 등) 를 두지 않는다.

## 두 엔진의 구현

[실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 에서 정한 두 엔진:

```
JIT 엔진  ──  nokc repl / jupyter 가 사용
AOT 엔진  ──  nokc build / run / test 가 사용
```

두 엔진은 같은 컴파일 파이프라인 (nok-hir → nok-mir → 표준 MLIR) 을 거치며, 마지막 중간 표현 이후에만 갈라진다. 자세한 내용은 [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 과 [컴파일 파이프라인](../000-meta/002-pipeline.md) 에서 다룬다.

## 세 사용자 표면의 구현

```
표면              구현 명령
────────         ──────
REPL              nokc repl
Jupyter (로컬)    nokc jupyter        (사용자가 직접 실행하지 않고 Jupyter 가 호출)
Jupyter (WASM)    별도 wasm32 빌드     (브라우저 안에서 작동)
AOT 직접 실행     nokc build → 산출물 실행
```

각 표면의 구현은 후속 문서에서 다룬다.

## 패키지 빌드 산출물

[소스 단위](../000-meta/003-source-units.md) 의 패키지가 `libfoo.a` 한 개로 컴파일된다.

```
foo/
├── lib.nok                  ← 입력
└── ...
       ↓ nokc build
foo/target/
├── libfoo.a                 ← 산출물 (정적 archive)
└── ...
```

[archive](../000-meta/007-glossary.md#archive) 안의 구체적 형식은 [패키지 아카이브](903-package-archive.md) 에서 정의한다.

## NHIR: 제네릭 함수의 휴대용 IR

제네릭 함수는 호출 지점에서 [specialize](../000-meta/007-glossary.md#specialize) 해야 하므로 바이너리 코드만으로는 부족하다. 직렬화된 IR 인 NHIR (nok [HIR](../000-meta/007-glossary.md#hir-high-level-ir-/-mir-mid-level-ir)) bytecode 를 archive 안에 함께 담는다.

자세한 형식은 [NHIR bytecode 포맷](904-nhir-bytecode.md) 에서 정의한다.

## 단일 의존성: `libnokr.a`

빌드 산출물은 사용자 패키지의 `libfoo.a`, 런타임 `libnokr.a`, 표준 라이브러리 `libnok_*.a` 로 구성된다. 외부 OS 라이브러리 외에 추가 의존성이 없다.

자세한 빌드 흐름은 [AOT 빌드 파이프라인](902-aot-pipeline.md) 에서 정의한다.
