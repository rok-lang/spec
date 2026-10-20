# 8.1. 객체 헤더 ABI

[객체 레이아웃과 메모리 공간](../007-runtime/701-object-layout.md) 의 16 바이트 헤더의 비트 배치를 정의합니다. 헤더 크기, `gc_flags` 비트, LOS 임계, type id 의 시작 값, 런타임 buffer type id 같은 GC 상수는 컴파일러와 런타임이 함께 쓰는 정의 하나 (`gc_abi`) 에서 옵니다.

## 8.1.1. 정확한 레이아웃

헤더는 다음과 같은 형태로 구성됩니다.

```
+---------+---------+-----------------+----------------+
| offset  |  size   |  field          |  type          |
+---------+---------+-----------------+----------------+
|    0    |    4    |  type_id        |  u32           |
|    4    |    4    |  gc_flags       |  u32           |
|    8    |    8    |  gc_word        |  u64 / ptr     |
|   16    |    var  |  payload        |  per type_id   |
+---------+---------+-----------------+----------------+
```

> [!IMPORTANT]
> 총 헤더 크기는 16 바이트이고 8 바이트로 정렬되며, 32-bit 와 64-bit 타깃에서 모두 동일합니다.

모든 객체는 최소 24 바이트입니다 (헤더와 payload word 하나, `gc_abi` 의 `MIN_OBJECT_SIZE`). 그래서 컴파일된 코드는 참조를 가진 어떤 객체든 첫 payload word 를 load 할 수 있습니다.

## 8.1.2. `type_id` (offset 0, u32, little-endian)

`type_id` 는 타입의 고유 식별자입니다. 32 비트 unsigned 값으로 little-endian 방식으로 저장되며, 값 `0` 은 예약되어 null type 또는 잘못된 객체를 표시하는 용도로 사용됩니다. 컴파일러는 프로세스 시작 시 `rok_gc_register_type` 을 호출하여 각 type_id 를 등록해 둡니다.

### type_id 의 할당

같은 빌드 안에서 같은 타입은 같은 id 를 갖습니다. 빌드 사이의 id 는 안정적이지 않으므로 직렬화에 쓰지 않습니다.

### 예약 범위

예약된 id 범위는 다음과 같습니다.

| 범위           | 용도                        |
|----------------|-----------------------------|
| `0`            | invalid / null              |
| `1`            | 런타임 — GC 문자열 (traced 아님) |
| `2`            | 런타임 — word-traced buffer |
| `3` ~ `0xFFFF` | 표준 라이브러리 / 런타임 (예약) |
| `0x10000` ~    | 사용자 / 다른 패키지 타입   |

표준 라이브러리 타입의 정확한 id 매핑은 표준 라이브러리 사양에서 정의됩니다. 컴파일러가 사용자 타입에 부여하는 id 는 `0x10000` 부터 시작합니다.

## 8.1.3. `gc_flags` (offset 4, u32)

`gc_flags` 필드는 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 상태 비트의 정확한 위치를 담고 있으며, 다음과 같이 배치됩니다.

```
bit  0  : MARK       (현재 GC 사이클 mark)
bit  1  : STICKY     (sticky mark-bit, old generation)
bit  2  : PINNED     (FFI 이동 금지)
bit  3  : FORWARDED  (evacuate 후. gc_word 가 새 위치)
bit  4  : LARGE      (LOS 객체. gc_word 가 size)
bit  5  : IMMORTAL   (수집 안 함)
bit  6 ~ 31 : 예약 / GC 구현 결정
```

