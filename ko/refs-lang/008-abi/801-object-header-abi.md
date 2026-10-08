# 8.1. 객체 헤더 ABI

[객체 레이아웃과 메모리 공간](../007-runtime/701-object-layout.md) 의 16 바이트 헤더의 정확한 비트 배치를 정의한다.
컴파일러와 런타임 양쪽이 이 배치를 정확히 따라야 한다.

## 8.1.1. 정확한 레이아웃

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
> 총 헤더 크기는 16 바이트, 8 바이트 정렬. 32-bit / 64-bit 타깃 모두 동일.

## 8.1.2. `type_id` (offset 0, u32, little-endian)

타입의 고유 식별자.

- 32 비트 unsigned, little-endian 으로 저장.
- 값 `0` 은 예약. null type / 잘못된 객체 표시.
- 컴파일러는 프로세스 시작 시 `nok_gc_register_type` 으로 각 type_id 를 등록한다.

### type_id 의 할당

- 컴파일러가 각 타입에 안정된 id 를 부여한다. 같은 빌드의 같은 타입은 항상 같은 id.
- 다른 빌드 사이의 id 안정성은 보장되지 않는다. 직렬화에 사용 금지.

### 예약 범위

| 범위           | 용도                        |
| -------------- | --------------------------- |
| `0`            | invalid / null              |
| `1` ~ `0xFFFF` | 표준 라이브러리 타입 (예약) |
| `0x10000` ~    | 사용자 / 다른 패키지 타입   |

표준 라이브러리 타입의 정확한 id 매핑은 표준 라이브러리 사양에서 정의한다.

## 8.1.3. `gc_flags` (offset 4, u32)

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 상태 비트의 정확한 비트 위치.

```
bit  0  : MARK       (현재 GC 사이클 mark)
bit  1  : STICKY     (sticky mark-bit, old generation)
bit  2  : PINNED     (FFI 이동 금지)
bit  3  : FORWARDED  (evacuate 후. gc_word 가 새 위치)
bit  4  : LARGE      (LOS 객체. gc_word 가 size)
bit  5  : IMMORTAL   (수집 안 함)
bit  6 ~ 31 : 예약 / GC 구현 결정
```

비트 검사와 설정은 atomic 연산이 필요한 경우 `u32` atomic 으로 한다 (mark / sweep 동시 수집 시).

### atomic 요구

| 연산                | atomic 필요?                                           |
| ------------------- | ------------------------------------------------------ |
| `MARK` set / read   | yes. concurrent mark 중                                |
| `STICKY` set / read | sticky 비트 갱신은 mark 후 STW 단계에서. non-atomic OK |
| `FORWARDED` set     | yes. concurrent evacuate 중                            |
| 다른 비트           | 보통 non-atomic OK                                     |

자세한 메모리 모델은 GC 구현의 결정. 본 사양은 어떤 비트가 어디에 있는지만 정의한다.

## 8.1.4. `gc_word` (offset 8, u64 또는 ptr)

상황에 따라 두 가지 의미를 갖는다.

| `LARGE` 비트                                                       | `gc_word` 의미                          |
| ------------------------------------------------------------------ | --------------------------------------- |
| 0 ([Immix](../000-meta/007-glossary.md#immix) 객체)                | forwarding 포인터 (or 0 / self-pointer) |
| 1 ([LOS](../000-meta/007-glossary.md#los-large-object-space) 객체) | 객체 크기 (페이로드 바이트 수)          |

### Immix 객체의 forwarding

- evacuate 안 됨: `FORWARDED` 비트 0, `gc_word` = 0 (또는 self-pointer, 구현 결정).
- evacuate 됨: `FORWARDED` 비트 1, `gc_word` = 새 위치의 포인터 (8 바이트 정렬).

mutator 가 객체에 접근할 때:

```
load forwarded_bit
if forwarded:
    load new_ptr from gc_word
    redirect access
else:
    access here
```

### LOS 객체의 size

- `gc_word` = 객체의 payload 크기 (바이트). 헤더는 포함하지 않는다.
- 64 비트 unsigned, little-endian.
- 객체의 총 크기 = 16 (헤더) + gc_word.

## 8.1.5. `payload` (offset 16+)

`type_id` 가 등록한 필드 레이아웃에 따라 컴파일러가 결정한다. 자세한 형식은:

- [`struct`](../002-types/203-struct.md) / [`enum`](../002-types/204-enum.md) / [`union`](../002-types/205-union.md) 의 컴파일러 자유 레이아웃.
- `#[repr(C)]` 부착 시 C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface).
- `#[repr(packed)]` 부착 시 정렬 무시.

GC 의 mark 단계는 `type_id → ScanTable` (자세한 형식은 [ScanTable 직렬화 포맷](802-scan-table.md)
에서 정의) 로 자식 포인터의 offset 을 찾는다.

## 8.1.6. 정렬 요구

| 위치                     | 정렬                                |
| ------------------------ | ----------------------------------- |
| 객체 시작 (헤더)         | 8 바이트                            |
| payload 시작 (offset 16) | 8 바이트 (자동. 헤더가 16 바이트라) |
| 각 필드                  | 자기 타입의 정렬                    |

`payload` 내부 필드의 정렬 요구가 8 바이트보다 큰 경우 (예: `#[repr(align(16))]`) 객체
시작이 그만큼 정렬된다.

## 8.1.7. 헤더 + payload 총 크기 계산

```
total_size = 16 + align_up(payload_size, max(8, payload_align))
```

`align_up(x, a) = (x + a - 1) & ~(a - 1)`.

이 크기가 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) bump pointer 의 증분량 ([할당](../007-runtime/702-allocation.md)).

## 8.1.8. 예: i32 객체

i32 한 개를 담는 GC 객체:

```
offset  0:  type_id = ID_I32     (4 바이트)
offset  4:  gc_flags = 0          (4 바이트)
offset  8:  gc_word = 0           (8 바이트)
offset 16:  i32 value             (4 바이트)
offset 20:  padding               (4 바이트, 8 바이트 정렬용)
total: 24 bytes
```

i32 한 개가 16 바이트 헤더 + 4 바이트 + 4 바이트 패딩으로 24 바이트.

이 오버헤드는 작은 객체에서 크게 보이지만, 대부분의 객체는 페이로드가 더 크기 (composite 구조)
때문에 비율이 줄어든다.

## 8.1.9. 예: 큰 객체 (List<i64> 의 buffer)

페이로드 16 KB 짜리 list buffer:

```
offset    0:  type_id = ID_BUFFER_I64  (4 바이트)
offset    4:  gc_flags = LARGE          (4 바이트)
offset    8:  gc_word = 16384            (페이로드 크기, 8 바이트)
offset   16:  i64[2048]                 (16384 바이트)
total: 16 + 16384 = 16400 bytes
```

`LARGE` 비트가 켜져 LOS 공간에 위치하고, `gc_word` 가 페이로드 크기를 담는다.

## 8.1.10. 헤더 변경 시의 호환성

본 헤더 레이아웃은 ABI 의 가장 핵심 부분이다. 변경 시:

- minor / patch: 변경 불가 (호환을 깬다).
- major: 변경 가능, ABI 버전 해시가 바뀐다.

이 안정성으로 같은 major 버전의 사용자 패키지, 표준 라이브러리, 런타임이 모두 호환된다.
