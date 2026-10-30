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
rokc rokhir-text path/to/foo.rokhir      # 또는 libfoo.a. 옆의 foo.rokhir 를 읽음
```

위 명령은 미리 빌드한 라이브러리의 [ROKHIR](904-rokhir-bytecode.md) 을 사람이 읽을 수 있는 형식으로 출력합니다. 버전, producer, meta 다음에 interface 파일마다 `--- <경로>` 와 그 텍스트 (구체 함수 본문이 `;` 로 잘린 소스) 가 이어집니다.

```
rokhir 6
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

머리줄은 `<severity>[<code>]: <message>` 이고 `--> path:line:col` 이 주 위치입니다. 주 span 은 `^`, 보조 span 은 `-` 로 밑줄을 긋고, 다른 파일의 보조 span 은 `::: path:line:col` 로 시작합니다. 끝의 `= note:` 줄은 맥락을, `= help:` 줄은 고치는 방법을 덧붙입니다. 찾지 못한 이름을 다른 곳의 항목이 가지고 있으면, help 는 그것을 들여오는 `use` 줄 (최대 셋: 패키지 자신의 항목 먼저, 그다음 표준 라이브러리, 그다음 의존 패키지) 이나, 패키지의 유일한 그런 항목이 가려져 있으면 빠진 `pub` 을 알립니다 (``add `use geo::Point;` ``; `` `geo::secret` has that name, but it is private to module `geo`: add `pub` to it ``).

주 span 은 잘못된 코드입니다: 타입이 맞지 않는 값이면 그 값을 담은 문장이 아니라 값 자체입니다. 메시지는 라이브러리 항목을 코드에 쓰는 이름으로 부릅니다 (`rok::core::marker::Send` 가 아니라 `List::sort`, `Send`, `print`). 사람용 형식은 이 명세의 인용 (`(spec 3.12.5)`) 을 빼고, JSON 형식은 글에 그대로 두고 `spec` 에 모아 줍니다.

### JSON 형식 (도구 통합)

도구 통합을 위해 JSON 형식으로도 출력할 수 있습니다.

```
rokc build --message-format=json
```

진단 하나가 stdout 의 한 줄 JSON 객체입니다 (사람용 형식은 stderr).

```json
{"type":"error","code":"E200","kind":"type","message":"...","spans":[{"file_name":"src/main.rok","line_start":5,"col_start":10,"byte_start":71,"byte_end":76,"is_primary":true,"label":"..."}],"notes":["..."],"spec":["..."]}
```

| 필드       | 뜻                                                                 |
|------------|--------------------------------------------------------------------|
| `type`     | `error` / `fatal` / `warning` / `note`                             |
| `code`     | 에러 종류의 코드 (아래 표)                                         |
| `kind`     | 에러 종류의 이름                                                   |
| `message`  | 메시지                                                             |
| `spans`    | 위치 목록. 첫 원소가 주 위치 (`is_primary: true`), 나머지는 보조 위치 |
| `notes`    | note 문자열 목록. help 항목은 `help: ` 로 시작                     |
| `spec`     | 메시지와 note 가 인용하는 절 (`spec 3.12.5`). 사람용 형식에서는 빠짐 |

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
| `E202` | `handoff`     | 넘긴 뒤의 사용 ([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---스레드-경계-marker)) |
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

