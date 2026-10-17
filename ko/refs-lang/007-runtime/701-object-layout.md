# 7.1. 객체 레이아웃과 메모리 공간

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 가 관리하는 모든 객체는 고정 헤더로 시작합니다. 본 문서에서는 그 헤더의 형식과 함께 객체가 자리잡을 수 있는 네 가지 메모리 공간을 정의합니다.

## 7.1.1. 바이트 헤더

모든 GC 객체의 처음 16 바이트는 헤더로 쓰입니다.

```
offset  0 :  type_id    u32       // 타입 식별자
offset  4 :  gc_flags   u32       // GC 상태 비트
offset  8 :  gc_word    u64       // 객체 총 크기 (evacuation 중에는 새 주소)
offset 16 :  payload ...           // 객체의 실제 필드들
```

헤더 자체와 payload 모두 8 바이트 정렬을 따릅니다. 또한 32-bit 와 64-bit 양쪽에서 동일한 레이아웃을 유지하기 때문에 Tier 1 플랫폼 matrix 의 32-bit 타깃에서도 헤더는 변함없이 16 바이트입니다.

### `type_id` (offset 0, 4 bytes)

타입의 고유 식별자로서, 컴파일러가 타입마다 부여합니다.

이 식별자는 여러 용도로 활용됩니다. 먼저 동적 디스패치에서는 [`any` 슬롯](../002-types/202-any-slot.md) 의 태그로 쓰이며, GC 의 필드 스캔에서는 `type_id → field layout` 매핑을 통해 필드 위치를 파악하는 데 사용됩니다. 그 밖에도 진단 메시지를 출력할 때 사용자 친화적인 타입 이름을 보여주는 역할을 합니다.

`type_id` 는 프로세스 수명 동안 안정적이며, 같은 타입은 항상 같은 id 를 갖게 됩니다.

### `gc_flags` (offset 4, 4 bytes)

GC 상태를 나타내는 비트들입니다.

