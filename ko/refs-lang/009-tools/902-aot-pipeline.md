# 9.2. AOT 빌드 파이프라인

이 장에서는 `rokc build` 의 단계별 흐름을 정의합니다. [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 과 [컴파일 파이프라인](../000-meta/002-pipeline.md) 에서 제시한 큰 그림을 구체적인 명령과 파일 흐름으로 구체화하여 명시 표기합니다.

## 9.2.1. 단계

전체 단계는 다음과 같이 13 단계로 진행됩니다.

```
1. package discovery       (find package root)
2. dependency resolve      (resolve deps to their .a)
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

`rokc build [path]` 의 path 부터 상위 방향으로 탐색하여 `main.rok` 또는 `lib.rok` 을 찾습니다. 만약 `[path]` 자체가 `lib.rok` 이거나 `main.rok` 이라면 그것이 곧 패키지가 되고, 그 외의 경우에는 디렉토리를 위로 한 단계씩 올라가며 찾습니다.

찾았다면 그 디렉토리가 패키지 루트가 됩니다 ([소스 단위](../000-meta/003-source-units.md) 참고).

## 9.2.3. 의존성 해소

이 단계에서는 [경로](../005-modules/507-paths.md) 와 [`use`](../005-modules/505-use.md) 의 외부 패키지 참조 (`use external_pkg::...`) 를 처리합니다.

의존성 명세 자체는 본 사양의 범위 밖이며, 패키지 매니저가 `rok.toml` 같은 메타 파일에서 의존성을 읽어 들여 `.a` 들의 위치를 결정하는 방식으로 다룹니다. 본 사양에서는 의존성 `.a` 파일이 명시적으로 지정된 경우의 처리만 다룹니다.

```
rokc build --extern foo=path/to/libfoo.a --extern bar=...
```

옵션을 통해 각 의존 패키지의 .a 위치를 명시합니다. 이 자리는 패키지 매니저가 자동으로 채워 주게 될 부분입니다.

## 9.2.4. 소스 파싱

[소스 파일](../001-lexical/101-source-files.md) 부터 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md) 까지의 어휘 / 구문 규칙에 따라 모든 `.rok` 파일을 AST 로 파싱합니다.

[모듈](../005-modules/504-modules.md) 의 디렉토리 모듈은 모든 .rok 파일을 합쳐서 한 모듈의 본체로 취급합니다.

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

HIR 은 타입 정보가 풍부하게 담긴 중간 표현이며, 위 4 ~ 5 단계의 결과를 모두 반영한 형태입니다. 이때 구조적으로 압축하여 같은 타입과 식별자는 공유 표현을 사용하게 됩니다.

## 9.2.8. specialize: 단형화

[제네릭](../002-types/210-generics.md) 과 [const 제네릭](../002-types/211-const-generics.md) 의 단형화를 수행합니다.

```
1. 모든 호출 지점 수집.
2. 각 호출의 구체 타입 튜플 결정.
3. (item_id, type_tuple) 마다 별개의 specialized HIR 생성.
4. 캐시. 같은 (item_id, type_tuple) 는 한 번만.
```

프로그램의 root (`main`, export 된 심볼, `#[export_name]` 항목, 런타임이 참조하는 심볼) 에서 도달할 수 없는 함수는 emit 하지 않습니다.

이 단계 덕분에 [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 가 vtable 없는 직접 호출을 할 수 있게 됩니다 ([트레잇-as-타입 금지](../002-types/212-trait-as-type-banned.md) 참고).

## 9.2.9. rok-mir 생성

[specialize](../000-meta/007-glossary.md#specialize) 된 HIR 을 rok-mir (MIR) 로 변환합니다.

MIR 은 HIR 보다 한층 더 낮은 수준의 IR 로, 이 단계에서 연산자 desugar 가 완료되고 ([연산자 디스패치](../003-expressions/301-operator-dispatch.md) 참고), 패턴 매칭의 decision tree 가 결정되며 ([`match` 표현식](../003-expressions/312-match-expression.md) 참고), closure 와 코루틴의 frame layout 도 함께 확정됩니다.

## 9.2.10. 표준 MLIR 텍스트 출력

[컴파일 파이프라인](../000-meta/002-pipeline.md) 과 [MLIR 다이얼렉트 사용](../008-abi/807-mlir-dialects.md) 의 표준 MLIR 다이얼렉트를 사용하여 `.mlir` 텍스트를 emit 합니다.

```
target/foo.mlir
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
         target/foo.mlir > target/foo.opt.mlir
```

정확한 패스 목록은 구현이 정하며, 위는 대표적인 구성입니다. `-O0` ~ `-O3` 같은 최적화 단계는 이후 `clang` 단계에 적용됩니다.

## 9.2.12. mlir-translate 호출

표준 도구인 `mlir-translate` 를 사용해서 MLIR 을 LLVM IR 로 옮깁니다.

```
mlir-translate --mlir-to-llvmir target/foo.opt.mlir > target/foo.ll
```

## 9.2.13. clang / lld 호출

표준 도구인 `clang` (또는 LLVM 의 `llc` + `lld` 조합) 으로 LLVM IR 을 네이티브 코드로 변환합니다.

```
clang -O2 -c target/foo.ll -o target/foo.o
ar rcs target/libfoo.a target/foo.o target/foo.hir.o
```

여기서 `foo.hir.o` 는 [RHIR bytecode 포맷](904-rhir-bytecode.md) 의 [RHIR](../000-meta/007-glossary.md#rhir-rok-hir-bytecode) 섹션을 담고 있는 obj 입니다.

## 9.2.14. 링킹 (실행 파일의 경우)

`main.rok` 패키지인 경우에는 다음과 같이 링킹을 수행합니다.

```
clang -O2 \
  target/foo.o \
  $ROK_HOME/lib/librokr.a \
  librok_bare.a librok_core.a librok_std.a \
  -o target/foo                         (-lc 등 필요시)
```

이 단계가 비로소 실행 가능한 바이너리를 만들어 내는 단계입니다.

## 9.2.15. 산출물

최종적으로 생성되는 산출물의 구조는 다음과 같습니다.

```
target/
├── foo.mlir            # debug (when --emit=mlir)
├── foo.ll              # debug (when --emit=llvmir)
├── foo.o               # obj of concrete code
├── foo.hir.o           # obj containing RHIR bytecode
├── libfoo.a            # package archive (for lib)
└── foo                 # executable (for main)
```

> debug 는 디버그를 의미하고, obj of concrete code 는 concrete 코드의 obj, obj containing RHIR bytecode 는 RHIR bytecode 가 담긴 obj 를 가리킵니다. package archive (for lib) 은 lib 인 경우의 패키지 archive 이고, executable (for main) 은 main 인 경우의 실행 파일입니다.

## 9.2.16. 증분 빌드

증분 빌드는 다음과 같은 흐름으로 동작합니다.

```
1. 변경된 .rok 파일만 다시 파싱.
2. 영향받은 specialization 만 다시 생성.
3. 변경된 obj 만 다시 컴파일.
4. archive 만 재구성.
```

이때 캐시 키는 파일 hash 에 `rokc` 버전 hash 와 의존 hash 를 더해 구성합니다.

자세한 증분 알고리듬은 `rokc` 가 결정합니다.

## 9.2.17. 컴파일러와 런타임의 분리

이 파이프라인은 코드 의존성 0 원칙을 지킵니다. 즉 `rokc` 는 런타임 (`librokr.a`) 을 빌드 시점에 참조하지 않습니다.

대신에 `extern "C"` 심볼 ([Extern "C" 심볼](../008-abi/805-extern-symbols.md)) 호출만을 emit 하고, 링크 시점에 가서 사용자 빌드가 `librokr.a` 를 함께 링크하는 방식으로 동작합니다.

이렇게 분리해 둔 덕분에 `rokc` 와 `librokr.a` 가 서로 독립된 SemVer 로 진화할 수 있게 됩니다.

## 9.2.18. 외부 도구 의존

`rokc build` 가 정상적으로 동작하려면 다음의 외부 도구들이 PATH 에 등록되어 있어야 합니다.

| 도구                       | 역할                 |
|----------------------------|----------------------|
| `mlir-opt`                 | MLIR 최적화          |
| `mlir-translate`           | MLIR → LLVM IR       |
| `clang` 또는 `llc` + `lld` | LLVM IR → obj / 링킹 |

`rokc` 는 이들 도구를 호출하기만 할 뿐이며, LLVM 라이브러리에 직접 link 하지는 않습니다 ([컴파일 파이프라인](../000-meta/002-pipeline.md) 참고).

만약 도구가 설치되어 있지 않은 경우에는 `rokc build` 가 명확한 에러를 내놓습니다 (어떤 도구를 어디에서 받을 수 있는지 함께 안내합니다).

## 9.2.19. 설치 레이아웃

설치된 toolchain 은 `$ROK_HOME` 아래에 다음과 같이 놓입니다.

```
$ROK_HOME/
├── bin/rokc
└── lib/
    ├── librokr.a              # 런타임
    └── rok/
        ├── core/              # rok::core
        ├── std/               # rok::std
        └── bare/              # rok::bare
```

`rokc` 는 런타임과 표준 라이브러리를 이 경로에서 찾습니다. `ROK_HOME` 이 설정되지 않으면 `rokc` 실행 파일이 있는 디렉토리의 상위 (`bin/` 의 `..`) 를 `ROK_HOME` 으로 봅니다. 저장소에서 직접 실행하는 개발 레이아웃은 위 경로에서 찾지 못했을 때의 fallback 으로 유지합니다.