구현 예약 범위에서 다음 비트를 씁니다 ([7.4.2](../007-runtime/704-barriers-and-roots.md#742-write-barrier-incremental-update)).

| 비트 | 이름       | 의미                                                                                       |
|------|------------|--------------------------------------------------------------------------------------------|
| 6    | `UNLOGGED` | 수집에서 살아남은 객체에 `STICKY` 와 함께 켬. write barrier 가 remembered set 에 처음 기록할 때 지움 |
| 7    | `CARDED`   | 8 KiB 이상의 old 객체에 `UNLOGGED` 대신 켬. barrier 가 512 B card 를 dirty 로 표시. card table 포인터는 헤더 바로 앞 word |
| 8    | `NOMOVE`   | 주소를 수로 쓴 객체. 수명과 무관하게 영구히 옮기지 않음. `rok_task_key` ([8.5](805-extern-symbols.md)) 가 켬 |
| 10   | `SATB`     | 동시 mark 중 barrier 가 이 객체의 참조를 기록하는 동안 켬. 경쟁하는 store 는 꺼질 때까지 기다림 ([7.5.3](../007-runtime/705-concurrent-collection.md#753-satb-barrier)) |
| 11   | `SCANNED`  | 동시 mark 에서 이 객체를 훑었음 |

### atomic 요구

STW 수집에서 `MARK` / `STICKY` / `FORWARDED` 는 non-atomic 입니다. 동시 mark 중 marker 와 mutator 가 함께 다루는 `MARK` / `SATB` / `SCANNED` 와, mutator 끼리 경쟁하는 `UNLOGGED` 의 해제는 atomic 입니다 ([7.4.2](../007-runtime/704-barriers-and-roots.md#742-write-barrier-incremental-update)).

## 8.1.4. `gc_word` (offset 8, u64 또는 ptr)

`gc_word` 는 객체의 총 크기 (헤더 16 바이트 포함, 8 의 배수) 입니다. Immix 객체와 LOS 객체에 같이 적용되며, 할당 fast path 가 인라인으로 씁니다 ([8.9.9](809-lowering-control.md#899-gc-alloc-fast--slow-path)).

full 수집의 evacuation 중에만 옮겨진 객체의 `FORWARDED` 비트가 켜지고 `gc_word` 가 새 주소를 가리킵니다. 같은 수집 안에서 모든 참조를 고친 뒤 ([7.3.1](../007-runtime/703-gc-algorithm.md#731-immix-mark-region)) 옛 자리는 버리므로, mutator 는 forwarding 을 보지 않습니다.

block 안의 객체는 `type_id` 0 을 갖지 않습니다. 수집기는 0 을 빈 자리의 표시로 씁니다.

### 수집기 전용 정보

수집기는 헤더 앞 (객체 주소보다 낮은 주소) 이나 별도의 side table 에 자체 bookkeeping 을 둘 수 있습니다. 컴파일러와 다른 코드는 객체 주소부터 시작하는 헤더와 payload 만 보며, 그 바깥은 수집기 구현의 영역입니다.

## 8.1.5. `payload` (offset 16+)

payload 영역은 `type_id` 가 등록한 필드 레이아웃에 따라 컴파일러가 결정합니다. 자세한 형식은 다음과 같이 나뉩니다.

- [`struct`](../002-types/203-struct.md) / [`enum`](../002-types/204-enum.md) / [`union`](../002-types/205-union.md) 의 경우 컴파일러 자유 레이아웃을 따릅니다.
- `#[repr(C)]` 를 부착한 경우 C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 를 따릅니다.
- `#[repr(packed)]` 를 부착한 경우 정렬을 무시합니다.

GC 의 mark 단계에서는 `type_id → ScanTable` (자세한 형식은 [ScanTable 직렬화 포맷](802-scan-table.md) 에서 정의합니다) 매핑을 통해 자식 포인터의 offset 을 찾아냅니다.

## 8.1.6. 정렬 요구

각 위치별 정렬 요구는 다음과 같습니다.

| 위치                     | 정렬                                |
|--------------------------|-------------------------------------|
| 객체 시작 (헤더)         | 8 바이트                            |
| payload 시작 (offset 16) | 8 바이트 (자동. 헤더가 16 바이트라) |
| 각 필드                  | 자기 타입의 정렬                    |

`payload` 내부 필드의 정렬 요구가 8 바이트보다 큰 경우 (예: `#[repr(align(16))]`) 객체의 시작 위치 또한 그만큼 정렬됩니다.

## 8.1.7. 헤더 + payload 총 크기 계산

헤더와 payload 를 합한 총 크기는 다음 식으로 계산합니다.

```
total_size = 16 + align_up(payload_size, max(8, payload_align))
```

여기서 `align_up(x, a) = (x + a - 1) & ~(a - 1)` 입니다.

이 크기가 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) bump pointer 의 증분량이 됩니다 ([할당](../007-runtime/702-allocation.md) 참고).

## 8.1.8. 예: i32 객체

i32 한 개를 담는 GC 객체의 레이아웃은 다음과 같습니다.

```
offset  0:  type_id = ID_I32     (4 바이트)
offset  4:  gc_flags = 0          (4 바이트)
offset  8:  gc_word = 0           (8 바이트)
offset 16:  i32 value             (4 바이트)
offset 20:  padding               (4 바이트, 8 바이트 정렬용)
total: 24 bytes
```

i32 한 개가 16 바이트 헤더와 4 바이트 값, 그리고 4 바이트 패딩을 합쳐 총 24 바이트가 됩니다.

이러한 오버헤드는 작은 객체에서는 크게 보이지만, 대부분의 객체는 페이로드가 더 커서 (composite 구조) 그 비율이 점차 줄어듭니다.

## 8.1.9. 예: 큰 객체 (List<i64> 의 buffer)

페이로드가 16 KB 인 list buffer 의 예시는 다음과 같습니다.

```
offset    0:  type_id = ID_BUFFER_I64  (4 바이트)
offset    4:  gc_flags = LARGE          (4 바이트)
offset    8:  gc_word = 16400            (총 크기, 8 바이트)
offset   16:  i64[2048]                 (16384 바이트)
total: 16 + 16384 = 16400 bytes
```

`LARGE` 비트가 켜져 있으므로 LOS 객체입니다. `gc_word` 는 총 크기입니다 (8.1.4).

## 8.1.10. 헤더 변경 시의 호환성

본 헤더 레이아웃은 ABI 의 가장 핵심적인 부분에 해당합니다. 변경 시에는 다음 규칙을 따릅니다.

- minor / patch 버전에서는 변경할 수 없습니다 (호환성을 깨뜨리기 때문입니다).
- major 버전에서는 변경이 가능하며, 이 경우 ABI 버전 해시가 함께 변환됩니다.

이러한 안정성 덕분에 같은 major 버전을 공유하는 사용자 패키지, 표준 라이브러리, 런타임이 모두 호환되어 동작합니다.
