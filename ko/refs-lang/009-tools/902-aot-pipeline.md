# 9.2. AOT 빌드 파이프라인

이 장에서는 `rokc build` 의 단계별 흐름을 정의합니다. [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 과 [컴파일 파이프라인](../000-meta/002-pipeline.md) 에서 제시한 큰 그림을 구체적인 명령과 파일 흐름으로 구체화하여 명시 표기합니다.

## 9.2.1. 단계

전체 단계는 다음과 같이 11 단계로 진행되며, 모두 `rokc` 안에서 실행됩니다.

```
1. package discovery       (find package root)
2. dependency resolve      (resolve deps to sources or prebuilt .rokhir + .a)
3. source parse            (.rok → AST)
4. name / visibility       (resolve)
5. type check              (type check)
6. rok-hir build           (HIR build)
7. specialize              (collect call-sites + monomorphize)
8. rok-mir build
9. code generation         (native at -O0; MLIR → LLVM at -O1 and above)
10. archive / link         (lld)
11. artifact .a / executable
```

> package discovery 는 패키지 발견을, dependency resolve 는 의존성 해소를, source parse 는 소스 파싱을, name / visibility 는 이름 / 가시성 해소를, type check 는 타입 검사를, rok-hir build 는 rok-hir 생성을, specialize 는 단형화를 의미합니다. collect call-sites + monomorphize 는 call-site 수집과 단형화를, rok-mir build 는 rok-mir 생성을, code generation 은 object 로의 코드 생성을, archive / link 는 archive 생성 또는 링크를 가리키며, artifact 는 산출물, executable 은 실행 파일을 의미합니다.

각 단계의 입력과 출력, 그리고 책임을 명확하게 분리합니다.

## 9.2.2. 패키지 발견

`rokc build [path]` 의 path 부터 상위 방향으로 탐색하여 `src/main.rok` 또는 `src/lib.rok` 가 있는 디렉토리를 찾습니다 (path 가 파일이면 그 디렉토리부터). 디렉토리를 위로 한 단계씩 올라가며 찾습니다.

찾았다면 그 디렉토리가 패키지 루트가 되고, 그 `src/` 가 패키지의 소스가 됩니다 ([소스 단위](../000-meta/003-source-units.md) 참고). 그 밖의 경우는 다음과 같습니다.

- path 가 찾은 패키지의 `src/` 밖의 파일 (패키지 소스 옆의 스크립트나 도구) 이거나 위로 올라가도 패키지가 없으면, 그 파일 하나로 된 익명 실행 패키지입니다 ([4.16.8](../004-functions/416-entry-point.md#4168-단일-파일-실행)). 그 파일 서브모듈은 그 옆에 있습니다.
- path 가 디렉토리이고 패키지가 없으면 에러입니다.

패키지 이름은 패키지 디렉토리의 이름 (익명 패키지는 파일 이름에서 `.rok` 를 뗀 것) 이며, 산출물 이름이 됩니다 (9.2.15). 루트 파일이 둘 다 있는 패키지는 두 target 으로 빌드합니다: 먼저 라이브러리, 그다음 그 라이브러리를 의존 패키지 `<name>` 으로 mount 하는 실행 파일 ([0.3.1](../000-meta/003-source-units.md#한-패키지의-라이브러리와-실행-파일)).

## 9.2.3. 의존성 해소

이 단계에서는 [경로](../005-modules/507-paths.md) 와 [`use`](../005-modules/505-use.md) 의 외부 패키지 참조 (`use external_pkg::...`) 를 처리합니다.

`rokc` 는 의존성을 스스로 해소하지 않습니다. front-end `rok` ([9.10](910-rok-frontend.md)) 가 `rok.rokon` / `rok.lock` 을 따라 path / git 의존 패키지를 준비하고, 각 패키지의 루트를 `--extern` 으로 `rokc` 에 넘깁니다.

```
rokc build --extern util=../util --extern parser=target/debug/deps/libparser.a
```

`--extern name=<경로>` 는 의존 패키지 하나를 최상위 경로 `name` 에 mount 합니다. `name` 은 `rok.rokon` 의 `deps` 키에서 `-` 를 `_` 로 바꾼 식별자입니다. 경로는 두 형태입니다.

| 경로                              | 의존 패키지                                                                  |
|-----------------------------------|------------------------------------------------------------------------------|
| `<dir>` / `<dir>/src/lib.rok`     | 소스. 그 `src/lib.rok` 를 같은 빌드에서 함께 컴파일                           |
| `<name>.rokhir` / `lib<name>.a`     | 미리 빌드한 패키지. 둘 중 하나를 주면 같은 디렉토리의 짝 파일을 찾음. ROKHIR ([9.4](904-rokhir-bytecode.md)) 을 읽어 선언과 제네릭을 얻고 archive 를 링크 |

- 사용자 코드는 `use name::..` 로 그 패키지의 항목을 씁니다.
- 의존 패키지 안의 `pkg::` 는 그 의존 패키지 자신을 가리킵니다.
- 의존 패키지는 별개의 패키지입니다. `pub` 항목만 바깥에서 쓸 수 있고, `pub(pkg)` / `pub(sub)` / 비공개 항목은 쓸 수 없습니다 ([5.7.14](../005-modules/507-paths.md#5714-패키지-사이-경로)).
- 사용자 패키지의 모듈이 의존 패키지와 같은 이름이면 에러입니다.

`src/lib.rok` 가 있는 패키지를 `rokc build` 하면 `lib<name>.a` 와 `<name>.rokhir` 를, `src/main.rok` 가 있으면 실행 파일 `<name>` 을 만듭니다. 둘 다 있는 패키지는 셋 모두를 만들며, 실행 파일은 그 라이브러리에 링크합니다. `--name <n>` 은 패키지 이름 (산출물 이름과 mount 이름) 을 정하며, 기본은 디렉토리 이름입니다.

### 미리 빌드한 의존 패키지

`rokc` 는 소스로 받은 `--extern` 을 주어진 순서대로 **한 번씩** 미리 빌드해 `target/[<triple>/]<profile>/deps/` 에 둡니다. 각 의존 패키지는 앞서 빌드한 의존 패키지를 모두 봅니다. 그 뒤의 컴파일은 소스 대신 이 `lib<name>.a` 와 `<name>.rokhir` 를 씁니다.

```
target/<profile>/deps/
├── lib<name>.a
├── <name>.rokhir
└── <name>.fp               # fingerprint
```

fingerprint 가 같으면 다시 빌드하지 않습니다. fingerprint 에 들어가는 것은 다음입니다.

- `rokc` 의 정체 (ROKHIR producer 와 같은 값)
- 빌드 옵션: `--release` / `-O*`, `-g`, `--target`, `--cfg` (`feature` 키 제외), `--bootstrap`
- 그 의존 패키지의 feature (`--extern-feature`)
- 그 의존 패키지의 `src/` 아래 모든 파일과 `rok.rokon`
- 그보다 먼저 빌드한 의존 패키지들의 fingerprint

미리 빌드한 패키지의 interface 는 소비하는 쪽의 feature 가 아니라 그 패키지가 빌드된 feature (그 ROKHIR meta, [9.4.6](904-rokhir-bytecode.md#946-meta)) 로 읽습니다.

컴파일하는 모든 명령이 이 경로를 씁니다: `build` / `run` / `test` / `bench` / `check` / `lint` / `doc` / `lsp` / `repl` / `jupyter`, 단일 파일 스크립트, 이름 있는 환경. REPL host 도 미리 빌드한 archive 를 링크합니다. 패키지도 `--target-dir` 도 없으면 시스템 임시 디렉토리의 `rokc-target/` 아래에 둡니다.

표준 라이브러리는 소스에서 빌드하며 ([9.3.6](903-package-archive.md#936-표준-라이브러리)) `rokc` 의 cache (9.2.19) 에 미리 빌드된 상태로 둡니다.

## 9.2.4. 소스 파싱

[소스 파일](../001-lexical/101-source-files.md) 부터 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md) 까지의 어휘 / 구문 규칙에 따라 모든 `.rok` 파일을 AST 로 파싱합니다.

[모듈](../005-modules/504-modules.md) 의 디렉토리 모듈은 모든 .rok 파일을 합쳐서 한 모듈의 본체로 취급합니다.

패키지 루트의 매니페스트 `rok.rokon` ([9.10.3](910-rok-frontend.md#9103-rokrokon--roklock)) 은 모듈 파일이 아닙니다. 매니페스트는 [ROKON](../../refs-standard-libraries/300-std/309-rokon.md) 값 하나 (`Package { .. }`) 로 읽으며, top-level 문 규칙을 적용하지 않고 컴파일하지도 않습니다.

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

## 9.2.10. Backend 선택

기본으로는 최적화 수준이 코드 생성기를 고르며 ([0.2.4](../000-meta/002-pipeline.md#024-backends)), `-C backend=native|llvm` 이나 `ROK_BACKEND` 로 바꿀 수 있습니다.

| 수준                     | Backend      |
|--------------------------|--------------|
| `-O0`                    | native       |
| `-O1` `-O2` `-O3` `-Os`  | MLIR → LLVM  |

native backend 는 {aarch64, x86_64} × {macOS, Linux} 를 지원합니다. 그 밖의 대상, [`__mlir_op` / `__mlir_type`](../../refs-standard-libraries/100-bare/109-mlir.md) 을 쓰는 모듈, MLIR plugin 을 쓰는 빌드, `-C profile-generate` / `-C profile-use` 빌드는 어느 수준에서나 MLIR → LLVM backend 를 씁니다. REPL 도 같은 선택을 따릅니다 (9.6.1). 두 backend 는 [8](../008-abi/README.md) 의 런타임 ABI 를 따르므로, 다른 수준으로 빌드한 미리 빌드된 의존 패키지까지 포함해 어느 쪽 object 든 함께 링크됩니다.

## 9.2.11. native backend

native backend 는 macOS 와 Linux 의 aarch64, x86_64 를 대상으로, 중간 IR 없이 rok-mir 에서 바로 기계어와 object 파일 (Mach-O, ELF) 을 만듭니다. 목표는 컴파일 속도이며, debug 빌드와 REPL 셀에는 LLVM 이 필요 없습니다. `-g` 이면 DWARF 줄 정보와 frame 정보를 냅니다.

## 9.2.12. MLIR → LLVM backend

rok-mir 를 표준 MLIR 로 lower 합니다 ([8.7](../008-abi/807-mlir-dialects.md)). `rokc` 는 그 MLIR 을 프로세스 안에서 파싱하고, LLVM 다이얼렉트까지 MLIR 패스 파이프라인을 돌린 뒤 LLVM IR 로 옮겨, 요청한 수준으로 최적화하고 대상 triple 의 object 를 만듭니다. IR 이 1 MiB 를 넘는 모듈은 여러 부분으로 나누어 병렬로 코드 생성합니다. 모두 외부 도구 없이 프로세스 안에서 실행되며, 중간 형태는 `--emit` 일 때만 파일로 씁니다 ([9.9.1](909-debug-tools.md#991-중간-산출물-출력---emit)).

```
target/debug/build/foo.o
```

## 9.2.13. archive

라이브러리 패키지는 `rokc` 가 직접 쓰는 표준 `ar` archive 에 object 를 codegen unit (9.2.16) 마다 하나씩 넣고, 그 옆에 인터페이스 파일 `foo.rokhir` ([ROKHIR 형식](904-rokhir-bytecode.md)) 을 씁니다.

```
target/debug/libfoo.a
target/debug/foo.rokhir
```

## 9.2.14. 링킹 (실행 파일의 경우)

실행 파일은 내장 lld 로 링크합니다:

```
foo.o + <deps>/lib*.a + <sysroot>/lib/librokr.a + target libc  →  target/debug/foo
```

Mach-O 실행 파일은 macOS SDK (Xcode Command Line Tools) 에 대해 링크합니다. Windows 실행 파일은 `rokc` 가 함께 가진 module definition (mingw-w64 에서 유래) 으로 만든 import library 와, UCRT 위의 rok 자신의 startup 에 대해 링크합니다. MSVC 도구나 Windows SDK 는 필요 없습니다. rok 의 Windows startup 은 `main` 전에 `.CRT$XCA`..`.CRT$XCZ` 의 C initializer 를 실행합니다. x86 Windows 의 import 이름은 mingw-w64 의 `lib32` 정의에서 오며 stdcall `@N` 장식을 쓰고, image 는 `/SAFESEH:NO` 로 링크합니다. Linux 대상은 native 든 cross 든 언제나 `<sysroot>/lib/rok/targets/<triple>/` 에 제공된 C 라이브러리에 대해 링크합니다: musl 은 정적으로, glibc 는 link stub 을 통해 (실행 파일은 glibc 2.34 이상이 필요). 다른 라이브러리는 `-L` 과 `LIBRARY_PATH` 로 찾습니다. 32-bit glibc 타깃의 dynamic loader 는 `/lib/ld-linux.so.2` (i686) 와 `/lib/ld-linux-armhf.so.3` (armv7hf) 입니다. `ROK_CC` 가 설정되어 있으면 그 C 컴파일러 드라이버가 최종 링크를 대신합니다 (`$ROK_CC <objects> <archives> -o <out>`). 이것은 선택적인 대체입니다. `--native-runtime` 으로 준 공유 라이브러리는 실행 파일 옆에 복사되며, 실행 파일은 rpath `@loader_path` (Mach-O) / `$ORIGIN` (ELF) 을 가집니다. Windows 는 실행 파일의 디렉토리에서 DLL 을 읽습니다. 실행 파일은 dead-strip 됩니다: 참조되지 않는 함수와 데이터는 제거하며, `release` 빌드는 같은 함수도 하나로 합칩니다 ([9.2.15](#9215-산출물)). Apple 대상에서 실행 파일에 디버그 정보가 있으면 내장 `dsymutil` (`rokc` 에 링크된 LLVM 의 것) 이 실행 파일 옆에 `.dSYM` bundle 을 씁니다.

## 9.2.15. 산출물

산출물은 cargo 와 같이 **profile 별** 디렉토리에 놓입니다. profile 은 `debug` (기본값) 와 `release` 이며, 최적화 단계가 `-O2` / `-O3` / `-Os` (`--release` 포함) 이면 `release` 입니다.

두 profile 은 cargo 처럼 다릅니다.

| Profile | 최적화 | 디버그 정보 | 링크 |
|---|---|---|---|
| `debug` | `-O0`, `-O1` | 있음 | 참조되지 않는 코드와 데이터 제거 |
| `release` | `-O2`, `-O3`, `-Os` | `-g` 일 때만 | 같은 함수도 하나로 합침 (객체가 주소를 취했다고 표시한 함수는 제외); `-g` 가 없으면 loader 가 필요로 하는 것 밖의 심볼 없음 |

디버그 정보는 빌드가 만드는 실행 파일이나 archive (`build`, `run`, `test`) 의 것이며, `--emit` 출력이나 REPL, Jupyter 셀의 것이 아닙니다. panic 은 어느 profile 에서든 소스 위치를 알립니다. 빌드의 객체는 출력 옆의 한 곳에서 링크되므로, 같은 입력은 같은 실행 파일을 만듭니다. profile 에는 `rok.rokon` 의 설정이 없습니다.

```
target/
├── debug/                      # profile (기본)
│   ├── foo                     # executable (src/main.rok)
│   ├── libfoo.a                # library archive (src/lib.rok)
│   ├── foo.rokhir              # library interface (9.4)
│   ├── build/                  # intermediates and --emit output (foo.mlir, ...)
│   └── deps/                   # prebuilt dependencies (lib<dep>.a, <dep>.rokhir, <dep>.fp)
├── release/                    # --release
│   └── ...
├── aarch64-unknown-linux-gnu/  # cross build (--target=<triple>)
│   └── debug/ ...
└── doc/                        # rok doc / rokc doc
```

| 산출물          | 경로                                                              |
|-----------------|-------------------------------------------------------------------|
| 실행 파일       | `target/<profile>/<name>`                                         |
| 라이브러리      | `target/<profile>/lib<name>.a`                                    |
| `--emit=<kind>` | `target/<profile>/build/<name>.<ext>` (`hir` `mir` `mlir` `ll` `s` `o`) |
| 중간 산출물     | `target/<profile>/build/`                                         |
| 의존 패키지     | `target/<profile>/deps/` (`lib<dep>.a`, `<dep>.rokhir`, `<dep>.fp`, 9.2.3) |
| 교차 빌드       | `target/<triple>/<profile>/...` (위와 같은 구조)                  |
| 문서            | `target/doc/`                                                     |
| 증분            | `target/<profile>/incremental/<name>/` (실행 파일의 codegen unit), `.../lib<name>/` (라이브러리의 것), target 루트마다의 `.../<name>-<hash>/` (front end), 9.2.16 |

루트 파일이 둘 다 있는 패키지는 두 산출물 `target/<profile>/<name>` 과 `target/<profile>/lib<name>.a` (+ `<name>.rokhir`) 를 가지며, `-o` 는 실행 파일의 이름을 정합니다.

이름 `foo` 는 패키지 이름입니다 (9.2.2). `-o` 는 산출물 하나의 경로를, `--target-dir=<dir>` 와 환경 변수 `ROKC_TARGET_DIR` 은 `target/` 자체의 위치를 바꿉니다. `rokc clean` / `rok clean` 은 `target/` 전체를 지웁니다. 라이브러리의 ROKHIR 은 archive 옆의 `foo.rokhir` 입니다 ([9.4](904-rokhir-bytecode.md)).

## 9.2.16. 증분 빌드

증분 빌드는 세 수준에서 동작합니다: 입력이 바뀌지 않은 패키지는 빌드하지 않고, 빌드하는 패키지는 바뀌지 않은 파일과 함수의 파싱과 lowering 한 본체를 지난 빌드에서 가져오며 (아래 front end), 그 코드는 codegen unit 으로 컴파일해 바뀐 unit 만 다시 컴파일합니다. `rokc build` / `rokc run` 은 이전 빌드의 fingerprint 와 지금의 것이 같으면 `Fresh` 를 출력하고 빌드를 건너뜁니다. fingerprint 는 다음을 포함합니다.

- 컴파일러의 정체 (ROKHIR producer 와 같은 값), 빌드 옵션, 진단 형식 (`--message-format`, 색), 입력 / 출력 경로, 설정되어 있으면 `ROK_CC`
- 패키지와 의존성의 `src/` 아래 파일, 그 `rok.rokon`, 그리고 빌드가 읽은 그 밖의 모든 파일 (`src/` 밖의 `#[path]` / `include_str!`)
- `include_str!` / `include_bytes!` ([6.11](../006-meta-system/611-include.md)) 가 읽는 파일과 그 내용
- lint 수준 (`-A` / `-W` / `-D`, 그리고 그것을 거친 매니페스트의 `lints`)
- toolchain 이 그 target 용으로 배포하는 런타임 라이브러리, C 런타임 객체, C 라이브러리, 링커 ([9.2.19](#9219-설치-레이아웃)), 미리 빌드한 의존성의 `.a`, MLIR plugin 파일, `-C profile-use` 파일과 profile 런타임. 각각을 크기, 수정 시각, 파일 정체 (inode 와 status change 시각; Windows 에서는 생성 시각) 로 보므로, 수정 시각을 유지한 복사도 변경입니다
- 산출물의 크기, 수정 시각, 파일 정체

빌드가 낸 경고는 lint 경고와 back end 의 것까지 기록해 두며, `Fresh` 인 빌드는 그 경고를 다시 출력합니다.

`--cfg`, `--target`, MLIR plugin, profile 옵션, `-O3` 가 없는 native 빌드는 `lib/rok-cache/` (9.2.19) 의 미리 빌드한 표준 라이브러리를 링크합니다. 폴더는 toolchain, 옵션, 소스의 hash 마다 하나이며, 처음 쓸 때 lock 아래에서 빌드합니다. `-O3` 에서는 표준 라이브러리를 프로그램과 함께 컴파일합니다 (whole-program 최적화). REPL 은 표준 라이브러리를 소스에서 컴파일합니다.

### Front end

패키지의 빌드 (`check`, `build`, `run`, `test`, `bench`, `doc`, `lint`) 는 front end 가 계산한 것을 `target/<profile>/incremental/<name>-<hash>/` 에 둡니다. `<hash>` 는 패키지 루트를 가리키며 (루트 파일이 둘 다 있는 패키지는 target 마다 자기 것을 가지며, `<hash>` 가 그 루트를 가리킴), 빌드 종류 (target, cfg, test 여부, 어떤 라이브러리가 미리 빌드되었는지) 마다 하위 디렉토리가 하나씩 있습니다. 같은 종류의 다음 빌드는 여기서 다음을 가져옵니다.

- 텍스트와 파싱 입력 (빌드의 확장 연산자, 패키지의 키워드, 메타 블록과 `#[literal]` 패턴, cfg) 이 바뀌지 않은 각 파일의 파싱.
- 각 plain 함수 본체의 lowering (이름 해소, 타입 검사, 제네릭 인스턴스화). plain 함수는 제네릭도 `const` 도 아닌 함수, 또는 제네릭이 아닌 `impl` 의 메서드입니다. 그 자신의 텍스트와 위치가 같고, 프로그램의 인터페이스가 같고, lowering 이 읽은 모든 시그니처, 인스턴스, 그 밖의 사실이 지금도 같고, 그 결과가 본체 밖에서 가리키는 모든 위치 앞의 텍스트가 같으면 재사용합니다.
- 인터페이스는 반환 타입을 적었거나 `()` 인 plain 본체와 plain 함수의 매개변수, 반환 타입을 뺀 모든 항목이며, 반환 타입을 추론하는 본체는 그 일부입니다 (호출자가 그 타입을 읽음). 본체를 고치면 그 본체를 다시 lowering 하고, 반환 타입을 추론하는 본체를 고치면 그 타입을 읽은 본체들도 다시 lowering 하며, plain 함수의 시그니처를 고치면 그것과 그 시그니처를 읽은 본체들을 다시 lowering 하며, 항목의 그 밖의 변경 (타입, 트레잇, `impl` 머리, 제네릭 함수, `use`) 은 모든 본체를 다시 lowering 합니다.
- 재사용한 lowering 이 만든 함수, 타입, 인스턴스는 깨끗한 빌드와 같이 번호가 매겨지며, 진단, 프로그램, 그 심볼은 깨끗한 빌드의 것과 같습니다.

시그니처, `impl` 등록, coherence, 프로그램 전체 검사는 모든 빌드에서 실행합니다. 라이브러리의 항목을 모으며 만든 것 (표준 라이브러리와 의존 패키지의 것) 과 라이브러리 파일의 pre-pass 파싱은 모든 빌드에서, 단독 파일의 빌드에서도 toolchain 의 cache (`lib/rok-cache/library/`, toolchain, 빌드 플래그, 라이브러리 소스의 해시마다 파일 하나) 에 보관합니다. `ROKC_INCREMENTAL` 은 이것을 옮기지도 끄지도 않습니다. 보관한 것이 꼭 있어야 하지는 않습니다: 없거나 망가졌거나 오래된 항목은 다시 계산합니다. 패키지 밖의 단독 파일은 `ROKC_INCREMENTAL` 이 디렉토리를 가리키지 않으면 자기 파싱과 본체를 보관하지 않습니다. `ROKC_INCREMENTAL=<dir>` 은 결과를 `<dir>/<name>-<hash>/` 에 두고, `ROKC_INCREMENTAL=0` 은 끕니다. `ROKC_INCR_VERIFY=1` 은 모든 본체를 따로 다시 lowering 해, 재사용한 lowering 이 다르면 빌드를 실패시킵니다. `-v` 는 파싱과 본체를 재사용한 파일 수를 보고합니다.

### Codegen unit

패키지 안에서 `Fresh` 가 아닌 빌드는 컴파일한 코드를 codegen unit 단위로 재사용합니다. rokc 는 패키지의 코드 (실행 파일이든 라이브러리든) 를 unit 으로 나눕니다: 패키지의 모듈마다 하나, 그 인스턴스를 컴파일하는 라이브러리의 모듈마다 하나 (인스턴스는 타입 인자에 따라 여러 unit 에 나뉠 수 있음), 그리고 프로그램에 한 번만 있는 것 (static, C `main`, GC 타입 등록) 을 위한 unit 하나입니다.

- unit 은 같은 target 디렉토리로의 지난 빌드 이후 그 키가 바뀌었을 때만 컴파일합니다. 아니면 `target/<profile>/incremental/<name>/` (라이브러리의 것은 `lib<name>/`) 에 둔 결과를 재사용하고, 그 뒤 모두 링크합니다.
- 익명 파일의 unit 은 패키지 없이 쓰는 target 디렉토리 (`--target-dir`, `ROKC_TARGET_DIR`, 없으면 `<시스템 임시 디렉토리>/rokc-target/`) 의 `[<triple>/]<profile>/incremental/<stem>-<경로의 hash>/` 에 둡니다. 기본 `rokc-target/` 은 캐시입니다: rokc 는 하루에 많아야 한 번, 일주일 동안 어떤 빌드도 쓰지 않은 파일을 지웁니다.
- unit 의 키는 그 함수들의 코드, 다른 unit 에서 쓰는 것의 시그니처와 코드 생성 정보 (함수의 시그니처가 바뀌면 호출자의 unit 을 다시 컴파일), 쓰는 static 과 GC 타입, target, 최적화 수준, 코드 생성 옵션, 컴파일러의 정체, 그리고 `-g` 이면 그 debug info 가 가리키는 소스 파일의 경로를 담습니다 (파일을 더하면 그것을 가리키는 unit 만 다시 컴파일).
- 심볼과 GC type id 는 다른 unit 의 내용에 의존하지 않으므로 (unit 은 GC type id 를 패키지의 공유 unit 이 정의한 심볼에서 읽음; [9.5.7](905-symbol-mangling.md#957-충돌-회피-typehash-의-안정성)), 산출물은 깨끗한 빌드가 만드는 것과 같습니다.
- 최적화 빌드도 unit 사이에서 inline 합니다 (ThinLTO, unit 별 결과도 cache 됨).
- profile-guided 빌드 (`-C profile-generate`, `-C profile-use`), MLIR plugin 이나 pass 를 쓰는 빌드는 통째로 컴파일합니다.
- `-v` 는 재사용하고 컴파일한 unit 의 수를, `-vv` 는 컴파일한 unit 의 이름을 보고합니다.

## 9.2.17. 컴파일러와 런타임의 분리

`rokc` 는 서드파티 crate 를 하나도 쓰지 않으며, 고정된 LLVM (22.1.8) 하나만 정적으로 링크합니다.

파이프라인은 컴파일러와 런타임을 떼어 둡니다. `rokc` 는 런타임 (`librokr.a`) 을 빌드 시점에 참조하지 않습니다.

대신에 `extern "C"` 심볼 ([Extern "C" 심볼](../008-abi/805-extern-symbols.md)) 호출만을 emit 하고, 링크 시점에 가서 사용자 빌드가 `librokr.a` 를 함께 링크하는 방식으로 동작합니다.

이렇게 분리해 둔 덕분에 `rokc` 와 `librokr.a` 가 서로 독립된 SemVer 로 진화할 수 있습니다.

## 9.2.18. 내장 toolchain

`rokc` 는 자기 완결적입니다: MLIR 과 lld 를 포함한 LLVM 22.1.8 이 그 안에 들어 있으며, `PATH` 에서 찾지 않습니다. 외부 컴파일러, 어셈블러, 링커를 실행하지 않습니다. 크로스 컴파일은 내장되어 있습니다: toolchain 은 지원하는 Linux 대상의 C 라이브러리 (musl, glibc link stub) 를 함께 제공합니다.

| 시스템에서 필요한 것   | 대상                                     |
|------------------------|------------------------------------------|
| macOS SDK (Xcode Command Line Tools, 재배포 불가) | Apple 대상     |
| 없음 (제공된 C 라이브러리) | Linux 대상                      |
| 없음 (제공된 module definition 으로 만든 import library, UCRT) | Windows 대상 |
| `ROK_CC` (선택)        | 최종 링크를 대체 (9.2.14)                |

## 9.2.19. 설치 레이아웃

설치 레이아웃은 rustup / cargo 식입니다. `ROK_HOME` 의 기본값은 `~/.rok` (Windows 에서는 `%USERPROFILE%\.rok`) 입니다.

```
$ROK_HOME/
├── bin/                         # proxy: 활성 toolchain 의 같은 이름 실행 파일을 실행
│   ├── rok
│   ├── rokc
│   ├── rokls
│   └── <name>                   # rok install 이 설치한 실행 파일 (9.10.5a)
├── toolchains/
│   └── <version>-<triple>/      # 자기 완결 toolchain
│       ├── bin/{rok, rokc, rokls}
│       └── lib/
│           ├── librokr.a        # 런타임
│           ├── libclang.{so,dylib,dll}  # libclang, rokc bindgen 만 로드 (std 1.7.6)
│           ├── clang/<major>/include/   # clang 의 내장 헤더 (그 resource 디렉토리)
│           ├── rok/lldb/rok_lldb.py      # -g 프로그램용 lldb formatter (9.9.6)
│           ├── rok/targets/<triple>/   # --target 이 링크하는 것: 런타임 (librokr.a / rokr.lib), Linux C 라이브러리,
│           │                           #   libclang_rt.profile.a (Windows clang_rt.profile.lib), host 의 것 포함
│           └── rok/{core, std, bare}/src/   # 표준 라이브러리 소스: tier 마다 패키지처럼 놓이며 루트는 src/lib.rok (5.4)
├── cache/                       # git, registry, native, 스크립트 (9.10.4)
├── envs/<name>/, envs/CURRENT   # 이름 있는 환경: rok.rokon, rok.lock, target/ (9.10.5)
├── installs.rokon               # rok install 이 bin/ 에 둔 것과 그 출처
└── config.rokon                 # Config { registry, toolchain }
```

`rokc` 는 표준 라이브러리와 런타임을 자기 실행 파일 기준 (sysroot = 실행 파일의 `..`) 에서 찾으며, `ROK_HOME` 을 보지 않습니다. 표준 라이브러리의 각 tier 는 라이브러리 패키지처럼 놓입니다: 루트 모듈은 `lib/rok/<tier>/src/lib.rok` 이고 모듈들은 `lib/rok/<tier>/src/` 아래에 있습니다. 구현 저장소에서 빌드한 `rokc` 는 같은 레이아웃을 저장소의 `library/<tier>/src/` 에서 읽습니다. 자신의 cache 는 toolchain 의 `lib/rok-cache/` 입니다 (저장소 빌드는 `target/<profile>/rok-cache/`). 여기에 미리 빌드한 표준 라이브러리를 toolchain, 빌드 옵션, 소스를 키로 두고 (9.2.16), `.facts/` 에 캐시한 대상 data layout 과 도구 정보를 둡니다. 활성 toolchain 의 선택은 [9.10.6](910-rok-frontend.md#9106-toolchain) 이 정합니다.

Linux (glibc) toolchain 은 자신이 만드는 실행 파일과 같이 glibc 2.34 이상이 필요합니다 (9.2.14).

릴리스에는 MLIR plugin 개발 archive (`rok-mlir-dev-<version>-<triple>`: 고정 LLVM / MLIR 헤더와, Windows 에서는 `rokc.lib`) 가 함께 올 수 있습니다 ([stdlib 1.10.2](../../refs-standard-libraries/100-bare/110-mlir-plugin.md)).
