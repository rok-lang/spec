# 9.4. NHIR bytecode 포맷

[패키지 아카이브](903-package-archive.md) 의 `.nok.hir` 섹션에 들어가는, 제네릭 함수의 nok-hir 직렬화 형식인 [NHIR](../000-meta/007-glossary.md#nhir-nok-hir-bytecode) bytecode 를 정의한다.

## 9.4.1. 동기

[AOT 빌드 파이프라인](902-aot-pipeline.md) 의 단형화는 호출 지점의 구체 타입으로 [specialize](../000-meta/007-glossary.md#specialize) 한다. 다른 패키지가 우리 패키지의 [generic](../000-meta/007-glossary.md#generic-/-제네릭) `fn` 을 호출하면 그 호출 지점에서 specialize 가 일어나야 한다.

이때 우리 패키지의 generic `fn` 에 대한 [HIR](../000-meta/007-glossary.md#hir-high-level-ir-/-mir-mid-level-ir) 이 필요하다. [concrete fn](../000-meta/007-glossary.md#concrete-fn-/-generic-fn) 의 obj 만으로는 부족하다.

해결책은 [archive](../000-meta/007-glossary.md#archive) 안에 generic `fn` 의 HIR 을 직렬화해 함께 담는 것이다. 다운스트림이 이를 읽어 specialize 한다.

## 9.4.2. 외부 라이브러리 미사용

의존성 0 원칙에 따라 직렬화 라이브러리도 사용하지 않는다. NHIR 은 직접 설계한다. 수백 줄 수준의 단순한 형식으로 충분하다.

## 9.4.3. 형식 영감

MLIR bytecode 포맷의 구조를 차용한다.

- magic + version: 버전 호환 검사.
- string section: 식별자 중복 제거.
- type section: 타입 표 (구조적 공유).
- item section: `fn` / `struct` / `enum` 등의 메타.
- body section: 각 `fn` body 의 expr tree (lazy load 가능).
- VarInt 인코딩: LEB128 변형 (PrefixVarInt).

## 9.4.4. 전체 레이아웃

```
NHIR file:
+-----------------------------------+
|  Header                           |  (16 bytes 고정)
|    magic       "NHIR"  (4 bytes)  |
|    version     u32                |
|    flags       u32                |
|    body_offset u32                |
+-----------------------------------+
|  Producer string                  |  (가변)
|    "nokc 1.2.3 (abi-hash:abc...)" |
+-----------------------------------+
|  String Section                   |  (식별자 / 문자열 풀)
+-----------------------------------+
|  Type Section                     |  (타입 표)
+-----------------------------------+
|  Item Section                     |  (fn / struct / enum 메타)
+-----------------------------------+
|  Body Section                     |  (각 fn 의 HIR expr tree)
+-----------------------------------+
```

## 9.4.5. Header

```
+--------+---------+---------+
| offset |  size   | field   |
+--------+---------+---------+
|   0    |    4    | magic   |     "NHIR"
|   4    |    4    | version |     u32 — 포맷 버전
|   8    |    4    | flags   |     u32 — 옵션 플래그
|  12    |    4    | body_offset | u32 — body section 의 시작 offset
+--------+---------+---------+
```

`magic` 으로 파일이 NHIR 인지 빠르게 확인한다. 잘못된 파일은 즉시 reject 한다.

`body_offset` 은 lazy load 를 위해 body 의 위치를 직접 가리킨다. 메타만 읽고 body 는 필요할 때 로드한다.

## 9.4.6. Producer String

```
varint(length) | utf8 bytes
```

nokc 버전 + [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 해시 등 디버그 / 호환 검사용. 사용자 친화적 텍스트.

## 9.4.7. VarInt 인코딩

MLIR bytecode 와 같은 PrefixVarInt 이며 LEB128 의 변형이다.

```
1-byte int  : 0xxxxxxx                    (0 ~ 127)
2-byte int  : 10xxxxxx xxxxxxxx           (0 ~ 16383)
3-byte int  : 110xxxxx xxxxxxxx xxxxxxxx  (0 ~ 2097151)
...
```

작은 정수는 1 바이트로, 큰 정수는 더 많은 바이트로 인코딩한다.

대부분의 정수 (offset, count, index) 는 값이 작아 평균 1 ~ 2 바이트로 인코딩된다.

## 9.4.8. String Section

식별자 / 문자열의 공유 풀.

```
varint(num_strings)
for each string:
    varint(length)
    utf8 bytes
```

다른 섹션은 index 로 참조한다. 같은 식별자가 여러 번 등장해도 한 번만 저장한다.

## 9.4.9. Type Section

타입 표.

```
varint(num_types)
for each type:
    type_kind (1 byte)         // primitive / struct / enum / sum / fn / ...
    추가 필드 (kind 별)
```

타입은 재귀적 구조이다. 다른 타입을 `index` 로 참조한다.

| kind | 의미           | 추가 필드                                   |
| ---- | -------------- | ------------------------------------------- |
| 0    | primitive      | `prim_id` (i8 ~ f64 등)                     |
| 1    | `struct`       | item_index, generic_args...                 |
| 2    | `enum`         | item_index, generic_args...                 |
| 3    | sum (`A \| B`) | element_count, element_indices...           |
| 4    | tuple          | element_count, element_indices...           |
| 5    | array          | element_index, length                       |
| 6    | `fn`           | param_count, param_indices..., return_index |
| 7    | `any`          | (없음)                                      |
| 8    | generic_param  | param_id                                    |
| ...  | ...            | ...                                         |

## 9.4.10. Item Section

함수 / 타입 / 트레잇 / `impl` 등의 항목 메타.

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

각 항목의 body 는 별도의 body section 에 두고 lazy load 한다.

## 9.4.11. Body Section

각 `fn` 의 HIR expression tree 직렬화.

```
for each fn body:
    // body_offset 으로 점프해서 시작
    varint(num_locals)
    serialize_expr_tree(...)
```

expression tree 의 직렬화는 재귀적 노드 + index 참조 형태이다.

```
expr_kind (1 byte)
varint(추가 정보)
재귀 호출 (자식 expr)
```

자세한 expr 노드 종류는 `nokc` 가 정한다.

## 9.4.12. Lazy load

다운스트림이 NHIR 을 읽을 때:

1. Header + Producer + String + Type + Item 은 처음부터 읽는다.
2. Body 는 그 `fn` 이 실제로 호출되어 specialize 가 필요할 때만 읽는다.
3. body_offset 으로 직접 random access 한다. 전체 파일을 읽지 않는다.

`mmap` 을 쓰면 OS 가 필요한 페이지만 자동 로드한다.

## 9.4.13. NHIR 의 심볼 매핑

각 항목은 맹글링 이름과 매핑된다.

NHIR 의 항목 메타 → 맹글링 이름 → archive 의 obj 안 심볼.

이 매핑으로 concrete `fn` 은 맹글링 심볼로 찾고, generic `fn` 은 NHIR 안의 항목으로 찾는다.

## 9.4.14. NHIR 검증 (디버그)

`nokc nhir-text path/to/libfoo.a` 는 NHIR 을 사람이 읽을 수 있는 형식으로 출력한다.

```
nhir 1.2.3
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

이 도구로 NHIR 이 정확히 직렬화 / 역직렬화되는지 검증한다.

## 9.4.15. 안정성

NHIR 포맷의 `version` 필드로 호환을 검사한다.

- `version` 같음 → 호환.
- `version` 다름 → fail with clear error (rebuild required).

[개요](../008-abi/README.md) 의 안정성 정책을 적용한다.

## 9.4.16. 크기

작은 패키지의 NHIR 은 보통 수십 KB 이하이다. 의존성이 많은 패키지도 수백 KB ~ 수 MB 수준이다.

archive 전체에서 `.nok.hir` 섹션이 가장 큰 부분일 수 있다 (concrete .o 보다 커질 수 있다).

## 9.4.17. 추후: `.nok.interface` 텍스트 포맷

향후 NHIR 의 텍스트 표현 형식을 추가할 수 있다 (Swift `.swiftinterface` 모티브).

- 사람이 직접 읽기 / git diff 친화.
- 외부 도구 (LSP, 분석) 가 활용.

본 사양 범위 외이며 확장 가능성만 표시한다.
