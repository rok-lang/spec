# 2. Rok tier

rok tier 는 안전 영역 표준 라이브러리 으로, 모든 target 에서 빌드되는 rok 의 실용 표면입니다. 사용자 코드의 90% 가 이 tier 의 항목을 사용하기 때문에, rok 은 rok 라는 언어가 일상적인 프로그래밍에서 어떤 모습으로 드러나는지를 결정하는 핵심 계층이라 할 수 있습니다.

## 위치

먼저 사양 문서가 자리잡은 위치는 다음과 같습니다.

- spec: `spec/ko/refs-standard-libraries/200-core/`

## 의존성

rok tier 는 위로는 `unsafe` 를 내부 사용 목적으로 의존하며, 아래로는 `std` 와 사용자 코드가 rok 을 의존자로 갖습니다. 따라서 rok 은 unsafe 와 std 사이를 잇는 안전 영역의 중심에 위치합니다.

## 사용자 import 정책

`use rok::core::**;` 가 **묵시 적용**되며, 모든 사용자 파일에 컴파일러가 자동으로 prepend 합니다. 그 결과 rok 의 모든 pub 항목 (List, Dict, Add, Display, Iterator, Mutex, task, ...) 이 별도 import 없이 사용 가능합니다.

또한 필요에 따라 좁게 명시하는 방식도 허용됩니다.

```rok
use rok::core::collections::List;           // 좁은 명시
use rok::core::ops::{Add, Sub};
```

> [!NOTE]
> rok 은 **별도 `rok::core::prelude` 모듈을 두지 않습니다**. `**` recursive glob 자체가
> prelude 역할 - 추가 indirection 없습니다.

자세한 정책은 [0.2 import 정책](../000-meta/002-import.md) 에서 다룹니다.

## 모듈 목차

이어서 rok tier 가 제공하는 모듈을 영역별로 정리합니다.

### 숫자 / niche

- [2.1. 숫자 타입 (`int`, `float`, primitive 통합)](201-numbers.md)
- [2.2. niche wrapper (`NonZero`, `NonNan`, `Finite`, `Positive`)](202-non-zero.md)

### 연산자 트레잇

- [2.3. 산술 / 비트 / 논리 (`Add`, `Sub`, `Mul`, `Div`, `Rem`, `Neg`, `Not`, `BitNot`, `BitAnd`, `BitOr`, `BitXor`, `Shl`, `Shr`, `LogicalAnd`, `LogicalOr`)](203-ops.md)
- [2.4. 복합 대입 (`AddAssign`, `SubAssign`, ..., `ShrAssign`)](204-ops-assign.md)
- [2.5. 비교 (`PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`)](205-cmp.md)
- [2.6. broadcast (`DotAdd`, `DotMul`, ..., `DotShr`)](206-broadcast.md)

### iter / stream

- [2.7. iter (`Iterator`, `IntoIterator`, `Range`, `RangeInclusive`)](207-iter.md)
- [2.8. `Stream` (async iterator)](208-stream.md)

### 컬렉션

- [2.9. 컬렉션 (`List`, `Dict`, `Set`, `Tuple`, `Slice`, `OrderedDict`, `OrderedSet`)](209-collections.md)

### 포맷 / 변환

- [2.10. fmt (`Display`, `Debug`, `Formatter`)](210-fmt.md)
- [2.11. 변환 (`From`, `Into`, `TryFrom`, `TryInto`, `saturating_into`, `wrapping_into`)](211-conversion.md)

### 메타 / 마커

- [2.12. 리플렉션 (`Reflect`, `Inspect`)](212-reflect.md)
- [2.13. marker (`Clone`, `Copy`, `Drop`, `NoneSentinel`)](213-marker.md)

### 에러 / 문자열

- [2.14. 에러 (`Error`, `Result` (보관용), `Option` (제거), `TypeError`, `ValueError`, `IndexError`, `OverflowError`)](214-error.md)
- [2.15. 문자열 (`str` 가변 단일, `char`, `c_char`)](215-string.md)

### 동시성 / 비동기

- [2.16. sync primitive (`Atomic`, `Mutex`, `RwLock`, `Cell`)](216-sync.md)
- [2.17. `Channel` - 메시지 패싱](217-channel.md)
- [2.18. `task::` - M:N 스케줄러 task API](218-task.md)
- [2.19. `par::` - 병렬 처리 API](219-par.md)
- [2.20. `Future` / `await` 통합](220-future.md)

### 암호

- [2.21. `crypto::` - 해시, MAC, AEAD, 키 교환, 서명](221-crypto.md)

> [!NOTE]
> 텐서 / 자동 미분 / SIMD / GPU 같은 과학 컴퓨팅 영역은 rok tier 에 두지 않고 **외부 라이브러리** 로 분리합니다. 외부 라이브러리는 [`rok::bare::mlir`](../100-bare/) 모듈을 통해 MLIR 다이얼렉트에 직접 접근할 수 있습니다.

## 핵심 디자인 원칙

rok tier 의 설계는 다음 네 가지 축을 따릅니다.

먼저 트레잇이 우선합니다. 모든 의미 결정은 트레잇 메서드 dispatch 로 이루어지며, `+` 연산은 `Add::add` 로, `[i]` 인덱싱은 `Index::index` 로 해소됩니다.

그다음 변환은 명시적입니다. `i64 → i32` 같이 정보 손실이 발생할 수 있는 변환은 컴파일 또는 런타임 에러로 거부되며, saturate 나 wrap 같은 정책은 per-op 단위로 명시되어야 합니다 (refs-standard-libraries 0.x - 변환 정책).

또한 **GC 친화** 를 지향합니다. 컬렉션과 동시성 객체는 GC 로 관리되며, 외부 자원의 `drop` 은 `using` 블록에서만 호출됩니다.

따라서 마지막으로 **target 독립성** 이 보장됩니다. 모든 항목은 모든 Tier 1 플랫폼에서 동일하게 동작하며, target 에 의존하는 항목은 `std` 로 분리됩니다.
