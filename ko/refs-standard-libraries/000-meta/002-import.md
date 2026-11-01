# 0.2. import 정책: prelude `rok::core::prelude`

> 이 문서는 표준 라이브러리의 묵시 import 정책을 정의합니다. 모든 사용자 파일은 `use` 없이 모듈 하나, `rok::core::prelude`의 이름을 봅니다. 그 밖의 `rok::core` 항목과 `rok::bare`, `rok::std`의 모든 항목은 경로로 부르거나 명시적으로 가져옵니다.

Rok은 모든 사용자 소스 파일 맨 앞에 다음 한 줄을 **묵시적으로** 붙입니다.

```rok
use rok::core::prelude::*;
```

`rok::core::prelude`는 `pub use` 재내보내기로 이루어진 평범한 모듈입니다([refs-lang 5.8](../../refs-lang/005-modules/508-re-export.md)). 목록은 아래(0.2.3)에 정해 두며 규범입니다. 이 문서에 적힌 이름만 prelude에 들어 있습니다.

prelude에는 enum 변형이 없습니다. 맨 변형 이름은 그 자리가 기대하는 타입으로 정해집니다([refs-lang 2.4.8](../../refs-lang/002-types/204-enum.md#248-기대-타입으로-정하는-맨-변형)). `let o: Ordering = Less;`, `x.load(SeqCst)`, `return Ok(v);`처럼 씁니다.

## 0.2.1. 핵심 설계 결정

| 모델                                 | 채택 언어              | Rok 채택  |
|--------------------------------------|------------------------|-----------|
| 명시 use만                           | C / C++                | (verbose) |
| core 라이브러리의 모든 공개 이름     | Swift, Python builtins | 거부      |
| **짧게 고른 prelude 모듈**           | Rust, Gleam, Elm       | **채택**  |

### 목록을 고르는 이유

`rok::core` 전체에 거는 묵시 glob(Rok 0.1)은 core의 모든 공개 항목을 모든 파일의 맨 이름으로 만들었습니다. 타입 160여 개, 트레잇 100개, 함수 90개에 enum 변형까지 전부입니다. 그중 여럿이 서로 겹쳤고(`iter`와 `stream`의 `Map`, `Filter`, `Zip`, `task`와 `par`의 `join`, `Ordering`과 `fmt::Alignment`의 변형 `Equal`), core에 항목이 하나 늘 때마다 사용자 코드의 이름 해석이 바뀌었습니다.

prelude에는 거의 모든 파일에서 쓰는 것만 둡니다. 컬렉션과 핵심 타입, `impl` 머리와 bound에 적는 트레잇, 출력과 패닉 함수, 그리고 core의 최상위 모듈입니다. 그래서 `task::sleep`과 `par::map`은 계속 짧게 씁니다. prelude의 이름은 이름공간마다 하나뿐이고(라이브러리 테스트가 모든 target에서 확인), 따라서 맨 prelude 이름은 모호해지지 않습니다. 묵시 import는 우선순위가 가장 낮으므로(0.2.4) prelude에 이름을 더해도 같은 이름을 정의하거나 가져오는 프로그램은 깨지지 않습니다.

## 0.2.2. `**` 재귀 glob

```rok
use foo::bar;            // 단일 항목
use foo::{a, b};         // 다중
use foo::*;              // 단일 레벨 glob (foo 의 직접 자식만)
use foo::**;             // recursive glob (foo + 하위 모듈 모두)
```

`**` 문법의 규범적 정의는 [refs-lang 5.5 use](../../refs-lang/005-modules/505-use.md)에 있습니다.

## 0.2.3. prelude 목록

아래 이름이 `rok::core::prelude`의 이름 전부입니다. Rok 어디서나 그렇듯 모듈, 타입, 트레잇이 이름공간 하나를 함께 쓰고 함수는 따로 씁니다. 그래서 `channel`, `panic`, `print`는 각각 모듈이면서 함수입니다.

**모듈 (27).** `rok::core`의 최상위 모듈입니다.

`array` `bench` `channel` `cmp` `collections` `convert` `crypto` `encoding` `error` `fmt` `future` `hint` `index` `iter` `marker` `num` `ops` `panic` `par` `print` `range` `reflect` `str` `stream` `sync` `task` `tuple`

**타입 (32).**

| 분류 | 이름 |
|---|---|
| 컬렉션 | `List` `Dict` `Set` `Deque` `BinaryHeap` `Slice` `StrBuf` |
| 수와 범위 | `Range` `RangeInclusive` `Int` `Float` |
| 일반 | `Result` `Ordering` `Duration` `Instant` |
| 동기화와 채널 | `Mutex` `RwLock` `Atomic` `Cell` `Sender` `Receiver` `Task` |
| 포매팅 | `Formatter` `FmtError` |
| core 에러 | `ParseError` `ValueError` `OverflowError` `ConversionError` `TypeError` `StackOverflowError` `OutOfMemoryError` `InternalError` |

**트레잇 (26).**

| 분류 | 이름 |
|---|---|
| 마커 | `Clone` `Copy` `Drop` `Default` `Send` `Sync` |
| 비교 | `PartialEq` `Eq` `PartialOrd` `Ord` `Hash` |
| 포매팅과 에러 | `Debug` `Display` `Error` |
| 반복과 비동기 | `Iterator` `IntoIterator` `FromIterator` `TryIterator` `Future` `IntoFuture` |
| 변환 | `From` `Into` `TryFrom` `TryInto` `FromStr` |
| 리플렉션 | `Inspect` (derive 포함, [refs-lang 6.8.11](../../refs-lang/006-meta-system/608-meta-blocks.md)) |

연산자 트레잇은 prelude에 없습니다. `ops::Add`로 쓰거나 `use rok::core::ops::Add;`로 가져옵니다. 연산자 자체는 이름이 필요 없으므로 어디서나 동작합니다([refs-lang 6.7.8](../../refs-lang/006-meta-system/607-operator-magic.md#678-매핑의-가시성)). 모든 `rok::core` 트레잇의 메서드도 마찬가지입니다([refs-lang 3.5](../../refs-lang/003-expressions/305-member-access-and-method-call.md)).

**함수 (11).**

`print` `println` `eprint` `eprintln` `panic` `assert` `assert_eq` `assert_ne` `unreachable` `channel` `channel_bounded`

**변형.** 없습니다. [refs-lang 2.4.8](../../refs-lang/002-types/204-enum.md#248-기대-타입으로-정하는-맨-변형)을 봅니다.

```rok
// 사용자 코드
fn main() {
    let xs: List<i32> = [1, 2, 3];        // prelude: List
    let total: i32 = xs.iter().sum();      // core 트레잇의 메서드: import 불필요
    println(f"{total}");                   // prelude: println

    let m = Mutex::new(0);                 // prelude: Mutex
    task::spawn(async { /* ... */ });      // prelude 모듈 task
    task::sleep(Duration::ms(10));         // prelude: Duration

    let n = Atomic::new(0);
    n.store(1, SeqCst);                    // 매개변수 타입으로 정한 변형
}
```

`rok::core`의 나머지는 prelude 모듈을 거쳐 부르거나 가져옵니다.

```rok
let p: future::Poll<i32> = future::Poll::Pending;
impl ops::Add for V { ... }
use rok::core::sync::AtomicOrdering;
let h = encoding::hex_encode("rok");
```

스코프에 없는 이름을 쓰면 무엇을 써야 하는지 알려 주는 에러가 납니다. 동작하는 가장 짧은 경로(`task::sleep`, `ops::Add`, `Ordering::Less`)와 그 이름을 가져오는 `use`를 함께 보여 줍니다.

## 0.2.4. 이름 충돌 정책

묵시 import는 우선순위가 가장 낮습니다. 이름을 찾을 때 모듈 자신의 항목과 그 모듈의 `use`를 먼저 봅니다([refs-lang 5.7.1](../../refs-lang/005-modules/507-paths.md#571-경로의-시작-식별자)). 사용자 항목의 이름이 prelude 이름과 같으면 그 모듈에서는 사용자 항목이 prelude 이름을 가리고, core 항목에는 경로로 여전히 접근할 수 있습니다.

```rok
struct List<T> { ... }                  // OK: 여기서 prelude 의 List 를 가림
let xs: rok::core::collections::List<i32> = [1, 2];   // 경로로 가리킨 core 의 List
```

`use` 경로는 prelude 이름으로 시작할 수 있습니다. `use task::sleep;`은 `rok::core::task::sleep`을 가져옵니다.

## 0.2.5. 묵시 적용되지 않는 계층

묵시적으로 적용하는 것은 prelude뿐입니다. unsafe와 std는 **명시해야만** 쓸 수 있습니다.

```rok
// unsafe: 위험 시그널
use rok::bare::ptr::{Ptr, ConstPtr};
use rok::bare::ffi::{CStr, CString};

// std: target-dep 시그널
use rok::std::fs;
use rok::std::time::SystemTime;
```

이 정책은 [0.1 tier 모델](001-tier-model.md)의 다음 표와 맞물립니다.

| tier        | 묵시 import            | 이유                   |
|-------------|------------------------|------------------------|
| `rok::core` | ✓ prelude (0.2.3)      | 모든 target / 안전     |
| `rok::bare` | ✗                      | 위험 시그널, 명시 강제 |
| `rok::std`  | ✗                      | target-dep, 명시 강제  |

이렇게 하면 코드가 어떤 영역을 쓰는지 파일 맨 위만 봐도 알 수 있습니다.

## 0.2.6. 적용 해제: `#![no_implicit_rok]`

특수한 상황에서는 묵시 import 자체를 끌 수 있습니다.

```rok
#![no_implicit_rok]   // 파일 최상단

// 이 파일에서는 묵시 import 안 됨
use rok::core::prelude::*;   // 같은 import 를 직접 적음
```

리터럴은 컴파일러가 정의하므로 계속 쓸 수 있고, 연산자와 core 트레잇의 메서드도 그대로입니다([refs-lang 6.7.8](../../refs-lang/006-meta-system/607-operator-magic.md#678-매핑의-가시성)). 잃는 것은 맨 이름뿐입니다.

학습용이나 데모 코드에서 각 항목이 어디서 오는지 한눈에 보여 줄 때 쓸모가 있습니다.

표준 라이브러리 자신의 모듈은 prelude를 가져오지 않습니다. 각 계층은 `rok::core`의 모든 공개 항목을 마지막 이름으로 봅니다(내부의 `use rok::core::**;`). 사용자 코드에는 이 import가 없습니다.

## 0.2.7. 가시성

prelude는 `pub` 항목만 재내보냅니다. 내부 구현(`pub(pkg)` / private)은 사용자에게 새어 나가지 않습니다. `rok::core::collections::raw`는 `pub(pkg)`이므로 `GcBuf`와 `buf_*` 함수는 `use`를 명시해 `rok::bare::mem::..`로만 씁니다.

목록 밖으로 이름을 평탄화하지 않고 모듈 경로를 그대로 보존합니다.

```rok
let m = Mutex::new(0);              // prelude: Mutex
let s = task::spawn(async { /* */ });      // prelude 모듈 task. spawn 은 task::spawn

// 둘은 같은 뜻입니다.
rok::core::sync::Mutex::new(0);
Mutex::new(0);
```

## 0.2.8. 다른 언어와의 비교

| 언어     | 묵시 스코프                                     | 변형                                       |
|----------|-------------------------------------------------|--------------------------------------------|
| Rust     | `std::prelude::rust_20xx`, 명시 목록            | `Some` `None` `Ok` `Err`만                 |
| Go       | universe block: 내장 함수와 타입 20여 개        | 한정 이름 (`time.Second`)                  |
| Swift    | `Swift`의 모든 공개 선언                        | 문맥 타입으로 정하는 `.less`               |
| Kotlin   | 기본 패키지 import                              | 문맥 기반 해석 (2.2)                       |
| **Rok**  | `rok::core::prelude`, 명시 목록 (0.2.3)         | 기대 타입으로 정하는 맨 이름 (모든 enum)   |
