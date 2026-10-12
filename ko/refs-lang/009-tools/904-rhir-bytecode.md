# 9.4. RHIR bytecode 포맷

[패키지 아카이브](903-package-archive.md) 의 `.rok.hir` 섹션에 들어가는, 제네릭 함수의 rok-hir 직렬화 형식인 [RHIR](../000-meta/007-glossary.md#rhir-rok-hir-bytecode) bytecode 를 정의합니다.

## 9.4.1. 동기

[AOT 빌드 파이프라인](902-aot-pipeline.md) 의 단형화는 호출 지점의 구체 타입을 기준으로 [specialize](../000-meta/007-glossary.md#specialize) 합니다. 따라서 다른 패키지가 한 패키지의 [generic](../000-meta/007-glossary.md#generic--제네릭) `fn` 을 호출하면 그 호출 지점에서 specialize 가 일어나야 합니다.

이때 호출되는 패키지의 generic `fn` 에 대한 [HIR](../000-meta/007-glossary.md#hir-high-level-ir--mir-mid-level-ir) 이 필요한데, [concrete fn](../000-meta/007-glossary.md#concrete-fn--generic-fn) 의 obj 만으로는 부족합니다.

해결책은 [archive](../000-meta/007-glossary.md#archive) 안에 generic `fn` 의 HIR 을 직렬화해 함께 담는 것이고, 다운스트림은 이를 읽어 specialize 합니다.

## 9.4.2. 외부 라이브러리 미사용

의존성 0 원칙에 따라 직렬화 라이브러리도 사용하지 않으며, RHIR 은 직접 설계합니다. 수백 줄 수준의 단순한 형식만으로도 충분합니다.

## 9.4.3. 형식 영감

MLIR bytecode 포맷의 구조를 차용합니다.

- magic + version: 버전 호환 검사.
- string section: 식별자 중복 제거.
- type section: 타입 표 (구조적 공유).
- item section: `fn` / `struct` / `enum` 등의 메타.
- body section: 각 `fn` body 의 expr tree (lazy load 가능).
- VarInt 인코딩: LEB128 변형 (PrefixVarInt).

## 9.4.4. 전체 레이아웃

```
RHIR file:
+-----------------------------------+
|  Header                           |  (16 bytes fixed)
|    magic       "RHIR"  (4 bytes)  |
|    version     u32                |
|    flags       u32                |
|    body_offset u32                |
+-----------------------------------+
|  Producer string                  |  (variable)
|    "rok 1.2.3 (abi-hash:abc...)" |
+-----------------------------------+
|  String Section                   |  (identifier / string pool)
+-----------------------------------+
|  Type Section                     |  (type table)
+-----------------------------------+
|  Item Section                     |  (fn / struct / enum metadata)
+-----------------------------------+
|  Body Section                     |  (HIR expr tree of each fn)
+-----------------------------------+
```

> 16 bytes fixed: 16 바이트 고정. variable: 가변. identifier / string pool: 식별자 / 문자열 풀. type table: 타입 표. metadata: 메타. HIR expr tree of each fn: 각 fn 의 HIR expr tree.

## 9.4.5. Header

```
+--------+---------+---------+
| offset |  size   | field   |
+--------+---------+---------+
|   0    |    4    | magic   |     "RHIR"
|   4    |    4    | version |     u32 - format version
|   8    |    4    | flags   |     u32 - option flags
|  12    |    4    | body_offset | u32 - start offset of body section
+--------+---------+---------+
```

> format version: 포맷 버전. option flags: 옵션 플래그. start offset of body section: body section 의 시작 offset.

`magic` 으로 파일이 RHIR 인지 빠르게 확인하며, 잘못된 파일은 즉시 reject 합니다.

`body_offset` 은 lazy load 를 위해 body 의 위치를 직접 가리키므로, 메타만 읽고 body 는 필요할 때 로드합니다.

## 9.4.6. Producer String

```
varint(length) | utf8 bytes
```

rok 버전과 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 해시 등을 담아 디버그와 호환 검사에 사용하며, 사용자 친화적인 텍스트입니다.

## 9.4.7. VarInt 인코딩

MLIR bytecode 와 같은 PrefixVarInt 이며 LEB128 의 변형입니다.

```
1-byte int  : 0xxxxxxx                    (0 ~ 127)
2-byte int  : 10xxxxxx xxxxxxxx           (0 ~ 16383)
3-byte int  : 110xxxxx xxxxxxxx xxxxxxxx  (0 ~ 2097151)
...
```

작은 정수는 1 바이트로, 큰 정수는 더 많은 바이트로 인코딩합니다.

대부분의 정수 (offset, count, index) 는 값이 작아 평균 1 ~ 2 바이트로 인코딩됩니다.

## 9.4.8. String Section

식별자와 문자열의 공유 풀입니다.

```
varint(num_strings)
for each string:
    varint(length)
    utf8 bytes
```

다른 섹션은 index 로 참조하며, 같은 식별자가 여러 번 등장해도 한 번만 저장합니다.

## 9.4.9. Type Section

타입 표입니다.

```
varint(num_types)
for each type:
    type_kind (1 byte)         // primitive / struct / enum / sum / fn / ...
    추가 필드 (kind 별)
```

타입은 재귀적 구조이며, 다른 타입을 `index` 로 참조합니다.

| kind | 의미          | 추가 필드                                   |
|------|---------------|---------------------------------------------|
| 0    | primitive     | `prim_id` (i8 ~ f64 등)                     |
| 1    | `struct`      | item_index, generic_args...                 |
| 2    | `enum`        | item_index, generic_args...                 |
| 3    | sum (`A \| B`) | element_count, element_indices...           |
| 4    | tuple         | element_count, element_indices...           |
| 5    | array         | element_index, length                       |
| 6    | `fn`          | param_count, param_indices..., return_index |
| 7    | `any`         | (없음)                                      |
| 8    | generic_param | param_id                                    |
| ...  | ...           | ...                                         |

## 9.4.10. Item Section

함수, 타입, 트레잇, `impl` 등의 항목 메타입니다.

```
varint(num_items)
for each item:
    item_kind (1 byte)            // fn / struct / enum / trait / impl / ...
    name_index (varint)
    visibility (1 byte)
    generics_info (varint)
    signature_info (varint)
    body_offset (u32)              // body section 안의 시작 위치 (fn 의 경우)
    추가 필드 (kind 별)
```

각 항목의 body 는 별도의 body section 에 두고 lazy load 합니다.

## 9.4.11. Body Section

각 `fn` 의 HIR expression tree 직렬화입니다.

```
for each fn body:
    // body_offset 으로 점프해서 시작
    varint(num_locals)
    serialize_expr_tree(...)
```

expression tree 의 직렬화는 재귀적 노드와 index 참조의 형태입니다.

```
expr_kind (1 byte)
varint(추가 정보)
재귀 호출 (자식 expr)
```

자세한 expr 노드 종류는 `rok` 가 정합니다.

## 9.4.12. Lazy load

다운스트림이 RHIR 을 읽을 때는 다음 순서를 따릅니다.

1. Header + Producer + String + Type + Item 은 처음부터 읽습니다.
2. Body 는 그 `fn` 이 실제로 호출되어 specialize 가 필요할 때만 읽습니다.
3. body_offset 으로 직접 random access 하며, 전체 파일을 읽지는 않습니다.

`mmap` 을 사용하면 OS 가 필요한 페이지만 자동으로 로드해 줍니다.

## 9.4.13. RHIR 의 심볼 매핑

각 항목은 맹글링 이름과 매핑됩니다.

RHIR 의 항목 메타 → 맹글링 이름 → archive 의 obj 안 심볼의 흐름으로 이어집니다.

이 매핑을 통해 concrete `fn` 은 맹글링 심볼로 찾고, generic `fn` 은 RHIR 안의 항목으로 찾습니다.

## 9.4.14. RHIR 검증 (디버그)

`rok rhir-text path/to/libfoo.a` 는 RHIR 을 사람이 읽을 수 있는 형식으로 출력합니다.

```
rhir 1.2.3
package: foo (hash: 3f8a1c2b)

types:
  [0] i32
  [1] f64
  [2] struct Point { x: f64; y: f64; }
  ...

items:
  [0] fn distance(p1: Point, p2: Point) -> f64
      body: at offset 1024
  ...
```

이 도구로 RHIR 이 정확히 직렬화 / 역직렬화되는지 검증합니다.

## 9.4.15. 안정성

RHIR 포맷의 `version` 필드로 호환을 검사합니다.

- `version` 같음 → 호환.
- `version` 다름 → fail with clear error (rebuild required).

[개요](../008-abi/README.md) 의 안정성 정책을 적용합니다.

## 9.4.16. 크기

작은 패키지의 RHIR 은 보통 수십 KB 이하이며, 의존성이 많은 패키지도 수백 KB 에서 수 MB 수준입니다.

archive 전체에서 `.rok.hir` 섹션이 가장 큰 부분일 수 있으며, concrete .o 보다도 커질 수 있습니다.
