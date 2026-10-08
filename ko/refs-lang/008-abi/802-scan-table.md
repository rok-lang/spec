# 8.2. ScanTable 직렬화 포맷

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 의 mark 단계는 각 객체의 자식 포인터들을 따라가야 한다. 어떤 offset 에 자식 포인터가
있는지를 알려주는 자료가 ScanTable. 본 문서는 그 직렬화 포맷을 정의한다.

## 8.2.1. 목적

```
GC mark loop:
  for each live object O:
    type_id = O.type_id
    scan_table = lookup(type_id)
    for each (offset, kind) in scan_table:
      child = *(O + offset)
      mark(child)
```

`lookup(type_id)` 는 컴파일러가 [객체 헤더 ABI](801-object-header-abi.md) 의 `nok_gc_register_type` 으로 프로세스 시작 시
등록한 결과를 사용한다.

## 8.2.2. 등록 호출

```
extern "C" {
    fn nok_gc_register_type(type_id: u32, scan_table: ConstPtr<u8>);
}
```

- `type_id`: 등록할 타입의 ID.
- `scan_table`: 직렬화된 ScanTable 의 시작 포인터.

ScanTable 의 메모리는 컴파일러가 .rodata 에 emit 한다. 프로세스 수명 동안 유효. 런타임은 읽기만
한다.

## 8.2.3. 직렬화 포맷

```
ScanTable Header (8 bytes, 4-byte aligned):
  offset  0:  total_size       u32         // 객체의 총 크기 (16 + payload)
  offset  4:  version          u16  = 1
  offset  6:  num_entries      u16         // 자식 포인터 entry 수

Entry array (8 bytes each):
  offset  8 + 8*i + 0:  field_offset  u32  // 객체 내 offset
  offset  8 + 8*i + 4:  field_kind    u32  // 자식 포인터 종류
```

총 크기 = 8 + 8 × num_entries. 모든 필드는 자연 정렬.

### Header

| 필드          | 타입 | 의미                                             |
| ------------- | ---- | ------------------------------------------------ |
| `version`     | u16  | 포맷 버전. 현재 = 1                              |
| `total_size`  | u32  | 객체의 총 크기 (16 바이트 헤더 + 정렬된 payload) |
| `num_entries` | u16  | 자식 포인터 entry 의 개수                        |

### Entry

각 entry 는 객체 안의 한 자식 포인터 슬롯을 가리킨다.

| 필드           | 타입 | 의미                             |
| -------------- | ---- | -------------------------------- |
| `field_offset` | u32  | 그 슬롯의 offset (객체 시작부터) |
| `field_kind`   | u32  | 슬롯의 종류                      |

### `field_kind` 종류

| 값  | 의미                                                             |
| --- | ---------------------------------------------------------------- |
| 0   | `STRONG_REF`: 일반 GC 객체 포인터 (mark 따라감)                  |
| 1   | `WEAK_REF`: 약한 참조 (mark 안 따라감, GC 수집 시 정리)          |
| 2   | `INTERIOR_REF`: 객체 내부를 가리킴 (forwarding 시 보정)          |
| 3   | `ANY_SLOT`: `tag + payload` 형태의 동적 슬롯. tag 검사 후 mark   |
| 4   | `SUM_PAYLOAD`: 합 타입의 payload. tag 가 어딘지 메타와 결합 필요 |
| 5 ~ | 예약 / 추가 종류                                                 |

각 종류가 어떻게 mark 되는지는 GC 구현의 처리. 본 사양은 어떤 종류가 있는지와
인코딩만 정의한다.

## 8.2.4. 인코딩 예: `struct Point { x: f64; y: f64; }`

자식 포인터가 없다 (필드 모두 원시 부동소수).

```
ScanTable {
  version      = 1
  total_size   = 32        // 16 헤더 + 8 + 8
  num_entries  = 0
}
```

8 바이트 (header only).

## 8.2.5. 인코딩 예: `struct Pair { a: List<i32>; b: List<f64>; }`

