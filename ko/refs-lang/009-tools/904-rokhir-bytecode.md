# 9.4. ROKHIR 포맷

미리 빌드한 라이브러리 패키지의 인터페이스 파일 `<name>.rokhir`을 정의합니다. [ROKHIR](../000-meta/007-glossary.md#rokhir-rok-hir-bytecode) 파일은 그 패키지의 `lib<name>.a`([패키지 아카이브](903-package-archive.md)) 옆에 놓입니다. 아카이브의 멤버는 아닙니다.

## 9.4.1. 동기

[AOT 빌드 파이프라인](902-aot-pipeline.md)의 단형화는 호출 지점의 구체 타입을 기준으로 [특수화](../000-meta/007-glossary.md#specialize)합니다. 따라서 다른 패키지가 한 패키지의 [제네릭](../000-meta/007-glossary.md#generic--제네릭) `fn`을 호출하면 그 호출 지점에서 특수화가 일어나야 합니다. [구체 fn](../000-meta/007-glossary.md#concrete-fn--generic-fn)의 오브젝트 코드만으로는 부족합니다. 그래서 다운스트림이 읽을 인터페이스를 따로 둡니다.

단형화의 입력은 AST입니다. 그래서 ROKHIR의 본문(body)은 **잘라 낸 패키지 소스**입니다. 다운스트림의 파서가 이 소스를 읽습니다.

## 9.4.2. 외부 라이브러리 미사용

의존성 0 원칙에 따라 직렬화 라이브러리도 쓰지 않습니다. ROKHIR 형식은 직접 설계합니다.

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

`str` 값은 PrefixVarInt 길이(9.4.5) 뒤에 오는 UTF-8 바이트입니다. 문자열, 타입, 항목 섹션은 두지 않습니다.

## 9.4.4. 헤더와 producer

| 필드          | 크기 | 의미                                                         |
|---------------|------|--------------------------------------------------------------|
| `magic`       | 4    | `"ROKH"`                                                       |
| `version`     | u32  | 포맷 버전(`6`). 레이아웃이나 본문 내용이 바뀌면 올림. 2는 `features` 메타, 3은 본문의 `.wit` 파일을 더함. 4는 `include_str!` / `include_bytes!` 데이터를 인라인함. 5는 넘김 요약을 더함. 6은 요약의 출구 종류를 더함 |
| `flags`       | u32  | 예약 (0)                                                     |
| `body_offset` | u32  | 본문의 시작 오프셋                                          |

producer 문자열은 `rokc <버전> (<compiler id>)`입니다. compiler id는 그 `rokc` 실행 파일의 크기와 수정 시각(`<size>-<mtime>`, 16진수)입니다.

## 9.4.5. VarInt 인코딩

MLIR 바이트코드와 같은 PrefixVarInt이고 LEB128의 변형입니다. 첫 바이트 앞쪽에 이어지는 1 비트의 개수가 뒤따르는 바이트 수입니다. 값은 나머지 비트에 빅 엔디언으로 놓입니다.

```
1-byte int  : 0xxxxxxx                    (0 ~ 127)
2-byte int  : 10xxxxxx xxxxxxxx           (0 ~ 16383)
3-byte int  : 110xxxxx xxxxxxxx xxxxxxxx  (0 ~ 2097151)
...
```

## 9.4.6. Meta

| 필드             | 의미                                                                              |
|------------------|-----------------------------------------------------------------------------------|
| name             | 패키지의 마운트 이름. 심볼에 들어가므로 `--extern <name>=`과 같아야 함           |
| hash             | 패키지 해시 ([0.3.3](../000-meta/003-source-units.md#033-패키지-식별자)). 심볼 맹글링에 쓰는 값. 표준 라이브러리 패키지 `rok`에서는 계층마다 하나씩 든 `pkg=hash;...` 목록 |
| source dir       | 빌드 때 패키지 소스가 있던 디렉터리. 본문 파일 경로의 기준. 다운스트림은 디스크가 아니라 본문을 읽음 |
| gc type offset / count | 이 패키지의 GC 타입 id 구간 `[offset, offset + count)` (9.4.8)              |
| features         | 패키지를 빌드할 때 쓴 `feature` cfg 값. 다운스트림은 빌드의 다른 cfg와 이 feature 값으로 본문을 파싱함 |

## 9.4.7. Body

본문에는 패키지의 **모든 소스 파일**이 (경로, 텍스트) 쌍으로 들어갑니다. 각 파일은 다음과 같이 잘려 있습니다.

- 구체(concrete) 함수의 본문은 `;`로 잘라 냅니다. 줄바꿈은 남기므로 진단의 줄 번호가 원본과 같습니다. 이런 함수는 아카이브의 심볼로 링크됩니다.
- 다운스트림이 단형화하거나 인라인해야 하는 본문은 남깁니다. 제네릭 함수, `async` / `yield` 함수, `#[inline]` 함수, `extern "C"` 정의, 트레잇의 기본 메서드, 제네릭 `impl`의 메서드가 여기에 해당합니다.
- 시그니처, 타입, 트레잇, `impl`, `use` 같은 선언은 그대로 둡니다.
- 남긴 텍스트의 `include_str!` / `include_bytes!`는 읽어 들인 리터럴로 바꿉니다(줄바꿈 유지). 그래서 다운스트림은 포함 파일을 읽지 않고, 다른 곳으로 옮긴 라이브러리도 그대로 동작합니다.

소스 파일 뒤에는 패키지의 `ffi::wit` 항목이 읽는 `.wit` 파일([stdlib 1.8](../../refs-standard-libraries/100-bare/108-wit.md))이 그 `deps/`와 함께 통째로 옵니다. 경로는 source dir 기준이고 `..`로 그 밖으로 나갈 수 있습니다. 다운스트림은 WIT를 디스크가 아니라 body에서 읽습니다.

본문 뒤에는 패키지 함수의 넘김 요약(hand-off summary, [stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync-스레드-경계-마커))이 옵니다. 요약은 탭으로 필드를 구분한 줄로 이루어집니다.

- `g <root> <mutex|rwlock|channel> <line> <function> <file>`: `<root>`를 넘김
- `s <root> <root>`: 하나의 객체
- `i <root> <root>`: 첫째가 사라지면 둘째를 함께 가져감
- `t <guard parameter> <root>`: 그 가드에 묶임
- `x <tag|*> [<root>]`: 반환한 값의 태그가 그 값인 출구에서 `<root>`가 넘긴 값 가운데 있음. 태그는 `bool`이면 0 / 1, 합이나 enum이면 변형, carrier면 에러 1, 값 0, 2 + 값의 태그이고, `*`는 알 수 없음을 뜻함. `x` 줄이 없으면 모든 출구가 모든 `g` 루트를 넘김

루트는 매개변수 인덱스나 `r`(반환값)이고, 그 뒤에 `.`으로 구분한 필드 경로가 옵니다. 튜플 인덱스는 31번 비트가 켜져 있습니다. 이름이 `#fn`(`#fne`는 클로저, 환경 먼저)이고 그 뒤에 U+001F로 구분한 매개변수 타입 키가 붙은 항목이 있습니다. 이 항목은 그 매개변수 타입으로 값처럼 쓰인 함수의 요약을 합칩니다.

## 9.4.8. 분리 컴파일을 위한 규칙

미리 빌드한 라이브러리는 프로그램 전체를 한 번에 보지 못합니다. 그래서 다음 규칙으로 패키지 사이를 맞춥니다.

- **`any` tag.** `any` 슬롯의 tag는 타입 이름의 31비트 해시입니다. 그래서 따로 빌드한 패키지가 같은 타입에 같은 tag를 씁니다.
- **`any` seed.** 라이브러리가 `any`에 넣는 타입을 프로그램이 알아야 합니다. 그래서 라이브러리는 그 타입을 쓰는 작은 Rok 함수를 본문 끝에 덧붙입니다. 프로그램의 닫힌 세계 `any` 디스패처가 이 타입도 포함합니다.
- **디스패처 심볼.** `any` 디스패처는 `rok.any.<hash>` 심볼입니다. 라이브러리 안에서는 weak로, 프로그램에서는 strong으로 정의합니다. 링크하면 프로그램의 정의가 이깁니다.
- **합 항 순서.** 합의 항은 내부 타입 번호가 아니라 구조적 키로 정렬합니다([2.6.4](../002-types/206-sum-types.md#정규-순서)). 그래서 패키지 사이에서 태그가 일치합니다.
- **GC type id.** 패키지마다 GC 타입 id 구간(9.4.6)이 따로 있습니다. 한 프로그램에서 두 패키지의 구간이 겹치면 에러입니다.
- **메서드 심볼.** 의존 패키지의 메서드는 안정된 경로 `<모듈>::<타입>.<메서드>[__by_<트레잇>]`으로 맹글링됩니다([9.5](905-symbol-mangling.md)).

## 9.4.9. 읽기와 거부

`--extern name=<name>.rokhir`이나 `--extern name=lib<name>.a`는 각각 옆에 있는 짝 파일을 찾습니다([9.2.3](902-aot-pipeline.md#923-의존성-해소)). 다운스트림은 본문을 소스 오버레이로 파싱해 선언을 얻습니다. 잘린 함수는 미리 빌드된 심볼로 선언하고, 제네릭은 그 자리에서 단형화합니다. 그런 다음 아카이브를 링크합니다.

다음이 하나라도 다르면 ROKHIR을 거부하고 "다시 빌드하라(rebuild)"는 에러를 냅니다.

- `version`
- producer (다른 `rokc`가 만든 ROKHIR)
- name (`--extern` 이름과 ROKHIR의 mount 이름)

## 9.4.10. ROKHIR 검사

`rokc rokhir-text <file>`은 ROKHIR의 버전, producer, 메타(`features:` 포함), 인터페이스 파일의 경로와 텍스트, 요약을 출력합니다([9.9.2](909-debug-tools.md#992-rokhir-검사)). `rokc inspect`([9.9.14](909-debug-tools.md#9914-패키지-검사))는 아카이브와 함께 요약을 보여 줍니다.

## 9.4.11. 안정성

ROKHIR은 같은 `rokc`가 만든 것만 읽습니다(9.4.9). 포맷이 바뀌면 `version`을 올립니다. 버전이 다르면 다시 빌드해야 합니다. [개요](../008-abi/README.md)의 안정성 정책을 적용합니다.

표준 라이브러리는 ROKHIR로 배포하지 않습니다([9.3.6](903-package-archive.md#936-표준-라이브러리)).
