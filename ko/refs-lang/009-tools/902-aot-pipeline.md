# 9.2. AOT 빌드 파이프라인

이 장에서는 `rokc build` 의 단계별 흐름을 정의합니다. [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 과 [컴파일 파이프라인](../000-meta/002-pipeline.md) 에서 제시한 큰 그림을 구체적인 명령과 파일 흐름으로 구체화하여 명시 표기합니다.

## 9.2.1. 단계

전체 단계는 다음과 같이 13 단계로 진행됩니다.

```
1. package discovery       (find package root)
2. dependency resolve      (resolve deps to sources or prebuilt .rhir + .a)
3. source parse            (.rok → AST)
4. name / visibility       (resolve)
5. type check              (type check)
6. rok-hir build           (HIR build)
7. specialize              (collect call-sites + monomorphize)
8. rok-mir build
9. standard MLIR text emit
10. mlir-opt invoke        (external tool)
11. mlir-translate invoke  (external tool)
12. clang / lld invoke     (external tool)
13. artifact .a / executable
```

> package discovery 는 패키지 발견을, dependency resolve 는 의존성 해소를, source parse 는 소스 파싱을, name / visibility 는 이름 / 가시성 해소를, type check 는 타입 검사를, rok-hir build 는 rok-hir 생성을, specialize 는 단형화를 의미합니다. collect call-sites + monomorphize 는 call-site 수집과 단형화를, rok-mir build 는 rok-mir 생성을, standard MLIR text emit 은 표준 MLIR 텍스트 출력을 가리키며, invoke 는 호출, external tool 은 외부 도구, artifact 는 산출물, executable 은 실행 파일을 의미합니다.

각 단계의 입력과 출력, 그리고 책임을 명확하게 분리합니다.

## 9.2.2. 패키지 발견

`rokc build [path]` 의 path 부터 상위 방향으로 탐색하여 `main.rok` 또는 `lib.rok` 을 찾습니다. 만약 `[path]` 자체가 `lib.rok` 이거나 `main.rok` 이라면 그것이 곧 패키지가 되고, 그 외의 경우에는 (path 가 파일이면 그 디렉토리부터) 디렉토리를 위로 한 단계씩 올라가며 찾습니다.

찾았다면 그 디렉토리가 패키지 루트가 됩니다 ([소스 단위](../000-meta/003-source-units.md) 참고). 그 밖의 경우는 다음과 같습니다.

- 한 디렉토리에 `main.rok` 와 `lib.rok` 가 함께 있으면 에러입니다.
- path 가 파일이고 위로 올라가도 패키지가 없으면, 그 파일 하나로 된 익명 실행 패키지입니다 ([4.16.8](../004-functions/416-entry-point.md#4168-단일-파일-실행)). 파일 서브모듈 (`mod foo;`) 은 에러입니다.
- path 가 디렉토리이고 패키지가 없으면 에러입니다.

패키지 이름은 루트 디렉토리의 이름 (익명 패키지는 파일 이름에서 `.rok` 를 뗀 것) 이며, 산출물 이름이 됩니다 (9.2.15).

## 9.2.3. 의존성 해소

이 단계에서는 [경로](../005-modules/507-paths.md) 와 [`use`](../005-modules/505-use.md) 의 외부 패키지 참조 (`use external_pkg::...`) 를 처리합니다.

`rokc` 는 의존성을 스스로 해소하지 않습니다. front-end `rok` ([9.10](910-rok-frontend.md)) 가 `rok.rokon` / `rok.lock` 을 따라 path / git 의존 패키지를 준비하고, 각 패키지의 루트를 `--extern` 으로 `rokc` 에 넘깁니다.

```
rokc build --extern util=../util --extern parser=target/debug/deps/libparser.a
```

`--extern name=<경로>` 는 의존 패키지 하나를 최상위 경로 `name` 에 mount 합니다. `name` 은 `rok.rokon` 의 `deps` 키에서 `-` 를 `_` 로 바꾼 식별자입니다. 경로는 두 형태입니다.

| 경로                              | 의존 패키지                                                                  |
|-----------------------------------|------------------------------------------------------------------------------|
| `<dir>` / `<dir>/lib.rok`         | 소스. 그 `lib.rok` 를 같은 빌드에서 함께 컴파일                               |
| `<name>.rhir` / `lib<name>.a`     | 미리 빌드한 패키지. 둘 중 하나를 주면 같은 디렉토리의 짝 파일을 찾음. RHIR ([9.4](904-rhir-bytecode.md)) 을 읽어 선언과 제네릭을 얻고 archive 를 링크 |

- 사용자 코드는 `use name::..` 로 그 패키지의 항목을 씁니다.
- 의존 패키지 안의 `pkg::` 는 그 의존 패키지 자신을 가리킵니다.
- 의존 패키지는 별개의 패키지입니다. `pub` 항목만 바깥에서 쓸 수 있고, `pub(pkg)` / `pub(sub)` / 비공개 항목은 쓸 수 없습니다 ([5.7.14](../005-modules/507-paths.md#5714-패키지-사이-경로)).
- 사용자 패키지의 모듈이 의존 패키지와 같은 이름이면 에러입니다.

라이브러리 패키지를 `rokc build` 하면 `lib<name>.a` 와 `<name>.rhir` 를 함께 만듭니다. `--name <n>` 은 패키지 이름 (산출물 이름과 mount 이름) 을 정하며, 기본은 디렉토리 이름입니다.

### 미리 빌드한 의존 패키지

`rokc` 는 소스로 받은 `--extern` 을 주어진 순서대로 **한 번씩** 미리 빌드해 `target/[<triple>/]<profile>/deps/` 에 둡니다. 각 의존 패키지는 앞서 빌드한 의존 패키지를 모두 봅니다. 그 뒤의 컴파일은 소스 대신 이 `lib<name>.a` 와 `<name>.rhir` 를 씁니다.

```
target/<profile>/deps/
├── lib<name>.a
├── <name>.rhir
└── <name>.fp               # fingerprint
```

fingerprint 가 같으면 다시 빌드하지 않습니다. fingerprint 에 들어가는 것은 다음입니다.

- `rokc` 의 정체 (RHIR producer 와 같은 값)
- 빌드 옵션: `--release` / `-O*`, `-g`, `--target`, `--cfg`, `--bootstrap`
- 그 의존 패키지의 모든 `.rok` 파일과 `rok.rokon`
- 그보다 먼저 빌드한 의존 패키지들의 fingerprint

컴파일하는 모든 명령이 이 경로를 씁니다: `build` / `run` / `test` / `bench` / `check` / `lint` / `doc` / `lsp` / `repl` / `jupyter`, 단일 파일 스크립트, 이름 있는 환경. REPL host 도 미리 빌드한 archive 를 링크합니다. 패키지도 `--target-dir` 도 없으면 시스템 임시 디렉토리의 `rokc-target/` 아래에 둡니다.

표준 라이브러리는 미리 빌드하지 않고 빌드마다 소스로 bootstrap 합니다 ([9.3.6](903-package-archive.md#936-표준-라이브러리)).

## 9.2.4. 소스 파싱

[소스 파일](../001-lexical/101-source-files.md) 부터 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md) 까지의 어휘 / 구문 규칙에 따라 모든 `.rok` 파일을 AST 로 파싱합니다.

[모듈](../005-modules/504-modules.md) 의 디렉토리 모듈은 모든 .rok 파일을 합쳐서 한 모듈의 본체로 취급합니다.

패키지 루트의 매니페스트 `rok.rokon` ([9.10.3](910-rok-frontend.md#9103-rokrokon--roklock)) 은 모듈 파일이 아닙니다. 매니페스트는 [RokON](../../refs-standard-libraries/300-std/309-rokon.md) 값 하나 (`Package { .. }`) 로 읽으며, top-level 문 규칙을 적용하지 않고 컴파일하지도 않습니다.

### Scanner pre-pass

[`#[keyword]` 매직](../006-meta-system/606-keyword-magic.md) 의 scanner pre-pass 도 이 단계에서 함께 수행합니다. 먼저 모든 .rok 파일을 얕게 스캔한 뒤, `#[keyword(...)]`, `#[infix(...)]`, `#[literal(...)]` 등록 정보를 수집하고, 정식 lexer / parser 에 그 등록 정보를 전달합니다.

## 9.2.5. 이름 / 가시성 해소

[`use` (가져오기)](../005-modules/505-use.md), [가시성](../005-modules/506-visibility.md), [경로](../005-modules/507-paths.md), [재내보내기 (`pub use`)](../005-modules/508-re-export.md) 의 규칙으로 모든 경로와 식별자를 항목에 매핑합니다.

이 단계에서는 `rok::core::**` 묵시적 import 를 적용하고 ([표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md) 을 참고하되, `rok::bare::*` 와 `rok::std::*` 는 명시 import 가 필요한 항목으로 제외합니다), 가시성 검사를 수행해서 부적절한 접근은 에러로 처리하며, import 충돌 검사도 함께 진행합니다.

## 9.2.6. 타입 검사

[2. 타입 시스템](../002-types/README.md) 의 규칙으로 모든 식의 타입을 결정합니다.

이 과정에서 점진과 정적 슬롯의 결정을 수행하고 ([`any` 슬롯](../002-types/202-any-slot.md) 참고), 트레잇 바운드 검사를 진행하며 ([제네릭](../002-types/210-generics.md) 참고), 분기 결과 타입을 확정합니다 ([3. 표현식](../003-expressions/README.md) 참고). 또한 합 타입과 `any` 슬롯의 좁히기도 함께 처리합니다.

## 9.2.7. rok-hir 생성

타입 검사가 끝난 AST 를 rok-hir ([HIR](../000-meta/007-glossary.md#hir-high-level-ir--mir-mid-level-ir)) 자료구조로 변환합니다.

HIR 은 타입 정보가 풍부하게 담긴 중간 표현이며, 위 4 ~ 5 단계의 결과를 모두 반영한 형태입니다. 이때 구조적으로 압축하여 같은 타입과 식별자는 공유 표현을 사용합니다.

## 9.2.8. specialize: 단형화

[제네릭](../002-types/210-generics.md) 과 [const 제네릭](../002-types/211-const-generics.md) 의 단형화를 수행합니다.

```
1. 모든 호출 지점 수집.
2. 각 호출의 구체 타입 튜플 결정.
3. (item_id, type_tuple) 마다 별개의 specialized HIR 생성.
4. 캐시. 같은 (item_id, type_tuple) 는 한 번만.
```

프로그램의 root (`main`, export 된 심볼, `#[export_name]` 항목, 런타임이 참조하는 심볼) 에서 도달할 수 없는 함수는 emit 하지 않습니다. 라이브러리 빌드에서도 같아서, 패키지의 `pub` 항목과 그 root 에서 닿지 않는 함수는 archive 에 넣지 않습니다.

이 단계 덕분에 [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 가 vtable 없는 직접 호출을 할 수 있습니다 ([트레잇-as-타입 금지](../002-types/212-trait-as-type-banned.md) 참고).

## 9.2.9. rok-mir 생성

[specialize](../000-meta/007-glossary.md#specialize) 된 HIR 을 rok-mir (MIR) 로 변환합니다.

MIR 은 HIR 보다 한층 더 낮은 수준의 IR 로, 이 단계에서 연산자 desugar 가 완료되고 ([연산자 디스패치](../003-expressions/301-operator-dispatch.md) 참고), 패턴 매칭의 decision tree 가 결정되며 ([`match` 표현식](../003-expressions/312-match-expression.md) 참고), closure 와 코루틴의 frame layout 도 함께 확정됩니다.

## 9.2.10. 표준 MLIR 텍스트 출력

[컴파일 파이프라인](../000-meta/002-pipeline.md) 과 [MLIR 다이얼렉트 사용](../008-abi/807-mlir-dialects.md) 의 표준 MLIR 다이얼렉트를 사용하여 `.mlir` 텍스트를 emit 합니다.

```
target/debug/build/foo.mlir
```

[8. 런타임 ABI / MLIR lowering](../008-abi/README.md) 의 lowering 규칙에 따라 모든 rok 의미를 표준 다이얼렉트로 변환합니다.

`rokc` 는 MLIR 라이브러리 바인딩 (C++, Rust 등) 을 쓰지 않습니다. MLIR 을 텍스트로 emit 한 뒤, 이어지는 단계에서 외부 도구 `mlir-opt`, `mlir-translate`, `clang` 을 프로세스로 호출합니다.

여기까지가 `rokc` 의 책임 영역입니다.

## 9.2.11. mlir-opt 호출

표준 MLIR 도구인 `mlir-opt` 를 호출하여 정리 패스와 LLVM 다이얼렉트로의 변환 패스를 적용합니다. `mlir-opt` 에는 `-O2` 같은 최적화 단계 옵션이 없으므로, 패스를 이름으로 나열합니다.

```
mlir-opt --canonicalize --cse \
         --convert-scf-to-cf --convert-cf-to-llvm --convert-arith-to-llvm \
         --convert-func-to-llvm --reconcile-unrealized-casts \
         target/debug/build/foo.mlir > target/debug/build/foo.opt.mlir
```

정확한 패스 목록은 구현이 정하며, 위는 대표적인 구성입니다. `-O0` ~ `-O3` 같은 최적화 단계는 이후 `clang` 단계에 적용됩니다.

## 9.2.12. mlir-translate 호출

표준 도구인 `mlir-translate` 를 사용해서 MLIR 을 LLVM IR 로 옮깁니다.

```
mlir-translate --mlir-to-llvmir target/debug/build/foo.opt.mlir > target/debug/build/foo.ll
```

## 9.2.13. object 생성

LLVM IR 은 같은 LLVM 의 `clang` 으로 object 로 바꿉니다.

```
clang -O2 -c target/debug/build/foo.ll -o target/debug/build/foo.o
ar rcs target/debug/libfoo.a target/debug/build/foo.o
```

라이브러리 패키지는 archive 옆에 인터페이스 파일 `foo.rhir` ([RHIR 포맷](904-rhir-bytecode.md)) 도 씁니다.

## 9.2.14. 링킹 (실행 파일의 경우)

```
$ROK_CC target/debug/build/foo.o <deps>/lib*.a <sysroot>/lib/librokr.a -o target/debug/foo
```

링크는 `ROK_CC` 로 합니다. 없으면 Apple 에서는 `xcrun clang`, 그 밖에는 LLVM 의 `clang` 입니다. Apple 에서 `-g` 이면 링크 뒤에 `dsymutil` 을 실행합니다.

## 9.2.15. 산출물

산출물은 cargo 와 같이 **profile 별** 디렉토리에 놓입니다. profile 은 `debug` (기본값) 와 `release` 이며, 최적화 단계가 `-O2` / `-O3` / `-Os` (`--release` 포함) 이면 `release` 입니다.

```
target/
├── debug/                      # profile (기본)
│   ├── foo                     # executable (for main)
│   ├── libfoo.a                # package archive (for lib)
│   ├── foo.rhir                # interface (for lib, 9.4)
│   ├── foo.o                   # the archive's object (for lib)
│   ├── build/                  # intermediates and --emit output (foo.mlir, ...)
│   └── deps/                   # prebuilt dependencies (lib<dep>.a, <dep>.rhir, <dep>.fp)
├── release/                    # --release
│   └── ...
├── aarch64-unknown-linux-gnu/  # cross build (--target=<triple>)
│   └── debug/ ...
└── doc/                        # rok doc / rokc doc
```

| 산출물          | 경로                                                              |
|-----------------|-------------------------------------------------------------------|
| 실행 파일       | `target/<profile>/<name>`                                         |
| 라이브러리      | `target/<profile>/lib<name>.a` 와 `target/<profile>/<name>.o`     |
| `--emit=<kind>` | `target/<profile>/build/<name>.<ext>` (`hir` `mir` `mlir` `ll` `s` `o`) |
| 중간 산출물     | `target/<profile>/build/`                                         |
| 의존 패키지     | `target/<profile>/deps/` (`lib<dep>.a`, `<dep>.rhir`, `<dep>.fp`, 9.2.3) |
| 교차 빌드       | `target/<triple>/<profile>/...` (위와 같은 구조)                  |
| 문서            | `target/doc/`                                                     |

이름 `foo` 는 패키지 이름입니다 (9.2.2). `-o` 는 산출물 하나의 경로를, `--target-dir=<dir>` 와 환경 변수 `ROKC_TARGET_DIR` 은 `target/` 자체의 위치를 바꿉니다. `rokc clean` / `rok clean` 은 `target/` 전체를 지웁니다. 라이브러리의 RHIR 은 archive 옆의 `foo.rhir` 입니다 ([9.4](904-rhir-bytecode.md)).

## 9.2.16. 증분 빌드

증분 빌드는 패키지 단위입니다. `rokc build` / `rokc run` 은 이전 빌드의 fingerprint 와 지금의 것이 같으면 `Fresh` 를 출력하고 빌드를 건너뜁니다. fingerprint 는 다음을 포함합니다.

- 컴파일러의 정체 (RHIR producer 와 같은 값), 빌드 옵션, 진단 형식 (`--message-format`, 색), 입력 / 출력 경로, `PATH`
- 패키지와 의존성에서 읽을 수 있는 모든 `.rok` 파일과 `rok.rokon`
- 미리 빌드한 의존성의 `.a`, MLIR plugin 파일
- 산출물의 크기와 수정 시각

빌드가 낸 경고는 기록해 두며, `Fresh` 인 빌드는 그 경고를 다시 출력합니다.

## 9.2.17. 컴파일러와 런타임의 분리

이 파이프라인은 코드 의존성 0 원칙을 지킵니다. 즉 `rokc` 는 런타임 (`librokr.a`) 을 빌드 시점에 참조하지 않습니다.

대신에 `extern "C"` 심볼 ([Extern "C" 심볼](../008-abi/805-extern-symbols.md)) 호출만을 emit 하고, 링크 시점에 가서 사용자 빌드가 `librokr.a` 를 함께 링크하는 방식으로 동작합니다.

이렇게 분리해 둔 덕분에 `rokc` 와 `librokr.a` 가 서로 독립된 SemVer 로 진화할 수 있습니다.

## 9.2.18. 외부 도구 의존

| 도구             | 역할                 |
|------------------|----------------------|
| `mlir-opt`       | MLIR 최적화          |
| `mlir-translate` | MLIR → LLVM IR       |
| `clang`          | LLVM IR → object (같은 LLVM 의 것) |
| `ROK_CC` / `xcrun clang` | 마지막 링크  |

LLVM 은 21 ~ 22 를 씁니다. `PATH` 에서 먼저 찾고, 없으면 알려진 위치 (Homebrew, apt.llvm.org) 를 찾습니다. `rokc` 는 이 도구를 프로세스로 부를 뿐 LLVM 라이브러리를 링크하지 않습니다 ([0.2.4](../000-meta/002-pipeline.md#024-mlir-텍스트-직접-생성)). 도구가 없으면 찾아본 위치를 밝히는 에러입니다.

## 9.2.19. 설치 레이아웃

설치 레이아웃은 rustup / cargo 식입니다. `ROK_HOME` 의 기본값은 `~/.rok` 입니다.

```
$ROK_HOME/
├── bin/                         # proxy: 활성 toolchain 의 같은 이름 실행 파일을 실행
│   ├── rok
│   ├── rokc
│   └── rokls
├── toolchains/
│   └── <version>-<triple>/      # 자기 완결 toolchain
│       ├── bin/{rok, rokc, rokls}
│       └── lib/
│           ├── librokr.a        # 런타임
│           └── rok/{core, std, bare}/   # 표준 라이브러리 소스
├── cache/                       # git, registry, 스크립트 lock (9.10.4)
├── envs/<name>/, envs/CURRENT   # 이름 있는 환경 (9.10.5)
└── config.rokon                 # Config { registry, toolchain }
```

`rokc` 는 표준 라이브러리와 런타임을 자기 실행 파일 기준 (sysroot = 실행 파일의 `..`) 에서 찾으며, `ROK_HOME` 을 보지 않습니다. 활성 toolchain 의 선택은 [9.10.6](910-rok-frontend.md#9106-toolchain) 이 정합니다.
