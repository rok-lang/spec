# 0.2. import 정책: `use rok::core::**;` 묵시 적용

> 이 문서는 표준 라이브러리의 묵시 import 정책을 정의합니다. Rok 가상 네임스페이스 아래에서 자동으로 보이는 영역은 `rok::core`뿐입니다. `rok::bare`와 `rok::std`는 명시적으로 가져와야 합니다.

Rok은 모든 사용자 소스 파일 맨 앞에 다음 한 줄을 **묵시적으로** 붙입니다.

```rok
use rok::core::**;
```

`**`는 **재귀 glob(recursive glob)**입니다. `rok` 모듈 자신과 그 아래 모든 하위 모듈(`rok::core::ops`, `rok::core::iter`, `rok::core::collections`, `rok::core::sync`, `rok::core::task`, ...)의 pub 항목을 한꺼번에 가져옵니다.

> [!NOTE]
> Rok에는 **`rok::core::prelude` 모듈이 따로 없습니다**. Rust의 `std::prelude::v1::*`나 Haskell의 `Prelude` 같은 **재내보내기(re-export) 모듈** 관습은 따르지 않습니다. `**` 재귀 glob이 그 자리를 **직접** 대신합니다. 메커니즘이 단순해지고 간접 계층도 하나 사라집니다.

## 0.2.1. 핵심 설계 결정

| 모델                                     | 채택 언어      | Rok 채택  |
|------------------------------------------|----------------|-----------|
| 별도 `prelude` 모듈 + 자동 import        | Rust / Haskell | **거부**  |
| 명시 use 만                              | C / C++        | (verbose) |
| **`use rok::core::**;` 묵시 + 명시 보강** | Rok 자체       | **채택**  |

### 별도 prelude 모듈을 거부하는 이유

전통적인 prelude에는 단점이 여럿 있습니다. 먼저 **간접 계층(indirection)** 문제입니다. `rok::core::prelude::v1::List`처럼 **재내보내기 경로**가 실제 정의 경로와 따로 놉니다. 다음으로 `prelude::v1` / `prelude::v2`처럼 epoch 단위로 버전을 관리해야 합니다. Rust 2024에서 prelude에 항목을 추가한 일이 그 예입니다. 자기모순도 피할 수 없습니다. 표준 라이브러리 자신이 prelude를 가져오면 순환이 생기므로 `#![no_prelude]` 같은 탈출구가 따로 필요합니다. 무엇보다 `**`로 이미 재귀 glob을 쓸 수 있다면 prelude 모듈은 **뻔히 중복된(trivially redundant)** 층일 뿐입니다.

Rok의 해결책은 간단합니다. `**` 자체를 언어에 들였으므로 자동 적용도 `**`로, 사용자의 명시적 import도 `**`로 합니다. 둘은 같은 메커니즘 위에서 동작합니다.

## 0.2.2. `**` 재귀 glob

```rok
use foo::bar;            // 단일 항목
use foo::{a, b};         // 다중
use foo::*;              // 단일 레벨 glob (foo 의 직접 자식만)
use foo::**;             // recursive glob (foo + 하위 모듈 모두)
```

`**` 문법의 규범적 정의는 [refs-lang 5.5 use](../../refs-lang/005-modules/505-use.md)에 있습니다.

## 0.2.3. 묵시 적용: `use rok::core::**;`

```rok
// 사용자 코드
fn main() {
    let xs: List<i32> = [1, 2, 3];        // rok::core::collections::List
    let total: i32 = xs.iter().sum();      // rok::core::iter::Iterator
    println(f"{total}");                   // rok::core::print::println

    let m = Mutex::new(0);                 // rok::core::sync::Mutex
    task::spawn(async { /* ... */ });              // rok::core::task::spawn

    let n = Atomic::new(0);                // rok::core::sync::Atomic
}
```

위 코드는 use 문을 하나도 적지 않아도 동작합니다. 컴파일러가 `use rok::core::**;`를 **묵시적으로** 앞에 붙이기 때문입니다.

### 이름 충돌 정책

