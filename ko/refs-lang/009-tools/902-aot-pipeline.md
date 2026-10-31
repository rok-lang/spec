# 9.2. AOT 빌드 파이프라인

이 장은 `rokc build`의 단계별 흐름을 정의합니다. [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md)과 [컴파일 파이프라인](../000-meta/002-pipeline.md)에서 그린 큰 그림을 구체적인 명령과 파일 흐름으로 옮깁니다.

## 9.2.1. 단계

빌드는 다음 11단계를 거칩니다. 모든 단계가 `rokc` 안에서 실행됩니다.

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

> package discovery는 패키지 발견, dependency resolve는 의존성 해소, source parse는 소스 파싱, name / visibility는 이름 / 가시성 해소, type check는 타입 검사, rok-hir build는 rok-hir 생성, specialize는 단형화입니다. collect call-sites + monomorphize는 호출 지점 수집과 단형화, rok-mir build는 rok-mir 생성, code generation은 오브젝트 코드 생성, archive / link는 아카이브 생성이나 링크입니다. artifact는 산출물, executable은 실행 파일입니다.

각 단계의 입력, 출력, 책임은 분명히 나뉩니다.

## 9.2.2. 패키지 발견

`rokc build [path]`의 path에서 시작해 디렉터리를 한 단계씩 올라가며 `src/main.rok`이나 `src/lib.rok`이 있는 디렉터리를 찾습니다. path가 파일이면 그 파일이 있는 디렉터리부터 찾습니다.

찾으면 그 디렉터리가 패키지 루트가 되고, 그 `src/`가 패키지의 소스가 됩니다([소스 단위](../000-meta/003-source-units.md) 참고). 그 밖의 경우는 다음과 같습니다.

