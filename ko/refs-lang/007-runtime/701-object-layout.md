# 7.1. 객체 레이아웃과 메모리 공간

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 가 관리하는 모든 객체는 고정 헤더로 시작한다. 본 문서는 그 헤더 형식과 객체가 살 수 있는 네 가지 메모리 공간을 정의한다.

## 7.1.1. 바이트 헤더

모든 GC 객체의 처음 16 바이트는 헤더다.

```
offset  0 :  type_id    u32       // 타입 식별자
offset  4 :  gc_flags   u32       // GC 상태 비트
offset  8 :  gc_word    u64       // forwarding 포인터 또는 LOS 크기
offset 16 :  payload ...           // 객체의 실제 필드들
```

- 8 바이트 정렬: 헤더 자체와 payload 모두 해당.
- 32-bit 와 64-bit 양쪽에서 동일한 레이아웃. Tier 1 platform matrix 의 32-bit 타깃에서도 16 바이트.

### `type_id` (offset 0, 4 bytes)

타입의 고유 식별자. 컴파일러가 타입마다 부여한다.

용도:

- 동적 디스패치: [`any` 슬롯](../002-types/202-any-slot.md) 의 태그로 쓴다.
- GC 의 필드 스캔: `type_id → field layout` 매핑으로 필드 위치를 파악한다.
- 진단 메시지: 사용자 친화적 타입 이름을 출력한다.

`type_id` 는 프로세스 수명 동안 안정적이다. 같은 타입은 항상 같은 id 를 갖는다.

### `gc_flags` (offset 4, 4 bytes)

GC 상태 비트.

