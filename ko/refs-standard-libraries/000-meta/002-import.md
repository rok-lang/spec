# 0.2. import 정책 — `use rok::core::**;` 묵시 적용

> 본 문서는 표준 라이브러리 묵시 import 정책을 정의합니다. rok 가상 네임스페이스 아래 `rok::core` 영역만 자동 가시화되며, `rok::bare` 와 `rok::std` 는 명시 import 가 필요합니다.

rok 은 모든 사용자 소스 파일 앞에 **묵시적으로** 다음 한 줄을 prepend 합니다.

```rok
use rok::core::**;
```

여기서 `**` 는 **recursive glob** 으로, `rok` 모듈 자신은 물론 그 아래 모든 하위 모듈 (`rok::core::ops`, `rok::core::iter`, `rok::core::collections`, `rok::core::sync`, `rok::core::task`, ...) 의 pub 항목까지 한꺼번에 끌어옵니다.

> [!NOTE]
> rok 은 **별도 `rok::core::prelude` 모듈을 두지 않습니다**. Rust `std::prelude::v1::*` 나 Haskell `Prelude` 식 **re-export 모듈** 관습은 거부했고, 그 자리를 `**` recursive glob 이 **직접** 대신합니다. 메커니즘이 단순해지는 것은 물론 추가 indirection 도 사라집니다.

## 0.2.1. 핵심 디자인 결정

| 모델                                     | 채택 언어      | rok 채택  |
|------------------------------------------|----------------|-----------|
| 별도 `prelude` 모듈 + 자동 import        | Rust / Haskell | **거부**  |
| 명시 use 만                              | C / C++        | (verbose) |
| **`use rok::core::**;` 묵시 + 명시 보강** | rok 자체       | **채택**  |

### 왜 별도 prelude 모듈을 거부하는가

전통적 prelude 가 안고 있는 단점은 한둘이 아닙니다. 우선 **indirection** 문제가 있는데, `rok::core::prelude::v1::List` 처럼 **re-export 경로** 가 실제 정의 경로와 분리되어 버립니다. 거기에 `prelude::v1` / `prelude::v2` 같은 epoch 단위로 버전을 관리해야 하는 부담도 따라옵니다 (Rust 2024 prelude 추가가 그 학습 사례입니다). 자기 모순도 피할 수 없습니다. 표준 라이브러리 자체가 prelude 를 import 하면 cycle 이 발생해 `#![no_prelude]` 같은 별도 escape 를 마련해야 합니다. 무엇보다 `**` 가 이미 재귀 glob 을 제공한다면 prelude 모듈은 **trivially redundant** 한 layer 에 지나지 않습니다.

rok 의 해결책은 깔끔합니다. `**` 자체를 언어 차원에 끌어들였기에 자동 적용도 `**` 로, 사용자 명시도 `**` 로 같은 메커니즘 위에서 굴러갑니다.

## 0.2.2. `**` recursive glob

```rok
use foo::bar;            // 단일 항목
use foo::{a, b};         // 다중
use foo::*;              // 단일 레벨 glob (foo 의 직접 자식만)
use foo::**;             // recursive glob (foo + 하위 모듈 모두)
```

`**` syntax 의 본문 정의는 [refs-lang 5.5 use](../../refs-lang/005-modules/505-use.md) 가 담당합니다.

## 0.2.3. 묵시 적용 — `use rok::core::**;`

```rok
// 사용자 코드
fn main() {
    let xs: List<i32> = [1, 2, 3];        // rok::core::collections::List
    let total: i32 = xs.iter().sum();      // rok::core::iter::Iterator
    println(f"{total}");                   // rok::core::print::println

    let m = Mutex::new(0);                 // rok::core::sync::Mutex
    task::spawn(async { /* ... */ });              // rok::core::task::spawn

    let h = handle::new();                 // rok::core::sync::Handle
}
```

위 코드는 어떤 use 문도 적지 않은 채로 작동합니다. 컴파일러가 **묵시적으로** `use rok::core::**;` 를 prepend 해 주기 때문입니다.

### 이름 충돌 정책

