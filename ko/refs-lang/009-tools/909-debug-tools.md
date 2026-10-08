# 9.9. 디버그 / 진단 도구

`nokc` 가 제공하는 디버그 / 진단 도구와 외부 도구 통합을 정의한다.

## 9.9.1. 중간 산출물 출력: `--emit`

[AOT 빌드 파이프라인](902-aot-pipeline.md) 의 단계별 결과물을 덤프할 수 있다.

```
nokc build --emit=hir          # nok-hir 텍스트 출력
nokc build --emit=mir           # nok-mir 텍스트 출력
nokc build --emit=mlir          # 표준 MLIR 텍스트
nokc build --emit=llvmir        # LLVM IR
nokc build --emit=asm           # 어셈블리
nokc build --emit=obj           # object 파일 (default)
```

다음 위치에 출력된다.

```
target/foo.hir
target/foo.mir
target/foo.mlir
target/foo.ll
target/foo.s
target/foo.o
```

여러 단계를 한 번에 지정하려면 `--emit=mlir,llvmir,asm` 처럼 쉼표로 구분한다.

## 9.9.2. NHIR 검사

```
nokc nhir-text path/to/libfoo.a
```

[NHIR bytecode](904-nhir-bytecode.md) 를 사람이 읽을 수 있는 형식으로 출력한다.

```
nhir 1.2.3
package: foo (hash: 3f8a1c2b)
nokc version: 1.0.0
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

[NHIR](../000-meta/007-glossary.md#nhir-nok-hir-bytecode) 의 정확성 / 호환성 검증에 쓴다.

## 9.9.3. 심볼 demangle

```
nokc demangle <symbol>
nokc demangle _N3f8a1c2b_geom_distance_a7c91d8e44b2f190
```

[심볼 맹글링](905-symbol-mangling.md) 의 mangled 심볼을 원본 형식으로 출력한다.

```
package:  3f8a1c2b
path:     geom
name:     distance
generics: a7c91d8e44b2f190 (type tuple hash)
```

## 9.9.4. 진단 / 에러 메시지

### 표준 형식

```
error: 'count' is not declared
  --> src/main.nok:5:10
   |
 5 |     let x = count + 1;
   |             ^^^^^ unknown identifier
   |
   = note: did you mean 'counter'?
```

[소스 파일](../001-lexical/101-source-files.md) 의 파일 + 줄 + 컬럼 진단 단위를 사용한다.

### JSON 형식 (도구 통합)

```
nokc build --message-format=json
```

각 줄이 한 JSON 객체이다.

```json
{"type":"error","code":"E001","message":"...","spans":[{"file_name":"...","line_start":5,"col_start":10,...}]}
```

IDE 와 CI 시스템이 진단을 기계로 판독할 수 있다.

### 진단 카테고리

| 코드 prefix     | 영역               |
| --------------- | ------------------ |
| `E001` ~ `E099` | 어휘 / 구문        |
| `E100` ~ `E199` | 이름 해소 / 가시성 |
| `E200` ~ `E299` | 타입 시스템        |
| `E300` ~ `E399` | 메타 / 매직        |
| `E400` ~ `E499` | 빌드 / 링크        |
| `E500` ~        | 추가 카테고리      |

자세한 코드와 메시지는 `nokc` 가 결정한다. 본 사양은 영역별 코드 prefix 만 보장한다.

## 9.9.5. 로그 / 진행 표시

### 표준 출력

```
   Compiling foo v0.1.0 (path/to/foo)
    Building target/libfoo.a
    Finished release [optimized] target(s) in 1.23s
```

### `--verbose` / `-v`

```
   Compiling foo v0.1.0 (path/to/foo)
   Stage: parsing 23 files (45ms)
   Stage: type-check (89ms)
   Stage: specialize 142 instances (67ms)
   Stage: MLIR text emit (34ms)
   Stage: mlir-opt (456ms)
   Stage: mlir-translate (123ms)
   Stage: clang -c (234ms)
   Stage: ar/link (45ms)
   Finished in 1.10s
```

각 단계의 시간을 표시한다. 빌드 병목 진단에 쓴다.

### `-vv` (매우 자세히)

각 단계의 내부 정보 (처리한 파일 목록, [specialize](../000-meta/007-glossary.md#specialize) 의 (item, type) 쌍 등) 를 출력한다.

## 9.9.6. 디버거 통합: gdb / lldb

### DWARF 디버그 정보

`nokc build --debug` 또는 `-g` 로 DWARF 디버그 정보를 emit 한다.

- 함수 이름은 demangled 형태로 표시한다 (디버거가 이해).
- 변수 이름과 타입은 원본 nok 식별자 그대로.
- 줄 번호는 원본 .nok 파일의 줄.

### gdb / lldb 사용

```
gdb ./target/my_binary
(gdb) break main
(gdb) run
(gdb) print x
$1 = (i32) 42
(gdb) bt
#0  main at src/main.nok:5
...
```

표준 gdb 와 lldb 명령어가 nok 코드에 그대로 동작한다.

### macOS / Windows 디버거

- macOS: lldb / Xcode debugger.
- Windows: Visual Studio debugger / WinDbg (PDB 디버그 정보).

각 플랫폼의 기본 디버거가 nok 바이너리를 처리한다.

## 9.9.7. profiler 통합

### perf (Linux)

```
perf record ./target/my_binary
perf report
```

CPU 프로파일링이다. nok 함수가 demangled 이름으로 표시된다.

### Tracy / Chrome trace (선택)

표준 라이브러리에 tracing 라이브러리가 추가되면 `tracy` 또는 Chrome `chrome://tracing` 과 호환되는 trace 출력을 할 수 있다.