`debug` profile 빌드와, `-g` 를 준 `release` 빌드 ([9.2.15](902-aot-pipeline.md#9215-산출물)) 는 DWARF 디버그 정보를 emit 합니다. Windows 대상에서는 그 대신 같은 정보를 담은 CodeView 와 PDB 를 냅니다. 이때 다음과 같은 특성을 갖습니다.

- 함수의 `DW_AT_name` 은 Rok 경로 (`geom::distance`) 이며, linkage name (맹글링된 심볼) 은 두지 않습니다. 디버거가 `_Y` 맹글링을 모르기 때문입니다.
- 줄 번호는 원본 .rok 파일의 줄을 가리킵니다.
- 지역 변수는 원본 이름과 타입으로 보입니다. 원시 타입 (정수, 부동소수, `bool`, `char`) 외에 struct (필드 이름), 튜플, `str`, `any`, sum 타입, 배열의 DWARF 타입을 냅니다. sum 타입과 enum 은 DWARF variant part 입니다: artificial discriminant 멤버와, 변형마다 (데이터 없는 변형 포함) `DW_AT_discr_value` 를 가진 `DW_TAG_variant` 하나씩이며, 각각은 `<Sum>::<Variant>` 라는 이름의 struct 를 담습니다. 두 backend 가 같은 형태를 쓰며, 디버거가 variant part 를 읽도록 compile unit 은 `DW_LANG_Rust` 를 씁니다 (Rok 의 DWARF 언어 코드는 없음). `GcBuf<T>` 참조는 그 객체 (헤더, 그 다음 `T` 원소들) 를 가리키는 포인터로 기술합니다. `any` 값은 그 안에 저장된 타입들에 대한 variant part 이며, discriminant 는 저장된 tag 입니다. discriminant 는 artificial `i32` 입니다 (저장된 tag; `throws` carrier 의 `bool` tag 는 바이트). 함수 값 (`fn` 타입이나 클로저) 은 그 타입 이름 (`fn(i32) -> i32`) 의 구조체이며, 멤버는 `code` (`DW_TAG_subroutine_type` 을 가리키는 포인터: 환경, 그다음 매개변수) 와 `env` (그 환경; 클로저의 타입을 알면 환경 객체를 가리키는 포인터) 입니다. 그래서 디버거가 대상 함수를 보여 줍니다. 참조로 캡처한 변수는 그 값으로 보입니다. 컴파일러가 만드는 함수 (C `main`, static 초기화 guard) 도 subprogram 을 가집니다. 컴파일러가 만든 지역 변수 (이름이 `#` 이나 `__` 로 시작) 는 빠집니다.
- 블록마다 lexical block scope 를 내므로 가려진 (shadowed) 이름은 해당 블록 안에서만 보입니다.
- 스택 슬롯을 가진 지역은 debug / release 모두 `llvm.alloca` 하나에 놓입니다. 그래서 `-g` 는 최적화 수준과 무관하게 같은 슬롯에 debug 정보를 붙입니다.

### gdb / lldb 사용

표준 gdb 와 lldb 명령어가 Rok 코드에 그대로 동작합니다.

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

### Formatter

gdb 는 DWARF 만으로 합, enum, `any` 의 활성 변형만 보여 줍니다. lldb 에는 그 단위의 언어에 대한 plugin 이 없어 모든 변형을 나열합니다. toolchain 은 활성 변형만 보여 주는 lldb formatter `lib/rok/lldb/rok_lldb.py` 를 함께 배포합니다 (변형 이름을 summary 로, 그 필드를 children 으로; box 된 `any` payload 는 그 box 를 거쳐). `rokc --print lldbinit` ([9.1.4](901-rokc-cli.md#rokc---print-lldbinit-lldb-formatter)) 이 그것을 로드하는 lldb 명령을 출력합니다.

```
lldb -O "$(rokc --print lldbinit)" ./target/debug/my_binary
rokc --print lldbinit >> ~/.lldbinit
```

### macOS / Windows 디버거

각 플랫폼의 기본 디버거가 Rok 바이너리를 처리합니다.

- macOS 에서는 lldb 또는 Xcode debugger 를 사용하면 됩니다.
- Windows 에서는 Visual Studio debugger 또는 WinDbg (PDB 디버그 정보) 를 사용하면 됩니다.

## 9.9.7. profiler 통합

### perf (Linux)

CPU 프로파일링은 `perf` 로 수행할 수 있습니다.

```
perf record ./target/release/my_binary
perf report
```

이 경우 Rok 함수가 demangled 이름으로 표시됩니다.

### Profile-guided optimization

Profile-guided optimization ([9.1.2](901-rokc-cli.md#rokc-build-aot-빌드)) 은 세 단계입니다:

```
rokc build --release -C profile-generate=prof    # 1. 계측 빌드
./target/release/my_binary <대표 입력>            # 2. 실행, prof/default_*.profraw 를 씀
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
| `ROK_GC_LOG=1`            | 수집마다 종류 (minor / full), pause 시간, mark 수, sweep 결과를 stderr 에 출력. 동시 full 수집은 시작 pause (minor 수집) 와 끝 pause 를 따로 기록하며, sweep 결과는 살아 있는 바이트의 전 -> 후 |
| `ROK_GC_STATS=1`          | 종료 시 수집 횟수 (minor / full), pause 시간, 수집기 CPU 시간을 stderr 에 출력. 둘째 줄은 block 수 / 상주 크기 / evacuation 횟수 / 옮긴 바이트 |
| `ROK_GC_CONCURRENT=auto\|always\|off` | full 수집의 동시 mark. `auto` 는 old 객체 32 MiB 이상일 때 ([7.5](../007-runtime/705-concurrent-collection.md)) |
| `ROK_GC_EVAC=always`      | 테스트용. full 수집마다 evacuation ([7.3.1](../007-runtime/703-gc-algorithm.md#731-immix-mark-region)) |
| `ROK_GC_STRESS=N`         | 테스트용. 할당 N 번마다 수집 (그 8 번째마다 full)                            |
| `ROK_HEAP_DUMP=on_oom`    | OOM 시 합계, 바이트 기준 상위 32 개 타입, 가장 큰 큰 객체 32 개를 stderr 에 출력. 최선 노력: 할당하지 않고 lock 을 잡지 않음. 형식은 런타임이 정함 |
| `ROK_WORKERS=<n>`         | executor worker 수 N (기본값: 온라인 CPU 수) ([7.6.8](../007-runtime/706-scheduler.md#768-thread-per-core-옵션)) |
| `ROK_THREAD_PER_CORE=1`   | thread-per-core: task 를 자기 worker 에 고정하고 stealing 하지 않음 (7.6.8) |
| `ROK_DYNAMIC_WORKERS=1`   | idle worker 는 park 하고, 모든 활성 worker 가 바쁘면 다시 씀 ([7.7.8](../007-runtime/707-io-and-timer.md#778-worker-수의-동적-조절)) |

설정하지 않았거나 비어 있는 변수는 기본값을 가집니다. 나열된 것 밖의 값은 변수, 값, 받는 값을 알리는 경고 한 줄을 stderr 에 출력하며, 그 변수는 기본값을 가집니다.

## 9.9.9. REPL / Jupyter 디버그

| 수단                 | 효과                                                                 |
|----------------------|----------------------------------------------------------------------|
| `<식>?`              | 식을 평가하고 `Debug` 형태로 출력 ([9.7.7](907-jit-cells.md#977-시각화--출력)) |
| `it`                 | 식 셀이 마지막으로 보여 준 값. 사용자의 `it` 이 우선 ([9.7.7](907-jit-cells.md#977-시각화--출력)) |
| `:type <식>`, `:t`   | 식의 정적 타입을 출력 (실행하지 않음)                                |
| `:doc <이름>`        | 이름의 시그니처와 문서 주석을 출력. Jupyter `inspect_request` 와 같은 텍스트 ([9.8.6](908-jupyter-integration.md#986-inspect_request-도움말)). 메서드는 `:doc xs.len` |
| `:time <식>`         | 식을 셀로 평가하고 단계별 시간 (`ROK_REPL_TIME=1` 처럼) 과 합계를 stderr 에 출력 |
| `:vars`              | 바인딩을 `name: type` 으로, 이어서 항목의 시그니처를 나열              |
| `:load <file>`       | 실행 중인 세션에 프로그램을 로드 ([9.7.10](907-jit-cells.md#9710-프로그램-로드-repl-path)). 그 항목이 같은 이름 항목을 대신함. `:save` 로 쓴 스크립트는 그 셀을 하나씩 다시 실행 |
| `:reload`            | 마지막으로 로드한 프로그램을 다시 로드                               |
| `:edit`              | `$VISUAL` 이나 `$EDITOR` (없으면 `vi`, Windows 는 `notepad`) 로 임시 파일을 열고, 그 내용을 셀로 실행. 셀이 실패하면 다음 `:edit` 에서 그 텍스트를 다시 엶 |
| `:save <file>`       | 세션을 스크립트로 씀: 항목, 이어서 실행된 셀. 같은 타입으로 다시 선언한 이름은 대입으로, 다른 타입이면 파일 끝까지 이어지는 블록을 엶 ([5.1.4](../005-modules/501-variable-declaration.md#514-같은-스코프-내-재선언-금지)). `it` 은 이후 셀이 읽는 곳에서만 선언. 첫 줄은 `// rok repl session (:save); :load replays it cell by cell` 이고 각 셀 앞에 `// :cell` 줄이 옴: 스크립트를 `:load` 하면 셀을 하나씩 다시 실행하므로 다시 선언한 이름은 마지막 바인딩을 유지하고, `rokc run` 은 프로그램으로 실행 |
| `:opt [<level>]`     | 최적화 수준 (`-O` 처럼 `0`, `1`, `2`, `3`, `s`, `z`) 을 출력하거나 이후 셀에 설정 |
| `:reset`             | 세션을 처음 상태로 (라이브러리 경로의 `use` 는 다시 넣음)               |
| `:add <dep> ..`      | 의존성을 추가하고 (현재 환경으로 `rok install`, 패키지의 세션에서는 `rok add`) 세션에 붙임 ([9.10.5](910-rok-frontend.md#9105-환경)) |
| `:help`, `:h`        | 모든 명령과 별칭을 나열                                               |
| `:quit`, `:q`, `:exit` | 종료 (입력이 끝나도 종료)                                          |
| `exit`, `quit`, `exit()`, `quit()` | 셀에 단독으로 있으면 종료. 세션이 그 이름 (바인딩, 항목, `use`) 을 정의하면 그쪽이 우선 |
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

메타 확장과 desugar 를 모두 거친 전체 AST 를 Rok 소스로 보려면 `--emit=ast` (`<name>.ast.rok`) 를 씁니다.

## 9.9.11. 벤치마크

마이크로벤치마크는 다음 명령으로 실행할 수 있습니다.

```
rok bench
```

`rokc bench` 가 벤치마크 하네스를 빌드하고 `rok bench` ([9.10](910-rok-frontend.md)) 가 그것을 실행하며 ([9.1.2](901-rokc-cli.md#rokc-bench-벤치마크)), `#[bench]` 가 부착된 함수들을 `-O2` 빌드에서 하나씩 측정합니다. 각 함수는 `Bench` 핸들을 받아 `b.iter` 로 클로저를 측정하므로, 클로저 밖의 setup 은 측정되지 않습니다. 측정 방법은 다음과 같습니다:

1. **클록.** 모노토닉 나노초 클록. 고정 오버헤드는 표본마다 한 번이 아니라 `iters` 번 호출을 측정해 상각합니다.
2. **보정.** `--iters` 를 주지 않으면, 표본 하나가 약 100 ms 가 될 때까지 `iters` 를 키워 클록 양자화와 오버헤드를 무시할 수준으로 만듭니다.
3. **Warm-up.** 약 표본 하나 분량의 시간만큼 호출을 돌려 버리며, 캐시 / 분기 예측기 / 페이징을 안정시킵니다.
4. **표본.** `--samples` 개 (기본 100) 표본을 취합니다; 각 표본은 `iters` 번 호출의 총시간을 기록하고, 호출 한 번의 시간은 그것을 `iters` 로 나눈 값입니다.
5. **통계.** 평균 · 중앙값 · 표준편차 · 중앙값 절대편차 (MAD) · 최소 · 최대를 계산합니다. Tukey 울타리 (1.5 IQR mild, 3 IQR severe) 를 벗어난 표본은 이상치로 보고하되 요약 수치에서는 뺍니다. 보고 수치는 **중앙값 (+/- MAD)** 으로, 가끔 느린 표본에 강건합니다.
6. **GC 인지.** 표본마다 런타임의 수집 카운터를 전후로 읽어, 수집이 일어난 표본은 이상치로 분류해 따로 셉니다. GC 일시정지가 중앙값을 왜곡하지 않습니다.

`black_box(x)` 는 옵티마이저가 꿰뚫어 볼 수 없는 항등 함수입니다; 측정 클로저의 입력과 결과에 적용해 작업이 밖으로 끌려 나가거나 접혀 사라지지 않게 하십시오. `b.bytes(n)` / `b.items(n)` 은 호출당 시간을 처리량 (`MiB/s`, `Melem/s`) 으로 바꿉니다.

출력은 터미널이든 아니든 사람이 읽는 단위로 정렬되며, `--format=rokbench` 는 회귀 도구를 위한 안정적인 벤치당 한 줄 기계 가독 형식을 줍니다 ([9.1.2](901-rokc-cli.md#rokc-bench-벤치마크)). `b.iter` 를 한 번도 부르지 않은 벤치는 아무것도 측정하지 않았다고 보고하며, 이는 실패가 아닌 경고입니다. `--iters` / `--samples` 로 표본을 조절합니다.

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

`rokc test` 는 하네스를 빌드할 뿐 테스트를 실행하지 않습니다. 실행은 `rok test` 가 하며 실패한 테스트가 있으면 `1` 입니다. 테스트마다 별도 프로세스이므로 테스트의 panic (`101`) 이 러너의 `101` 이 되지는 않습니다.

CI 가 코드별로 다른 처리 (재시도, 알림 등) 를 적용할 수 있습니다.

## 9.9.13. LSP 서버

`rokc` 는 LSP (Language Server Protocol) 모드를 제공합니다. 같은 서버를 독립 바이너리 `rokls` (= `rok lsp`) 로도 제공합니다. stdio 로 통신하며, 진단, hover (멤버 포함), go-to-definition (정의하는 이름의 범위를 돌려줌), references, rename, 문서 심볼, 멤버 / 경로 완성, 시그니처 도움말, 빠른 수정, 포맷, semantic token (정적 문법이 알아볼 수 없는 매크로 호출과 사용자 정의 키워드, 리터럴 접미사, 연산자, 속성) 을 제공합니다. references 와 rename 은 버퍼 안에서만 동작합니다.

```
rokc lsp
rokls
```

이 모드를 사용하면 VS Code, Vim, Emacs 등 LSP 클라이언트가 자동 완성, hover, go-to-definition, 진단 등을 받아옵니다. 내부적으로는 `rokc check` 의 점진적 결과를 활용합니다.

- **자동 완성.** `x.` 뒤에서는 `x` 타입의 필드와 메서드입니다. 표준 라이브러리나 다른 패키지의 타입이면 `pub` 인 것만 내놓으며, `#[doc(hidden)]` 멤버와 연산자 트레잇 (`a + b`, `xs[i]` 처럼 쓰는 것) 의 메서드는 빼놓습니다. `a::` 뒤에서는 (`use a::b::` 안에서도) `a` 아래의 모듈과 항목입니다. 그 밖에서는 커서 위치에서 보이는 지역 변수, 버퍼의 항목, 경로 없이 쓸 수 있는 이름 (core 라이브러리의 이름과 `use` 항목이 가져온 이름), 키워드입니다. REPL 과 Jupyter 커널도 같은 엔진으로 완성합니다 ([9.7.8](907-jit-cells.md#978-입력-단위-repl-식별)).
- **hover 와 정의.** 함수, 메서드, 필드 위의 hover 는 표준 라이브러리의 것도 포함해 시그니처와 그 문서 주석을 보여 줍니다. `x.name` 의 go-to-definition 은 `x` 의 타입에서 `name` 을 찾으며, 표준 라이브러리 항목이면 라이브러리 소스의 그 항목으로 갑니다.
- **시그니처 도움말.** 호출의 괄호 안에서 (`(` 와 `,` 에서 요청) 호출되는 함수의 시그니처와 문서 주석을 보여 주고, 커서가 있는 매개변수를 활성으로 표시합니다. 메서드는 receiver 의 타입에서 찾으며, 그 `self` 는 세지 않습니다.
- **빠른 수정.** 편집을 담은 진단 (예: ``convert with `as i32` ``) 은 그 편집을 `quickfix` 코드 액션으로 내놓습니다. 편집이 없으면 무엇을 쓸지 말하는 help 줄이 코드 액션이 됩니다: ``add `;` `` 는 진단 span 의 끝에 `;` 를 넣고, ``add `use a::b;` `` 는 그 줄을 파일의 마지막 최상위 `use` 뒤에 (없으면 맨 앞의 `//!` 줄과 내부 속성 뒤에) 넣으며, ``rok add <name>`` 은 명령 `rok.add` 가 됩니다. `workspace/executeCommand` 를 받으면 서버는 `rok.add` 를 패키지 디렉터리에서 `rok add <name>` 으로 (스크립트면 그 파일의 디렉터리에서 `rok add <name> <file>` 로) 실행합니다. `rok` 은 `$ROK` 가 가리키는 것, 없으면 서버 옆의 것을 쓰며, 실행 뒤 열린 버퍼를 다시 검사합니다. 다른 명령은 실행하지 않으며 플래그도 받지 않습니다.
- **rename.** `prepareRename` 은 rename 이 받아들일 때 커서 아래 이름의 범위를 돌려줍니다. 버퍼 밖에서 선언된 이름이면 어디서 왔는지 말하는 메시지로 실패합니다. 예: ``cannot rename `println`: it is defined in the standard library``. 키워드이면 `` `let` is a keyword `` 입니다.
- **문서 심볼.** 심볼의 range 는 수식어 (`pub`, `const`, ..) 부터 닫는 `}` 나 `;` 까지 항목 전체를 덮고, selection range 는 이름입니다.
- **진단 범위.** span 이 빈 진단은 그 위치의 토큰 위에, 줄 끝이면 그 앞의 토큰 위에 보고합니다.
- **손으로 시작했을 때.** 터미널에서 시작하면 서버는 먼저 표준 에러에 ``rokls speaks the language server protocol on stdin; start it from an editor (see `rok help lsp`)`` 를 출력합니다. 클라이언트가 `initialize` 를 보내기 전에 입력이 끝나면 ``rokls: input ended before the editor sent `initialize` `` 를 출력하고 1 로 끝납니다. `initialize` 뒤에는 LSP 표준대로 `shutdown` 을 받았으면 0, 아니면 1 로 끝납니다.

자세한 LSP 명세는 본 사양 외이며 (LSP 표준 + rokc 구현 결정), 별도로 정의됩니다.

## 9.9.14. 패키지 검사

미리 빌드한 라이브러리 패키지의 정보를 한 번에 확인할 수 있습니다.

```
rokc inspect path/to/libfoo.a        # 또는 foo.rokhir. 짝 파일을 함께 읽음
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

지금까지 살펴본 도구들이 Rok 의 개발과 디버그 경험의 핵심을 이룹니다. 표준 도구 (gdb, perf, jupyter) 와의 통합과 Rok 고유 도구 (`rokc demangle`, `rokc rokhir-text` 등) 가 결합되어 풍부한 진단 환경을 제공합니다.

이로써 9 장 (실행과 툴) 을 마무리하며, 본 사양 전체가 큰 그림을 완성합니다.