`**` 도 끌어오는 이름의 사용자 정의 항목은 그 모듈에서 그것을 가립니다: 이름 찾기는 묵시 `use rok::core::**` 보다 모듈 자신의 항목과 그 `use` 를 먼저 봅니다 ([refs-lang 5.7.1](../../refs-lang/005-modules/507-paths.md#571-경로의-시작-식별자)). core 항목은 경로나 별칭으로 계속 닿을 수 있습니다.

```rok
struct List<T> { ... }                  // OK — 여기서 rok::core::collections::List 를 가림
let xs: rok::core::collections::List<i32> = [1, 2];   // 경로로 가리킨 core 의 List
```

## 0.2.4. 묵시 적용 **안 되는** tier

묵시 처리되는 것은 `use rok::core::**;` 뿐입니다. unsafe 와 std 는 **명시 강제** 입니다.

```rok
// unsafe — 위험 시그널
use rok::bare::{Ptr, ConstPtr};
use rok::bare::ffi::{CStr, CString};

// std — target-dep 시그널
use rok::std::fs;
use rok::std::time::Instant;
```

이 정책은 [0.1 tier 모델](001-tier-model.md) 의 다음 표와 그대로 맞물립니다.

| tier           | 묵시 import | 이유                    |
|----------------|-------------|-------------------------|
| `rok::core::**` | ✓           | 모든 target / 안전      |
| `rok::bare::**` | ✗           | 위험 시그널 — 명시 강제 |
| `rok::std::**` | ✗           | target-dep — 명시 강제  |

이렇게 해 두면 사용자가 어떤 영역으로 진입하고 있는지 파일 상단만 봐도 즉시 드러납니다.

## 0.2.5. 묵시 무시 — `#![no_implicit_rok]`

특수한 상황에서는 묵시 `use rok::core::**;` 자체를 꺼 버릴 수도 있습니다.

```rok
#![no_implicit_rok]   // 파일 최상단

// 이 파일에서는 묵시 import 안 됨
use rok::core::ops::Add;
let r = a + b;   // OK (Add 명시)

let m = Mutex::new(0);   // ERROR — Mutex 미 import
```

리터럴은 컴파일러가 정의하므로 계속 쓸 수 있습니다. 잃는 것은 이름 (과 import 하지 않은 트레잇의 연산자, [refs-lang 6.7.8](../../refs-lang/006-meta-system/607-operator-magic.md#678-매핑의-가시성)) 뿐입니다.

쓸 일이 생기는 곳은 다음과 같습니다. 표준 라이브러리 자체가 자기 자신을 import 하면 cycle 이 발생하므로 이 attribute 가 필요하고, `no_std` 나 `no_rok` 환경 (드물지만 임베디드) 에서도 마찬가지입니다. 학습용이나 데모 코드에서 어떤 항목이 어디서 오는지 한눈에 보여 주고 싶을 때도 유용한 도구입니다.

## 0.2.6. `**` 의 가시성 규칙

`**` 는 **pub 항목만** 노출합니다. 내부 구현 (`pub(pkg)` / private) 은 외부 사용자에게 절대로 새어 나가지 않습니다.

```rok
mod sync {                          // rok::core::sync
    pub struct Mutex<T> { ... }
    pub fn lock<T>(ref m: Mutex<T>) -> MutexGuard<T> { ... }

    pub(pkg) struct Internal { ... }    // pkg 내부만
    fn helper() { ... }                  // private
}

// 사용자 코드 — 묵시 use rok::core::**; 후
let m: Mutex<i32> = Mutex::new(0);       // OK
rok::core::sync::Internal::new();              // ERROR — pub(pkg)
rok::core::sync::helper();                     // ERROR — private
```

또한 이름을 평탄화하지 않고 모듈 경로를 그대로 보존합니다.

```rok
// 묵시 use rok::core::**; 후
let m = Mutex::new(0);              // OK — Mutex 직접 노출 (pub at rok::core::sync 라 reachable)
let s = task::spawn(async { /* */ });      // OK — task 모듈 자체도 reachable, spawn 은 task::spawn

// 둘 다 같은 의미.
rok::core::sync::Mutex::new(0);
Mutex::new(0);
```

## 0.2.7. 다른 언어와의 비교

| 언어    | 모델                                        | indirection                        |
|---------|---------------------------------------------|------------------------------------|
| C       | 모든 #include 명시                          | 0 (직접)                           |
| Rust    | `std::prelude::v1::*` re-export 모듈 + 자동 | 1 (prelude module)                 |
| Haskell | `Prelude` re-export 모듈 + 자동             | 1                                  |
| Python  | built-ins (언어 magic)                      | 0 (langauge-level)                 |
| **rok** | `use rok::core::**;` 묵시                    | **0** (직접 — `**` 가 곧 메커니즘) |

rok 이 다른 언어와 갈라지는 가장 큰 지점은 prelude 모듈 indirection 이 없다는 사실입니다. `**` 가 **언어 차원** 에서 recursive glob 을 제공하기에 사용자도 표준 라이브러리도 같은 메커니즘을 공유합니다.

이 결정 하나로 explicit > implicit 와 no extra indirection 두 원칙을 모두 만족시킬 수 있습니다.
