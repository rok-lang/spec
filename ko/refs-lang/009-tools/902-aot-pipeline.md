# 9.2. AOT 빌드 파이프라인

`nokc build` 의 단계별 흐름을 정의한다. [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 과 [컴파일 파이프라인](../000-meta/002-pipeline.md) 에서 제시한 큰 그림을 구체적인 명령과 파일 흐름으로 풀어 쓴다.

## 9.2.1. 단계

```
1. 패키지 발견          (find package root)
2. 의존성 해소          (resolve deps to their .a)
3. 소스 파싱            (.nok → AST)
4. 이름 / 가시성 해소    (resolve)
5. 타입 검사            (type check)
6. nok-hir 생성         (HIR build)
7. specialize           (call-site 수집 + 단형화)
8. nok-mir 생성
9. 표준 MLIR 텍스트 출력
10. mlir-opt 호출        (외부 도구)
11. mlir-translate 호출  (외부 도구)
12. clang / lld 호출    (외부 도구)
13. 산출물 .a / 실행 파일
```

각 단계의 입력, 출력, 책임을 명확하게 분리한다.

## 9.2.2. 패키지 발견

`nokc build [path]` 의 path 부터 상위로 탐색하여 `main.nok` 또는 `lib.nok` 을 찾는다.

- `[path]` 가 `lib.nok` 또는 `main.nok` 이면 그것이 패키지이다.
- 그 외에는 디렉토리를 위로 올라가며 찾는다.

찾으면 그 디렉토리가 패키지 루트가 된다 ([소스 단위](../000-meta/003-source-units.md)).

## 9.2.3. 의존성 해소

[경로](../005-modules/507-paths.md) 와 [`use`](../005-modules/505-use.md) 의 외부 패키지 참조 (`use external_pkg::...`).

- 현재 nok 사양 범위 안에서는 의존성 명세가 없다.
- 향후 패키지 매니저가 `Nok.toml` 같은 메타 파일에서 의존성을 읽고 `.a` 들의 위치를 결정한다.
- 본 사양은 의존성 .a 파일이 명시적으로 지정된 경우의 처리만 다룬다.

```
nokc build --extern foo=path/to/libfoo.a --extern bar=...
```

옵션으로 각 의존 패키지의 .a 위치를 명시한다. 패키지 매니저가 자동으로 채워 주는 자리이다.

## 9.2.4. 소스 파싱

[소스 파일](../001-lexical/101-source-files.md) 부터 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md) 까지의 어휘 / 구문 규칙에 따라 모든 `.nok` 파일을 AST 로 파싱한다.

[모듈](../005-modules/504-modules.md) 의 디렉토리 모듈은 모든 .nok 파일을 합쳐 한 모듈의 본체로 다룬다.

### Scanner pre-pass

[`#[keyword]` 매직](../006-meta-system/606-keyword-magic.md) 의 scanner pre-pass 도 이 단계에서 수행한다.

- 모든 .nok 파일을 얕게 스캔한다.
- `#[keyword(...)]`, `#[infix(...)]`, `#[literal(...)]` 등록을 수집한다.
- 정식 lexer / parser 에 등록 정보를 전달한다.

## 9.2.5. 이름 / 가시성 해소

[`use` (가져오기)](../005-modules/505-use.md), [가시성](../005-modules/506-visibility.md), [경로](../005-modules/507-paths.md), [재내보내기 (`pub use`)](../005-modules/508-re-export.md) 의 규칙으로 모든 경로와 식별자를 항목에 매핑한다.

- 가시성 검사. 부적절한 접근은 에러.
- import 충돌 검사.
- 자동 prelude (`core::prelude`) 를 적용한다.

## 9.2.6. 타입 검사

[2. 타입 시스템](../002-types/README.md) 의 규칙으로 모든 식의 타입을 결정한다.

- 점진 vs 정적 슬롯 결정 ([`any` 슬롯](../002-types/202-any-slot.md)).
- 트레잇 바운드 검사 ([제네릭](../002-types/210-generics.md)).
- 분기 결과 타입 ([3. 표현식](../003-expressions/README.md)).
- 합 타입 / `any` 슬롯의 좁히기.

## 9.2.7. nok-hir 생성

