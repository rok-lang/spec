# 8.1. 객체 헤더 ABI

[객체 레이아웃과 메모리 공간](../007-runtime/701-object-layout.md)에서 정한 16바이트 헤더의 비트 배치를 정의합니다. 헤더 크기, `gc_flags` 비트, LOS 임계값, type id 시작 값, 런타임 버퍼 type id 같은 GC 상수는 정의 하나(`gc_abi`)에서 옵니다. 컴파일러와 런타임이 이 정의를 함께 씁니다.

## 8.1.1. 정확한 레이아웃

헤더의 형태는 다음과 같습니다.

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
> 헤더의 총 크기는 16바이트이고 8바이트로 정렬됩니다. 32비트 타깃과 64비트 타깃에서 똑같습니다.

모든 객체는 최소 24바이트입니다(헤더와 페이로드 word 하나, `gc_abi`의 `MIN_OBJECT_SIZE`). 그래서 컴파일된 코드는 참조를 쥔 객체라면 어느 것이든 첫 페이로드 word를 읽어도 됩니다.

## 8.1.2. `type_id`(오프셋 0, u32, 리틀 엔디언)

`type_id`는 타입의 고유 식별자입니다. 32비트 부호 없는 값이며 little-endian으로 저장합니다. 값 `0`은 예약된 값으로, null 타입이나 잘못된 객체를 나타냅니다. 컴파일러는 프로세스 시작 시 `rok_gc_register_type`을 호출해 각 type_id를 등록합니다.

### type_id의 할당

한 빌드 안에서는 같은 타입이 같은 id를 받습니다. 빌드가 달라지면 id도 달라질 수 있으므로 직렬화에 쓰지 않습니다.

### 예약 범위

예약된 id 범위는 다음과 같습니다.

| 범위           | 용도                        |
|----------------|-----------------------------|
| `0`            | invalid / null              |
| `1`            | 런타임 (GC 문자열, traced 아님) |
| `2`            | 런타임 (word-traced buffer) |
| `3`            | 런타임 (weak cell, `Weak<T>`) |
| `4` ~ `0xFFFF` | 표준 라이브러리 / 런타임 (예약) |
| `0x10000` ~    | 사용자 / 다른 패키지 타입   |

표준 라이브러리 타입의 정확한 id 매핑은 표준 라이브러리 사양에서 정의합니다. 컴파일러가 사용자 타입에 주는 id는 `0x10000`부터 시작합니다.

## 8.1.3. `gc_flags`(오프셋 4, u32)

`gc_flags` 필드에는 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 상태 비트가 다음 위치에 놓입니다.

```
bit  0  : MARK       (현재 GC 사이클 mark)
bit  1  : STICKY     (sticky mark-bit, old generation)
bit  2  : PINNED     (FFI 이동 금지)
bit  3  : FORWARDED  (evacuate 후. gc_word 가 새 위치)
bit  4  : LARGE      (LOS 객체. gc_word 가 size)
bit  5  : IMMORTAL   (수집 안 함)
bit  6 ~ 31 : 예약 / GC 구현 결정
```