두 자식 포인터.

```
ScanTable {
  version      = 1
  total_size   = 32        // 16 헤더 + 8 (List ptr) + 8
  num_entries  = 2

  entry[0]: field_offset = 16, field_kind = STRONG_REF (0)
  entry[1]: field_offset = 24, field_kind = STRONG_REF (0)
}
```

8 + 16 = 24 바이트.

## 8.2.6. 인코딩 예: `enum` 변형

```
enum Result {
    Ok(i32),
    Err(str),
    Pending,
}
```

Tag (4 바이트) + 패딩 (4 바이트) + 가장 큰 변형 payload (8 바이트, str 포인터). 총 32 바이트
객체.

ScanTable 은 변형 별로 다르게 처리해야 한다. `field_kind` 의 `SUM_PAYLOAD` 사용.

```
ScanTable {
  version      = 1
  total_size   = 32        // 16 헤더 + 4 tag + 4 padding + 8 payload (max)
  num_entries  = 1

  entry[0]: field_offset = 16, field_kind = SUM_PAYLOAD (4)
                                            // tag 는 16, payload 는 24
}
```

`SUM_PAYLOAD` 는 추가 메타가 필요하다. tag 의 위치와, 각 tag 에 대해 어떤 자식이 있는지.
이 추가 메타는 entry 뒤에 연속된 보조 영역에 둘 수 있다 (자세한 형식은 GC 구현 결정).

본 사양은 `SUM_PAYLOAD` kind 가 추가 정보를 요구한다는 사실까지 정의한다.

## 8.2.7. ANY 슬롯

`any` 슬롯 ([`any` 슬롯](../002-types/202-any-slot.md)) 은 `tag + payload` 형태로 어떤 타입이든 담는다.

```
ANY 슬롯 레이아웃:
  offset  0:  tag       u32
  offset  8:  payload   8 바이트 (작은 값 직접, 큰 값은 ptr)
```

ScanTable entry:

```
field_offset = (그 슬롯의 offset)
field_kind = ANY_SLOT (3)
```

GC 가 mark 시 tag 를 보고 payload 의 의미를 결정한다. 작은 원시 타입이면 무시, 포인터형이면
mark.

## 8.2.8. WEAK_REF

약한 참조는 살아있게 만드는 효과가 없다.

- mark 단계: weak `ref` 는 따라가지 않는다.
- sweep 후: weak `ref` 가 가리키던 객체가 수집되었으면 weak `ref` 슬롯을 0 으로 클리어한다.

이 메커니즘은 캐시나 옵저버 패턴 등에 유용하다. 정확한 라이브러리 인터페이스는 표준 라이브러리
사양에서 정한다.

## 8.2.9. INTERIOR_REF

객체 내부의 (시작이 아닌) 위치를 가리킨다.

- 슬라이스나 부분 view 등에서 발생한다.
- forwarding 시 옛 객체와 새 객체의 상대 offset 으로 보정한다.

자세한 처리는 GC 구현의 결정.

## 8.2.10. 직렬화의 endianness

모든 필드는 little-endian. nok 은 little-endian 타깃을 우선한다 ([플랫폼 타깃](../000-meta/006-platform-targets.md) 의 모든
Tier 1 타깃이 little-endian).

big-endian 타깃 지원이 추가되면 그때 직렬화 endianness 를 little 로 고정할지, 호스트
endianness 를 따를지 결정한다.

## 8.2.11. 변경 호환

ScanTable 포맷의 version 필드가 호환을 다룬다.

- 같은 major version 안에서 version 필드만으로 호환.
- 다른 major version 은 별도 .a 가 필요하다.

## 8.2.12. 등록 시점

```
process_start:
  for each registered type:
    nok_gc_register_type(type_id, &scan_table)
  ↓
  user main()
```

컴파일러가 `nok_register_types()` 같은 진입 함수에 모든 등록 호출을 넣고, 런타임 초기화
직후 호출한다.
