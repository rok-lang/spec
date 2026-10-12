# 9.9. 디버그 / 진단 도구

이 절에서는 `rok` 가 제공하는 디버그 / 진단 도구와 외부 도구 통합을 정의합니다.

## 9.9.1. 중간 산출물 출력: `--emit`

[AOT 빌드 파이프라인](902-aot-pipeline.md) 의 단계별 결과물을 덤프할 수 있습니다.

```
rok build --emit=hir          # rok-hir 텍스트 출력
rok build --emit=mir           # rok-mir 텍스트 출력
rok build --emit=mlir          # 표준 MLIR 텍스트
rok build --emit=llvmir        # LLVM IR
rok build --emit=asm           # 어셈블리
rok build --emit=obj           # object 파일
```

산출물은 다음 위치에 출력됩니다.

```
target/foo.hir
target/foo.mir
target/foo.mlir
target/foo.ll
target/foo.s
target/foo.o
```

여러 단계를 한 번에 지정하려면 `--emit=mlir,llvmir,asm` 처럼 쉼표로 구분하면 됩니다. 종류가 하나면 `-o <path>` 로 위치를 바꿀 수 있습니다. `--emit` 을 주면 실행 파일이나 archive 는 만들지 않습니다.

## 9.9.2. RHIR 검사

> [!NOTE]
> `rok rhir-text` 는 아직 제공하지 않습니다 (RHIR, [9.4](904-rhir-bytecode.md) 미구현).

```
rok rhir-text path/to/libfoo.a
```

위 명령은 [RHIR bytecode](904-rhir-bytecode.md) 를 사람이 읽을 수 있는 형식으로 출력해 줍니다.

```
rhir 1.2.3
package: foo (hash: 3f8a1c2b)
rok version: 1.0.0
abi version: ...

types:
  [0] i32
  [1] f64
  [2] struct Point { x: f64; y: f64; }
  [3] fn (Point, Point) -> f64
  ...

items:
  [0] pub fn distance: type [3]
      generics: [T: Num]
      body: at offset 1024
  ...
```