사용자가 정의한 항목의 이름이 `**`로 들어오는 이름과 같으면 그 모듈에서는 사용자 항목이 core 항목을 가립니다. 이름을 찾을 때 묵시 `use rok::core::**`보다 모듈 자신의 항목과 그 모듈의 `use`를 먼저 보기 때문입니다([refs-lang 5.7.1](../../refs-lang/005-modules/507-paths.md#571-경로의-시작-식별자)). core 항목에는 경로나 별칭으로 여전히 접근할 수 있습니다.

```rok
struct List<T> { ... }                  // OK: 여기서 rok::core::collections::List 를 가림
let xs: rok::core::collections::List<i32> = [1, 2];   // 경로로 가리킨 core 의 List
```

## 0.2.4. 묵시 적용되지 않는 계층

묵시적으로 적용하는 것은 `use rok::core::**;`뿐입니다. unsafe와 std는 **명시해야만** 쓸 수 있습니다.

```rok
// unsafe: 위험 시그널
use rok::bare::ptr::{Ptr, ConstPtr};
use rok::bare::ffi::{CStr, CString};

// std: target-dep 시그널
use rok::std::fs;
use rok::std::time::Instant;
```

이 정책은 [0.1 tier 모델](001-tier-model.md)의 다음 표와 맞물립니다.

| tier           | 묵시 import | 이유                    |
|----------------|-------------|-------------------------|
| `rok::core::**` | ✓           | 모든 target / 안전      |
| `rok::bare::**` | ✗           | 위험 시그널, 명시 강제 |
| `rok::std::**` | ✗           | target-dep, 명시 강제  |

이렇게 하면 코드가 어떤 영역을 쓰는지 파일 맨 위만 봐도 알 수 있습니다.

## 0.2.5. 적용 해제: `#![no_implicit_rok]`

특수한 상황에서는 묵시 `use rok::core::**;` 자체를 끌 수 있습니다.

```rok
#![no_implicit_rok]   // 파일 최상단

// 이 파일에서는 묵시 import 안 됨
use rok::core::ops::Add;
let r = a + b;   // OK (Add 명시)

let m = Mutex::new(0);   // ERROR: Mutex 미 import
```

리터럴은 컴파일러가 정의하므로 계속 쓸 수 있습니다. 잃는 것은 이름과, 가져오지 않은 트레잇의 연산자뿐입니다([refs-lang 6.7.8](../../refs-lang/006-meta-system/607-operator-magic.md#678-매핑의-가시성)).

이 속성을 쓰는 곳은 다음과 같습니다. 표준 라이브러리는 자기 자신을 가져오면 순환이 생기므로 이 속성이 필요합니다. 드물지만 임베디드 같은 `no_std`나 `no_rok` 환경도 마찬가지입니다. 학습용이나 데모 코드에서 각 항목이 어디서 오는지 한눈에 보여 줄 때도 쓸모가 있습니다.

## 0.2.6. `**`의 가시성 규칙

`**`는 **pub 항목만** 드러냅니다. 내부 구현(`pub(pkg)` / private)은 외부 사용자에게 새어 나가지 않습니다.

```rok
mod sync {                          // rok::core::sync
    pub struct Mutex<T> { ... }
    pub fn lock<T>(ref m: Mutex<T>) -> MutexGuard<T> { ... }

    pub(pkg) struct Internal { ... }    // pkg 내부만
    fn helper() { ... }                  // private
}

// 사용자 코드 (묵시 use rok::core::**; 후)
let m: Mutex<i32> = Mutex::new(0);       // OK
rok::core::sync::Internal::new();              // ERROR: pub(pkg)
rok::core::sync::helper();                     // ERROR: private
```

이름을 평탄화하지 않고 모듈 경로를 그대로 보존합니다.

```rok
// 묵시 use rok::core::**; 후
let m = Mutex::new(0);              // OK: Mutex 직접 노출 (pub at rok::core::sync 라 reachable)
let s = task::spawn(async { /* */ });      // OK: task 모듈 자체도 reachable, spawn 은 task::spawn

// 둘 다 같은 의미.
rok::core::sync::Mutex::new(0);
Mutex::new(0);
```

## 0.2.7. 다른 언어와 비교

| 언어    | 모델                                        | indirection                        |
|---------|---------------------------------------------|------------------------------------|
| C       | 모든 #include 명시                          | 0 (직접)                           |
| Rust    | `std::prelude::v1::*` re-export 모듈 + 자동 | 1 (prelude module)                 |
| Haskell | `Prelude` re-export 모듈 + 자동             | 1                                  |
| Python  | built-ins (언어 magic)                      | 0 (langauge-level)                 |
| **Rok** | `use rok::core::**;` 묵시                    | **0** (직접: `**`가 곧 메커니즘)  |

Rok이 다른 언어와 가장 크게 다른 점은 prelude 모듈이라는 간접 계층이 없다는 것입니다. 재귀 glob `**`가 **언어 차원**의 기능이므로 사용자와 표준 라이브러리가 같은 메커니즘을 씁니다.

이 결정 하나로 '명시가 묵시보다 낫다(explicit > implicit)'와 '간접 계층을 더 두지 않는다(no extra indirection)'는 두 원칙을 모두 만족합니다.
