# 9.4. ROKHIR 포맷

미리 빌드한 라이브러리 패키지의 인터페이스 파일 `<name>.rokhir` 를 정의합니다. [ROKHIR](../000-meta/007-glossary.md#rokhir-rok-hir-bytecode) 파일은 그 패키지의 `lib<name>.a` ([패키지 아카이브](903-package-archive.md)) 옆에 놓이며, archive 의 멤버가 아닙니다.

## 9.4.1. 동기

[AOT 빌드 파이프라인](902-aot-pipeline.md) 의 단형화는 호출 지점의 구체 타입을 기준으로 [specialize](../000-meta/007-glossary.md#specialize) 합니다. 따라서 다른 패키지가 한 패키지의 [generic](../000-meta/007-glossary.md#generic--제네릭) `fn` 을 호출하면 그 호출 지점에서 specialize 가 일어나야 합니다. [concrete fn](../000-meta/007-glossary.md#concrete-fn--generic-fn) 의 obj 만으로는 부족하므로, 다운스트림이 읽을 인터페이스를 따로 둡니다.

단형화의 입력은 AST 입니다. 그래서 ROKHIR 의 body 는 **잘라 낸 패키지 소스** 이며, 다운스트림의 파서가 그것을 읽습니다.

## 9.4.2. 외부 라이브러리 미사용

의존성 0 원칙에 따라 직렬화 라이브러리도 사용하지 않으며, ROKHIR 은 직접 설계합니다.

## 9.4.3. 전체 레이아웃

```
ROKHIR file (little-endian):
+-----------------------------------+
|  Header                           |  (16 bytes fixed)
|    magic       "ROKH"    (4 bytes)  |
|    version     u32  (= 6)         |
|    flags       u32                |
|    body_offset u32                |
+-----------------------------------+
|  Producer string        str       |  "rokc <version> (<compiler id>)"
+-----------------------------------+
|  Meta                             |
|    name                 str       |  mount name (--extern <name>=)
|    hash                 str       |  package hash (0.3.3)
|    source dir           str       |
|    gc type offset       varint    |
|    gc type count        varint    |
|    features  varint n, n × str    |
+-----------------------------------+
|  Body (at body_offset)            |
|    file count           varint    |
|    file × n: path str, text str   |
+-----------------------------------+
|  Summaries                        |
|    count                varint    |
|    n × (function str, summary str) |
+-----------------------------------+
```

`str` 은 PrefixVarInt 길이 (9.4.5) 뒤의 UTF-8 바이트입니다. string / type / item section 은 두지 않습니다.

## 9.4.4. Header 와 Producer

| 필드          | 크기 | 의미                                                         |
|---------------|------|--------------------------------------------------------------|
| `magic`       | 4    | `"ROKH"`                                                       |
| `version`     | u32  | 포맷 버전 (`6`). 레이아웃이나 body 의 내용이 바뀌면 올림: 2 는 `features` meta, 3 은 body 의 `.wit` 파일을 더함, 4 는 `include_str!` / `include_bytes!` 데이터를 inline 함, 5 는 hand-off summary 를 더함, 6 은 summary 의 exit 종류를 더함 |
| `flags`       | u32  | 예약 (0)                                                     |
| `body_offset` | u32  | body 의 시작 offset                                          |

producer 문자열은 `rokc <버전> (<compiler id>)` 입니다. compiler id 는 그 `rokc` 실행 파일의 크기와 수정 시각 (`<size>-<mtime>`, 16 진수) 입니다.

## 9.4.5. VarInt 인코딩

MLIR bytecode 와 같은 PrefixVarInt 이며 LEB128 의 변형입니다. 첫 바이트의 앞쪽 1 비트 개수가 뒤따르는 바이트 수이고, 값은 나머지 비트에 big-endian 으로 놓입니다.

```
1-byte int  : 0xxxxxxx                    (0 ~ 127)
2-byte int  : 10xxxxxx xxxxxxxx           (0 ~ 16383)
3-byte int  : 110xxxxx xxxxxxxx xxxxxxxx  (0 ~ 2097151)
...
```

## 9.4.6. Meta

| 필드             | 의미                                                                              |
|------------------|-----------------------------------------------------------------------------------|
| name             | 패키지의 mount 이름. 심볼에 들어가므로 `--extern <name>=` 과 같아야 함            |
| hash             | 패키지 해시 ([0.3.3](../000-meta/003-source-units.md#033-패키지-식별자)). 심볼 맹글링에 쓰인 값. 표준 라이브러리 패키지 `rok` 에서는 tier 마다 하나씩인 `pkg=hash;...` 목록 |
| source dir       | 빌드 때 패키지 소스가 있던 디렉토리. body 의 파일 경로의 기준이며, 다운스트림은 디스크가 아니라 body 를 읽음 |
| gc type offset / count | 이 패키지의 GC type id 구간 `[offset, offset + count)` (9.4.8)              |
| features         | 패키지를 빌드할 때의 `feature` cfg 값. downstream 은 빌드의 다른 cfg 와 이 feature 로 body 를 파싱합니다 |

## 9.4.7. Body

body 는 패키지의 **모든 소스 파일** 을 (경로, 텍스트) 쌍으로 담습니다. 각 파일은 다음과 같이 잘려 있습니다.

- 구체 (concrete) 함수의 본문은 `;` 로 잘라 냅니다. 줄바꿈은 남기므로 진단의 줄 번호가 원본과 같습니다. 이 함수들은 archive 의 심볼로 링크됩니다.
- 다운스트림이 단형화하거나 inline 해야 하는 본문은 남깁니다: 제네릭 함수, `async` / `yield` 함수, `#[inline]` 함수, `extern "C"` 정의, 트레잇의 기본 메서드, 제네릭 `impl` 의 메서드.
- 시그니처, 타입, 트레잇, `impl`, `use` 같은 선언은 그대로 둡니다.
- 남긴 텍스트의 `include_str!` / `include_bytes!` 는 그것이 읽은 리터럴로 바꿉니다 (줄 바꿈 유지). 그래서 다운스트림은 포함 파일을 읽지 않으며, 옮긴 라이브러리도 그대로 동작합니다.

소스 파일 뒤에는 패키지의 `ffi::wit` 항목이 읽는 `.wit` 파일이 ([stdlib 1.8](../../refs-standard-libraries/100-bare/108-wit.md)) 그 `deps/` 와 함께 통째로 옵니다. 경로는 source dir 기준이며 `..` 로 그 밖으로 나갈 수 있습니다. 다운스트림은 WIT 를 디스크가 아니라 body 에서 읽습니다.

body 뒤에는 패키지 함수들의 hand-off summary ([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---스레드-경계-marker)) 가 옵니다. summary 는 tab 으로 구분한 필드의 줄들입니다:

- `g <root> <mutex|rwlock|channel> <line> <function> <file>`: `<root>` 를 넘김;
- `s <root> <root>`: 하나의 객체;
- `i <root> <root>`: 첫째가 사라지면 둘째를 함께 가져감;
- `t <guard parameter> <root>`: 그 guard 에 묶임.
- `x <tag|*> [<root>]`: 반환한 값이 그 tag 를 가지는 exit 들에서 (`*`: 알 수 없음; `bool` 의 0 / 1, 합이나 enum 변형, carrier 의 경우 에러는 1, 값은 0, 2 + 값의 tag), `<root>` 가 넘긴 것들 가운데 있음. `x` 줄이 없으면 모든 exit 가 모든 `g` root 를 넘김.

root 는 매개변수 index 나 `r` (반환값) 이며, 그 뒤에 `.` 으로 구분한 필드 경로가 옵니다 (tuple index 는 bit 31 이 켜짐). `#fn` (`#fne`: 클로저, 환경 먼저) 이라는 이름 뒤에 U+001F 로 구분한 매개변수 타입 key 가 붙은 항목은 그 매개변수 타입으로 값으로 쓰인 함수들의 summary 를 합칩니다.

## 9.4.8. 분리 컴파일을 위한 규칙

라이브러리를 미리 빌드하면 프로그램 전체를 한 번에 보지 못하므로, 다음 규칙으로 패키지 사이를 맞춥니다.

- **`any` tag.** `any` 슬롯의 tag 는 타입 이름의 31 비트 해시입니다. 그래서 따로 빌드한 패키지가 같은 타입에 같은 tag 를 씁니다.
- **`any` seed.** 라이브러리가 `any` 에 넣는 타입을 프로그램이 알도록, 라이브러리는 그 타입들을 쓰는 작은 rok 함수를 body 끝에 덧붙입니다. 프로그램의 닫힌 세계 `any` dispatcher 가 이 타입들도 포함합니다.
- **dispatcher 심볼.** `any` dispatcher 는 `rok.any.<hash>` 심볼이며, 라이브러리 안에서는 weak, 프로그램에서는 strong 으로 정의됩니다. 링크하면 프로그램의 것이 이깁니다.
- **합 항 순서.** 합의 항은 내부 타입 번호가 아니라 구조적 키로 정렬하므로 ([2.6.4](../002-types/206-sum-types.md#정규-순서)) 패키지 사이에서 tag 가 일치합니다.
- **GC type id.** 패키지마다 GC type id 구간 (9.4.6) 을 따로 가집니다. 한 프로그램에서 두 패키지의 구간이 겹치면 에러입니다.
- **메서드 심볼.** 의존 패키지의 메서드는 안정된 경로 `<모듈>::<타입>.<메서드>[__by_<트레잇>]` 으로 맹글링됩니다 ([9.5](905-symbol-mangling.md)).

## 9.4.9. 읽기와 거부

`--extern name=<name>.rokhir` 나 `--extern name=lib<name>.a` 는 서로 옆의 짝 파일을 찾습니다 ([9.2.3](902-aot-pipeline.md#923-의존성-해소)). 다운스트림은 body 를 소스 overlay 로 파싱해 선언을 얻고, 잘린 함수는 미리 빌드된 심볼로 선언하며, 제네릭은 그 자리에서 단형화한 뒤 archive 를 링크합니다.

다음이 하나라도 다르면 ROKHIR 을 거부하고 "다시 빌드하라 (rebuild)" 는 에러를 냅니다.

- `version`
- producer (다른 `rokc` 가 만든 ROKHIR)
- name (`--extern` 이름과 ROKHIR 의 mount 이름)

## 9.4.10. ROKHIR 검사

`rokc rokhir-text <file>` 은 ROKHIR 의 버전, producer, meta (`features:` 포함) 와 interface 파일들의 경로와 텍스트, 그리고 summary 를 출력합니다 ([9.9.2](909-debug-tools.md#992-rokhir-검사)). `rokc inspect` ([9.9.14](909-debug-tools.md#9914-패키지-검사)) 는 archive 와 함께 요약을 보여 줍니다.

## 9.4.11. 안정성

ROKHIR 은 같은 `rokc` 가 만든 것만 읽습니다 (9.4.9). 포맷이 바뀌면 `version` 을 올리며, 버전이 다르면 다시 빌드해야 합니다. [개요](../008-abi/README.md) 의 안정성 정책을 적용합니다.

표준 라이브러리는 ROKHIR 로 배포하지 않습니다 ([9.3.6](903-package-archive.md#936-표준-라이브러리)).