본 사양은 외부 도구 통합 가능성만 표시한다. 구체 라이브러리는 추후 결정한다.

## 9.9.8. 메모리 / GC 도구

### GC 로그

런타임 환경 변수:

```
NOK_GC_LOG=1 ./target/my_binary
```

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 사이클마다 pause time, mark count, sweep result 등을 stderr 에 출력한다.

### heap dump

```
NOK_HEAP_DUMP=on_oom ./target/my_binary
```

OOM 시 heap 의 객체 분포와 큰 객체 등을 dump 한다.

자세한 형식은 GC 구현이 결정한다.

## 9.9.9. REPL / Jupyter 디버그

### JIT 모듈 dump

REPL / Jupyter 안에서:

```
:dump_mlir         # 마지막 cell 의 MLIR
:dump_llvmir       # 마지막 cell 의 LLVM IR
:dump_asm          # 마지막 cell 의 어셈블리
```

[JIT](../000-meta/007-glossary.md#jit-just-in-time) 의 내부 단계를 대화식으로 확인한다.

### specialize 캐시 inspect

```
:spec_cache        # 현재 spec_cache 의 항목 목록
:spec_cache clear   # 캐시 비우기 (강제 재컴파일)
```

JIT 의 캐시 동작 디버그.

## 9.9.10. 메타 블록 디버그

[사용자 정의 메타 블록](../006-meta-system/608-meta-blocks.md) 의 derive / attribute 가 어떤 코드를 생성하는지 보여준다.

```
nokc build --emit=expanded
```

[템플릿 토큰과 확장](../006-meta-system/609-template-tokens.md) 의 메타 확장 결과 AST 를 출력한다. 사용자 derive 의 디버그용.

## 9.9.11. 벤치마크

```
nokc test --bench
```

`#[bench]` 가 부착된 함수들을 마이크로벤치마크로 실행한다.

자세한 벤치마크 인프라는 표준 라이브러리 / 도구 결정.

## 9.9.12. CI / 테스트 통합

### CI 모드

```
nokc build --message-format=json --no-color > build.log
```

machine-readable 출력이며 CI 시스템이 파싱한다.

### 종료 코드

[`nokc` CLI 인터페이스](901-nokc-cli.md) 의 종료 코드 정의:

| 코드  | 의미             |
| ----- | ---------------- |
| `0`   | 성공             |
| `1`   | 사용자 코드 에러 |
| `2`   | CLI 에러         |
| `101` | nokc 내부 panic  |

CI 가 코드별로 다른 처리를 한다 (재시도, 알림 등).

## 9.9.13. LSP 서버

`nokc` 는 LSP (Language Server Protocol) 모드도 제공한다.

```
nokc lsp
```

VS Code, Vim, Emacs 등 LSP 클라이언트가 자동 완성, hover, go-to-definition, 진단 등을 받아온다.

내부적으로 `nokc check` 의 점진적 결과를 활용한다.

자세한 LSP 명세는 본 사양 외이다 (LSP 표준 + nokc 구현 결정).

## 9.9.14. 패키지 검사

```
nokc inspect path/to/libfoo.a
```

[archive](../000-meta/007-glossary.md#archive) 의 전체 정보 (exports, [generic](../000-meta/007-glossary.md#generic-/-제네릭) items, 의존, 빌드 메타 등) 를 보여준다.

```
Package: foo
Hash: 3f8a1c2b
nokc version: 1.0.0
ABI version: ...

Exports:
  pub fn distance: fn(Point, Point) -> f64
  pub struct Point { ... }
  ...

Generic items (NHIR):
  pub fn id<T>(x: T) -> T
  ...

Concrete .o files:
  foo.concrete.o (15 KB)
  foo.concrete.o.1 (8 KB)
```

패키지의 외부에 보이는 모습을 한눈에 보여준다.

## 9.9.15. 정리

위 도구들이 nok 의 개발과 디버그 경험의 핵심이다. 표준 도구 (gdb, perf, jupyter) 와의 통합과 nok 고유 도구 (`nokc demangle`, `nokc nhir-text` 등) 를 결합한다.

이로써 9 장 (실행과 툴) 을 마무리하고 본 사양 전체가 큰 그림을 완성한다.
