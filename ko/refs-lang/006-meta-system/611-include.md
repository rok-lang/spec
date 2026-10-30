# 6.11. 컴파일 시점 파일 포함

## 6.11.1. `include_str!` / `include_bytes!`

```rok
const TEMPLATE: str = include_str!("templates/page.html");
static ICON: [u8; 1406] = include_bytes!("res/icon.png");
```

`include_str!("path")`는 파일 `path`의 내용을 `str` 값으로 평가합니다. `include_bytes!("path")`는 그 바이트를 `[u8; N]` 값으로 평가합니다. `N`은 파일 길이입니다. 결과는 바이트 문자열([1.7.5](../001-lexical/107-string-literals.md#175-바이트-문자열))과 같고 `Slice<u8>` 규칙도 따릅니다. 인자는 정확히 문자열 리터럴 하나입니다. 경로는 호출이 들어 있는 소스 파일의 디렉터리를 기준으로 합니다. 컴파일러는 파일을 컴파일 시점에 읽습니다. 결과는 상수이므로 리터럴이 올 수 있는 곳이면 어디에나 올 수 있습니다. `const`와 `static`의 초기화식도 그런 곳입니다.

파일을 읽을 수 없으면 컴파일 에러입니다. `include_str!`에서 내용이 올바른 UTF-8이 아니어도 컴파일 에러입니다. 포함한 파일은 빌드의 입력입니다. 파일을 바꾸면 그 파일을 쓰는 곳을 다시 빌드합니다([9.2.16](../009-tools/902-aot-pipeline.md#9216-증분-빌드)).

`cfg!`([6.3.5](603-cfg.md#635-cfg-컴파일-타임-bool-값))와 마찬가지로, 이 둘은 사용자 정의 매크로가 아니라 컴파일러 내장 형태입니다.