| 비트        | 의미                                                                                    |
| ----------- | --------------------------------------------------------------------------------------- |
| `mark`      | 현재 GC 사이클에서 도달했는지                                                           |
| `sticky`    | [sticky mark-bit](../000-meta/007-glossary.md#sticky-mark-bit) (세대 추적)              |
| `pinned`    | [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 등으로 이동 금지 표시 |
| `forwarded` | 이미 evacuate 되어 새 위치로 옮겨짐 (gc_word 가 forwarding 포인터)                      |
| `large`     | [LOS](../000-meta/007-glossary.md#los-large-object-space) (Large Object Space) 객체     |
| `immortal`  | 영구 살아있음 (수집 안 함)                                                              |
| ...         | 추가 비트는 GC 구현이 결정                                                              |

정확한 비트 배치는 GC 구현이 결정한다. 위 표는 의미상 필요한 비트 종류를 나열한다.

### `gc_word` (offset 8, 8 bytes)

상황에 따라 두 가지 의미를 갖는다.

- 일반 객체 ([Immix](../000-meta/007-glossary.md#immix) 공간): evacuate 시 forwarding 포인터. `forwarded` 비트가 켜지면 이 슬롯이 새 위치를 가리킨다.
- LOS 객체: 객체 크기 (페이로드 바이트 수). LOS 는 객체마다 크기가 다르므로 헤더에 저장한다.

### `payload` (offset 16+)

객체의 실제 필드들. 컴파일러가 필드 순서와 정렬을 결정한다 ([`struct`](../002-types/203-struct.md) / [`enum`](../002-types/204-enum.md) / [`union`](../002-types/205-union.md)).

`#[repr(C)]` 같은 속성이 붙으면 그 표현 규칙을 따른다 ([내장 속성](../006-meta-system/602-built-in-attributes.md)).

## 7.1.2. 메모리 공간

GC 는 객체를 네 공간 중 하나에 배치한다.

### 1. Immortal

`const`, 리터럴 인스턴스, 부팅 시 생성되는 정적 객체 등.

- bump pointer 로 할당. 가장 빠르다.
- 수집하지 않는다. 프로세스 수명 동안 살아있다.
- `gc_flags` 의 `immortal` 비트가 켜진다. GC 가 그냥 지나친다.

[`const`](../005-modules/502-const.md) 의 `const` 표현식 결과, [`static`](../005-modules/503-static.md) 의 초깃값 등이 여기에 들어간다.

### 2. Immix

대부분의 일반 객체가 사는 공간. Immix mark-region 알고리듬을 쓴다.

- 32 KB [block](../000-meta/007-glossary.md#block-블록) 단위로 OS 에서 메모리를 받는다.
- 각 block 은 128 B line 단위로 분할된다.
- mark-sweep 이 기본, opportunistic evacuation (단편화가 심할 때만 copy).
- sticky mark-bit 으로 세대를 추적한다.

자세한 알고리듬은 [GC 알고리듬](703-gc-algorithm.md) 에서 정의한다.

### 3. Large Object Space (LOS)

특정 크기 이상의 객체는 이동 없이 별도 공간에 둔다.

| 비트 폭 | LOS 임계 |
| ------- | -------- |
| 64-bit  | ≥ 8 KB   |
| 32-bit  | ≥ 2 KB   |

- non-moving mark-sweep. 수집은 하지만 evacuate 는 하지 않는다.
- 한 객체 = 한 page (mmap 으로 직접 할당).
- `gc_flags` 의 `large` 비트가 켜진다.

큰 객체를 evacuate 하면 복사 비용이 막대하다. 그래서 큰 객체는 제자리에 두고 mark-sweep 만 한다.

### 4. Non-Moving (FFI pinned)

[`extern` 블록](../005-modules/509-extern-block.md) 의 FFI 함수에 주소를 넘긴 객체는 외부 코드가 그 주소를 들고 있는 동안 이동하면 안 된다.

- non-moving 공간에 둔다.
- `gc_flags` 의 `pinned` 비트가 켜진다.
- 사용자가 `pin` / `unpin` 을 명시적으로 호출한다 (표준 라이브러리 함수).

이 공간도 mark-sweep 으로 수집한다. 단 evacuate 는 하지 않는다.

## 7.1.3. 공간 사이의 이동

GC 는 다음 경로로 객체를 이동시킨다.

```
TLAB (할당) → Immix
Immix → Immix (evacuate, 단편화 시)
Immix → LOS (큰 객체 승급 — 이론상 가능, 일반적으론 처음부터 LOS)
Immix / LOS → Non-Moving (pin 호출 시)
Non-Moving → Immix / LOS (unpin 호출 시)
Immortal — 진입 / 이동 없음
```

대부분의 객체는 사는 동안 자기 공간 안에서 이동하지 않는다.

## 7.1.4. 헤더 비트 폭의 동기

16 바이트 헤더는 시간/공간 트레이드오프의 결과다.

- 작은 객체 (예: 4 바이트 i32) 도 헤더가 16 바이트라 메모리 효율이 떨어진다.
- 16 바이트면 type_id, flags, forwarding 을 모두 담을 수 있어 알고리듬이 단순하다.
- 32/64 비트 양쪽에서 레이아웃이 같아 크로스 플랫폼 코드가 단순해진다.

이 트레이드오프는 대부분의 객체가 4 바이트 이상이라는 가정 위에서 합리적이다.

## 7.1.5. type_id 등록: `nok_gc_register_type`

각 타입의 필드 레이아웃 정보를 GC 에 등록한다.

```
nok_gc_register_type(type_id, scan_table)
```

- `type_id`: 그 타입의 식별자.
- `scan_table`: 필드 가운데 GC 객체를 가리키는 포인터들의 offset 목록 (+ 추가 메타).

GC 는 객체를 마크할 때 `type_id` 로 `scan_table` 을 찾아 자식 포인터들을 따라간다.

이 등록은 컴파일러가 `extern "C"` 호출로 프로세스 시작 시점에 수행한다. 자세한 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 는 후속 사양에서 정의한다.

## 7.1.6. 객체의 총 크기 계산

```
size = 16  (헤더) + payload_size  (필드들의 정렬된 크기)
```

`payload_size` 는 컴파일러가 `type_id → size` 표로 알고 있다. 일반 (Immix) 객체는 `type_id` 한 번의 lookup 으로 크기가 결정된다.

LOS 객체는 헤더의 `gc_word` 에서 크기를 직접 읽는다.