| 비트        | 의미                                                                                    |
|-------------|-----------------------------------------------------------------------------------------|
| `mark`      | 현재 GC 사이클에서 도달했는지                                                           |
| `sticky`    | [sticky mark-bit](../000-meta/007-glossary.md#sticky-mark-bit) (세대 추적)              |
| `pinned`    | [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 등으로 이동 금지 표시 |
| `forwarded` | 이미 evacuate 되어 새 위치로 옮겨짐 (gc_word 가 forwarding 포인터)                      |
| `large`     | [LOS](../000-meta/007-glossary.md#los-large-object-space) (Large Object Space) 객체     |
| `immortal`  | 영구 살아있음 (수집 하지 않습니다)                                                      |
| ...         | 추가 비트는 GC 구현이 결정                                                              |

정확한 비트 배치는 GC 구현이 결정하며, 위 표는 의미상 필요한 비트의 종류를 나열한 것일 뿐입니다.

### `gc_word` (offset 8, 8 bytes)

객체의 총 크기 (헤더 포함) 입니다. evacuation 중 옮겨진 객체는 `forwarded` 비트가 켜지고 이 슬롯이 새 위치를 가리킵니다 ([8.1.4](../008-abi/801-object-header-abi.md#814-gc_word-offset-8-u64-또는-ptr)).

### `payload` (offset 16+)

객체의 실제 필드들이 자리잡는 영역입니다. 필드의 순서와 정렬은 컴파일러가 결정하며, 이는 [`enum`](../002-types/204-enum.md), [`union`](../002-types/205-union.md) 에도 해당합니다. [`struct`](../002-types/203-struct.md) 값은 그 자체로 heap 객체가 아니며, heap 객체 (list 버퍼, 클로저 환경, coroutine frame) 안에 저장되면 여기에 inline 으로 배치됩니다 ([8.8.3](../008-abi/808-lowering-types.md#883-struct-의-표현)).

`#[repr(C)]` 같은 속성이 붙어 있다면 그에 해당하는 표현 규칙을 따르게 됩니다 ([내장 속성](../006-meta-system/602-built-in-attributes.md)).

## 7.1.2. 메모리 공간

GC 는 객체를 다음 네 공간 중 하나에 배치합니다.

### 1. Immortal

`const`, 리터럴 인스턴스, 부팅 시 생성되는 정적 객체 등이 이 공간에 들어갑니다.

할당은 bump pointer 방식으로 이루어져 가장 빠르며, 수집은 하지 않으므로 프로세스 수명 동안 그대로 살아 있습니다. `gc_flags` 의 `immortal` 비트가 켜지면 GC 가 그 객체를 그냥 지나치게 됩니다.

[`const`](../005-modules/502-const.md) 의 `const` 표현식 결과나 [`static`](../005-modules/503-static.md) 의 초깃값 등이 여기에 자리잡습니다.

### 2. Immix

대부분의 일반 객체가 사는 공간으로, Immix mark-region 알고리듬을 사용합니다.

32 KiB [block](../000-meta/007-glossary.md#block-블록) 과 128 B line 으로 나눕니다. mark-sweep 이 기본이고, full 수집이 단편화된 block 의 객체를 옮길 수 있습니다 (opportunistic evacuation). 세대는 sticky mark-bit 으로 나눕니다 ([7.3](703-gc-algorithm.md)).

### 3. Large Object Space (LOS)

8 KiB 이상이거나 8 바이트보다 큰 정렬을 요구하는 객체는 LOS 에 하나씩 따로 할당하며, `gc_flags` 의 `LARGE` 비트를 켭니다. LOS 는 mark-sweep 으로 수집하고 객체를 옮기지 않습니다. old LOS 객체는 card table 을 가집니다 ([7.4.2](704-barriers-and-roots.md#742-write-barrier-incremental-update)).

### 4. 고정된 객체 (FFI pinned)

[`extern` 블록](../005-modules/509-extern-block.md) 의 FFI 함수에 넘긴 GC 포인터는 호출이 끝날 때까지 인자가 루트에 있으므로 옮겨지지 않습니다 ([7.4.1](704-barriers-and-roots.md#루트가-가리키는-객체는-옮기지-않음)). 외부 코드가 호출 뒤에도 주소를 들고 있으려면 `rok_gc_pin` 으로 고정합니다. 고정된 객체는 `gc_flags` 의 `PINNED` 비트가 켜지며 제자리에 남습니다. `rok_gc_unpin` 이 고정을 풉니다.

고정된 객체도 mark-sweep 으로 수집하되, evacuate 는 하지 않습니다.

## 7.1.3. 공간 사이의 이동

```
TLAB (할당) → Immix
Immix → Immix         (full 수집의 evacuation)
Immix / LOS ↔ 고정    (pin / unpin, 제자리)
Immortal              (이동 없음)
```

객체는 공간을 옮겨 다니지 않습니다. 주소가 바뀌는 것은 evacuation 뿐입니다.

## 7.1.4. 헤더 비트 폭의 동기

16 바이트 헤더는 시간과 공간 사이의 트레이드오프 끝에 나온 결과입니다.

작은 객체, 예컨대 4 바이트 i32 같은 경우에는 헤더가 16 바이트나 차지하기 때문에 메모리 효율이 떨어진다는 단점이 있습니다. 그렇지만 16 바이트면 type_id, flags, forwarding 을 모두 한 번에 담을 수 있어 알고리듬이 단순해진다는 장점이 있고, 32 비트와 64 비트 양쪽에서 레이아웃이 동일하게 유지되어 크로스 플랫폼 코드도 함께 단순해집니다.

이러한 트레이드오프는 대부분의 객체가 4 바이트 이상이라는 가정 위에서 합리적인 선택이 됩니다.

## 7.1.5. type_id 등록: `rok_gc_register_type`

각 타입의 필드 레이아웃 정보는 GC 에 등록해 두어야 합니다.

```
rok_gc_register_type(type_id, scan_table)
```

여기에서 `type_id` 는 그 타입의 식별자이고, `scan_table` 은 필드 가운데 GC 객체를 가리키는 포인터들의 offset 목록과 추가 메타 정보를 담고 있습니다.

GC 는 객체를 마크할 때 `type_id` 를 사용해 `scan_table` 을 찾고, 그를 통해 자식 포인터들을 따라가게 됩니다.

이 등록은 컴파일러가 `extern "C"` 호출을 통해 프로세스 시작 시점에 수행하며, 등록 ABI 는 [Extern "C" 심볼](../008-abi/805-extern-symbols.md) 에서 정의합니다.

## 7.1.6. 객체의 총 크기 계산

```
size = 16  (헤더) + payload_size  (필드들의 정렬된 크기)
```

`payload_size` 는 컴파일러가 `type_id → size` 표를 통해 알고 있습니다. 따라서 일반적인 Immix 객체의 경우 `type_id` 한 번의 lookup 으로 크기가 결정됩니다.

LOS 객체는 헤더의 `gc_word` 에서 크기를 직접 읽어옵니다.
