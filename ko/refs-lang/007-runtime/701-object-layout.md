# 7.1. 객체 레이아웃과 메모리 공간

[GC](../000-meta/007-glossary.md#gc-garbage-collector)가 관리하는 모든 객체는 고정 헤더로 시작합니다. 이 절은 헤더 형식과 객체가 놓일 수 있는 네 가지 메모리 공간을 정의합니다.

> [!NOTE]
> 이 절은 비규범(참고용)입니다. 다만 16바이트 헤더는 [8.1](../008-abi/801-object-header-abi.md)에서 ABI로 고정하므로 규범입니다. 메모리 공간은 참조 런타임을 설명하며 어느 릴리스에서든 바뀔 수 있습니다.

## 7.1.1. 16바이트 헤더

모든 GC 객체의 처음 16바이트는 헤더입니다.

```
offset  0 :  type_id    u32       // 타입 식별자
offset  4 :  gc_flags   u32       // GC 상태 비트
offset  8 :  gc_word    u64       // 객체 총 크기 (evacuation 중에는 새 주소)
offset 16 :  payload ...           // 객체의 실제 필드들
```

헤더와 페이로드는 모두 8바이트로 정렬됩니다. 레이아웃은 32비트와 64비트에서 같습니다. 그래서 Tier 1 플랫폼 표의 32비트 타깃에서도 헤더는 16바이트입니다.

### `type_id`(오프셋 0, 4바이트)

타입의 고유 식별자입니다. 컴파일러가 타입마다 하나씩 붙입니다.

쓰임은 여러 가지입니다. 동적 디스패치에서는 [`any` 슬롯](../002-types/202-any-slot.md)의 태그로 씁니다. GC는 필드를 스캔할 때 `type_id → field layout` 매핑으로 필드 위치를 찾습니다. 진단 메시지는 이 값으로 읽기 쉬운 타입 이름을 보여 줍니다.

`type_id` 값은 프로세스가 끝날 때까지 바뀌지 않습니다. 같은 타입의 id는 항상 같습니다.

### `gc_flags`(오프셋 4, 4바이트)

GC 상태를 나타내는 비트입니다.

| 비트        | 의미                                                                                    |
|-------------|-----------------------------------------------------------------------------------------|
| `mark`      | 현재 GC 사이클에서 도달했는지                                                           |
| `sticky`    | [sticky mark-bit](../000-meta/007-glossary.md#sticky-mark-bit) (세대 추적)              |
| `pinned`    | [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 등으로 이동 금지 표시 |
| `forwarded` | 이미 evacuation으로 새 위치로 옮겨짐 (gc_word가 forwarding 포인터)                     |
| `large`     | [LOS](../000-meta/007-glossary.md#los-large-object-space) (Large Object Space) 객체     |
| `immortal`  | 영구 살아있음 (수집하지 않습니다)                                                       |
| ...         | 추가 비트는 GC 구현이 결정                                                              |

정확한 비트 배치는 GC 구현이 정합니다. 위 표에는 의미상 필요한 비트의 종류만 적었습니다.

### `gc_word`(오프셋 8, 8바이트)

헤더를 포함한 객체의 총 크기입니다. evacuation 중에 옮겨진 객체는 `forwarded` 비트가 켜지고, 이 슬롯이 새 위치를 가리킵니다([8.1.4](../008-abi/801-object-header-abi.md#814-gc_word오프셋-8-u64-또는-ptr)).

### `payload`(오프셋 16+)

객체의 필드가 놓이는 영역입니다. 필드 순서와 정렬은 컴파일러가 정합니다. [`enum`](../002-types/204-enum.md)과 [`union`](../002-types/205-union.md)도 마찬가지입니다. [`struct`](../002-types/203-struct.md) 값은 그 자체로는 힙 객체가 아닙니다. 리스트 버퍼, 클로저 환경, 코루틴 프레임 같은 힙 객체 안에 저장되면 이 영역에 인라인으로 놓입니다([8.8.3](../008-abi/808-lowering-types.md#883-struct의-표현)).

`#[repr(C)]` 같은 속성이 붙으면 그 속성의 표현 규칙을 따릅니다([내장 속성](../006-meta-system/602-built-in-attributes.md)).

## 7.1.2. 메모리 공간

GC는 객체를 다음 네 공간 중 하나에 배치합니다.

### 1. Immortal

`const`, 리터럴 인스턴스, 부팅 때 만드는 정적 객체 등이 이 공간에 들어갑니다.

할당은 bump pointer 방식이라 가장 빠릅니다. GC가 이 공간을 수집하지 않으므로 객체는 프로세스가 끝날 때까지 살아 있습니다. `gc_flags`의 `immortal` 비트가 켜져 있으면 GC는 그 객체를 건너뜁니다.

[`const`](../005-modules/502-const.md)의 `const` 표현식 결과나 [`static`](../005-modules/503-static.md)의 초깃값 등이 여기에 놓입니다.

### 2. Immix

대부분의 일반 객체가 사는 공간입니다. Immix mark-region 알고리듬을 씁니다.

공간은 32 KiB [block](../000-meta/007-glossary.md#block-블록)과 128 B line으로 나뉩니다. 기본은 mark-sweep입니다. full 수집은 단편화된 block의 객체를 옮길 수 있습니다(opportunistic evacuation). 세대는 sticky mark-bit로 나눕니다([7.3](703-gc-algorithm.md)).

### 3. Large Object Space (LOS)

8 KiB 이상이거나 8바이트보다 큰 정렬을 요구하는 객체, 그리고 비밀 버퍼(`rok_gc_secret_new`, [8.5.1](../008-abi/805-extern-symbols.md#rok_gc_secret_new--rok_gc_secret_wipe))는 LOS에 하나씩 따로 할당하고 `gc_flags`의 `LARGE` 비트를 켭니다. GC는 LOS를 mark-sweep으로 수집하고 객체를 옮기지 않습니다. old LOS 객체에는 card table이 있습니다([7.4.2](704-barriers-and-roots.md#742-write-barrier-incremental-update)). 비밀 버퍼에는 `WIPE` 비트([8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags오프셋-4-u32))가 있습니다. 이 버퍼를 해제하는 sweep은 먼저 페이로드를 0으로 덮어씁니다.

### 4. 고정된 객체(FFI pinned)

[`extern` 블록](../005-modules/509-extern-block.md)의 FFI 함수에 넘긴 GC 포인터는 호출이 끝날 때까지 옮겨지지 않습니다. 인자가 루트에 있기 때문입니다([7.4.1](704-barriers-and-roots.md#루트가-가리키는-객체는-옮기지-않음)). 외부 코드가 호출 뒤에도 주소를 들고 있어야 한다면 `rok_gc_pin`으로 고정합니다. 고정된 객체는 `gc_flags`의 `PINNED` 비트가 켜지고 제자리에 남습니다. 고정은 `rok_gc_unpin`으로 풉니다.

고정된 객체도 mark-sweep으로 수집하지만 evacuation으로 옮기지는 않습니다.

## 7.1.3. 공간 사이의 이동

```
TLAB (할당) → Immix
Immix → Immix         (full 수집의 evacuation)
Immix / LOS ↔ 고정    (pin / unpin, 제자리)
Immortal              (이동 없음)
```

객체는 공간 사이를 옮겨 다니지 않습니다. 주소를 바꾸는 것은 evacuation뿐입니다.

## 7.1.4. 헤더 비트 폭의 동기

16바이트 헤더는 시간과 공간 사이에서 절충한 결과입니다.

4바이트 i32 같은 작은 객체에는 16바이트 헤더가 메모리 효율을 떨어뜨립니다. 하지만 16바이트에는 type_id, flags, forwarding을 한꺼번에 담을 수 있어서 알고리듬이 단순해집니다. 32비트와 64비트에서 레이아웃이 같으므로 크로스 플랫폼 코드도 단순해집니다.

대부분의 객체가 4바이트 이상이라고 가정하면 이 절충은 합리적인 선택입니다.

## 7.1.5. type_id 등록: `rok_gc_register_type`

각 타입의 필드 레이아웃은 GC에 등록해야 합니다.

```
rok_gc_register_type(type_id, scan_table)
```

`type_id`는 그 타입의 식별자입니다. `scan_table`에는 GC 객체를 가리키는 포인터 필드의 오프셋 목록과 추가 메타 정보가 들어 있습니다.

GC는 객체를 마크할 때 `type_id`로 `scan_table`을 찾고, 그 표를 따라 자식 포인터를 추적합니다.

컴파일러는 프로세스가 시작할 때 `extern "C"` 호출로 타입을 등록합니다. 등록 ABI는 [Extern "C" 심볼](../008-abi/805-extern-symbols.md)에서 정의합니다.

## 7.1.6. 객체의 총 크기 계산

```
size = 16  (헤더) + payload_size  (필드들의 정렬된 크기)
```

컴파일러는 `type_id → size` 표로 `payload_size` 값을 압니다. 그래서 일반 Immix 객체의 크기는 `type_id`로 표를 한 번 찾으면 정해집니다.

LOS 객체는 헤더의 `gc_word`에서 크기를 바로 읽습니다.
