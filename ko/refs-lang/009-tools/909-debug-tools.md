# 9.9. 디버그 / 진단 도구

이 절에서는 `rokc` 가 제공하는 디버그 / 진단 도구와 외부 도구 통합을 정의합니다.

## 9.9.1. 중간 산출물 출력: `--emit`

[AOT 빌드 파이프라인](902-aot-pipeline.md) 의 단계별 결과물을 덤프할 수 있습니다.

```
rokc build --emit=ast           # 메타 확장과 desugar 뒤의 전체 AST 를 rok 소스로
rokc build --emit=hir          # rok-hir 텍스트 출력
rokc build --emit=mir           # rok-mir 텍스트 출력
rokc build --emit=mlir          # 표준 MLIR 텍스트
rokc build --emit=llvmir        # LLVM IR
rokc build --emit=asm           # 어셈블리
rokc build --emit=obj           # object 파일
rokc build --emit=expanded      # 메타 확장 결과 (9.9.10)
```

`mlir` 과 `llvmir` 은 최적화 수준과 관계없이 MLIR → LLVM backend 에서, `asm` 과 `obj` 는 수준이 고른 backend 에서 나옵니다 ([9.2.10](902-aot-pipeline.md#9210-backend-선택)).

산출물은 다음 위치에 출력됩니다.

```
target/debug/build/foo.ast.rok
target/debug/build/foo.hir
target/debug/build/foo.mir
target/debug/build/foo.mlir
target/debug/build/foo.ll
target/debug/build/foo.s
target/debug/build/foo.o
target/debug/build/foo.expanded.rok
```

여러 단계를 한 번에 지정하려면 `--emit=mlir,llvmir,asm` 처럼 쉼표로 구분하면 됩니다. 종류가 하나면 `-o <path>` 로 위치를 바꿀 수 있습니다. `--emit` 을 주면 실행 파일이나 archive 는 만들지 않습니다.

## 9.9.2. ROKHIR 검사

```
rokc rokhir-text path/to/foo.rokhir      # 또는 libfoo.a — 옆의 foo.rokhir 를 읽음
```

위 명령은 미리 빌드한 라이브러리의 [ROKHIR](904-rokhir-bytecode.md) 을 사람이 읽을 수 있는 형식으로 출력합니다. 버전, producer, meta 다음에 interface 파일마다 `--- <경로>` 와 그 텍스트 (구체 함수 본문이 `;` 로 잘린 소스) 가 이어집니다.

```
rokhir 2
producer: rokc 0.1.0 (1f2a40-17f3a9c2e1b0c4d8)
package: geom (hash: 3f8a1c2b)
source: /home/me/geom
gc types: 4096..4160
features: tls

--- lib.rok
pub struct Point { pub x: f64; pub y: f64; }
pub fn distance(p1: Point, p2: Point) -> f64;
pub fn id<T>(x: T) -> T { x }
```

이 출력은 [ROKHIR](../000-meta/007-glossary.md#rokhir-rok-hir-bytecode) 의 정확성과 호환성을 검증할 때 활용하면 됩니다.

## 9.9.3. 심볼 demangle

```
rokc demangle <symbol>
rokc demangle _Y3f8a1c2b_geom_distance_a7c91d8e44b2f190
```

이 명령은 [심볼 맹글링](905-symbol-mangling.md) 의 mangled 심볼을 원본 형식으로 출력합니다. 심볼을 여러 개 주면 블록을 빈 줄로 나눕니다. 맹글링은 루트 `rok::` 를 남기지 않으므로, 경로의 첫 세그먼트가 `core` / `std` / `bare` 이면 `rok::` 를 붙여 표준 라이브러리 경로로 보여 줍니다.

```
package:  3f8a1c2b
path:     geom
name:     distance
generics: a7c91d8e44b2f190 (type tuple hash)
```

## 9.9.4. 진단 / 에러 메시지

### 표준 형식

진단 메시지는 다음과 같은 표준 형식으로 출력됩니다.

```
error[E100]: cannot find value `count` in this scope
  --> src/main.rok:5:13
   |
 2 |     let counter = 0;
   |         ------- similar name defined here
...
 5 |     let x = count + 1;
   |             ^^^^^ not found in this scope
   |
   = help: did you mean `counter`?
```

이 형식은 [소스 파일](../001-lexical/101-source-files.md) 의 파일 + 줄 + 컬럼 진단 단위를 사용합니다.

머리줄은 `<severity>[<code>]: <message>` 이고 `--> path:line:col` 이 주 위치입니다. 주 span 은 `^`, 보조 span 은 `-` 로 밑줄을 긋고, 다른 파일의 보조 span 은 `::: path:line:col` 로 시작합니다. 끝의 `= note:` 줄은 맥락을, `= help:` 줄은 고치는 방법을 덧붙입니다.

### JSON 형식 (도구 통합)

도구 통합을 위해 JSON 형식으로도 출력할 수 있습니다.

```
rokc build --message-format=json
```

진단 하나가 stdout 의 한 줄 JSON 객체입니다 (사람용 형식은 stderr).

```json
{"type":"error","code":"E200","kind":"type","message":"...","spans":[{"file_name":"src/main.rok","line_start":5,"col_start":10,"byte_start":71,"byte_end":76,"is_primary":true,"label":"..."}],"notes":["..."]}
```

| 필드       | 뜻                                                                 |
|------------|--------------------------------------------------------------------|
| `type`     | `error` / `fatal` / `warning` / `note`                             |
| `code`     | 에러 종류의 코드 (아래 표)                                         |
| `kind`     | 에러 종류의 이름                                                   |
| `message`  | 메시지                                                             |
| `spans`    | 위치 목록. 첫 원소가 주 위치 (`is_primary: true`), 나머지는 보조 위치 |
| `notes`    | note 문자열 목록. help 항목은 `help: ` 로 시작                     |

span 은 `file_name`, `line_start`, `col_start`, `byte_start`, `byte_end`, `is_primary`, `label` 을 가집니다. 위치 없이 파일만 아는 진단은 `{"file_name": .., "is_primary": true}` 만 둡니다.

이 형식 덕분에 IDE 와 CI 시스템이 진단을 기계로 판독할 수 있습니다.

### 진단 카테고리

진단 코드는 영역별로 다음과 같이 분류됩니다.

| 코드 prefix     | 영역               |
|-----------------|--------------------|
| `E001` ~ `E099` | 어휘 / 구문        |
| `E100` ~ `E199` | 이름 해소 / 가시성 |
| `E200` ~ `E299` | 타입 시스템        |
| `E300` ~ `E399` | 메타 / 매직        |
| `E400` ~ `E499` | 빌드 / 링크        |
| `E500` ~        | 추가 카테고리      |

코드는 에러 종류마다 하나입니다. 같은 종류의 진단은 같은 코드를 가집니다.

| 코드   | `kind`        | 종류                         |
|--------|---------------|------------------------------|
| `E001` | `lexical`     | 어휘                         |
| `E002` | `parse`       | 구문                         |
| `E100` | `resolve`     | 이름 해소                    |
| `E101` | `visibility`  | 가시성                       |
| `E102` | `module`      | 모듈 / 파일                  |
| `E200` | `type`        | 타입                         |
| `E201` | `pattern`     | 패턴 / 완전성                |
| `E300` | `meta`        | 메타 블록 / 속성 / 키워드 매직 |
| `E400` | `lowering`    | HIR / MIR lowering           |
| `E401` | `codegen`     | 코드 생성                    |
| `E402` | `linkage`     | 외부 도구 / 링크             |
| `E500` | `unsupported` | 컴파일러가 처리하지 않는 구문 |
| `E501` | `internal`    | 컴파일러 내부                |
| `E502` | `other`       | 그 밖                        |
| `E503` | `lint`        | lint ([6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어), `rokc lint`) |

본 사양은 영역별 코드 prefix 를 보장합니다. 개별 코드가 생기면 같은 영역 안에서 나뉩니다.

## 9.9.5. 로그 / 진행 표시

### 표준 출력

기본 빌드 출력은 다음과 같은 형태로 표시됩니다.

```
   Compiling foo (path/to/foo)
    Building target/debug/libfoo.a
    Finished in 1.23s
```

진행 줄은 stderr 에 나오며, 동사 목록은 [9.1.8](901-rokc-cli.md#918-출력-형식) 에 있습니다.

### `--verbose` / `-v`

상세 모드에서는 단계별 시간을 함께 표시합니다.

```
   Compiling foo (path/to/foo)
   Stage: front end (134ms)
    Building target/debug/foo
   Stage: codegen + link (812ms)
    Finished in 0.95s
```

각 단계의 시간을 표시하므로, 빌드 병목을 진단할 때 유용합니다. 단계는 `front end` 와 `codegen + link` (`--emit` 이면 `emit <ext>`) 입니다.

### `ROK_TIME_PASSES`

`ROK_TIME_PASSES=1` 은 컴파일러 패스마다 걸린 실제 시간을 stderr 에 출력합니다.

### `-vv` (매우 자세히)

`-v` 에 더해 처리한 파일마다 `Reading <file>` 줄을, [specialize](../000-meta/007-glossary.md#specialize) 한 (item, type) 쌍마다 `Specialized <path>(sig)` 줄을 출력합니다.

## 9.9.6. 디버거 통합: gdb / lldb

### DWARF 디버그 정보

`rokc build -g` 또는 `--debug` (같은 뜻이며, profile 이름 `debug` 와는 별개) 를 사용하면 DWARF 디버그 정보를 emit 합니다. Windows 대상에서는 그 대신 같은 정보를 담은 CodeView 와 PDB 를 냅니다. 이때 다음과 같은 특성을 갖습니다.

- 함수의 `DW_AT_name` 은 rok 경로 (`geom::distance`) 이며, linkage name (맹글링된 심볼) 은 두지 않습니다. 디버거가 `_Y` 맹글링을 모르기 때문입니다.
- 줄 번호는 원본 .rok 파일의 줄을 가리킵니다.
- 지역 변수는 원본 이름과 타입으로 보입니다. 원시 타입 (정수, 부동소수, `bool`, `char`) 외에 struct (필드 이름), 튜플, `str`, `any`, sum 타입, 배열의 DWARF 타입을 냅니다. sum 타입과 enum 은 DWARF variant part 입니다: artificial discriminant 멤버와, 변형마다 (데이터 없는 변형 포함) `DW_AT_discr_value` 를 가진 `DW_TAG_variant` 하나씩이며, 각각은 `<Sum>::<Variant>` 라는 이름의 struct 를 담습니다. 두 backend 가 같은 형태를 쓰며, 디버거가 variant part 를 읽도록 compile unit 은 `DW_LANG_Rust` 를 씁니다 (rok 의 DWARF 언어 코드는 없음). `GcBuf<T>` 참조는 그 객체 (헤더, 그 다음 `T` 원소들) 를 가리키는 포인터로 기술합니다. `any` 값은 그 안에 저장된 타입들에 대한 variant part 이며, discriminant 는 저장된 tag 입니다. 참조로 캡처한 변수는 그 값으로 보입니다. 컴파일러가 만드는 함수 (C `main`, static 초기화 guard) 도 subprogram 을 가집니다.
- 블록마다 lexical block scope 를 내므로 가려진 (shadowed) 이름은 해당 블록 안에서만 보입니다.
- 스택 슬롯을 가진 지역은 debug / release 모두 `llvm.alloca` 하나에 놓입니다. 그래서 `-g` 는 최적화 수준과 무관하게 같은 슬롯에 debug 정보를 붙입니다.

### gdb / lldb 사용

표준 gdb 와 lldb 명령어가 rok 코드에 그대로 동작합니다.

```
gdb ./target/debug/my_binary
(gdb) break main
(gdb) run
(gdb) print x
$1 = (i32) 42          # 원시 타입 변수만
(gdb) bt
#0  main at src/main.rok:5
...
```

### macOS / Windows 디버거

각 플랫폼의 기본 디버거가 rok 바이너리를 처리합니다.

- macOS 에서는 lldb 또는 Xcode debugger 를 사용하면 됩니다.
- Windows 에서는 Visual Studio debugger 또는 WinDbg (PDB 디버그 정보) 를 사용하면 됩니다.

## 9.9.7. profiler 통합

### perf (Linux)

CPU 프로파일링은 `perf` 로 수행할 수 있습니다.

```
perf record ./target/release/my_binary
perf report
```

이 경우 rok 함수가 demangled 이름으로 표시됩니다.

### Profile-guided optimization

Profile-guided optimization ([9.1.2](901-rokc-cli.md#rokc-build-aot-빌드)) 은 세 단계입니다:

```
rokc build --release -C profile-generate=prof    # 1. 계측 빌드
./target/release/my_binary <대표 입력>            # 2. 실행 — prof/default_*.profraw 를 씀
rokc profdata merge -o app.profdata prof          # 3. 병합한 뒤
rokc build --release -C profile-use=app.profdata  #    profile 로 다시 빌드
```

선택인 context-sensitive 단계는 `-C profile-use=app.profdata -C cs-profile-generate=cs` 로 빌드해 다시 실행하고, `cs` 를 `app.profdata` 와 합쳐 최종 profile 을 만듭니다. `wasm32-wasi` 는 지원하고 `wasm32-unknown-unknown` 은 지원하지 않습니다.

### Tracy / Chrome trace (선택)

표준 라이브러리의 tracing 표면을 통해 `tracy` 또는 Chrome `chrome://tracing` 과 호환되는 trace 를 출력할 수 있습니다. 구체적인 라이브러리 선택은 본 사양의 범위 밖이며, 별도 라이브러리 spec 에서 다룹니다.

## 9.9.8. 메모리 / GC 도구

런타임은 다음 환경 변수를 읽습니다.

| 변수                      | 효과                                                                         |
|---------------------------|------------------------------------------------------------------------------|
| `ROK_GC_LOG=1`            | 수집마다 종류 (minor / full), pause 시간, mark 수, sweep 결과를 stderr 에 출력 |
| `ROK_GC_STATS=1`          | 종료 시 수집 횟수 (minor / full), pause 시간, 수집기 CPU 시간을 stderr 에 출력. 둘째 줄은 block 수 / 상주 크기 / evacuation 횟수 / 옮긴 바이트 |
| `ROK_GC_CONCURRENT=auto\|always\|off` | full 수집의 동시 mark. `auto` 는 old 객체 32 MiB 이상일 때 ([7.5](../007-runtime/705-concurrent-collection.md)) |
| `ROK_GC_EVAC=always`      | 테스트용. full 수집마다 evacuation ([7.3.1](../007-runtime/703-gc-algorithm.md#731-immix-mark-region)) |
| `ROK_GC_STRESS=N`         | 테스트용. 할당 N 번마다 수집 (그 8 번째마다 full)                            |
| `ROK_HEAP_DUMP=on_oom`    | OOM 시 heap 의 객체 분포와 큰 객체를 stderr 에 출력. 형식은 런타임이 정함    |

## 9.9.9. REPL / Jupyter 디버그

| 수단                 | 효과                                                                 |
|----------------------|----------------------------------------------------------------------|
| `:type <식>`         | 식의 타입을 출력 (실행하지 않음)                                     |
| `:reset`             | 세션을 처음 상태로 (라이브러리 경로의 `use` 는 다시 넣음)               |
| `:add <dep> ..`      | 현재 환경에 의존성을 추가하고 세션에 붙임 (`rok` 경유, [9.10.5](910-rok-frontend.md#9105-환경)) |
| `:help`, `:quit`     | 도움말, 종료                                                         |
| `ROK_REPL_TIME=1`    | 셀마다 단계별 시간과, 새로 컴파일한 함수 / 앞 셀에서 재사용한 함수 수 ([9.6.4](906-jit-engine.md#964-라이브러리-코드의-재사용)) |

셀의 MLIR / LLVM IR 을 보려면 같은 코드를 `main.rok` 로 두고 `rokc build --emit=mlir,llvmir` 을 씁니다 (9.9.1).

## 9.9.10. 메타 블록 디버그

[사용자 정의 메타 블록](../006-meta-system/608-meta-blocks.md) 의 derive / attribute 가 어떤 코드를 생성하는지 확인할 수 있습니다.

```
rokc build --emit=expanded      # → target/<profile>/build/<name>.expanded.rok
```

이 옵션은 사용자 패키지의 [메타 확장](../006-meta-system/609-template-tokens.md) 결과를 소스 텍스트로 출력합니다. 확장마다 다음 주석 뒤에 생성된 소스가 이어지며, 생성물이 다시 확장된 결과는 바깥 확장 뒤에 이어집니다.

```
// expanded from this #[derive(MyHash)] (src/geom.rok:12:1)
impl MyHash for Point { ... }
```

메타 확장과 desugar 를 모두 거친 전체 AST 를 rok 소스로 보려면 `--emit=ast` (`<name>.ast.rok`) 를 씁니다.

## 9.9.11. 벤치마크

마이크로벤치마크는 다음 명령으로 실행할 수 있습니다.

```
rokc bench
```

이 명령 ([9.1.2](901-rokc-cli.md#rokc-bench-벤치마크)) 은 `#[bench]` 가 부착된 함수들을 `-O2` 빌드에서 하나씩 실행하고, 호출 한 번의 평균 시간과 표본 사이의 표준편차를 `ns/iter (+/- sd)` 로 출력합니다. `--iters` / `--samples` 로 표본을 조절합니다.

## 9.9.12. CI / 테스트 통합

### CI 모드

CI 환경에서는 machine-readable 출력을 사용하면 좋습니다.

```
rokc build --message-format=json --no-color > build.log
```

이 출력은 CI 시스템이 파싱하기에 적합합니다.

### 종료 코드

[`rokc` CLI 인터페이스](901-rokc-cli.md) 의 종료 코드는 다음과 같이 정의됩니다.

| 코드  | 의미             |
|-------|------------------|
| `0`   | 성공             |
| `1`   | 사용자 코드 에러 |
| `2`   | CLI 에러         |
| `101` | rokc 내부 panic  |

`rokc test` 는 실패한 테스트가 있으면 `1` 입니다. 테스트마다 별도 프로세스이므로 테스트의 panic (`101`) 이 rokc 의 `101` 이 되지는 않습니다.

CI 가 코드별로 다른 처리 (재시도, 알림 등) 를 적용할 수 있습니다.

## 9.9.13. LSP 서버

`rokc` 는 LSP (Language Server Protocol) 모드를 제공합니다. 같은 서버를 독립 바이너리 `rokls` (= `rok lsp`) 로도 제공합니다. stdio 로 통신하며, 진단, hover (멤버 포함), go-to-definition (정의하는 이름의 범위를 돌려줌), references, rename, 문서 심볼, 멤버 / 경로 완성, 포맷을 제공합니다. references 와 rename 은 버퍼 안에서만 동작합니다.

```
rokc lsp
rokls
```

이 모드를 사용하면 VS Code, Vim, Emacs 등 LSP 클라이언트가 자동 완성, hover, go-to-definition, 진단 등을 받아옵니다. 내부적으로는 `rokc check` 의 점진적 결과를 활용합니다.

자세한 LSP 명세는 본 사양 외이며 (LSP 표준 + rokc 구현 결정), 별도로 정의됩니다.

## 9.9.14. 패키지 검사

미리 빌드한 라이브러리 패키지의 정보를 한 번에 확인할 수 있습니다.

```
rokc inspect path/to/libfoo.a        # 또는 foo.rokhir — 짝 파일을 함께 읽음
```

이 명령은 ROKHIR ([9.4](904-rokhir-bytecode.md)) 과 [archive](../000-meta/007-glossary.md#archive) 의 정보를 다음과 같이 표시합니다.

```
Package: foo
Hash: 3f8a1c2b
rokc version: 0.1.0
ROKHIR version: 1
GC type ids: 4096..4160

Exports:
  pub fn distance(p1: Point, p2: Point) -> f64
  pub struct Point
  ...

Generic items (ROKHIR):
  pub fn id<T>(x: T) -> T
  ...

Concrete .o files:
  foo.o (15 KB)
```

`rokc rokhir-text <file>` 은 ROKHIR 의 버전, producer, meta 와 interface 파일 전체를 출력합니다 (9.9.2).

이를 통해 패키지의 외부에 보이는 모습을 한눈에 파악할 수 있습니다.

## 9.9.15. 정리

지금까지 살펴본 도구들이 rok 의 개발과 디버그 경험의 핵심을 이룹니다. 표준 도구 (gdb, perf, jupyter) 와의 통합과 rok 고유 도구 (`rokc demangle`, `rokc rokhir-text` 등) 가 결합되어 풍부한 진단 환경을 제공합니다.

이로써 9 장 (실행과 툴) 을 마무리하며, 본 사양 전체가 큰 그림을 완성합니다.
