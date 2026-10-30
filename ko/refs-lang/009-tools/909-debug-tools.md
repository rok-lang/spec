# 9.9. 디버그 / 진단 도구

이 절은 `rokc`의 디버그 / 진단 도구와 외부 도구 통합을 정의합니다.

## 9.9.1. 중간 산출물 출력: `--emit`

[AOT 빌드 파이프라인](902-aot-pipeline.md)의 단계별 결과를 덤프할 수 있습니다.

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

`mlir`과 `llvmir`은 최적화 수준과 관계없이 MLIR → LLVM 백엔드에서 나옵니다. `asm`과 `obj`는 최적화 수준에 따라 고른 백엔드에서 나옵니다([9.2.10](902-aot-pipeline.md#9210-백엔드-선택)).

산출물은 다음 위치에 씁니다.

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

여러 단계를 한 번에 지정하려면 `--emit=mlir,llvmir,asm`처럼 쉼표로 구분합니다. 종류가 하나면 `-o <path>`로 위치를 바꿀 수 있습니다. `--emit`을 주면 실행 파일이나 아카이브는 만들지 않습니다.

## 9.9.2. ROKHIR 검사

```
rokc rokhir-text path/to/foo.rokhir      # 또는 libfoo.a - 옆의 foo.rokhir 를 읽음
```

이 명령은 미리 빌드한 라이브러리의 [ROKHIR](904-rokhir-bytecode.md)을 사람이 읽을 수 있는 형식으로 출력합니다. 버전, producer, 메타 다음에 인터페이스 파일마다 `--- <경로>`와 그 텍스트가 이어집니다. 텍스트는 구체 함수 본문을 `;`로 잘라 낸 소스입니다.

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

이 출력은 [ROKHIR](../000-meta/007-glossary.md#rokhir-rok-hir-bytecode)의 정확성과 호환성을 검증할 때 쓸 수 있습니다.

## 9.9.3. 심볼 디맹글링

```
rokc demangle <symbol>
rokc demangle _Y3f8a1c2b_geom_distance_a7c91d8e44b2f190
```

이 명령은 [심볼 맹글링](905-symbol-mangling.md)으로 맹글링한 심볼을 원본 형식으로 출력합니다. 심볼을 여러 개 주면 블록을 빈 줄로 나눕니다. 맹글링은 루트 `rok::`를 남기지 않습니다. 그래서 경로의 첫 세그먼트가 `core` / `std` / `bare`이면 `rok::`를 붙여 표준 라이브러리 경로로 보여 줍니다.

```
package:  3f8a1c2b
path:     geom
name:     distance
generics: a7c91d8e44b2f190 (type tuple hash)
```

## 9.9.4. 진단 / 에러 메시지

### 표준 형식

진단 메시지는 다음 표준 형식으로 출력합니다.

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

이 형식은 [소스 파일](../001-lexical/101-source-files.md)에서 정한 파일, 줄, 열 단위의 진단 위치를 씁니다.

머리줄은 `<severity>[<code>]: <message>`이고 `--> path:line:col`이 주 위치입니다. 주 span에는 `^`로, 보조 span에는 `-`로 밑줄을 긋습니다. 다른 파일의 보조 span은 `::: path:line:col`로 시작합니다. 끝의 `= note:` 줄은 맥락을, `= help:` 줄은 고치는 방법을 덧붙입니다. 찾지 못한 이름이 다른 곳에 항목으로 있으면 help는 그 항목을 가져오는 `use` 줄을 알립니다. 줄은 최대 세 개이고, 패키지 자신의 항목, 표준 라이브러리, 의존 패키지 순서로 고릅니다. 패키지에 그런 항목이 하나뿐이고 그 항목이 비공개이면 빠진 `pub`을 알립니다(``add `use geo::Point;` ``, `` `geo::secret` has that name, but it is private to module `geo`: add `pub` to it ``).

### JSON 형식(도구 통합)

도구와 통합할 때는 JSON 형식으로도 출력할 수 있습니다.

```
rokc build --message-format=json
```

진단 하나가 stdout의 JSON 객체 한 줄입니다. 사람용 형식은 stderr에 나옵니다.

```json
{"type":"error","code":"E200","kind":"type","message":"...","spans":[{"file_name":"src/main.rok","line_start":5,"col_start":10,"byte_start":71,"byte_end":76,"is_primary":true,"label":"..."}],"notes":["..."]}
```

| 필드       | 뜻                                                                 |
|------------|--------------------------------------------------------------------|
| `type`     | `error` / `fatal` / `warning` / `note`                             |
| `code`     | 에러 종류의 코드(아래 표)                                          |
| `kind`     | 에러 종류의 이름                                                   |
| `message`  | 메시지                                                             |
| `spans`    | 위치 목록. 첫 원소가 주 위치(`is_primary: true`), 나머지는 보조 위치 |
| `notes`    | note 문자열 목록. help 항목은 `help: `로 시작                     |

span에는 `file_name`, `line_start`, `col_start`, `byte_start`, `byte_end`, `is_primary`, `label`이 있습니다. 위치 없이 파일만 아는 진단은 `{"file_name": .., "is_primary": true}`만 둡니다.

IDE와 CI 시스템은 이 형식으로 진단을 기계적으로 읽을 수 있습니다.

### 진단 카테고리

진단 코드는 영역별로 다음과 같이 나뉩니다.

| 코드 접두사     | 영역               |
|-----------------|--------------------|
| `E001` ~ `E099` | 어휘 / 구문        |
| `E100` ~ `E199` | 이름 해소 / 가시성 |
| `E200` ~ `E299` | 타입 시스템        |
| `E300` ~ `E399` | 메타 / 매직        |
| `E400` ~ `E499` | 빌드 / 링크        |
| `E500` ~        | 추가 카테고리      |

코드는 에러 종류마다 하나입니다. 같은 종류의 진단에는 같은 코드가 붙습니다.

| 코드   | `kind`        | 종류                         |
|--------|---------------|------------------------------|
| `E001` | `lexical`     | 어휘                         |
| `E002` | `parse`       | 구문                         |
| `E100` | `resolve`     | 이름 해소                    |
| `E101` | `visibility`  | 가시성                       |
| `E102` | `module`      | 모듈 / 파일                  |
| `E200` | `type`        | 타입                         |
| `E201` | `pattern`     | 패턴 / 완전성                |
| `E202` | `handoff`     | 넘긴 뒤의 사용([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync-스레드-경계-마커)) |
| `E300` | `meta`        | 메타 블록 / 속성 / 키워드 매직 |
| `E400` | `lowering`    | HIR / MIR lowering           |
| `E401` | `codegen`     | 코드 생성                    |
| `E402` | `linkage`     | 외부 도구 / 링크             |
| `E500` | `unsupported` | 컴파일러가 처리하지 않는 구문 |
| `E501` | `internal`    | 컴파일러 내부                |
| `E502` | `other`       | 그 밖                        |
| `E503` | `lint`        | 린트([6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어), `rokc lint`) |

이 사양은 영역별 코드 접두사를 보장합니다. 개별 코드가 생기면 같은 영역 안에서 세분합니다.

## 9.9.5. 로그 / 진행 표시

### 표준 출력

기본 빌드 출력은 다음과 같습니다.

```
   Compiling foo (path/to/foo)
    Building target/debug/libfoo.a
    Finished in 1.23s
```

진행 줄은 stderr에 나옵니다. 동사 목록은 [9.1.8](901-rokc-cli.md#918-출력-형식)에 있습니다.

### `--verbose` / `-v`

상세 모드에서는 단계별 시간을 함께 표시합니다.

```
   Compiling foo (path/to/foo)
   Stage: front end (134ms)
    Building target/debug/foo
   Stage: codegen + link (812ms)
    Finished in 0.95s
```

각 단계의 시간이 나오므로 빌드 병목을 찾을 때 쓸 수 있습니다. 단계는 `front end`와 `codegen + link`(`--emit`이면 `emit <ext>`)입니다.

### `ROK_TIME_PASSES`

`ROK_TIME_PASSES=1`은 컴파일러 패스마다 걸린 실제 시간을 stderr에 출력합니다.

### `-vv`(매우 자세히)

`-v`의 출력에 더해 처리한 파일마다 `Reading <file>` 줄을 출력합니다. [특수화](../000-meta/007-glossary.md#specialize)한 (항목, 타입) 쌍마다 `Specialized <path>(sig)` 줄도 출력합니다.

## 9.9.6. 디버거 통합: gdb / lldb

### DWARF 디버그 정보

`debug` 프로필 빌드와 `-g`를 준 `release` 빌드([9.2.15](902-aot-pipeline.md#9215-산출물))는 다음과 같은 DWARF 디버그 정보를 내보냅니다. Windows 타깃에서는 그 대신 같은 정보를 담은 CodeView를 PDB에 냅니다.

- 함수의 `DW_AT_name`은 Rok 경로(`geom::distance`)입니다. linkage name(맹글링된 심볼)은 두지 않습니다. 디버거가 `_Y` 맹글링을 모르기 때문입니다.
- 줄 번호는 원본 .rok 파일의 줄을 가리킵니다.
- 지역 변수는 원본 이름과 타입으로 보입니다. 원시 타입(정수, 부동소수, `bool`, `char`) 외에 struct(필드 이름), 튜플, `str`, `any`, 합 타입, 배열의 DWARF 타입을 냅니다.
- 합 타입과 enum은 DWARF variant part입니다. variant part에는 artificial discriminant 멤버가 하나 있고, 변형마다(데이터 없는 변형 포함) `DW_AT_discr_value`가 붙은 `DW_TAG_variant`가 하나씩 있습니다. 각 변형은 이름이 `<Sum>::<Variant>`인 struct를 담습니다. 두 백엔드가 같은 형태를 씁니다. 디버거가 variant part를 읽도록 컴파일 단위는 `DW_LANG_Rust`를 씁니다(Rok의 DWARF 언어 코드는 없음).
- `GcBuf<T>` 참조는 그 객체(헤더, 그다음 `T` 원소)를 가리키는 포인터로 기술합니다.
- `any` 값은 그 안에 저장된 타입의 variant part이고, discriminant는 저장된 태그입니다. discriminant는 artificial `i32`입니다. `throws` carrier의 `bool` 태그는 바이트입니다.
- 함수 값(`fn` 타입이나 클로저)은 그 타입 이름(`fn(i32) -> i32`)의 struct입니다. 멤버 `code`는 `DW_TAG_subroutine_type`을 가리키는 포인터이고, 그 매개변수는 환경, 그다음 함수 매개변수입니다. 멤버 `env`는 그 환경이고, 클로저의 타입을 알면 환경 객체를 가리키는 포인터입니다. 그래서 디버거가 대상 함수를 보여 줍니다. 참조로 캡처한 변수는 그 값으로 보입니다.
- 컴파일러가 만드는 함수(C `main`, static 초기화 가드)에도 subprogram이 있습니다. 컴파일러가 만든 지역 변수(이름이 `#`이나 `__`로 시작)는 빠집니다.
- 블록마다 lexical block scope를 냅니다. 그래서 가려진(shadowed) 이름은 해당 블록 안에서만 보입니다.
- 스택 슬롯이 있는 지역 변수는 debug와 release 모두 `llvm.alloca` 하나에 놓입니다. 그래서 `-g`는 최적화 수준과 상관없이 같은 슬롯에 디버그 정보를 붙입니다.
- 블록마다 lexical block scope를 냅니다. 그래서 가려진(shadowed) 이름은 해당 블록 안에서만 보입니다.
- 스택 슬롯이 있는 지역 변수는 debug와 release 모두 `llvm.alloca` 하나에 놓입니다. 그래서 `-g`는 최적화 수준과 상관없이 같은 슬롯에 디버그 정보를 붙입니다.

### gdb / lldb 사용

표준 gdb와 lldb 명령어가 Rok 코드에서 그대로 동작합니다.

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

### 포매터

gdb는 DWARF만으로 합, enum, `any`의 활성 변형만 보여 줍니다. lldb에는 그 단위 언어의 플러그인이 없어서 모든 변형을 나열합니다. toolchain은 활성 변형만 보여 주는 lldb 포매터 `lib/rok/lldb/rok_lldb.py`를 함께 배포합니다. 포매터는 변형 이름을 summary로, 그 필드를 children으로 보여 줍니다. 박스에 담긴 `any` 페이로드는 그 박스를 거쳐 보여 줍니다. `rokc --print lldbinit`([9.1.4](901-rokc-cli.md#rokc---print-lldbinit-lldb-포매터))은 이 포매터를 로드하는 lldb 명령을 출력합니다.

```
lldb -O "$(rokc --print lldbinit)" ./target/debug/my_binary
rokc --print lldbinit >> ~/.lldbinit
```

### macOS / Windows 디버거

각 플랫폼의 기본 디버거가 Rok 바이너리를 처리합니다.

- macOS에서는 lldb나 Xcode 디버거를 씁니다.
- Windows에서는 Visual Studio 디버거나 WinDbg(PDB 디버그 정보)를 씁니다.

## 9.9.7. 프로파일러 통합

### perf (Linux)

CPU 프로파일링은 `perf`로 할 수 있습니다.

```
perf record ./target/release/my_binary
perf report
```

Rok 함수는 디맹글링한 이름으로 표시됩니다.

### 프로필 기반 최적화

Profile-guided optimization([9.1.2](901-rokc-cli.md#rokc-build-aot-빌드))은 세 단계로 이루어집니다.

```
rokc build --release -C profile-generate=prof    # 1. 계측 빌드
./target/release/my_binary <대표 입력>            # 2. 실행 - prof/default_*.profraw 를 씀
rokc profdata merge -o app.profdata prof          # 3. 병합한 뒤
rokc build --release -C profile-use=app.profdata  #    profile 로 다시 빌드
```

선택 단계인 context-sensitive 단계에서는 `-C profile-use=app.profdata -C cs-profile-generate=cs`로 빌드해 다시 실행합니다. 그런 다음 `cs`를 `app.profdata`와 합쳐 최종 프로필을 만듭니다. `wasm32-wasi`는 지원하고 `wasm32-unknown-unknown`은 지원하지 않습니다.

### Tracy / Chrome trace(선택)

표준 라이브러리의 tracing 표면으로 `tracy`나 Chrome `chrome://tracing`과 호환되는 트레이스를 출력할 수 있습니다. 구체적인 라이브러리 선택은 이 사양의 범위 밖이고, 별도 라이브러리 사양에서 다룹니다.

## 9.9.8. 메모리 / GC 도구

런타임은 다음 환경 변수를 읽습니다.

| 변수                      | 효과                                                                         |
|---------------------------|------------------------------------------------------------------------------|
| `ROK_GC_LOG=1`            | 수집마다 종류(minor / full), 일시 정지 시간, mark 수, sweep 결과를 stderr에 출력. 동시 full 수집은 시작 일시 정지(minor 수집)와 끝 일시 정지를 따로 기록함. sweep 결과는 살아 있는 바이트의 전 -> 후 |
| `ROK_GC_STATS=1`          | 종료할 때 수집 횟수(minor / full), 일시 정지 시간, 수집기 CPU 시간을 stderr에 출력. 둘째 줄은 블록 수 / 상주 크기 / evacuation 횟수 / 옮긴 바이트 |
| `ROK_GC_CONCURRENT=auto\|always\|off` | full 수집의 동시 mark. `auto`는 old 객체가 32 MiB 이상일 때([7.5](../007-runtime/705-concurrent-collection.md)) |
| `ROK_GC_EVAC=always`      | 테스트용. full 수집마다 evacuation([7.3.1](../007-runtime/703-gc-algorithm.md#731-immix-mark-region)) |
| `ROK_GC_STRESS=N`         | 테스트용. 할당 N번마다 수집(그 8번째마다 full)                             |
| `ROK_HEAP_DUMP=on_oom`    | OOM이 나면 합계, 바이트 기준 상위 32개 타입, 가장 큰 큰 객체 32개를 stderr에 출력. 최선 노력이며 할당하지 않고 락을 잡지 않음. 형식은 런타임이 정함 |
| `ROK_WORKERS=<n>`         | executor 워커 수 N(기본값은 온라인 CPU 수)([7.6.8](../007-runtime/706-scheduler.md#768-thread-per-core-옵션)) |
| `ROK_THREAD_PER_CORE=1`   | thread-per-core. task를 자기 워커에 고정하고 훔쳐 가지 않음(7.6.8) |
| `ROK_DYNAMIC_WORKERS=1`   | 쉬는 워커는 재우고, 활성 워커가 모두 바쁘면 다시 씀([7.7.8](../007-runtime/707-io-and-timer.md#778-워커-수의-동적-조절)) |

설정하지 않았거나 비어 있는 변수는 기본값을 씁니다. 표에 없는 값을 주면 stderr에 경고를 한 줄 출력합니다. 경고는 변수, 준 값, 받는 값을 알립니다. 그 변수는 기본값을 씁니다.

## 9.9.9. REPL / Jupyter 디버그

| 수단                 | 효과                                                                 |
|----------------------|----------------------------------------------------------------------|
| `:type <식>`         | 식의 타입을 출력(실행하지 않음)                                      |
| `:reset`             | 세션을 처음 상태로 되돌림(라이브러리 경로의 `use`는 다시 넣음)         |
| `:add <dep> ..`      | 의존성을 추가하고(현재 환경에서는 `rok install`, 패키지의 세션에서는 `rok add`) 세션에 붙임([9.10.5](910-rok-frontend.md#9105-환경)) |
| `:help`, `:quit`     | 도움말, 종료                                                         |
| `ROK_REPL_TIME=1`    | 셀마다 단계별 시간과, 새로 컴파일한 함수 / 앞 셀에서 재사용한 함수 수([9.6.4](906-jit-engine.md#964-라이브러리-코드의-재사용)) |

셀의 MLIR / LLVM IR을 보려면 같은 코드를 `main.rok`으로 두고 `rokc build --emit=mlir,llvmir`을 씁니다(9.9.1).

## 9.9.10. 메타 블록 디버그

[사용자 정의 메타 블록](../006-meta-system/608-meta-blocks.md)의 derive와 속성이 어떤 코드를 생성하는지 확인할 수 있습니다.

```
rokc build --emit=expanded      # → target/<profile>/build/<name>.expanded.rok
```

이 옵션은 사용자 패키지의 [메타 확장](../006-meta-system/609-template-tokens.md) 결과를 소스 텍스트로 출력합니다. 확장마다 다음 주석 뒤에 생성된 소스가 이어집니다. 생성물을 다시 확장한 결과는 바깥 확장 뒤에 이어집니다.

```
// expanded from this #[derive(MyHash)] (src/geom.rok:12:1)
impl MyHash for Point { ... }
```

메타 확장과 디슈거링을 모두 거친 전체 AST를 Rok 소스로 보려면 `--emit=ast`(`<name>.ast.rok`)를 씁니다.

## 9.9.11. 벤치마크

마이크로벤치마크는 다음 명령으로 실행할 수 있습니다.

```
rok bench
```

`rokc bench`가 벤치마크 하네스를 빌드하고, `rok bench`([9.10](910-rok-frontend.md))가 하네스를 실행합니다([9.1.2](901-rokc-cli.md#rokc-bench-벤치마크)). 하네스는 `#[bench]`가 붙은 함수를 `-O2` 빌드에서 하나씩 측정합니다. 각 함수는 `Bench` 핸들을 받아 `b.iter`로 클로저를 측정합니다. 그래서 클로저 밖의 준비 코드는 측정되지 않습니다. 측정 방법은 다음과 같습니다.

1. **클록.** 모노토닉 나노초 클록을 씁니다. 표본마다 호출 한 번이 아니라 `iters`번의 호출을 측정해 고정 오버헤드를 상각합니다.
2. **보정.** `--iters`를 주지 않으면 표본 하나가 약 100 ms가 될 때까지 `iters`를 키웁니다. 그러면 클록 양자화와 오버헤드가 무시할 수준이 됩니다.
3. **Warm-up.** 표본 하나 정도의 시간 동안 호출을 돌리고 결과는 버립니다. 이렇게 캐시, 분기 예측기, 페이징을 안정시킵니다.
4. **표본.** 표본을 `--samples`개(기본 100개) 얻습니다. 각 표본은 `iters`번 호출한 총시간을 기록합니다. 호출 한 번의 시간은 총시간을 `iters`로 나눈 값입니다.
5. **통계.** 평균 · 중앙값 · 표준편차 · 중앙값 절대편차(MAD) · 최소 · 최대를 계산합니다. Tukey 울타리(1.5 IQR mild, 3 IQR severe)를 벗어난 표본은 이상치로 보고하고 요약 수치에서는 뺍니다. 보고 수치는 **중앙값(+/- MAD)** 형태이므로 가끔 나오는 느린 표본에 흔들리지 않습니다.
6. **GC 인지.** 표본마다 런타임의 수집 카운터를 전후로 읽습니다. 수집이 일어난 표본은 이상치로 분류해 따로 셉니다. 그래서 GC 일시 정지가 중앙값을 왜곡하지 않습니다.

`black_box(x)`는 옵티마이저가 꿰뚫어 볼 수 없는 항등 함수입니다. 측정 클로저의 입력과 결과에 적용하세요. 그러면 작업이 밖으로 끌려 나가거나 접혀 사라지지 않습니다. `b.bytes(n)` / `b.items(n)`은 호출당 시간을 처리량(`MiB/s`, `Melem/s`)으로 바꿉니다.

터미널 출력은 사람이 읽는 단위로 정렬됩니다. 터미널이 아니면 회귀 도구가 읽는 안정된 기계용 형식을 벤치마다 한 줄씩 출력합니다. `--iters` / `--samples`로 표본을 조절합니다.

## 9.9.12. CI / 테스트 통합

### CI 모드

CI 환경에서는 기계가 읽을 수 있는 출력을 권장합니다.

```
rokc build --message-format=json --no-color > build.log
```

이 출력은 CI 시스템이 파싱하기 좋습니다.

### 종료 코드

[`rokc` CLI 인터페이스](901-rokc-cli.md)의 종료 코드는 다음과 같습니다.

| 코드  | 의미             |
|-------|------------------|
| `0`   | 성공             |
| `1`   | 사용자 코드 에러 |
| `2`   | CLI 에러         |
| `101` | rokc 내부 panic  |

`rokc test`는 하네스를 빌드할 뿐 테스트를 실행하지 않습니다. 실행은 `rok test`가 하고, 실패한 테스트가 있으면 `1`로 끝납니다. 테스트마다 별도 프로세스에서 돌므로 테스트의 panic(`101`)이 러너의 `101`이 되지는 않습니다.

CI는 코드마다 다른 처리(재시도, 알림 등)를 할 수 있습니다.

## 9.9.13. LSP 서버

`rokc`에는 LSP(Language Server Protocol) 모드가 있습니다. 같은 서버를 독립 바이너리 `rokls`(= `rok lsp`)로도 배포합니다. 서버는 stdio로 통신합니다. 지원하는 기능은 다음과 같습니다.

- 진단, hover(멤버 포함), go-to-definition(정의하는 이름의 범위를 돌려줌)
- references, rename, 문서 심볼
- 멤버 / 경로 완성, 포맷

references와 rename은 버퍼 안에서만 동작합니다.

```
rokc lsp
rokls
```

이 모드에서 VS Code, Vim, Emacs 같은 LSP 클라이언트는 자동 완성, hover, go-to-definition, 진단 등을 받습니다. 서버는 내부에서 `rokc check`의 증분 결과를 씁니다.

LSP의 세부 사항은 이 사양의 범위 밖입니다. LSP 표준과 rokc 구현 결정으로 따로 정합니다.

## 9.9.14. 패키지 검사

미리 빌드한 라이브러리 패키지의 정보를 한 번에 볼 수 있습니다.

```
rokc inspect path/to/libfoo.a        # 또는 foo.rokhir - 짝 파일을 함께 읽음
```

이 명령은 ROKHIR([9.4](904-rokhir-bytecode.md))과 [아카이브](../000-meta/007-glossary.md#archive)의 정보를 다음과 같이 표시합니다.

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

`rokc rokhir-text <file>`은 ROKHIR의 버전, producer, 메타와 인터페이스 파일 전체를 출력합니다(9.9.2).

패키지가 바깥에 무엇을 노출하는지 한눈에 볼 수 있습니다.

## 9.9.15. 정리

이 절의 도구는 Rok 개발과 디버그의 중심입니다. 표준 도구(gdb, perf, jupyter)를 통합하고, 그 위에 Rok 고유 도구(`rokc demangle`, `rokc rokhir-text` 등)를 더해 진단 환경을 이룹니다.

9장(실행과 도구)은 여기서 끝납니다. 이로써 이 사양 전체의 큰 그림이 완성됩니다.
