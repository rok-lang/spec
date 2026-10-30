# 0.1. 3-tier 모델

Rok 표준 라이브러리는 세 계층(tier)으로 나뉩니다. tier마다 의존할 수 있는
영역이 다르고, 사용자 코드의 import 규칙도 다릅니다.

```
rok::bare   (base, runtime extern "C" 만 의존)
  ↓
rok::core     (foundation, 안전, 모든 target; uses bare internally)
  ↓
rok::std   (target-dep; uses foundation + bare)
```

세 계층은 모두 가상 네임스페이스 `rok::` 아래에 있습니다([refs-lang 5.7.4](../../refs-lang/005-modules/507-paths.md#574-rok-표준-라이브러리-가상-네임스페이스)).

세 층으로 나누는 이유는 분명합니다. 메모리 안전성을 깨뜨릴 수 있는 원시
연산은 좁은 영역에 가두어야 감사하기 쉽습니다. 모든 플랫폼에서 똑같이 동작해야 하는
핵심 추상은 OS 의존 코드와 섞이지 않아야 이식성을 보장할 수 있습니다.
플랫폼마다 가용성이 다른 기능은 그 사실을 시그니처와 import 지점에서 드러내야
사용자가 wasm 빌드와 네이티브 빌드를 일관되게 다룰 수 있습니다. 세 tier는 이
세 요구를 하나씩 맡습니다.

## 0.1.1. tier 별 책임

### `rok::bare`

맨 아래의 `rok::bare` tier는 runtime의 `extern "rok"` 심볼에만 의존합니다.
이 tier는 `Ptr<T>`, `ConstPtr<T>`, `transmute`, raw allocation, pin/unpin, volatile
같은 원시 도구를 노출하므로, 사용하려면 항상 명시적인 `use`가
필요합니다. 명시 자체가 위험 신호입니다. 코드 리뷰에서 위험 영역이 한눈에 보이도록
일부러 둔 장치입니다.

```rok
use rok::bare::ptr::CPtr;
use rok::bare::alloc::{alloc, Layout};
let layout = try Layout::array::<u8>(1024);
let p: CPtr<u8> = unsafe { alloc(layout) };    // 실패 시 null (1.4 alloc 참고)
```

### `rok::core` (foundation)

가운데의 foundation tier인 `rok::core`에는 `rok::core::collections`, `rok::core::iter`, `rok::core::ops` 같은 모듈이 있으며, 내부적으로만 `rok::bare`에 의존합니다. 숫자 타입, 산술 트레잇,
iter, 컬렉션, fmt, 변환, sync, task, future 등 언어를 일상적으로 쓰는 데
필요한 추상은 거의 모두 이 층에 모여 있습니다. 사용자 코드에는 **`use rok::core::**;`
가 묵시적으로 자동 적용**되므로(자세한 정책은 [0.2 import 정책](002-import.md) 참고)
별도 import 없이 바로 쓸 수 있습니다.

```rok
let xs: List<i32> = [1, 2, 3];        // List 직접 사용, 묵시 use rok::core::**; 가 가져옴
let r = a + b;                         // Add 트레잇 자동
let m = Mutex::new(0);                 // sync::Mutex 도 자동
task::spawn(async { /* ... */ });              // task 모듈도 자동
```

이 자동 적용을 위한 별도의 `rok::core::prelude` 모듈은 없습니다.
`**` recursive glob 자체가 prelude 역할을 하므로 간접 계층(indirection)을 더
둘 자리가 없습니다.

### `rok::std`

맨 위의 `rok::std` tier는 필요에 따라 foundation과 `rok::bare`를 모두 사용합니다. 이 층에는
`fs`, `net`, `time`, `thread`(OS 스레드), `io`, `env`, `process`, `path`처럼
플랫폼에 의존하는 항목이 있습니다. target마다 가용성이 다르므로, 사용하는 쪽은
어떤 플랫폼 기능을 쓰는지 분명히 드러내도록 항상 명시적으로 import
해야 합니다.

```rok
use rok::std::fs;
use rok::std::time::Instant;
```

## 0.1.2. 의존성 규칙 (강제)

다음과 같은 의존성이 허용됩니다.

| 의존하는 측 | 의존 가능 대상                                          |
|-------------|---------------------------------------------------------|
| `rok::bare`  | runtime extern "C"                                      |
| `rok::core`  | `rok::bare`                                              |
| `rok::std`  | `rok::core`, `rok::bare`                                  |
| 사용자 코드 | `rok::core` (묵시), `rok::std` (명시), `rok::bare` (명시) |

다음 방향의 의존은 **금지**입니다. foundation(`rok::core`)은 `rok::std`를 import해서는
안 됩니다. foundation은 모든 target에서 빌드되어야 하므로, 플랫폼 의존 코드를
끌어들이는 순간 이식성이 무너집니다. `rok::bare`도 foundation이나 `rok::std`를 import해서는
안 됩니다. 저수준 층이 고수준 층에 의존하기 시작하면 계층 구조 자체가 의미를
잃습니다. 마지막으로 개별 `rok::std::*` 모듈 사이의 순환 의존도 금지됩니다. 그래서 각 std
모듈은 독립적으로 빌드할 수 있는 단위로 유지됩니다.

이 규칙이 다스리는 대상은 tier가 직접 적는 것, 곧 그 tier의 `use`와 경로입니다. 묵시 `use rok::core::**`는 여전히 모든 tier에 bare 자신의 시그니처가 쓰는 언어 어휘(0.4의 slot 항목, `List`, `Slice`, `Result`, `panic`)를 공급합니다(1.6, 1.7).

## 0.1.3. 빌드 단위

세 tier는 각각 별도 빌드 단위(별도 archive)로 처리되며, 컴파일 단위로 캐시할 수 있습니다.
링크 순서는 `rok::std` → `rok::core` → `rok::bare` → runtime이고, 사용자
패키지가 그 맨 위에 놓입니다. 컴파일러가 `-NL <tier>` 메커니즘으로 자동 링크하므로, 사용자가
의존성 선언에 추가할 필요는 없습니다([0.5.4 제공 형태와 링크 메커니즘](../../refs-lang/000-meta/005-standard-libraries-tiers.md#054-제공-형태와-링크-메커니즘) 참고).

## 0.1.4. target 가용성

`rok::bare`와 foundation(`rok::core`)은 wasm32를 포함한 모든 [Tier 1 플랫폼](../../refs-lang/000-meta/006-platform-targets.md)
에서 빌드됩니다. 반면 `rok::std`는 target에 따라 가용성이
다릅니다. 예를 들어 `rok::std::thread`는 wasm32에서 지원되지 않습니다. 브라우저 환경은 main
thread 하나로만 동작하기 때문입니다. `rok::std::fs`도 wasm32-unknown-unknown에서는
쓸 수 없고 WASI 타깃(`wasm32-wasi`, `wasm32-wasi-threads`)에서만 제공됩니다. 가용성은 각 `rok::std::*` 모듈 spec의 가용성 절에 명시합니다.
그래도 API 자체는 모든 타깃에서 빌드됩니다. 타깃에 없는 연산은 런타임에 `IoErrorKind::Unsupported`로
실패합니다. 브라우저에서 stdout / stderr는 host 콘솔로 가고, stdin은 EOF 상태입니다.

## 0.1.5. 진화 정책

세 tier는 안정성 정책도 다릅니다. `rok::bare` API는 안정적이지 않고
컴파일러, 런타임과 함께 바뀌므로, 사용자 코드는 버전을 고정(pin)하기를
권장합니다. foundation(`rok::core`) API는 SemVer 안정성을 목표로 하며, 직접적인 변경에는 major 버전
bump가 따릅니다. `rok::std` API는 target별 가용성 차이만큼 변동이 크지만, 시그니처
자체는 안정적으로 유지됩니다.
