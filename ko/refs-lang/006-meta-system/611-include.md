# 6.11. 컴파일 시점 파일 포함

## 6.11.1. `include_str!` / `include_bytes!`

```rok
const TEMPLATE: str = include_str!("templates/page.html");
static ICON: [u8; 1406] = include_bytes!("res/icon.png");
```

`include_str!("path")` 는 파일 `path` 의 내용을 `str` 로, `include_bytes!("path")` 는 그 바이트를 `[u8; N]` 으로 평가합니다 (`N` 은 파일의 길이이며, 그 `Slice<u8>` 규칙을 포함해 바이트 문자열 [1.7.5](../001-lexical/107-string-literals.md#175-바이트-문자열) 과 같음). 인자는 정확히 문자열 리터럴 하나입니다. 경로는 호출을 담은 소스 파일의 디렉토리 기준입니다. 파일은 컴파일 시점에 읽으며, 결과는 상수이므로 `const` 와 `static` 의 초기화식을 포함해 리터럴이 올 수 있는 곳이면 어디든 올 수 있습니다.

파일을 읽을 수 없거나, `include_str!` 의 경우 내용이 올바른 UTF-8 이 아니면 컴파일 에러입니다. 포함한 파일은 빌드의 입력입니다: 바꾸면 그것을 쓰는 곳이 다시 빌드됩니다 ([9.2.16](../009-tools/902-aot-pipeline.md#9216-증분-빌드)).

`cfg!` ([6.3.5](603-cfg.md#635-cfg-컴파일-타임-bool-값)) 처럼 이것들은 사용자 정의 매크로가 아니라 컴파일러 내장 형태입니다.
