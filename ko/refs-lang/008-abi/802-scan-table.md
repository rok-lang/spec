# 8.2. ScanTable 직렬화 포맷

[GC](../000-meta/007-glossary.md#gc-garbage-collector)의 mark 단계는 각 객체의 자식 포인터를 따라가야 합니다. 그러려면 자식 포인터가 어느 오프셋에 있는지 알려 주는 자료가 필요합니다. 이 자료를 ScanTable이라고 부릅니다. 이 문서는 ScanTable의 직렬화 포맷을 정의합니다.

## 8.2.1. 목적

GC의 mark 루프는 다음과 같이 동작합니다.

```
GC mark loop:
  for each live object O:
    type_id = O.type_id
    scan_table = lookup(type_id)
    for each (offset, kind) in scan_table:
      child = *(O + offset)
      mark(child)
```

`lookup(type_id)`는 컴파일러가 프로세스 시작 시 [객체 헤더 ABI](801-object-header-abi.md)의 `rok_gc_register_type`으로 등록해 둔 결과를 씁니다.

스택(shadow stack)과 `static` 루트도 같은 바이트 오프셋 기반 walk로 훑습니다. 루트 슬롯 하나는 ScanTable entry 하나처럼 다룹니다. `any` 루트는 페이로드 word까지 덮습니다.

## 8.2.2. 등록 호출

등록은 다음 외부 함수를 호출해서 합니다.

```
extern "C" {
    fn rok_gc_register_type(type_id: u32, scan_table: ConstPtr<u8>);
}
```

- `type_id`: 등록할 타입의 ID입니다.
- `scan_table`: 직렬화된 ScanTable의 시작 포인터입니다.

컴파일러는 ScanTable 메모리를 .rodata에 내보냅니다. 그래서 이 메모리는 프로세스 수명 동안 유효합니다. 런타임은 이 영역을 읽기만 합니다.

## 8.2.3. 직렬화 포맷

ScanTable은 8바이트 헤더 뒤에 entry 배열이 이어지는 구조입니다.

```
ScanTable Header (8 bytes, 4-byte aligned):
  offset  0:  total_size       u32         // 객체의 총 크기 (16 + payload)
  offset  4:  version          u16  = 1
  offset  6:  num_entries      u16         // 자식 포인터 entry 수

Entry array (8 bytes each):
  offset  8 + 8*i + 0:  field_offset  u32  // 객체 내 offset
  offset  8 + 8*i + 4:  field_kind    u32  // 자식 포인터 종류
```

전체 크기는 8 + 8 × num_entries이며, 모든 필드는 자연 정렬을 따릅니다.

### 헤더

헤더의 필드는 다음과 같습니다.

| 필드          | 타입 | 의미                                             |
|---------------|------|--------------------------------------------------|
| `version`     | u16  | 포맷 버전 (bit 0 ~ 14, 현재 = 1). bit 15 = 배열 table |
| `total_size`  | u32  | 객체의 총 크기 (16바이트 헤더 + 정렬된 payload) |
| `num_entries` | u16  | 자식 포인터 entry의 개수                         |

`version`의 bit 15가 1이면 **배열 table**입니다. 이때 `total_size`는 원소 stride이고, entry의 `field_offset`은 원소 시작이 기준입니다. entry는 페이로드 전체에 걸쳐 원소마다 되풀이해 적용합니다. 컴파일러는 GC 버퍼의 원소 table을 이 형태로 씁니다.

### 엔트리

각 entry는 객체 안의 자식 포인터 슬롯 하나를 가리킵니다.

| 필드           | 타입 | 의미                             |
|----------------|------|----------------------------------|
| `field_offset` | u32  | 그 슬롯의 offset (객체 시작부터) |
| `field_kind`   | u32  | 슬롯의 종류                      |

### `field_kind` 종류

`field_kind`에 올 수 있는 값과 의미는 다음과 같습니다.

| 값  | 의미                                                             |
|-----|------------------------------------------------------------------|
| 0   | `STRONG_REF`: 일반 GC 객체 포인터 (mark 따라감)                  |
| 1   | `WEAK_REF`: 약한 참조 (mark 안 따라감, GC 수집 시 정리)          |
| 2   | `INTERIOR_REF`: 객체 내부를 가리킴 (forwarding 시 보정)          |
| 3   | `ANY_SLOT`: `tag + payload` 형태의 동적 슬롯. tag 검사 후 mark   |
| 4   | `SUM_PAYLOAD`: 합 타입의 tag와 변형별 참조 (8.2.6). bit 8..31은 descriptor 위치   |
| 5   | `AMBIGUOUS_WORD`: 참조일 수도 아닐 수도 있는 word. 가리키는 객체를 살리되 옮기지 않음 (type id `2` buffer, untagged `union`) |
| 6 ~ | 예약 / 추가 종류                                                 |

각 종류를 실제로 어떻게 표시(mark)하는지는 GC 구현이 정합니다. 이 사양은 어떤 종류가 있는지와 그 인코딩만 정의합니다.

## 8.2.4. 인코딩 예: `struct Point { x: f64; y: f64; }`

이 struct에는 자식 포인터가 없습니다(모든 필드가 원시 부동소수점 타입).

```
ScanTable {
  version      = 1
  total_size   = 32        // 16 헤더 + 8 + 8
  num_entries  = 0
}
```

따라서 ScanTable은 헤더만 있는 8바이트입니다.

## 8.2.5. 인코딩 예: `struct Pair { a: List<i32>; b: List<f64>; }`

이 struct에는 자식 포인터가 두 개 있습니다.

```
ScanTable {
  version      = 1
  total_size   = 32        // 16 헤더 + 8 (List ptr) + 8
  num_entries  = 2

  entry[0]: field_offset = 16, field_kind = STRONG_REF (0)
  entry[1]: field_offset = 24, field_kind = STRONG_REF (0)
}
```

전체 크기는 8 + 16 = 24바이트입니다.

## 8.2.6. 인코딩 예: `enum` 변형

```
enum Result {
    Ok(i32),
    Err(str),
    Pending,
}
```

이 타입의 객체는 tag(4바이트), 패딩(4바이트), 가장 큰 변형의 페이로드(8바이트, str 포인터)로 이루어집니다. 총 32바이트입니다.

변형마다 참조의 위치가 다르므로 `SUM_PAYLOAD` entry 하나와 그 descriptor를 씁니다.

- entry의 `field_offset`은 tag(u32)의 오프셋입니다.
- `field_kind`의 하위 바이트는 `4`이고, bit 8..31은 descriptor의 위치(table 시작부터 u32 word 단위)입니다.
- descriptor는 `n` 뒤에, 참조가 있는 변형마다 `{tag, num_entries, entries_index}`를 둡니다.
- 변형의 entry는 `{tag 로부터의 offset, field_kind}` 형태입니다. 중첩된 합 타입은 다시 `SUM_PAYLOAD`로 씁니다.
- descriptor에 tag가 없는 변형에는 참조가 없습니다.

```
ScanTable {
  version = 1, total_size = 40, num_entries = 1
  entry[0]: field_offset = 16, field_kind = 4 | (desc << 8)    // tag 는 offset 16
  desc:     n = 1
            { tag = 1 (Err), num_entries = 1, entries_index = e }
  e:        { offset = 8, field_kind = STRONG_REF }             // Err 의 str ptr
}
```

enum / 합 타입 페이로드의 참조는 정밀합니다. 그래서 페이로드를 거쳐 닿은 객체도 evacuation으로 옮겨질 수 있습니다. `AMBIGUOUS_WORD`는 type id `2` 버퍼와, tag가 없어 해석할 수 없는 사용자 `union`의 저장소에만 씁니다.

## 8.2.7. ANY 슬롯

`any` 슬롯([`any` 슬롯](../002-types/202-any-slot.md))은 `tag + payload` 형태로 어떤 타입의 값이든 담을 수 있습니다.

```
ANY 슬롯 레이아웃.
  offset  0:            tag       u32
  offset  ptr 정렬:     payload   usize (포인터 폭 이하 숫자·bool·char 는 직접, 그 밖은 box ptr)
```

이에 대응하는 ScanTable entry는 다음과 같습니다.

```
field_offset = (그 슬롯의 offset)
field_kind = ANY_SLOT (3)
```

페이로드가 box 참조인지는 tag가 알려 줍니다.

- tag의 **bit 31**이 1이면 페이로드는 box 참조이고, 0이면 값 자체입니다.
- tag로 타입을 비교할 때는 bit 31을 뺀 값으로 비교합니다.

그래서 GC는 별도의 표 없이 페이로드를 정밀하게 훑고, 옮긴 box의 주소를 고쳐 쓸 수 있습니다. 따로 컴파일한 패키지끼리 표를 합치지 않아도 됩니다.

## 8.2.8. WEAK_REF

약한 참조는 객체를 살려 두지 않습니다.

- mark 단계는 weak `ref`를 따라가지 않습니다.
- sweep 뒤에 weak `ref`가 가리키던 객체가 수집되었으면 런타임이 weak `ref` 슬롯을 0으로 지웁니다.

이 메커니즘은 캐시나 옵저버 패턴 등에 쓸모가 있습니다. 라이브러리 인터페이스는 `Weak<T>`입니다([stdlib 1.6.9](../../refs-standard-libraries/100-bare/106-gc-pin.md#169-weakt-약한-참조)).

## 8.2.9. INTERIOR_REF

INTERIOR_REF는 객체 안에서 시작이 아닌 위치를 가리킵니다.

- 슬라이스나 부분 뷰 등에서 생깁니다.
- forwarding 때는 옛 객체와 새 객체 사이의 상대 오프셋으로 보정합니다.

정확한 처리 방식은 GC 구현이 정합니다.

## 8.2.10. 직렬화의 엔디언

모든 필드는 little-endian입니다. Rok은 little-endian 타깃을 우선하고, 직렬화 포맷도 이에 맞춥니다. [플랫폼 타깃](../000-meta/006-platform-targets.md)의 Tier 1 타깃은 모두 little-endian입니다.

big-endian 타깃 지원을 추가할 때 직렬화 endianness를 little로 고정할지, 호스트 endianness를 따를지 정합니다.

## 8.2.11. 변경 호환

ScanTable 포맷의 호환성은 version 필드로 다룹니다.

- 같은 major version 안에서는 version 필드만으로 호환성을 처리합니다.
- major version이 다르면 별도의 .a가 필요합니다.

## 8.2.12. 런타임 예약 type id

다음 세 type id는 런타임이 직접 정의합니다. 컴파일러는 이 id의 ScanTable을 등록하지 않습니다([8.1.2](801-object-header-abi.md#812-type_id오프셋-0-u32-리틀-엔디언)).

| type id | 의미              | scan                                                    |
|---------|-------------------|---------------------------------------------------------|
| `1`     | GC 문자열         | payload는 바이트이며 훑지 않음                     |
| `2`     | word-traced buffer | payload의 모든 word를 모호한 word(`AMBIGUOUS_WORD`)로 훑음    |
| `3`     | weak cell (`Weak<T>`) | `WEAK_REF` word 하나 (8.2.8), `rok_gc_weak_new`로 만듦  |

id `1`과 `2`는 type id `1` 또는 `2`를 넘겨 `rok_gc_alloc_slow`를 호출해 할당합니다([8.5.1](805-extern-symbols.md#851-gc-영역)). 표준 라이브러리의 [`GcBuf<T>`](../../refs-standard-libraries/200-core/209-collections.md#2912-gcbuft-원시-gc-버퍼)가 이 위에 놓입니다.

## 8.2.13. 등록 시점

등록은 프로세스가 시작될 때 한꺼번에 합니다.

```
process_start:
  for each registered type:
    rok_gc_register_type(type_id, &scan_table)
  ↓
  user main()
```

컴파일러는 모든 등록 호출을 모은 함수 `rok.register_types`를 내보냅니다. 컴파일러가 생성하는 C-ABI `main`은 `rok_runtime_init` 직후 이 함수를 호출하고 사용자 진입점으로 들어갑니다.

```
main(argc, argv):                // 컴파일러 생성, C ABI
  rok_runtime_init()
  rok.register_types()           // rok_gc_register_type 호출 모음
  → 사용자 진입점 (4.16)
```

GC 객체 할당의 slow path는 `rok_gc_alloc_slow(size, align, type_id)`입니다([7.2.3](../007-runtime/702-allocation.md#723-slow-path-rok_gc_alloc_slow)). 컴파일러가 주는 type id는 `0x10000`부터 시작합니다([8.1.2](801-object-header-abi.md#812-type_id오프셋-0-u32-리틀-엔디언)).

표의 바이트는 프로그램에 한 번만 들어갑니다. 각 패키지의 `rok.register_types`는 표의 내용으로 이름을 붙인 숨은 정의를 가리킵니다. 여러 패키지가 같은 레이아웃을 각자의 id로 등록해도 링크 결과에는 그 정의가 한 번만 남습니다.