타입 검사된 AST 를 nok-hir ([HIR](../000-meta/007-glossary.md#hir-high-level-ir-/-mir-mid-level-ir)) 자료구조로 변환한다.

- HIR 은 타입 정보가 풍부한 중간 표현이다.
- 위 4 ~ 5 의 결과를 모두 반영한 형태이다.
- 구조적으로 압축한다. 같은 타입과 식별자는 공유 표현을 쓴다.

## 9.2.8. specialize: 단형화

[제네릭](../002-types/210-generics.md) 과 [const 제네릭](../002-types/211-const-generics.md) 의 단형화.

```
1. 모든 호출 지점 수집.
2. 각 호출의 구체 타입 튜플 결정.
3. (item_id, type_tuple) 마다 별개의 specialized HIR 생성.
4. 캐시. 같은 (item_id, type_tuple) 는 한 번만.
```

이 단계 덕분에 [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 가 vtable 없는 직접 호출을 할 수 있다 ([트레잇-as-타입 금지](../002-types/212-trait-as-type-banned.md)).

## 9.2.9. nok-mir 생성

[specialize](../000-meta/007-glossary.md#specialize) 된 HIR → nok-mir (MIR).

- 더 낮은 수준의 IR 이다.
- 연산자 desugar 완료 ([연산자 디스패치](../003-expressions/301-operator-dispatch.md)).
- 패턴 매칭의 decision tree 결정 ([`match` 표현식](../003-expressions/312-match-expression.md)).
- closure 와 코루틴의 frame layout 확정.

## 9.2.10. 표준 MLIR 텍스트 출력

[컴파일 파이프라인](../000-meta/002-pipeline.md) 과 [MLIR 다이얼렉트 사용](../008-abi/807-mlir-dialects.md) 의 표준 MLIR 다이얼렉트로 `.mlir` 텍스트를 emit 한다.

```
target/foo.mlir
```

[8. 런타임 ABI / MLIR lowering](../008-abi/README.md) 의 lowering 규칙에 따라 모든 nok 의미를 표준 다이얼렉트로 변환한다.

이 단계까지가 nokc 의 책임이다.

## 9.2.11. mlir-opt 호출

표준 MLIR 도구 `mlir-opt` 를 호출한다. 표준 패스 (인라인, dce, simplification 등) 를 적용한다.

```
mlir-opt -O2 ... target/foo.mlir > target/foo.opt.mlir
```

## 9.2.12. mlir-translate 호출

표준 도구 `mlir-translate` 로 MLIR 을 LLVM IR 로 옮긴다.

```
mlir-translate --mlir-to-llvmir target/foo.opt.mlir > target/foo.ll
```

## 9.2.13. clang / lld 호출

표준 도구 `clang` (또는 LLVM 의 `llc` + `lld`) 으로 LLVM IR 을 네이티브 코드로 변환한다.

```
clang -O2 -c target/foo.ll -o target/foo.o
ar rcs target/libfoo.a target/foo.o target/foo.hir.o
```

`foo.hir.o` 는 [NHIR bytecode 포맷](904-nhir-bytecode.md) 의 [NHIR](../000-meta/007-glossary.md#nhir-nok-hir-bytecode) 섹션을 담은 obj 이다.

## 9.2.14. 링킹 (실행 파일의 경우)

`main.nok` 패키지인 경우:

```
clang -O2 \
  target/foo.o \
  libnokr.a \
  libnok_unsafe.a libnok_core.a libnok_std.a \
  -o target/foo                         (-lc 등 필요시)
```

이 단계가 실행 가능한 바이너리를 만든다.

## 9.2.15. 산출물

```
target/
├── foo.mlir            # 디버그 (--emit=mlir 시)
├── foo.ll              # 디버그 (--emit=llvmir 시)
├── foo.o               # concrete 코드의 obj
├── foo.hir.o           # NHIR bytecode 가 든 obj
├── libfoo.a            # 패키지 archive (lib 의 경우)
└── foo                 # 실행 파일 (main 의 경우)
```

## 9.2.16. 증분 빌드

```
1. 변경된 .nok 파일만 다시 파싱.
2. 영향받은 specialization 만 다시 생성.
3. 변경된 obj 만 다시 컴파일.
4. archive 만 재구성.
```

캐시 키는 파일 hash + nokc 버전 hash + 의존 hash 이다.

자세한 증분 알고리듬은 `nokc` 가 결정한다.

## 9.2.17. 컴파일러와 런타임의 분리

코드 의존성 0 원칙. `nokc` 는 런타임 (`libnokr.a`) 을 빌드 시점에 참조하지 않는다.

대신 `extern "C"` 심볼 ([Extern "C" 심볼](../008-abi/805-extern-symbols.md)) 호출만 emit 한다. 링크 시점에 사용자 빌드가 `libnokr.a` 를 함께 링크한다.

이렇게 분리한 덕분에 `nokc` 와 `libnokr.a` 가 독립된 SemVer 로 진화할 수 있다.

## 9.2.18. 외부 도구 의존

`nokc build` 가 동작하려면 다음 외부 도구가 PATH 에 있어야 한다.

| 도구                       | 역할                 |
| -------------------------- | -------------------- |
| `mlir-opt`                 | MLIR 최적화          |
| `mlir-translate`           | MLIR → LLVM IR       |
| `clang` 또는 `llc` + `lld` | LLVM IR → obj / 링킹 |

`nokc` 는 이들을 호출만 한다. LLVM 라이브러리에 직접 link 하지 않는다 ([컴파일 파이프라인](../000-meta/002-pipeline.md)).

설치되어 있지 않으면 `nokc build` 는 명확한 에러를 낸다 (어느 도구를 어디서 받을 수 있는지 안내한다).