구현 예약 범위에서는 다음 비트를 씁니다([7.4.2](../007-runtime/704-barriers-and-roots.md#742-write-barrier-incremental-update)).

| 비트 | 이름       | 의미                                                                                       |
|------|------------|--------------------------------------------------------------------------------------------|
| 6    | `UNLOGGED` | 수집에서 살아남은 객체에 `STICKY`와 함께 켬. write barrier가 객체를 remembered set에 처음 기록할 때 끔    |
| 7    | `CARDED`   | 8 KiB 이상인 old 객체에 `UNLOGGED` 대신 켬. barrier가 512 B card를 dirty로 표시. card table 포인터는 헤더 바로 앞 word    |
| 8    | `NOMOVE`   | 주소를 수로 쓰는 객체. 수명과 상관없이 절대 옮기지 않음. `rok_task_key` ([8.5](805-extern-symbols.md))가 켬  |
| 9    | `NOEVAC`   | 수집기 내부용이며 full 수집 한 번 동안만 씀. 모호한 word(root, 또는 런타임 word 버퍼(type id `2`)의 word)가 객체를 가리키므로 그 수집의 evacuation은 객체를 제자리에 둠([7.3.1](../007-runtime/703-gc-algorithm.md#opportunistic-evacuation)). evacuation 전에 후보 block의 그런 객체에 켜고, evacuation이 그 객체를 지날 때 끔      |
| 10   | `SATB`     | 동시 mark 중 barrier가 이 객체의 참조를 기록하는 동안 켬. 경쟁하는 store는 비트가 꺼질 때까지 기다림([7.5.3](../007-runtime/705-concurrent-collection.md#753-satb-배리어))   |
| 11   | `SCANNED`  | 동시 mark에서 이 객체를 훑었음  |
| 12   | `WIPE`     | 비밀 버퍼(`rok_gc_secret_new`, [8.5.1](805-extern-symbols.md#rok_gc_secret_new--rok_gc_secret_wipe)). 이 버퍼를 해제하는 sweep은 먼저 페이로드를 0으로 지움. 이 비트는 끄지 않음    |

### 원자성 요구

STW 수집에서 `MARK` / `STICKY` / `FORWARDED`는 atomic이 아닙니다. 동시 mark 중에는 marker와 mutator가 `MARK` / `SATB` / `SCANNED`를 함께 다루므로 이 비트는 atomic입니다. mutator끼리 경쟁하는 `UNLOGGED` 해제도 atomic입니다([7.4.2](../007-runtime/704-barriers-and-roots.md#742-write-barrier-incremental-update)).

## 8.1.4. `gc_word`(오프셋 8, u64 또는 ptr)

`gc_word`는 객체의 총 크기(16바이트 헤더 포함, 8의 배수)입니다. Immix 객체와 LOS 객체 모두에 적용됩니다. 할당 fast path가 이 값을 인라인으로 씁니다([8.9.9](809-lowering-control.md#899-gc-할당-fast--slow-path)).

옮겨진 객체의 `FORWARDED` 비트가 켜지고 `gc_word`가 새 주소를 가리키는 것은 full 수집의 evacuation 중뿐입니다. 수집기는 같은 수집 안에서 모든 참조를 고치고([7.3.1](../007-runtime/703-gc-algorithm.md#731-immix-mark-region)) 옛 자리를 버립니다. 그래서 mutator는 forwarding을 보지 않습니다.

block 안의 객체는 `type_id`가 0이 아닙니다. 수집기는 0을 빈 자리 표시로 씁니다.

### 수집기 전용 정보

수집기는 헤더 앞(객체 주소보다 낮은 주소)이나 별도의 side table에 자체 관리 정보를 둘 수 있습니다. 컴파일러와 다른 코드는 객체 주소에서 시작하는 헤더와 페이로드만 봅니다. 그 바깥은 수집기 구현의 영역입니다.

## 8.1.5. `payload`(오프셋 16+)

페이로드 영역은 `type_id`에 등록한 필드 레이아웃에 따라 컴파일러가 정합니다. 형식은 다음과 같습니다.

- [`struct`](../002-types/203-struct.md) / [`enum`](../002-types/204-enum.md) / [`union`](../002-types/205-union.md)은 레이아웃을 컴파일러가 자유롭게 정합니다.
- `#[repr(C)]`를 붙이면 C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface)를 따릅니다.
- `#[repr(packed)]`를 붙이면 정렬을 무시합니다.

GC의 mark 단계는 `type_id → ScanTable` 매핑으로 자식 포인터의 오프셋을 찾습니다. 매핑 형식은 [ScanTable 직렬화 포맷](802-scan-table.md)에서 정의합니다.

## 8.1.6. 정렬 요구

위치별 정렬 요구는 다음과 같습니다.

| 위치                     | 정렬                                |
|--------------------------|-------------------------------------|
| 객체 시작 (헤더)         | 8바이트                            |
| payload 시작 (offset 16) | 8바이트 (헤더가 16바이트이므로 자동) |
| 각 필드                  | 자기 타입의 정렬                    |

`payload` 안 필드의 정렬 요구가 8바이트보다 크면(예: `#[repr(align(16))]`) 페이로드의 시작을 그만큼 정렬합니다. 헤더는 정렬된 주소의 16바이트 앞에 놓입니다. 그 앞의 틈은 수집기가 씁니다([8.1.4](#814-gc_word오프셋-8-u64-또는-ptr)).

## 8.1.7. 헤더 + payload 총 크기 계산

헤더와 payload를 합한 총 크기는 다음 식으로 계산합니다.

```
total_size = 16 + align_up(payload_size, max(8, payload_align))
```

여기서 `align_up(x, a) = (x + a - 1) & ~(a - 1)`입니다.

이 크기가 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) bump pointer의 증가량입니다([할당](../007-runtime/702-allocation.md) 참고).

## 8.1.8. 예: i32 객체

i32 하나를 담는 GC 객체의 레이아웃은 다음과 같습니다.

```
offset  0:  type_id = ID_I32     (4 바이트)
offset  4:  gc_flags = 0          (4 바이트)
offset  8:  gc_word = 24          (총 크기, 8 바이트)
offset 16:  i32 value             (4 바이트)
offset 20:  padding               (4 바이트, 8 바이트 정렬용)
total: 24 bytes
```

i32 하나가 16바이트 헤더, 4바이트 값, 4바이트 패딩을 합쳐 총 24바이트를 차지합니다.

작은 객체에서는 이 오버헤드가 커 보입니다. 하지만 대부분의 객체는 페이로드가 더 크므로(복합 구조) 비율이 줄어듭니다.

## 8.1.9. 예: 큰 객체(List<i64>의 버퍼)

페이로드가 16 KB인 리스트 버퍼의 예입니다.

```
offset    0:  type_id = ID_BUFFER_I64  (4 바이트)
offset    4:  gc_flags = LARGE          (4 바이트)
offset    8:  gc_word = 16400            (총 크기, 8 바이트)
offset   16:  i64[2048]                 (16384 바이트)
total: 16 + 16384 = 16400 bytes
```

`LARGE` 비트가 켜져 있으므로 LOS 객체입니다. `gc_word`는 총 크기입니다(8.1.4).

## 8.1.10. 헤더 변경 시의 호환성

이 헤더 레이아웃은 ABI의 가장 중심입니다. 바꿀 때는 다음 규칙을 따릅니다.

- minor / patch 버전에서는 바꾸면 안 됩니다. 바꾸면 호환성이 깨집니다.
- major 버전에서는 바꿀 수 있습니다. 이때 ABI 버전 해시도 함께 바뀝니다.

이 안정성 덕분에 같은 major 버전을 쓰는 사용자 패키지, 표준 라이브러리, 런타임은 모두 서로 호환됩니다.
