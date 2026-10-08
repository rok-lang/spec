# 9.5. 심볼 맹글링 스킴

[AOT 빌드 파이프라인](902-aot-pipeline.md) 의 [specialize](../000-meta/007-glossary.md#specialize) 결과로 같은 함수가 구체 타입마다 별개 함수가 된다. 이들에 고유한 심볼 이름을 부여하는 규칙을 정의한다.

## 9.5.1. 형태

```
mangled_symbol  ::= "_N" pkg_hash "_" mod_path "_" item_name "_" type_hash
pkg_hash        ::= [0-9a-f]{8}            // package SHA-256 prefix
mod_path        ::= identifier ("_" identifier)*
item_name       ::= identifier
type_hash       ::= [0-9a-f]{16}           // generic args SHA-256 prefix
                  | ""                       // concrete (non-generic) item
```

```
_N <pkg_hash_8> _ <path> _ <name> _ <typehash_16>
```

예시:

```
_N3f8a1c2b_std_vec_push_a7c91d8e44b2f190
   |       |       |    |
   |       |       |    └── typehash (16 글자, 제네릭 인자 hash)
   |       |       └── 이름 (push)
   |       └── path (std_vec)
   └── pkg hash (8 글자)
```

## 9.5.2. 부분별 의미

### `_N`: nok 심볼 prefix

모든 nok 심볼은 `_N` 으로 시작한다. C 심볼과의 충돌을 피한다.

### `<pkg_hash_8>`

[소스 단위](../000-meta/003-source-units.md) 의 패키지 해시이다. 내용 기반 SHA-256 의 앞 8 글자이다.

```
3f8a1c2b
```

같은 내용의 패키지는 어디서든 같은 hash 가 된다. 서로 다른 패키지에 있는 같은 이름의 함수끼리도 충돌하지 않는다.

### `<path>`: 모듈 경로

[경로](../005-modules/507-paths.md) 의 모듈 경로에서 `::` 를 `_` 로 바꾸어 표현한다.

```
std::vec      → std_vec
crate::geom   → geom               (crate:: 는 패키지 hash 가 이미 식별)
foo::inner::sub → foo_inner_sub
```

### `<name>`: 항목 이름

[식별자](../001-lexical/104-identifiers.md) 의 식별자. Unicode 식별자도 쓸 수 있다.

ASCII 가 아닌 식별자는 Punycode 인코딩 (또는 다른 안전한 인코딩) 으로 변환한다. 예:

```
α → __1z (가상 인코딩 — 정확한 형식은 nokc 가 결정)
변수1 → __6n1k... (가상)
```

대부분의 항목은 ASCII 라서 그대로 사용한다.

### `<typehash_16>`: 타입 튜플 해시

[제네릭](../002-types/210-generics.md) 의 구체 타입 튜플을 SHA-256 으로 해싱한 앞 16 글자.

```
fn id<T>(x: T) -> T

호출 id(42i32) → typehash = SHA-256("(i32)")[0..16]
호출 id(3.14f64) → typehash = SHA-256("(f64)")[0..16]
```

### concrete `fn` 의 typehash

제네릭 인자가 없는 함수는 typehash 가 빈 문자열이거나 생략한다.

```
_N3f8a1c2b_geom_compute_main           (typehash 없음)
or
_N3f8a1c2b_geom_compute_main_           (빈 typehash)
```

자세한 표기는 `nokc` 가 정한다. 본 사양은 제네릭 인자가 없을 때 typehash 를 명확히 식별할 수 있는 형태일 것만 요구한다.

## 9.5.3. 비교: Rust `_ZN`

Rust 의 itanium-style 맹글링:

```
_ZN3std3vec10push_inner17h2d4c3e9d8f1a2b3cE
```

- 길이 prefix 마다 길이 명시 (`3std3vec`).
- [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 식별자 hash (17h...).
- 끝 `E`.

Itanium 호환이지만 읽기 어렵고 길이 prefix 가 인코딩에 비효율적이다.

nok 의 `_N...` 은 읽기 쉽고 길이 prefix 가 없다 (구분자 `_`). demangle 도 단순하다.

## 9.5.4. demangle 도구

```
nokc demangle <symbol>
```

```
$ nokc demangle _N3f8a1c2b_std_vec_push_a7c91d8e44b2f190
package:  3f8a1c2b
path:     std::vec
name:     push
generics: a7c91d8e44b2f190 (type tuple hash)
```

## 9.5.5. 예약된 prefix

| Prefix   | 의미                                                                           |
| -------- | ------------------------------------------------------------------------------ |
| `_N...`  | nok 일반 심볼                                                                  |
| `nok_*`  | `extern` "C" 런타임 함수 ([Extern "C" 심볼](../008-abi/805-extern-symbols.md)) |
| `_NK...` | (예약) keyword 핸들러                                                          |
| `_NL...` | (예약) literal 핸들러                                                          |
| `_NI...` | (예약) infix / prefix / postfix 매직                                           |
| `_NM...` | (예약) 메타 블록 (impl_derive 등) 의 생성 코드                                 |

추가 prefix 는 향후 등장할 수 있다. 현재는 `_N` (일반) 만 사용한다.

## 9.5.6. 심볼 길이

대부분의 nok 심볼:

```
_N + 8 + _ + path (수십) + _ + 이름 (수~수십) + _ + 16
≈ 30 ~ 80 글자
```

매우 큰 path 와 깊은 [generic](../000-meta/007-glossary.md#generic-/-제네릭) 의 경우 100 글자를 넘길 수 있다. 디버거와 도구가 무리 없이 처리한다 (보통 2KB 이하 권장).

## 9.5.7. 충돌 회피: typehash 의 안정성

제네릭 인자의 타입 튜플은 결정론적으로 직렬화한 뒤 SHA-256 으로 해싱한다.

```
타입 튜플 → canonical 직렬화 → SHA-256 → 앞 16 글자
```

같은 타입은 빌드마다 같은 hash 가 된다. 다른 빌드의 같은 specialization 도 같은 심볼 이름이 된다.

이 안정성 덕분에 분리 빌드, 증분 빌드, 의존 패키지 사이의 심볼 매칭이 가능하다.

### canonical 직렬화 규칙

타입 튜플의 직렬화는 결정론적이다.

- 원시 타입 → 짧은 식별자 (`i32` / `f64` / 등).
- 사용자 타입 → `{pkg_hash}_{path}_{name}` + generic 인자.
- 합 타입 → 항들을 순서 정규화한 뒤 결합 ([합 타입 (`A | B`)](../002-types/206-sum-types.md) 의 집합 의미론).
- `struct` → 필드 타입들 (이름 무관).

자세한 정규화 알고리듬은 `nokc` 가 결정한다. 본 사양은 결정론적이어야 한다는 점까지만 정한다.

## 9.5.8. `extern` "C" 함수: 맹글링 미적용

[`extern` 블록](../005-modules/509-extern-block.md) 의 외부 함수는 해당 ABI 의 표준 명명을 사용한다. nok 맹글링을 적용하지 않는다.

```nok
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;     // 심볼: "malloc" (그대로)
}
```

`#[export_name("name")]` 으로 nok 함수를 외부에 노출할 때 이름을 지정할 수 있다 ([`extern` 블록 ([FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 선언)](../005-modules/509-extern-block.md)).

```nok
#[export_name("nok_compute")]
pub fn compute(x: i32, y: i32) -> i32 { x + y }
// 심볼: "nok_compute" (맹글링 안 됨)
```

이 두 메커니즘 덕분에 nok 와 C 의 자연스러운 통합이 이루어진다.

## 9.5.9. main 함수의 심볼

[소스 단위](../000-meta/003-source-units.md) 의 진입점 함수는 플랫폼의 일반 main 심볼을 사용한다.

```
main          (Linux / macOS)
WinMain       (Windows GUI)
_start        (no-OS)
```

플랫폼별 진입점이 nokc 가 생성한 wrapper 안에서 사용자의 main 함수를 호출한다.

## 9.5.10. 디버그 정보

각 mangled 심볼은 DWARF / PDB 디버그 정보에 원본 이름도 포함한다.

```
debugger 출력:
  fn distance<f64> at geom.nok:42
    (mangled: _N3f8a1c2b_geom_distance_a7c91d8e44b2f190)
```

이 정보 덕분에 gdb, lldb, VS Code 가 nok 함수를 원본 이름으로 표시한다.

## 9.5.11. 안정성

심볼 맹글링 스킴은 ABI 의 일부이다. 변경 시 [개요](../008-abi/README.md) 의 안정성 정책을 따른다.

- 같은 major version 에서는 호환.
- 다른 major version 에서는 break, ABI 해시가 다르다.
