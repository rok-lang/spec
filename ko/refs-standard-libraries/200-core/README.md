# 2. core 계층

core 계층(`rok::core`)은 안전한 표준 라이브러리입니다. 모든 타깃에서 빌드되는 Rok의 실용 표면입니다. 사용자 코드의 약 90%가 이 계층의 항목을 씁니다. 그래서 일상적인 프로그래밍에서 Rok이 어떤 모습인지는 이 계층이 정합니다.

## 위치

사양 문서는 다음 위치에 있습니다.

- spec: `spec/ko/refs-standard-libraries/200-core/`

## 의존성

core 계층은 위로는 내부 용도로 `rok::bare`에 의존합니다. 아래로는 `rok::std`와 사용자 코드가 core 계층에 의존합니다. 그래서 core 계층은 `rok::bare`와 `rok::std` 사이에서 안전 영역의 중심에 있습니다.

## 사용자 import 정책

`use rok::core::prelude::*;`는 **묵시적으로 적용**됩니다. 컴파일러가 모든 사용자 파일 앞에 이 줄을 붙입니다. prelude의 이름(List, Dict, Display, Iterator, Mutex, task 등)은 따로 가져오지 않고 쓸 수 있고, core의 나머지는 prelude 모듈을 거치거나(`ops::Add`, `task::sleep`) `use`로 가져옵니다.

필요하면 가져올 항목을 좁게 적어도 됩니다.

```rok
use rok::core::collections::List;           // 좁은 명시
use rok::core::ops::{Add, Sub};           // 연산자 트레잇은 prelude 에 없음
```

