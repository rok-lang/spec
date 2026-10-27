# 0.1. 3-tier 모델

Rok 표준 라이브러리는 세 계층 (tier) 으로 나뉩니다. 각 tier 마다 의존할 수 있는
영역이 다르고, 사용자 코드에서의 import 규칙 역시 다릅니다.

```
rok::bare   (base, runtime extern "C" 만 의존)
  ↓
rok::core     (foundation, 안전, 모든 target; uses bare internally)
  ↓
rok::std   (target-dep; uses foundation + bare)
```

세 계층 모두 가상 네임스페이스 `rok::` 아래에 자리 잡습니다 ([refs-lang 5.7.4](../../refs-lang/005-modules/507-paths.md#574-rok-표준-라이브러리-가상-네임스페이스)).

이렇게 세 층으로 나누는 이유는 분명합니다. 메모리 안전성을 깨뜨릴 수 있는 원시
연산은 좁은 영역에 가두어야 감사하기 쉽고, 모든 플랫폼에서 동일하게 동작해야 하는
핵심 추상은 OS 의존 코드와 섞이지 않을 때 비로소 이식성을 보장할 수 있기 때문입니다.
플랫폼별로 가용성이 갈리는 기능은 그 사실을 시그니처와 import 지점에서 드러내야
사용자가 wasm 빌드와 네이티브 빌드를 일관되게 다룰 수 있습니다. 세 tier 는 바로 이
세 가지 요구를 각각 한 층씩 책임집니다.

## 0.1.1. tier 별 책임

### `rok::bare`

가장 아래의 `rok::bare` tier 는 runtime 의 `extern "rok"` 심볼만을 의존 대상으로 삼습니다.
여기서 `Ptr<T>`, `ConstPtr<T>`, `transmute`, raw allocation, pin/unpin, volatile
같은 원시 도구들을 노출하므로, 사용자가 끌어 쓸 때는 반드시 명시적인 `use` 가
필요합니다. 명시는 곧 위험 시그널이며, 코드 리뷰 시점에 위험 영역이 한눈에 보이도록
의도된 장치입니다.

```rok
use rok::bare::ptr::CPtr;
use rok::bare::alloc::{alloc, Layout};
let layout = try Layout::array::<u8>(1024);
let p: CPtr<u8> = unsafe { alloc(layout) };    // 실패 시 null (1.4 alloc 참고)
```

### `rok::core` (foundation)

가운데의 foundation tier 인 `rok::core` 에는 `rok::core::collections`, `rok::core::iter`, `rok::core::ops` 같은 모듈이 자리 잡으며 내부적으로만 `rok::bare` 에 의존합니다. 숫자 타입, 산술 트레잇,
iter, 컬렉션, fmt, 변환, sync, task, future 등 언어를 일상적으로 쓰는 데
필요한 거의 모든 추상이 이 층에 모여 있습니다. 사용자 코드에서는 **`use rok::core::**;`
가 묵시적으로 자동 적용** 되므로 (자세한 정책은 [0.2 import 정책](002-import.md) 참고)
별도의 import 없이 바로 사용할 수 있습니다.

```rok
let xs: List<i32> = [1, 2, 3];        // List 직접 사용, 묵시 use rok::core::**; 가 가져옴
let r = a + b;                         // Add 트레잇 자동
let m = Mutex::new(0);                 // sync::Mutex 도 자동
task::spawn(async { /* ... */ });              // task 모듈도 자동
```

이 자동 적용을 위해 별도의 `rok::core::prelude` 모듈을 두지 않는다는 점은 짚어둘 만합니다.
`**` recursive glob 자체가 prelude 의 역할을 그대로 수행하므로 추가적인 indirection 이
들어설 자리가 없습니다.

### `rok::std`

가장 위의 `rok::std` tier 는 필요할 때 foundation 과 `rok::bare` 를 모두 사용합니다. 이 층에는
`fs`, `net`, `time`, `thread` (OS 스레드), `io`, `env`, `process`, `path` 처럼
플랫폼에 의존적인 항목들이 자리합니다. target 별로 가용성이 달라지므로, 사용 측은
어떤 플랫폼 기능에 발을 들이고 있는지 명확히 드러내기 위해 항상 명시적으로 import
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

반대로 다음 방향의 의존은 **금지** 입니다. 우선 foundation (`rok::core`) 이 `rok::std` 를 import 하는 일은
허용되지 않는데, foundation 은 모든 target 에서 빌드되어야 하므로 플랫폼 의존 코드를
끌어들이는 순간 이식성이 무너집니다. `rok::bare` 가 foundation 또는 `rok::std` 를 import 하는
것도 막혀 있습니다. 저수준 층이 고수준 층에 의존하기 시작하면 계층 자체가 의미를
잃기 때문입니다. 끝으로 개별 `rok::std::*` 모듈끼리의 순환 의존도 차단되어, 각 std
모듈은 서로 독립적으로 빌드 가능한 단위로 유지됩니다.

이 규칙이 다스리는 것은 tier 가 적는 것, 즉 그 `use` 와 경로입니다. 묵시 `use rok::core::**` 는 여전히 모든 tier 에 bare 자신의 시그니처가 쓰는 언어의 어휘 (0.4 의 slot 항목, `List`, `Slice`, `Result`, `panic`) 를 공급합니다 (1.6, 1.7).

## 0.1.3. 빌드 단위

세 tier 는 각자 별도의 빌드 단위 (별도 archive) 로 처리되어 컴파일 시점별로 캐시할 수 있습니다.
링크 순서는 `rok::std` → `rok::core` → `rok::bare` → runtime 으로 흐르고, 그 위에 사용자
패키지가 가장 위에 얹힙니다. 컴파일러가 `-NL <tier>` 메커니즘으로 자동 링크하므로, 사용자가
의존성 선언에 추가할 필요는 없습니다 ([0.5.4 제공 형태와 링크 메커니즘](../../refs-lang/000-meta/005-standard-libraries-tiers.md#054-제공-형태와-링크-메커니즘) 참고).

## 0.1.4. target 가용성

`rok::bare` 와 foundation (`rok::core`) 은 모든 [Tier 1 플랫폼](../../refs-lang/000-meta/006-platform-targets.md)
에서 빌드되며, 여기에는 wasm32 도 포함됩니다. 반면 `rok::std` 는 target 에 따라 가용성이
갈립니다. 가령 `rok::std::thread` 는 wasm32 에서 지원되지 않는데, 브라우저 환경이 main
thread only 로 동작하기 때문입니다. `rok::std::fs` 역시 wasm32-unknown-unknown 에서는
사용할 수 없고 WASI 타깃 (`wasm32-wasi`, `wasm32-wasi-threads`) 에서만 제공됩니다. 이러한 가용성은 각 `rok::std::*` 모듈 spec 의 가용성 절에 명시됩니다.
API 는 모든 타깃에서 빌드됩니다: 타깃에 없는 연산은 런타임에 `IoErrorKind::Unsupported` 로
실패합니다. 브라우저에서 stdout / stderr 는 host 콘솔로 가고 stdin 은 EOF 입니다.

## 0.1.5. 진화 정책

세 tier 는 안정성 정책 면에서도 결이 다릅니다. `rok::bare` API 는 stable 하지 않으며
컴파일러 및 런타임과 함께 진화하므로, 사용자 코드는 버전을 pin 하는 편이
권장됩니다. foundation (`rok::core`) API 는 SemVer 안정성을 목표로 하고, 직접적인 변경은 major 버전
bump 를 동반합니다. `rok::std` API 는 target 별 가용성 차이만큼 변동성이 크지만, 시그니처
자체는 stable 하게 유지됩니다.
