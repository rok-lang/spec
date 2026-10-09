# 9.5. 심볼 맹글링 스킴

[AOT 빌드 파이프라인](902-aot-pipeline.md) 의 [specialize](../000-meta/007-glossary.md#specialize) 결과로 같은 함수가 구체 타입마다 별개의 함수로 갈라지게 됩니다. 이렇게 갈라진 함수들에 고유한 심볼 이름을 부여하기 위한 규칙을 여기에서 정의해 두겠습니다.

## 9.5.1. 형태

심볼의 전체적인 문법은 다음과 같이 정리할 수 있습니다.

```
mangled_symbol  ::= "_Y" pkg_hash "_" mod_path "_" item_name "_" type_hash
pkg_hash        ::= [0-9a-f]{8}            // package SHA-256 prefix
mod_path        ::= identifier ("_" identifier)*
item_name       ::= identifier
type_hash       ::= [0-9a-f]{16}           // generic args SHA-256 prefix
                  | ""                       // concrete (non-generic) item
```

이를 한 줄로 풀어 보면 다음과 같은 모습이 됩니다.

```
_Y <pkg_hash_8> _ <path> _ <name> _ <typehash_16>
```

실제 예시는 아래와 같습니다.

```
_Y3f8a1c2b_std_vec_push_a7c91d8e44b2f190
   |       |       |    |
   |       |       |    +-- typehash (16 chars, hash of generic args)
   |       |       +-- name (push)
   |       +-- path (std_vec)
   +-- pkg hash (8 chars)
```

> 16 chars, hash of generic args: 16 글자, 제네릭 인자 hash. name: 이름. path: 경로. pkg hash: 패키지 해시.

## 9.5.2. 부분별 의미

각 부분이 어떤 역할을 하는지 차례대로 살펴보겠습니다.

### `_Y`: rok 심볼 prefix

모든 rok 심볼은 `_Y` 로 시작합니다. 밑줄 + 대문자로 시작하는 이름은 C 에서 구현 예약 영역이므로 일반 C 심볼과 충돌하지 않으며, 이미 널리 쓰이는 맹글링 스킴 (C++ Itanium 의 `_Z`, Rust v0 의 `_R`, D 의 `_D`, Swift 의 `$s`) 과도 겹치지 않도록 `_Y` 를 골랐습니다. 또한 `nm` 이나 링커 에러 출력에서 rok 이 발행한 심볼을 한눈에 구분할 수 있습니다. prefix 바로 뒤에는 소문자 16진수 `<pkg_hash_8>` 이 오므로, 대문자 한 글자가 이어지는 예약 prefix (`_YK` 등, [9.5.5](#955-예약된-prefix)) 와는 첫 글자만으로 구분됩니다.

### `<pkg_hash_8>`

[소스 단위](../000-meta/003-source-units.md) 의 패키지 해시에 해당합니다. 내용 기반의 SHA-256 해시 가운데 앞 8 글자를 잘라 사용합니다.

```
3f8a1c2b
```

같은 내용을 가진 패키지라면 어디에서 빌드하더라도 같은 해시가 나오게 됩니다. 덕분에 서로 다른 패키지에 들어 있는 같은 이름의 함수끼리도 충돌할 일이 없습니다.

### `<path>`: 모듈 경로

[경로](../005-modules/507-paths.md) 에서 정의한 모듈 경로의 `::` 를 `_` 로 바꾸어 표현합니다.

```
rok::std::vec      → std_vec
pkg::geom   → geom               (pkg:: 는 패키지 hash 가 이미 식별)
foo::inner::sub → foo_inner_sub
```

### `<name>`: 항목 이름

[식별자](../001-lexical/104-identifiers.md) 에서 정의한 식별자가 들어갑니다. `<path>` 의 각 세그먼트도 같은 규칙으로 escape 합니다.

`_` 가 구분자이므로, 식별자 안의 `[A-Za-z0-9]` 이외 문자는 다음과 같이 escape 합니다.

| 원문 | 표기 |
|---|---|
| `[A-Za-z0-9]` | 그대로 |
| `_` | `_0` |
| 그 밖의 문자 (유니코드 포함) | `_1<소문자 16진 코드포인트>_` |

식별자는 숫자로 시작하지 않으므로 구분자 `_` 뒤에는 `0` / `1` 이 오지 않습니다. 따라서 escape 와 구분자는 항상 구별됩니다.

```
mod a { fn b }   → _Y<pkg>_a_b_
fn a__b          → _Y<pkg>__a_0_0b_
fn α             → _Y<pkg>___13b1__
```

### `<typehash_16>`: 타입 튜플 해시

[제네릭](../002-types/210-generics.md) 의 구체 타입 튜플을 SHA-256 으로 해싱한 뒤 앞 16 글자를 가져와 사용합니다.

```
fn id<T>(x: T) -> T

call id(42i32)   → typehash = SHA-256("(i32)")[0..16]
call id(3.14f64) → typehash = SHA-256("(f64)")[0..16]
```

> call: 호출.

### concrete `fn` 의 typehash

제네릭 인자를 받지 않는 함수의 경우 typehash 를 빈 문자열로 두거나 아예 생략하게 됩니다.

```
_Y3f8a1c2b_geom_compute_main           (no typehash)
or
_Y3f8a1c2b_geom_compute_main_           (empty typehash)
```

> no typehash: typehash 없습니다. empty typehash: 빈 typehash.

자세한 표기 방식은 `rokc` 가 정해 주게 됩니다. 본 사양에서는 제네릭 인자가 없을 때 typehash 를 명확히 식별할 수 있는 형태일 것까지만 요구하고 있습니다.

## 9.5.3. 비교: Rust `_ZN`

Rust 의 itanium-style 맹글링은 다음과 같은 모습을 띱니다.

```
_ZN3std3vec10push_inner17h2d4c3e9d8f1a2b3cE
```

- 길이 prefix 마다 길이를 명시합니다 (`3std3vec`).
- [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 식별자 hash 가 들어갑니다 (17h...).
- 끝에 `E` 가 붙습니다.

Itanium 호환이라는 장점이 있지만, 읽기가 어렵고 길이 prefix 가 인코딩 측면에서 비효율적이라는 단점이 있습니다.

이에 비해 rok 의 `N...` 은 읽기가 쉽고 길이 prefix 도 따로 두지 않습니다 (구분자 `_` 로 구분). demangle 과정도 단순하게 가져갈 수 있습니다.

## 9.5.4. demangle 도구

심볼을 사람이 읽기 좋게 풀어 보고 싶을 때는 다음 명령을 사용할 수 있습니다.

```
rokc demangle <symbol>
```

실행 결과는 다음과 같은 모습이 됩니다.

```
$ rokc demangle _Y3f8a1c2b_std_vec_push_a7c91d8e44b2f190
package:  3f8a1c2b
path:     rok::std::vec
name:     push
generics: a7c91d8e44b2f190 (type tuple hash)
```

## 9.5.5. 예약된 prefix

사용 중인 prefix 와 예약된 prefix 는 다음과 같습니다.

| Prefix   | 의미                                                                           |
|----------|--------------------------------------------------------------------------------|
| `_Y...`  | rok 일반 심볼                                                                  |
| `rok_*`  | `extern` "C" 런타임 함수 ([Extern "C" 심볼](../008-abi/805-extern-symbols.md)) |
| `_YK...` | (예약) keyword 핸들러                                                          |
| `_YL...` | (예약) literal 핸들러                                                          |
| `_YI...` | (예약) infix / prefix / postfix 매직                                           |
| `_YM...` | (예약) 메타 블록 (impl_derive 등) 의 생성 코드                                 |

`_YK` / `_YL` / `_YI` / `_YM` 은 이름 공간만 예약된 상태이며, 일반 심볼은 모두 `_Y` prefix 로 발행됩니다.

## 9.5.6. 심볼 길이

대부분의 rok 심볼은 다음과 같은 길이 분포를 보입니다.

```
_Y + 8 + _ + path (수십) + _ + 이름 (수~수십) + _ + 16
≈ 30 ~ 80 글자
```

매우 큰 path 와 깊은 [generic](../000-meta/007-glossary.md#generic--제네릭) 이 결합되는 경우 100 글자를 넘어갈 수도 있습니다. 디버거나 도구들은 이 정도 길이는 무리 없이 처리해 주게 됩니다 (보통은 2KB 이하를 권장하고 있습니다).

## 9.5.7. 충돌 회피: typehash 의 안정성

제네릭 인자의 타입 튜플은 결정론적인 방식으로 직렬화한 뒤 SHA-256 으로 해싱하게 됩니다.

```
타입 튜플 → canonical 직렬화 → SHA-256 → 앞 16 글자
```

같은 타입이라면 빌드를 몇 번 다시 하더라도 동일한 해시가 나옵니다. 다른 빌드에서 일어난 같은 specialization 도 같은 심볼 이름을 갖게 됩니다.

이러한 안정성 덕분에 분리 빌드와 증분 빌드, 그리고 의존 패키지 사이의 심볼 매칭이 가능해집니다.

### canonical 직렬화 규칙

타입 튜플의 직렬화는 결정론적으로 이루어집니다.

- 원시 타입은 짧은 식별자로 표기됩니다 (`i32` / `f64` 등).
- 사용자 타입은 `{pkg_hash}**{path}**{name}` 에 generic 인자를 더한 형태가 됩니다.
- 합 타입의 경우 항들을 순서 정규화한 뒤 결합합니다 ([합 타입 (`A | B`)](../002-types/206-sum-types.md) 의 집합 의미론을 따릅니다).
- `struct` 는 필드 타입들로 구성됩니다 (필드 이름은 무관합니다).

자세한 정규화 알고리듬은 `rokc` 가 결정해 주게 됩니다. 본 사양에서는 결정론적이어야 한다는 점까지만 정해 두고 있습니다.

## 9.5.8. `extern` "C" 함수: 맹글링 미적용

[`extern` 블록](../005-modules/509-extern-block.md) 에서 선언한 외부 함수는 해당 ABI 의 표준 명명 규칙을 그대로 사용합니다. rok 의 맹글링은 적용하지 않습니다.

```rok
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;     // 심볼: "malloc" (그대로)
}
```

반대 방향으로, `#[export_name("name")]` 을 사용하면 rok 함수를 외부에 노출할 때 이름을 직접 지정해 둘 수 있습니다 ([`extern` 블록 ([FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 선언)](../005-modules/509-extern-block.md) 참고). 이런 항목은 도달 불가 함수 제거의 root 이므로 항상 emit 됩니다 ([9.2.8](902-aot-pipeline.md#928-specialize-단형화)).

```rok
#[export_name("rok_compute")]
pub fn compute(x: i32, y: i32) -> i32 { x + y }
// 심볼: "rok_compute" (맹글링 안 됨)
```

이 두 가지 메커니즘 덕분에 rok 과 C 사이의 자연스러운 통합이 이루어지게 됩니다.

## 9.5.9. main 함수의 심볼

[소스 단위](../000-meta/003-source-units.md) 의 진입점 함수는 플랫폼이 제공하는 일반 main 심볼을 그대로 사용합니다.

```
main          (Linux / macOS)
WinMain       (Windows GUI)
_start        (no-OS)
```

플랫폼별 진입점은 rokc 가 생성한 wrapper 안에서 사용자가 정의한 main 함수를 호출해 주게 됩니다. wrapper 는 `rok_runtime_init` 과 타입 등록 함수 `rok.register_types` ([8.2.13](../008-abi/802-scan-table.md#8213-등록-시점)) 를 차례로 호출한 뒤 진입 함수를 호출하고, [4.16.4](../004-functions/416-entry-point.md#4164-종료-코드와-미처리-예외) 의 규칙으로 종료 코드를 정한 다음 `rok_runtime_shutdown` 을 거쳐 반환합니다.

`main.rok` 가 top-level 문으로 이루어진 경우 ([4.16.2](../004-functions/416-entry-point.md#4162-top-level-문의-의미-암묵-main)), 컴파일러가 합성한 암묵 `main` 이 진입 함수가 됩니다. 암묵 `main` 은 일반 함수와 같은 규칙으로 맹글링되지만 사용자 코드에서 이름으로 참조할 수 없으며, wrapper 만이 이를 호출합니다.

## 9.5.10. 디버그 정보

각 mangled 심볼은 DWARF 와 PDB 디버그 정보 안에 원본 이름을 함께 포함합니다.

```
debugger 출력.
  fn distance<f64> at geom.rok:42
    (mangled: _Y3f8a1c2b_geom_distance_a7c91d8e44b2f190)
```

이 정보 덕분에 gdb, lldb, VS Code 같은 도구들이 rok 함수를 원본 이름으로 표시해 줄 수 있습니다.

## 9.5.11. 안정성

심볼 맹글링 스킴은 ABI 의 일부이기도 합니다. 따라서 변경하실 때는 [개요](../008-abi/README.md) 에 정리된 안정성 정책을 따르셔야 합니다.

- 같은 major version 안에서는 호환성이 유지됩니다.
- 다른 major version 사이에서는 호환이 깨지며, 이때는 ABI 해시도 달라지게 됩니다.