> [!NOTE]
> prelude 목록은 규범이며 enum 변형을 담지 않습니다. 맨 변형은 기대 타입으로 정해집니다
> ([refs-lang 2.4.8](../../refs-lang/002-types/204-enum.md#248-기대-타입으로-정하는-맨-변형)).

자세한 정책은 [0.2 import 정책](../000-meta/002-import.md)에 있습니다.

## 모듈 목차

core 계층의 모듈을 영역별로 나누면 다음과 같습니다. 항목 끝에는 그 장이 정의하는 `rok::core` 모듈을 적습니다.

### 숫자 / niche

- [2.1. 숫자 타입 (고정 폭, `Int`, `Float`)](201-numbers.md): `num`
- [2.2. niche wrapper (`NonZero`, `NonNan`, `Finite`, `Positive`)](202-non-zero.md): `num`

### 연산자 트레잇

- [2.3. 산술 / 비트 / 논리 (`Add`, `Sub`, `Mul`, `Div`, `Rem`, `Neg`, `Not`, `BitNot`, `BitAnd`, `BitOr`, `BitXor`, `Shl`, `Shr`, `LogicalAnd`, `LogicalOr`, `Index`)](203-ops.md): `ops`, `index`
- [2.4. 복합 대입 (`AddAssign`, `SubAssign`, ..., `ShrAssign`)](204-ops-assign.md): `ops`
- [2.5. 비교 (`PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`, `Hasher`)](205-cmp.md): `cmp`, `hash`
- [2.6. broadcast (`DotAdd`, `DotMul`, ..., `DotShr`)](206-broadcast.md): `ops`

### iter / stream

- [2.7. iter (`Iterator`, `IntoIterator`, `Range`, `RangeInclusive`)](207-iter.md): `iter`, `range`
- [2.8. `Stream` (async iterator)](208-stream.md): `stream`

### 컬렉션

- [2.9. 컬렉션 (`List`, `Dict`, `Set`, `Slice`, `Deque`, `BinaryHeap`, 배열, 튜플)](209-collections.md): `collections`, `array`, `tuple`

### 포맷 / 변환

- [2.10. fmt (`Display`, `Debug`, `Formatter`, `print`)](210-fmt.md): `fmt`, `print`
- [2.11. 변환 (`From`, `Into`, `TryFrom`, `TryInto`, `saturating_into`, `wrapping_into`)](211-conversion.md): `convert`

### 메타 / 마커

- [2.12. 리플렉션 (`Reflect`, `Inspect`)](212-reflect.md): `reflect`
- [2.13. marker (`Clone`, `Copy`, `Drop`, `NoneSentinel`)](213-marker.md): `marker`

### 에러 / 문자열

- [2.14. 에러 (`Error`, `Result`, `TypeError`, `ValueError`, `OverflowError`, `panic`)](214-error.md): `error`, `panic`
- [2.15. 문자열 (`str`, `StrBuf`, `char`, `c_char`)](215-string.md): `str`

### 동시성 / 비동기

- [2.16. sync primitive (`Atomic`, `Mutex`, `RwLock`, `Cell`)](216-sync.md): `sync`
- [2.17. `Channel` - 메시지 패싱](217-channel.md): `channel`
- [2.18. `task::` - M:N 스케줄러 task API](218-task.md): `task`
- `time` - `Duration` / `Instant`, std 시계와 함께 [3.3](../300-std/303-time.md)에서 정의
- [2.19. `par::` - 병렬 처리 API](219-par.md): `par`
- [2.20. `Future` / `await` 통합](220-future.md): `future`

### 암호

- [2.21. `crypto::` - 해시, MAC, AEAD, 키 교환, 서명](221-crypto.md): `crypto`

### 인코딩

- [2.22. `encoding::` - hex와 base64](222-encoding.md): `encoding`

### 입출력

- [2.23. io (`Read`, `BufRead`, `Write`, `Seek`, `AsyncRead`, `AsyncWrite`, `BufReader`, `Cursor`, 표준 입출력)](223-io.md): `io`

### 벤치마크 / 최적화 힌트

- `bench` - `#[bench]` 함수가 받는 핸들 `Bench`. 측정 방법은 [refs-lang 9.9.11](../../refs-lang/009-tools/909-debug-tools.md#9911-벤치마크)에 있습니다.

  ```rok
  pub struct Bench { /* 비공개 필드 */ }   // Debug: `Bench { name: "parse_1kb", .. }`
  impl Bench {
      pub fn iter<T>(ref self, f: fn() -> T);  // f를 보정, 예열, 표본 수집 후 보고
      pub fn set_bytes(ref self, n: i64);      // 호출당 바이트 수: 처리량 열을 더함
      pub fn set_items(ref self, n: i64);      // 호출당 항목 수: 처리량 열을 더함
  }
  ```
- `hint` - `black_box<T>(x: T) -> T`: `x`를 그대로 돌려주지만 최적화기가 들여다보지 못하므로, `x`를 만든 계산도 결과의 사용도 접혀 사라지지 않습니다. 벤치마크는 시간을 재는 구간의 입력과 결과를 이것으로 감쌉니다.

> [!NOTE]
> 텐서, 자동 미분, SIMD, GPU 같은 과학 컴퓨팅 영역은 core 계층에 두지 않고 **외부 라이브러리**로 분리합니다. 외부 라이브러리는 Rok 0.2.2에 도입할 예정인 백엔드 인터페이스로 컴파일러 백엔드에 닿습니다.

## 핵심 설계 원칙

core 계층의 설계는 다음 네 가지 축을 따릅니다.

첫째, 트레잇이 우선합니다. 모든 의미는 트레잇 메서드 디스패치로 정해집니다. `+` 연산은 `Add::add`로, `[i]` 인덱싱은 `Index::index`로 해소됩니다.

둘째, 변환은 명시적입니다. `i64 → i32`처럼 정보를 잃을 수 있는 변환은 컴파일 에러나 런타임 에러로 거부됩니다. 포화(saturate)나 랩(wrap) 같은 정책은 연산마다 적어야 합니다([2.11](211-conversion.md)).

셋째, **GC 친화**를 지향합니다. 컬렉션과 동시성 객체는 GC가 관리합니다. `Drop` 자원은 affine 값입니다. 마지막 소유자가 스코프를 벗어나는 곳이나 `using` 블록 끝에서 해제됩니다([2.13.4](213-marker.md)).

마지막으로 **타깃 독립성**을 보장합니다. 모든 항목은 모든 Tier 1 플랫폼에서 똑같이 동작합니다. 타깃에 의존하는 항목은 `std`로 분리합니다.