- path가 찾은 패키지의 `src/` 밖에 있는 파일(패키지 소스 옆의 스크립트나 도구)이면, 그 파일 하나로 된 익명 실행 패키지입니다. 위로 올라가도 패키지가 없을 때도 같습니다([4.16.8](../004-functions/416-entry-point.md#4168-단일-파일-실행)). 그 파일의 서브모듈은 파일 옆에 있습니다.
- path가 디렉터리인데 패키지가 없으면 에러입니다.

패키지 이름은 패키지 디렉터리의 이름이고, 산출물 이름이 됩니다(9.2.15). 익명 패키지는 파일 이름에서 `.rok`을 뗀 것이 이름입니다. 루트 파일이 둘 다 있는 패키지는 두 타깃으로 빌드합니다. 먼저 라이브러리를 빌드하고, 그다음 그 라이브러리를 의존 패키지 `<name>`으로 마운트하는 실행 파일을 빌드합니다([0.3.1](../000-meta/003-source-units.md#한-패키지의-라이브러리와-실행-파일)).

## 9.2.3. 의존성 해소

이 단계는 [경로](../005-modules/507-paths.md)와 [`use`](../005-modules/505-use.md)에 나오는 외부 패키지 참조(`use external_pkg::...`)를 처리합니다.

`rokc`는 의존성을 스스로 해소하지 않습니다. 프런트엔드 `rok`([9.10](910-rok-frontend.md))이 `rok.rokon` / `rok.lock`을 따라 path / git 의존 패키지를 준비하고, 각 패키지의 루트를 `--extern`으로 `rokc`에 넘깁니다.

```
rokc build --extern util=../util --extern parser=target/debug/deps/libparser.a
```

`--extern name=<경로>`는 의존 패키지 하나를 최상위 경로 `name`에 마운트합니다. `name`은 `rok.rokon`의 `deps` 키에서 `-`를 `_`로 바꾼 식별자입니다. 경로는 두 형태입니다.

| 경로                              | 의존 패키지                                                                  |
|-----------------------------------|------------------------------------------------------------------------------|
| `<dir>` / `<dir>/src/lib.rok`     | 소스. 그 `src/lib.rok`을 같은 빌드에서 함께 컴파일                           |
| `<name>.rokhir` / `lib<name>.a`     | 미리 빌드한 패키지. 둘 중 하나를 주면 같은 디렉터리의 짝 파일을 찾음. ROKHIR([9.4](904-rokhir-bytecode.md))을 읽어 선언과 제네릭을 얻고 아카이브를 링크 |

- 사용자 코드는 `use name::..`로 그 패키지의 항목을 씁니다.
- 의존 패키지 안의 `pkg::`는 그 의존 패키지 자신을 가리킵니다.
- 의존 패키지는 별개의 패키지입니다. 바깥에서는 `pub` 항목만 쓸 수 있고, `pub(pkg)` / `pub(sub)` / 비공개 항목은 쓸 수 없습니다([5.7.14](../005-modules/507-paths.md#5714-패키지-사이-경로)).
- 사용자 패키지의 모듈이 의존 패키지와 같은 이름이면 에러입니다.

`src/lib.rok`이 있는 패키지를 `rokc build`로 빌드하면 `lib<name>.a`와 `<name>.rokhir`이 나옵니다. `src/main.rok`이 있으면 실행 파일 `<name>`이 나옵니다. 둘 다 있는 패키지는 셋을 모두 만들고, 실행 파일을 그 라이브러리에 링크합니다. `--name <n>`은 패키지 이름(산출물 이름과 마운트 이름)을 정합니다. 기본값은 디렉터리 이름입니다.

### 미리 빌드한 의존 패키지

`rokc`는 소스로 받은 `--extern`을 주어진 순서대로 **한 번씩** 미리 빌드해 `target/[<triple>/]<profile>/deps/`에 둡니다. 각 의존 패키지는 앞서 빌드한 의존 패키지를 모두 봅니다. 이후 컴파일은 소스 대신 이 `lib<name>.a`와 `<name>.rokhir`을 씁니다.

```
target/<profile>/deps/
├── lib<name>.a
├── <name>.rokhir
└── <name>.fp               # fingerprint
```

fingerprint가 같으면 다시 빌드하지 않습니다. fingerprint에는 다음이 들어갑니다.

- `rokc`의 정체 (ROKHIR producer와 같은 값)
- 빌드 옵션: `--release` / `-O*`, `-g`, `--target`, `--cfg` (`feature` 키 제외), `--bootstrap`
- 그 의존 패키지의 feature (`--extern-feature`)
- 그 의존 패키지의 `src/` 아래 모든 파일과 `rok.rokon`
- 그보다 먼저 빌드한 의존 패키지의 fingerprint

미리 빌드한 패키지의 인터페이스는 소비하는 쪽의 feature가 아니라, 그 패키지를 빌드할 때 쓴 feature(그 ROKHIR meta, [9.4.6](904-rokhir-bytecode.md#946-meta))로 읽습니다.

컴파일하는 모든 명령이 이 경로를 씁니다. `build` / `run` / `test` / `bench` / `check` / `lint` / `doc` / `lsp` / `repl` / `jupyter`, 단일 파일 스크립트, 이름 있는 환경이 모두 해당합니다. REPL 호스트도 미리 빌드한 아카이브를 링크합니다. 패키지도 `--target-dir`도 없으면 시스템 임시 디렉터리의 단일 파일 캐시 아래에 둡니다(9.2.16).

표준 라이브러리는 소스에서 빌드하고([9.3.6](903-package-archive.md#936-표준-라이브러리)), 미리 빌드한 상태로 `rokc` 캐시(9.2.19)에 둡니다.

## 9.2.4. 소스 파싱

컴파일러는 [소스 파일](../001-lexical/101-source-files.md)부터 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)까지의 어휘 / 구문 규칙에 따라 모든 `.rok` 파일을 AST로 파싱합니다.

[모듈](../005-modules/504-modules.md)에서 정의한 디렉터리 모듈은 그 안의 모든 .rok 파일을 합쳐 한 모듈의 본체로 다룹니다.

패키지 루트의 매니페스트 `rok.rokon`([9.10.3](910-rok-frontend.md#9103-rokrokon--roklock))은 모듈 파일이 아닙니다. 매니페스트는 [ROKON](../../refs-standard-libraries/300-std/309-rokon.md) 값 하나(`Package { .. }`)로 읽습니다. 최상위 문 규칙을 적용하지 않고 컴파일하지도 않습니다.

### scanner pre-pass

[`#[keyword]` 매직](../006-meta-system/606-keyword-magic.md)의 scanner pre-pass도 이 단계에서 실행합니다. 먼저 모든 .rok 파일을 얕게 스캔해 `#[keyword(...)]`, `#[infix(...)]`, `#[literal(...)]` 등록 정보를 모읍니다. 그다음 그 정보를 정식 렉서와 파서에 넘깁니다.

## 9.2.5. 이름 / 가시성 해소

[`use`(가져오기)](../005-modules/505-use.md), [가시성](../005-modules/506-visibility.md), [경로](../005-modules/507-paths.md), [재내보내기(`pub use`)](../005-modules/508-re-export.md)의 규칙에 따라 모든 경로와 식별자를 항목에 연결합니다.

이 단계는 `rok::core::**` 묵시적 가져오기를 적용합니다([표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md) 참고). `rok::bare::*`와 `rok::std::*`는 명시적으로 가져와야 하므로 여기서 빠집니다. 또 가시성을 검사해 허용되지 않는 접근을 에러로 보고하고, 가져오기 충돌도 검사합니다.

## 9.2.6. 타입 검사

[2. 타입 시스템](../002-types/README.md)의 규칙으로 모든 식의 타입을 결정합니다.

이 과정에서 각 슬롯을 점진 슬롯으로 둘지 정적 슬롯으로 둘지 정합니다([`any` 슬롯](../002-types/202-any-slot.md) 참고). 트레잇 바운드를 검사하고([제네릭](../002-types/210-generics.md) 참고), 분기의 결과 타입을 확정합니다([3. 표현식](../003-expressions/README.md) 참고). 합 타입과 `any` 슬롯의 좁히기도 여기서 처리합니다.

## 9.2.7. rok-hir 생성

타입 검사가 끝난 AST를 rok-hir([HIR](../000-meta/007-glossary.md#hir-high-level-ir--mir-mid-level-ir)) 자료구조로 변환합니다.

HIR은 타입 정보를 많이 담은 중간 표현이고, 위 4~5단계의 결과를 모두 반영합니다. 구조를 압축해서 같은 타입과 식별자는 표현 하나를 함께 씁니다.

## 9.2.8. specialize: 단형화

[제네릭](../002-types/210-generics.md)과 [const 제네릭](../002-types/211-const-generics.md)을 단형화합니다.

```
1. 모든 호출 지점 수집.
2. 각 호출의 구체 타입 튜플 결정.
3. (item_id, type_tuple) 마다 별개의 specialized HIR 생성.
4. 캐시. 같은 (item_id, type_tuple) 는 한 번만.
```

프로그램 루트(`main`, 내보낸 심볼, `#[export_name]` 항목, 런타임이 참조하는 심볼)에서 닿을 수 없는 함수는 코드를 생성하지 않습니다. 라이브러리 빌드도 같습니다. 패키지의 `pub` 항목과 그 루트에서 닿지 않는 함수는 아카이브에 넣지 않습니다.

이 단계 덕분에 [AOT](../000-meta/007-glossary.md#aot-ahead-of-time)는 vtable 없이 직접 호출할 수 있습니다([트레잇-as-타입 금지](../002-types/212-trait-as-type-banned.md) 참고).

## 9.2.9. rok-mir 생성

[단형화](../000-meta/007-glossary.md#specialize)한 HIR을 rok-mir(MIR)로 변환합니다.

MIR은 HIR보다 한 단계 낮은 IR입니다. 이 단계에서 연산자 문법 설탕을 모두 풀고([연산자 디스패치](../003-expressions/301-operator-dispatch.md) 참고), 패턴 매칭의 결정 트리를 정합니다([`match` 표현식](../003-expressions/312-match-expression.md) 참고). 클로저와 코루틴의 프레임 레이아웃도 확정합니다.

## 9.2.10. 백엔드 선택

기본으로는 최적화 수준이 코드 생성기를 고릅니다([0.2.4](../000-meta/002-pipeline.md#024-백엔드)). `-C backend=native|llvm`이나 `ROK_BACKEND`로 바꿀 수 있습니다.

| 수준                     | 백엔드       |
|--------------------------|--------------|
| `-O0`                    | native       |
| `-O1` `-O2` `-O3` `-Os`  | MLIR → LLVM  |

native 백엔드는 {aarch64, x86_64} × {macOS, Linux}를 지원합니다. 그 밖의 타깃, [`__mlir_op` / `__mlir_type`](../../refs-standard-libraries/100-bare/109-mlir.md)을 쓰는 모듈, MLIR plugin을 쓰는 빌드, `-C profile-generate` / `-C profile-use` 빌드는 어느 수준에서나 MLIR → LLVM 백엔드를 씁니다. REPL도 같은 선택을 따릅니다(9.6.1). 두 백엔드는 [8](../008-abi/README.md)의 런타임 ABI를 따릅니다. 그래서 어느 쪽 오브젝트든 함께 링크할 수 있고, 다른 수준으로 미리 빌드한 의존 패키지도 마찬가지입니다.

## 9.2.11. 네이티브 백엔드

native 백엔드는 macOS와 Linux의 aarch64, x86_64를 대상으로 합니다. 중간 IR 없이 rok-mir에서 바로 기계어와 오브젝트 파일(Mach-O, ELF)을 만듭니다. 목표는 컴파일 속도이고, debug 빌드와 REPL 셀에는 LLVM이 필요 없습니다. `-g`를 주면 DWARF 줄 정보와 프레임 정보를 냅니다.

## 9.2.12. MLIR → LLVM 백엔드

rok-mir를 표준 MLIR로 낮춥니다([8.7](../008-abi/807-mlir-dialects.md)). `rokc`는 그 MLIR을 프로세스 안에서 파싱하고, LLVM 다이얼렉트까지 MLIR 패스 파이프라인을 돌립니다. 그 결과를 LLVM IR로 옮겨 요청한 수준으로 최적화하고, 타깃 트리플의 오브젝트를 만듭니다. IR이 1 MiB를 넘는 모듈은 여러 부분으로 나누어 병렬로 코드를 생성합니다. 이 모든 과정은 외부 도구 없이 프로세스 안에서 실행됩니다. 중간 형태는 `--emit`일 때만 파일로 씁니다([9.9.1](909-debug-tools.md#991-중간-산출물-출력---emit)).

```
target/debug/build/foo.o
```

## 9.2.13. 아카이브

라이브러리 패키지는 codegen unit(9.2.16)마다 오브젝트를 하나씩 만들어, `rokc`가 직접 쓰는 표준 `ar` 아카이브에 넣습니다. 그 옆에는 인터페이스 파일 `foo.rokhir`([ROKHIR 형식](904-rokhir-bytecode.md))을 씁니다.

```
target/debug/libfoo.a
target/debug/foo.rokhir
```

## 9.2.14. 링킹(실행 파일의 경우)

실행 파일은 내장 lld로 링크합니다.

```
foo.o + <deps>/lib*.a + <sysroot>/lib/librokr.a + target libc  →  target/debug/foo
```

Mach-O 실행 파일은 macOS SDK(Xcode Command Line Tools)에 대해 링크합니다. Windows 실행 파일은 두 가지에 대해 링크합니다. 하나는 `rokc`가 함께 싣고 있는 module definition(mingw-w64에서 유래)으로 만든 import library이고, 다른 하나는 UCRT 위에 올린 Rok 자체 startup입니다. MSVC 도구나 Windows SDK는 필요 없습니다. Rok의 Windows startup은 `main` 전에 `.CRT$XCA`..`.CRT$XCZ`의 C initializer를 실행합니다. x86 Windows의 import 이름은 mingw-w64의 `lib32` 정의에서 오고 stdcall `@N` 장식을 씁니다. 이미지는 `/SAFESEH:NO`로 링크합니다. Linux 타깃은 native든 cross든 언제나 `<sysroot>/lib/rok/targets/<triple>/`에 들어 있는 C 라이브러리에 대해 링크합니다. musl은 정적으로 링크하고, glibc는 link stub으로 링크합니다. 그래서 실행 파일에는 glibc 2.34 이상이 필요합니다. 다른 라이브러리는 `-L`과 `LIBRARY_PATH`로 찾습니다. 32비트 glibc 타깃의 동적 로더는 `/lib/ld-linux.so.2`(i686)와 `/lib/ld-linux-armhf.so.3`(armv7hf)입니다. `ROK_CC`를 설정하면 그 C 컴파일러 드라이버가 최종 링크를 대신합니다(`$ROK_CC <objects> <archives> -o <out>`). 이 대체는 선택 사항입니다. `--native-runtime`으로 준 공유 라이브러리는 실행 파일 옆에 복사하고, 실행 파일에는 rpath `@loader_path`(Mach-O) / `$ORIGIN`(ELF)을 넣습니다. Windows는 실행 파일이 있는 디렉터리에서 DLL을 읽습니다. 링커는 실행 파일에서 쓰지 않는 부분을 걷어 냅니다(dead-strip). 참조되지 않는 함수와 데이터를 지우고, `release` 빌드는 똑같은 함수도 하나로 합칩니다([9.2.15](#9215-산출물)). Apple 타깃에서 실행 파일에 디버그 정보가 있으면 내장 `dsymutil`(`rokc`에 링크된 LLVM의 것)이 실행 파일 옆에 `.dSYM` 번들을 씁니다.

## 9.2.15. 산출물

산출물은 cargo처럼 **프로필별** 디렉터리에 놓입니다. 프로필은 `debug`(기본값)와 `release`입니다. 최적화 단계가 `-O2` / `-O3` / `-Os`(`--release` 포함)이면 `release`입니다.

두 프로필은 cargo와 같은 방식으로 다릅니다.

| Profile | 최적화 | 디버그 정보 | 링크 |
|---|---|---|---|
| `debug` | `-O0`, `-O1` | 있음 | 참조되지 않는 코드와 데이터 제거 |
| `release` | `-O2`, `-O3`, `-Os` | `-g`일 때만 | 똑같은 함수도 하나로 합침(오브젝트가 주소를 쓴다고 표시한 함수는 제외). `-g`가 없으면 로더에 필요한 심볼만 남김 |

디버그 정보는 빌드가 만드는 실행 파일이나 아카이브(`build`, `run`, `test`)에 들어갑니다. `--emit` 출력이나 REPL, Jupyter 셀에는 들어가지 않습니다. panic은 어느 프로필에서든 소스 위치를 알립니다. 빌드의 오브젝트는 출력 옆 한 곳에서 링크하므로, 입력이 같으면 같은 실행 파일이 나옵니다. `rok.rokon`에는 프로필 설정이 없습니다.

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
| 증분            | `target/<profile>/incremental/<name>/` (실행 파일의 codegen unit), `.../lib<name>/` (라이브러리의 것), 타깃 루트마다의 `.../<name>-<hash>/` (프런트엔드), 9.2.16 |

루트 파일이 둘 다 있는 패키지에는 산출물이 두 개 있습니다. `target/<profile>/<name>`과 `target/<profile>/lib<name>.a`(+ `<name>.rokhir`)입니다. `-o`는 실행 파일의 이름을 정합니다.

이름 `foo`는 패키지 이름입니다(9.2.2). `-o`는 산출물 하나의 경로를 바꿉니다. `--target-dir=<dir>`와 환경 변수 `ROKC_TARGET_DIR`은 `target/` 자체의 위치를 바꿉니다. `rokc clean` / `rok clean`은 `target/` 전체를 지웁니다. 라이브러리의 ROKHIR은 아카이브 옆의 `foo.rokhir`입니다([9.4](904-rokhir-bytecode.md)).

## 9.2.16. 증분 빌드

증분 빌드는 세 수준에서 동작합니다. 첫째, 입력이 바뀌지 않은 패키지는 빌드하지 않습니다. 둘째, 빌드하는 패키지는 바뀌지 않은 파일의 파싱 결과와 바뀌지 않은 함수의 낮춘 본체를 지난 빌드에서 가져옵니다(아래 프런트엔드). 셋째, 코드는 codegen unit 단위로 컴파일하고, 바뀐 unit만 다시 컴파일합니다. `rokc build` / `rokc run`은 이전 빌드의 fingerprint가 지금과 같으면 `Fresh`를 출력하고 빌드를 건너뜁니다. fingerprint에는 다음이 들어갑니다.

- 컴파일러의 정체(ROKHIR producer와 같은 값), 빌드 옵션, 진단 형식(`--message-format`, 색), 입력 / 출력 경로, 설정되어 있으면 `ROK_CC`
- 패키지와 의존성의 `src/` 아래 파일, 그 `rok.rokon`, 그리고 빌드가 읽은 그 밖의 모든 파일(`src/` 밖의 `#[path]` / `include_str!`)
- `include_str!` / `include_bytes!`([6.11](../006-meta-system/611-include.md))가 읽는 파일과 그 내용
- 린트 수준(`-A` / `-W` / `-D`, 그리고 이 옵션을 거친 매니페스트의 `lints`)
- toolchain이 그 타깃용으로 배포하는 런타임 라이브러리, C 런타임 오브젝트, C 라이브러리, 링커([9.2.19](#9219-설치-레이아웃)), 미리 빌드한 의존성의 `.a`, MLIR plugin 파일, `-C profile-use` 파일과 프로필 런타임. 각 파일은 크기, 수정 시각, 파일 정체(inode와 상태 변경 시각, Windows에서는 생성 시각)로 비교하므로, 수정 시각을 유지한 복사도 변경으로 봄
- 산출물의 크기, 수정 시각, 파일 정체

빌드가 낸 경고는 린트 경고와 백엔드 경고까지 기록해 둡니다. `Fresh`인 빌드는 그 경고를 다시 출력합니다.

`--cfg`, `--target`, MLIR plugin, 프로필 옵션, `-O3`이 없는 native 빌드는 `lib/rok-cache/`(9.2.19)의 미리 빌드한 표준 라이브러리를 링크합니다. 폴더는 toolchain, 옵션, 소스의 해시마다 하나이고, 처음 쓸 때 잠금을 걸고 빌드합니다. `-O3`에서는 표준 라이브러리를 프로그램과 함께 컴파일합니다(전체 프로그램 최적화). REPL은 표준 라이브러리를 소스에서 컴파일합니다.

### 프런트엔드

패키지 빌드(`check`, `build`, `run`, `test`, `bench`, `doc`, `lint`)는 프런트엔드가 계산한 것을 `target/<profile>/incremental/<name>-<hash>/`에 둡니다. `<hash>`는 패키지 루트를 가리킵니다. 루트 파일이 둘 다 있는 패키지는 타깃마다 자기 디렉터리가 있고, 그 `<hash>`가 각 루트를 가리킵니다. 빌드 종류(타깃, cfg, 테스트 여부, 어떤 라이브러리를 미리 빌드했는지)마다 하위 디렉터리가 하나씩 있습니다. 같은 종류의 다음 빌드는 여기서 다음을 가져옵니다.

- 텍스트와 파싱 입력(빌드의 확장 연산자, 패키지의 키워드, 메타 블록과 `#[literal]` 패턴, cfg)이 바뀌지 않은 각 파일의 파싱 결과.
- 각 plain 함수 본체를 낮춘 결과(이름 해소, 타입 검사, 제네릭 인스턴스화). plain 함수는 제네릭도 `const`도 아닌 함수이거나, 제네릭이 아닌 `impl`의 메서드입니다. 재사용하려면 본체 자신의 텍스트와 위치가 같고, 프로그램의 인터페이스가 같아야 합니다. 낮출 때 읽은 모든 시그니처, 인스턴스, 그 밖의 사실도 지금과 같아야 합니다. 또 그 결과가 본체 밖에서 가리키는 모든 위치 앞의 텍스트가 같아야 합니다.
- 인터페이스는 반환 타입을 적었거나 `()`인 plain 본체, 그리고 plain 함수의 매개변수와 반환 타입을 뺀 모든 항목입니다. 반환 타입을 추론하는 본체는 인터페이스에 속합니다. 호출자가 그 타입을 읽기 때문입니다. 본체를 고치면 그 본체를 다시 낮춥니다. 반환 타입을 추론하는 본체를 고치면 그 타입을 읽은 본체도 다시 낮춥니다. plain 함수의 시그니처를 고치면 그 함수와, 그 시그니처를 읽은 본체를 다시 낮춥니다. 항목의 그 밖의 변경(타입, 트레잇, `impl` 머리, 제네릭 함수, `use`)은 모든 본체를 다시 낮춥니다.
- 재사용한 결과가 만든 함수, 타입, 인스턴스는 깨끗한 빌드와 같은 번호를 받습니다. 진단, 프로그램, 심볼도 깨끗한 빌드와 같습니다.

시그니처, `impl` 등록, coherence, 프로그램 전체 검사는 모든 빌드에서 실행합니다. 라이브러리 항목을 모으며 만든 것(표준 라이브러리와 의존 패키지의 것)과 라이브러리 파일의 pre-pass 파싱 결과는 toolchain 캐시(`lib/rok-cache/library/`)에 보관합니다. 캐시 파일은 toolchain, 빌드 플래그, 라이브러리 소스의 해시마다 하나이고, 단독 파일 빌드를 포함해 모든 빌드가 이 캐시를 씁니다. `ROKC_INCREMENTAL`은 이 캐시를 옮기지도 끄지도 않습니다. 보관한 것이 꼭 있어야 하지는 않습니다. 없거나 망가졌거나 오래된 항목은 다시 계산합니다. 패키지 밖의 단독 파일은 `ROKC_INCREMENTAL`이 디렉터리를 가리킬 때만 자기 파싱 결과와 본체를 보관합니다. `ROKC_INCREMENTAL=<dir>`은 결과를 `<dir>/<name>-<hash>/`에 두고, `ROKC_INCREMENTAL=0`은 보관을 끕니다. `ROKC_INCR_VERIFY=1`은 모든 본체를 따로 다시 낮추고, 재사용한 결과가 다르면 빌드를 실패로 끝냅니다. `-v`는 파싱 결과와 본체를 재사용한 파일 수를 보고합니다.

### 코드 생성 단위

패키지 안에서 `Fresh`가 아닌 빌드는 컴파일한 코드를 codegen unit 단위로 재사용합니다. rokc는 패키지의 코드(실행 파일이든 라이브러리든)를 unit으로 나눕니다. 패키지의 모듈마다 하나, 그 인스턴스를 컴파일하는 라이브러리의 모듈마다 하나, 그리고 프로그램에 한 번만 있는 것(static, C `main`, GC 타입 등록)을 위한 unit이 하나입니다. 인스턴스는 타입 인자에 따라 여러 unit에 나뉠 수 있습니다.

- unit은 같은 타깃 디렉터리에 출력한 지난 빌드 이후 키가 바뀌었을 때만 컴파일합니다. 키가 그대로면 `target/<profile>/incremental/<name>/`(라이브러리는 `lib<name>/`)에 둔 결과를 재사용합니다. 그 뒤 모두 링크합니다.
- 익명 파일의 unit은 패키지 없이 쓰는 타깃 디렉터리(`--target-dir`, `ROKC_TARGET_DIR`, 없으면 단일 파일 캐시)의 `[<triple>/]<profile>/incremental/<stem>-<경로의 hash>/`에 둡니다. 단일 파일 캐시는 Unix에서 `<시스템 임시 디렉토리>/rokc-target-<uid>/`(`<uid>`는 유효 사용자 ID)이고, Windows에서는 `%TEMP%\rokc-target\`입니다. Windows의 임시 디렉터리는 이미 사용자마다 따로 있습니다. 이 디렉터리는 캐시입니다. rokc는 하루에 많아야 한 번, 일주일 동안 어떤 빌드도 쓰지 않은 파일을 지웁니다.
- rokc는 단일 파일 캐시에서 찾은 것을 실행하므로, Unix에서는 이 디렉터리를 사용자 혼자만 쓰게 둡니다. rokc는 모드 `0700`으로 만들고, 쓰기 전에 심볼릭 링크가 아닌 디렉터리인지, 유효 사용자의 것인지, 그룹과 다른 사용자의 권한 비트가 없는지 확인합니다. 아니면 명령은 종료 코드 `1`과 ``rokc: error: <dir> is not private to you (owner uid N, mode 0755); remove it or set ROKC_TARGET_DIR``로 실패합니다. `--target-dir`와 `ROKC_TARGET_DIR`는 주어진 그대로 씁니다.
- unit의 키에는 그 함수들의 코드, 다른 unit에서 가져다 쓰는 것의 시그니처와 코드 생성 정보, 쓰는 static과 GC 타입, 타깃, 최적화 수준, 코드 생성 옵션, 컴파일러의 정체가 들어갑니다. `-g`이면 디버그 정보가 가리키는 소스 파일의 경로도 들어갑니다. 그래서 함수의 시그니처가 바뀌면 호출자의 unit을 다시 컴파일하고, 파일을 더하면 그 파일을 가리키는 unit만 다시 컴파일합니다.
- 심볼과 GC type id는 다른 unit의 내용에 의존하지 않습니다. unit은 GC type id를 패키지의 공유 unit이 정의한 심볼에서 읽습니다([9.5.7](905-symbol-mangling.md#957-충돌-회피-typehash의-안정성)). 그래서 산출물은 깨끗한 빌드가 만드는 것과 같습니다.
- 최적화 빌드도 unit 사이에서 인라인합니다(ThinLTO, unit별 결과도 캐시함).
- 프로필 기반 빌드(`-C profile-generate`, `-C profile-use`)와 MLIR plugin이나 pass를 쓰는 빌드는 통째로 컴파일합니다.
- `-v`는 재사용한 unit과 컴파일한 unit의 수를 보고하고, `-vv`는 컴파일한 unit의 이름을 보고합니다.

## 9.2.17. 컴파일러와 런타임의 분리

`rokc`는 서드파티 크레이트를 하나도 쓰지 않습니다. 고정된 LLVM(22.1.8) 하나만 정적으로 링크합니다.

파이프라인은 컴파일러와 런타임을 떼어 둡니다. `rokc`는 빌드할 때 런타임(`librokr.a`)을 참조하지 않습니다.

대신 `extern "C"` 심볼([Extern "C" 심볼](../008-abi/805-extern-symbols.md))을 호출하는 코드만 생성합니다. 링크할 때 사용자 빌드가 `librokr.a`를 함께 링크합니다.

이렇게 나누어 두었으므로 `rokc`와 `librokr.a`는 서로 독립된 SemVer로 발전할 수 있습니다.

## 9.2.18. 내장 toolchain

`rokc`는 자기 완결적입니다. MLIR과 lld를 포함한 LLVM 22.1.8이 그 안에 들어 있고, `PATH`에서 찾지 않습니다. 외부 컴파일러, 어셈블러, 링커를 실행하지 않습니다. 크로스 컴파일도 내장되어 있습니다. toolchain에는 지원하는 Linux 타깃의 C 라이브러리(musl, glibc link stub)가 함께 들어 있습니다.

| 시스템에서 필요한 것   | 타깃                                     |
|------------------------|------------------------------------------|
| macOS SDK (Xcode Command Line Tools, 재배포 불가) | Apple 타깃     |
| 없음 (제공된 C 라이브러리) | Linux 타깃                      |
| 없음 (제공된 module definition으로 만든 import library, UCRT) | Windows 타깃 |
| `ROK_CC` (선택)        | 최종 링크를 대체 (9.2.14)                |

## 9.2.19. 설치 레이아웃

설치 레이아웃은 rustup / cargo 식입니다. `ROK_HOME`의 기본값은 `~/.rok`(Windows에서는 `%USERPROFILE%\.rok`)입니다.

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

`rokc`는 표준 라이브러리와 런타임을 자기 실행 파일 기준으로 찾습니다(sysroot = 실행 파일의 `..`). `ROK_HOME`은 보지 않습니다. 표준 라이브러리의 각 계층은 라이브러리 패키지처럼 놓입니다. 루트 모듈은 `lib/rok/<tier>/src/lib.rok`이고 모듈은 `lib/rok/<tier>/src/` 아래에 있습니다. 구현 저장소에서 빌드한 `rokc`는 같은 레이아웃을 저장소의 `library/<tier>/src/`에서 읽습니다. 자기 캐시는 toolchain의 `lib/rok-cache/`입니다(저장소 빌드는 `target/<profile>/rok-cache/`). 이 캐시에는 toolchain, 빌드 옵션, 소스를 키로 미리 빌드한 표준 라이브러리가 있고(9.2.16), `.facts/`에는 캐시한 타깃 data layout과 도구 정보가 있습니다. 활성 toolchain은 [9.10.6](910-rok-frontend.md#9106-toolchain)이 정한 대로 고릅니다.

Linux(glibc) toolchain은 자신이 만드는 실행 파일과 마찬가지로 glibc 2.34 이상이 필요합니다(9.2.14).

릴리스에는 MLIR plugin 개발 아카이브(`rok-mlir-dev-<version>-<triple>`: 고정 LLVM / MLIR 헤더와, Windows에서는 `rokc.lib`)가 함께 올 수 있습니다([stdlib 1.10.2](../../refs-standard-libraries/100-bare/110-mlir-plugin.md)).