이 출력은 [RHIR](../000-meta/007-glossary.md#rhir-rok-hir-bytecode) 의 정확성과 호환성을 검증할 때 활용하면 됩니다.

## 9.9.3. 심볼 demangle

```
rok demangle <symbol>
rok demangle _Y3f8a1c2b_geom_distance_a7c91d8e44b2f190
```

이 명령은 [심볼 맹글링](905-symbol-mangling.md) 의 mangled 심볼을 원본 형식으로 출력해 줍니다. 심볼을 여러 개 주면 블록을 빈 줄로 나눕니다. 맹글링은 루트 `rok::` 를 남기지 않으므로, 경로의 첫 세그먼트가 `core` / `std` / `bare` 이면 `rok::` 를 붙여 표준 라이브러리 경로로 보여 줍니다.

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
error: 'count' is not declared
  --> src/main.rok:5:10
   |
 5 |     let x = count + 1;
   |             ^^^^^ unknown identifier
   |
   = note: did you mean 'counter'?
```

이 형식은 [소스 파일](../001-lexical/101-source-files.md) 의 파일 + 줄 + 컬럼 진단 단위를 사용합니다.

### JSON 형식 (도구 통합)

도구 통합을 위해 JSON 형식으로도 출력할 수 있습니다.

```
rok build --message-format=json
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
| `notes`    | note 문자열 목록                                                   |

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

현재 코드는 에러 종류마다 하나로 고정되어 있습니다. 같은 종류의 진단은 같은 코드를 가지며, 개별 진단 코드는 아직 없습니다.

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
| `E500` | `unsupported` | 아직 지원하지 않는 구문      |
| `E501` | `internal`    | 컴파일러 내부                |
| `E502` | `other`       | 그 밖                        |

본 사양은 영역별 코드 prefix 를 보장합니다. 개별 코드가 생기면 같은 영역 안에서 나뉩니다.

## 9.9.5. 로그 / 진행 표시

### 표준 출력

기본 빌드 출력은 다음과 같은 형태로 표시됩니다.

```
   Compiling foo (path/to/foo)
    Building target/libfoo.a
    Finished in 1.23s
```

진행 줄은 stderr 에 나오며, 동사 목록은 [9.1.8](901-rok-cli.md#918-출력-형식) 에 있습니다.

### `--verbose` / `-v`

상세 모드에서는 단계별 시간을 함께 표시합니다.

```
   Compiling foo (path/to/foo)
   Stage: front end (134ms)
    Building target/foo
   Stage: codegen + link (812ms)
    Finished in 0.95s
```

각 단계의 시간을 표시하므로, 빌드 병목을 진단할 때 유용합니다. 단계를 얼마나 잘게 나눌지는 `rok` 가 정합니다 (예: parse, type-check, specialize, mlir-opt, clang). 현재는 `front end` 와 `codegen + link` (`--emit` 이면 `emit <ext>`) 입니다.

### `-vv` (매우 자세히)

이 모드에서는 각 단계의 내부 정보 (처리한 파일 목록, [specialize](../000-meta/007-glossary.md#specialize) 의 (item, type) 쌍 등) 를 함께 출력합니다. 현재는 `-v` 와 같은 출력입니다.

## 9.9.6. 디버거 통합: gdb / lldb

### DWARF 디버그 정보

`rok build --debug` 또는 `-g` 옵션을 사용하면 DWARF 디버그 정보를 emit 합니다. 이때 다음과 같은 특성을 갖습니다.

- 함수 이름은 demangled 형태로 표시됩니다 (디버거가 이해합니다).
- 변수 이름과 타입은 원본 rok 식별자 그대로 유지됩니다.
- 줄 번호는 원본 .rok 파일의 줄을 가리킵니다.

### gdb / lldb 사용

표준 gdb 와 lldb 명령어가 rok 코드에 그대로 동작합니다.

```
gdb ./target/my_binary
(gdb) break main
(gdb) run
(gdb) print x
$1 = (i32) 42
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
perf record ./target/my_binary
perf report
```

이 경우 rok 함수가 demangled 이름으로 표시됩니다.

### Tracy / Chrome trace (선택)

표준 라이브러리의 tracing 표면을 통해 `tracy` 또는 Chrome `chrome://tracing` 과 호환되는 trace 를 출력할 수 있습니다. 구체적인 라이브러리 선택은 본 사양의 범위 밖이며, 별도 라이브러리 spec 에서 다룹니다.

## 9.9.8. 메모리 / GC 도구

### GC 로그

런타임 환경 변수를 통해 GC 로그를 활성화할 수 있습니다.

```
ROK_GC_LOG=1 ./target/my_binary
```

이 옵션은 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 사이클마다 pause time, mark count, sweep result 등을 stderr 에 출력합니다.

### heap dump

OOM 상황에서는 heap dump 를 남길 수 있습니다.

```
ROK_HEAP_DUMP=on_oom ./target/my_binary
```

OOM 시 heap 의 객체 분포와 큰 객체 등을 dump 하며, 자세한 형식은 GC 구현이 결정합니다.

## 9.9.9. REPL / Jupyter 디버그

### JIT 모듈 dump

REPL / Jupyter 안에서는 다음과 같은 명령으로 JIT 산출물을 확인할 수 있습니다.

```
:dump_mlir         # 마지막 cell 의 MLIR
:dump_llvmir       # 마지막 cell 의 LLVM IR
:dump_asm          # 마지막 cell 의 어셈블리
```

이를 통해 [JIT](../000-meta/007-glossary.md#jit-just-in-time) 의 내부 단계를 대화식으로 확인할 수 있습니다.

### specialize 캐시 inspect

캐시 동작도 명령으로 점검할 수 있습니다.

```
:spec_cache        # 현재 spec_cache 의 항목 목록
:spec_cache clear   # 캐시 비우기 (강제 재컴파일)
```

이 명령들은 JIT 의 캐시 동작 디버그에 사용합니다.

## 9.9.10. 메타 블록 디버그

[사용자 정의 메타 블록](../006-meta-system/608-meta-blocks.md) 의 derive / attribute 가 어떤 코드를 생성하는지 확인할 수 있습니다.

> [!NOTE]
> `--emit=expanded` 는 아직 제공하지 않습니다.

```
rok build --emit=expanded
```

이 옵션은 [템플릿 토큰과 확장](../006-meta-system/609-template-tokens.md) 의 메타 확장 결과 AST 를 출력해 주며, 사용자 derive 의 디버그 용도로 활용하면 됩니다.

## 9.9.11. 벤치마크

마이크로벤치마크는 다음 명령으로 실행할 수 있습니다.

```
rok bench
```

이 명령 ([9.1.2](901-rok-cli.md#rok-bench-벤치마크)) 은 `#[bench]` 가 부착된 함수들을 마이크로벤치마크로 실행하며, 자세한 벤치마크 인프라는 표준 라이브러리 / 도구에서 결정됩니다.

## 9.9.12. CI / 테스트 통합

### CI 모드

CI 환경에서는 machine-readable 출력을 사용하면 좋습니다.

```
rok build --message-format=json --no-color > build.log
```

이 출력은 CI 시스템이 파싱하기에 적합합니다.

### 종료 코드

[`rok` CLI 인터페이스](901-rok-cli.md) 의 종료 코드는 다음과 같이 정의됩니다.

| 코드  | 의미             |
|-------|------------------|
| `0`   | 성공             |
| `1`   | 사용자 코드 에러 |
| `2`   | CLI 에러         |
| `101` | rok 내부 panic  |

`rok test` 는 실패한 테스트가 있으면 `1` 입니다. 테스트마다 별도 프로세스이므로 테스트의 panic (`101`) 이 rok 의 `101` 이 되지는 않습니다.

CI 가 코드별로 다른 처리 (재시도, 알림 등) 를 적용할 수 있습니다.

## 9.9.13. LSP 서버

`rok` 는 LSP (Language Server Protocol) 모드를 둘 예정이며, 아직 제공하지 않습니다. 같은 서버를 독립 바이너리 `rokls` 로도 제공합니다.

```
rok lsp
rokls
```

이 모드를 사용하면 VS Code, Vim, Emacs 등 LSP 클라이언트가 자동 완성, hover, go-to-definition, 진단 등을 받아옵니다. 내부적으로는 `rok check` 의 점진적 결과를 활용합니다.

자세한 LSP 명세는 본 사양 외이며 (LSP 표준 + rok 구현 결정), 별도로 정의됩니다.

## 9.9.14. 패키지 검사

패키지 전체 정보를 한 번에 확인할 수 있습니다. 아직 제공하지 않습니다.

```
rok inspect path/to/libfoo.a
```

이 명령은 [archive](../000-meta/007-glossary.md#archive) 의 전체 정보 (exports, [generic](../000-meta/007-glossary.md#generic--제네릭) items, 의존, 빌드 메타 등) 를 다음과 같이 표시합니다.

```
Package: foo
Hash: 3f8a1c2b
rok version: 1.0.0
ABI version: ...

Exports:
  pub fn distance: fn(Point, Point) -> f64
  pub struct Point { ... }
  ...

Generic items (RHIR):
  pub fn id<T>(x: T) -> T
  ...

Concrete .o files:
  foo.concrete.o (15 KB)
  foo.concrete.o.1 (8 KB)
```

이를 통해 패키지의 외부에 보이는 모습을 한눈에 파악할 수 있습니다.

## 9.9.15. 정리

지금까지 살펴본 도구들이 rok 의 개발과 디버그 경험의 핵심을 이룹니다. 표준 도구 (gdb, perf, jupyter) 와의 통합과 rok 고유 도구 (`rok demangle`, `rok rhir-text` 등) 가 결합되어 풍부한 진단 환경을 제공합니다.

이로써 9 장 (실행과 툴) 을 마무리하며, 본 사양 전체가 큰 그림을 완성하게 됩니다.
