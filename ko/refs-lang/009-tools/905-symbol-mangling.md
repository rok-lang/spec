# 9.5. 심볼 맹글링 스킴

[AOT 빌드 파이프라인](902-aot-pipeline.md)의 [specialize](../000-meta/007-glossary.md#specialize) 결과로 함수 하나가 구체 타입마다 별개의 함수로 갈라집니다. 이 절은 갈라진 함수들에 고유한 심볼 이름을 붙이는 규칙을 정의합니다.

## 9.5.1. 형태

심볼의 문법은 다음과 같습니다.

```
mangled_symbol  ::= "_Y" pkg_hash "_" mod_path "_" item_name "_" type_hash
pkg_hash        ::= [0-9a-f]{8}            // package SHA-256 prefix
mod_path        ::= identifier ("_" identifier)*
item_name       ::= identifier
type_hash       ::= [0-9a-f]{16}           // generic args SHA-256 prefix
                  | ""                       // concrete (non-generic) item
```

한 줄로 풀어 쓰면 다음과 같습니다.

```
_Y <pkg_hash_8> _ <path> _ <name> _ <typehash_16>
```

실제 예는 다음과 같습니다.

```
_Y3f8a1c2b_std_vec_push_a7c91d8e44b2f190
   |       |       |    |
   |       |       |    +-- typehash (16 chars, hash of generic args)
   |       |       +-- name (push)
   |       +-- path (std_vec)
   +-- pkg hash (8 chars)
```

## 9.5.2. 부분별 의미

### `_Y`: Rok 심볼 prefix

모든 Rok 심볼은 `_Y`로 시작합니다. 밑줄 + 대문자로 시작하는 이름은 C에서 구현 예약 영역이므로 일반 C 심볼과 충돌하지 않으며, 이미 널리 쓰이는 맹글링 스킴 (C++ Itanium의 `_Z`, Rust v0의 `_R`, D의 `_D`, Swift의 `$s`)과도 겹치지 않도록 `_Y`를 골랐습니다. `nm`이나 링커 에러 출력에서 Rok이 만든 심볼을 바로 알아볼 수도 있습니다. prefix 바로 뒤에는 소문자 16진수 `<pkg_hash_8>`이 오므로, 대문자 한 글자가 이어지는 예약 prefix (`_YK` 등, [9.5.5](#955-예약된-prefix))와는 첫 글자만으로 구분됩니다.

### `<pkg_hash_8>`

[소스 단위](../000-meta/003-source-units.md)의 패키지 해시에 해당합니다. 내용으로 계산한 SHA-256 해시의 앞 8글자를 씁니다.

```
3f8a1c2b
```

내용이 같은 패키지는 어디에서 빌드하든 같은 해시가 나옵니다. 그래서 서로 다른 패키지에 있는 같은 이름의 함수끼리 충돌하지 않습니다.

### `<path>`: 모듈 경로

[경로](../005-modules/507-paths.md)에서 정의한 모듈 경로의 `::`를 `_`로 바꿉니다. 표준 라이브러리 경로는 `rok::` 루트 세그먼트를 빼고 (`rok::std::vec` → `std_vec`), 사용자 경로는 `pkg::`를 뺍니다.

```
rok::std::vec      → std_vec
pkg::geom   → geom               (pkg:: 는 패키지 hash 가 이미 식별)
foo::inner::sub → foo_inner_sub
```

### `<name>`: 항목 이름

[식별자](../001-lexical/104-identifiers.md)에서 정의한 식별자가 들어갑니다. `<path>`의 각 세그먼트도 같은 규칙으로 escape합니다.

`_`가 구분자이므로, 식별자 안의 `[A-Za-z0-9]` 이외 문자는 다음과 같이 escape합니다.

| 원문 | 표기 |
|---|---|
| `[A-Za-z0-9]` | 그대로 |
| `_` | `_0` |
| 그 밖의 문자 (유니코드 포함) | `_1<소문자 16진 코드포인트>_` |

식별자는 숫자로 시작하지 않으므로 구분자 `_` 뒤에는 `0` / `1`이 오지 않습니다. 따라서 escape와 구분자는 항상 구별됩니다.

```
mod a { fn b }   → _Y<pkg>_a_b_
fn a__b          → _Y<pkg>__a_0_0b_
fn α             → _Y<pkg>___13b1__
```

### 메서드의 이름

메서드는 `<타입>.<메서드>`를 이름으로, 타입이 정의된 모듈을 경로로 맹글링합니다. 같은 타입에 같은 이름의 메서드를 주는 트레잇이 여럿이면 뒤에 `__by_<트레잇>`이 붙습니다. 의존 패키지 ([9.2.3](902-aot-pipeline.md#923-의존성-해소))의 메서드도 이 안정된 경로 `<모듈>::<타입>.<메서드>[__by_<트레잇>]`으로 맹글링되므로, 따로 빌드한 패키지와 프로그램이 같은 심볼을 만듭니다. `.`와 `_`는 위 표대로 escape됩니다.

### 중첩 함수

중첩 함수 ([4.10](../004-functions/410-nested-functions.md))는 바깥 항목의 모듈에서 `<outer>/<name>`을 이름으로 맹글링합니다. `<outer>`는 바깥 함수의 이름입니다 (메서드면 `<타입>.<메서드>`, 트레잇 기본 본체면 `<트레잇>.<메서드>`, 중첩 함수면 재귀적으로 `<outer>/<name>`). 한 소스 파일에서 `<outer>/<name>`이 같은 중첩 함수가 둘 이상이면 소스 순서대로 `/2`, `/3`, ... 이 붙습니다. `/`는 다른 어떤 항목 이름에도 나오지 않으므로, 이 심볼은 모듈 경로나 메서드와 구별됩니다. `/`는 표대로 escape됩니다 (`_12f_`).

```
mod a { fn outer() { fn helper() } }   → _Y<pkg>_a_outer_12f_helper_
```

### `<typehash_16>`: 타입 튜플 해시

[제네릭](../002-types/210-generics.md)의 구체 타입 튜플을 SHA-256으로 해싱해 앞 16글자를 씁니다.

```
fn id<T>(x: T) -> T

call id(42i32)   → typehash = SHA-256("(i32)")[0..16]
call id(3.14f64) → typehash = SHA-256("(f64)")[0..16]
```

### concrete `fn` 의 typehash

제네릭 인자를 받지 않는 함수는 typehash를 빈 문자열로 두거나 아예 생략합니다.

```
_Y3f8a1c2b_geom_compute_main           (no typehash)
or
_Y3f8a1c2b_geom_compute_main_           (empty typehash)
```

정확한 표기는 `rokc`가 정합니다. 이 사양은 제네릭 인자가 없다는 사실을 typehash에서 모호하지 않게 식별할 수 있어야 한다는 것만 요구합니다.

## 9.5.3. 비교: Rust `_ZN`

Rust의 itanium-style 맹글링은 다음과 같습니다.

```
_ZN3std3vec10push_inner17h2d4c3e9d8f1a2b3cE
```

- 길이 prefix마다 길이를 명시합니다 (`3std3vec`).
- [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 식별자 hash가 들어갑니다 (17h...).
- 끝에 `E`가 붙습니다.

Itanium과 호환된다는 장점이 있지만, 읽기 어렵고 길이 prefix는 인코딩하기에 비효율적입니다.

이에 비해 Rok의 `N...`은 읽기 쉽고 길이 prefix가 없습니다(구성 요소는 `_`로 구분). demangle도 단순합니다.

## 9.5.4. demangle 도구

심볼을 사람이 읽을 수 있게 풀려면 다음 명령을 씁니다.

```
rokc demangle <symbol>
```

실행 결과는 다음과 같습니다.

```
$ rokc demangle _Y3f8a1c2b_std_vec_push_a7c91d8e44b2f190
package:  3f8a1c2b
path:     rok::std::vec
name:     push
generics: a7c91d8e44b2f190 (type tuple hash)
```

## 9.5.5. 예약된 prefix

사용 중인 prefix와 예약된 prefix는 다음과 같습니다.

| Prefix   | 의미                                                                           |
|----------|--------------------------------------------------------------------------------|
| `_Y...`  | Rok 일반 심볼                                                                  |
| `rok_*`  | `extern` "C" 런타임 함수 ([Extern "C" 심볼](../008-abi/805-extern-symbols.md)) |
| `_YK...` | (예약) keyword 핸들러                                                          |
| `_YL...` | (예약) literal 핸들러                                                          |
| `_YI...` | (예약) infix / prefix / postfix 매직                                           |
| `_YM...` | (예약) 메타 블록 (impl_derive 등)의 생성 코드                                 |

`_YK` / `_YL` / `_YI` / `_YM`은 이름 공간만 예약해 두었으며, 일반 심볼은 모두 `_Y` prefix로 만듭니다.

## 9.5.6. 심볼 길이

대부분의 Rok 심볼 길이는 다음 범위에 있습니다.

```
_Y + 8 + _ + path (수십) + _ + 이름 (수~수십) + _ + 16
≈ 30 ~ 80 글자
```

path가 매우 길고 [generic](../000-meta/007-glossary.md#generic--제네릭)이 깊게 중첩되면 100글자를 넘을 수도 있습니다. 디버거와 도구는 이 정도 길이를 문제없이 처리합니다(보통 2KB 이하를 권장합니다).

## 9.5.7. 충돌 회피: typehash 의 안정성

제네릭 인자의 타입 튜플은 결정론적으로 직렬화한 뒤 SHA-256으로 해싱합니다.

```
타입 튜플 → canonical 직렬화 → SHA-256 → 앞 16 글자
```

타입이 같으면 빌드를 몇 번 다시 하든 같은 해시가 나옵니다. 다른 빌드에서 일어난 같은 specialization도 같은 심볼 이름을 얻습니다. rokc가 클로저와 그 밖의 생성된 함수에 붙이는 이름은 그 함수가 적힌 항목(인스턴스면 그 타입 인자까지)에서 나오므로, 이 이름도 모든 빌드에서 같습니다.

이 안정성이 있어야 분리 빌드, 증분 빌드, 의존 패키지 사이의 심볼 매칭이 가능합니다.

### canonical 직렬화 규칙

타입 튜플은 다음 규칙으로 결정론적으로 직렬화합니다.

- 원시 타입은 짧은 식별자로 씁니다(`i32` / `f64` 등).
- 사용자 타입은 `{pkg_hash}**{path}**{name}`에 generic 인자를 더한 형태입니다.
- 합 타입은 항들의 순서를 정규화한 뒤 결합합니다([합 타입 (`A | B`)](../002-types/206-sum-types.md)의 집합 의미론을 따름).
- `struct`는 필드 타입들로 구성합니다(필드 이름은 상관없음).

정확한 정규화 알고리듬은 `rokc`가 정합니다. 이 사양은 알고리듬이 결정론적이어야 한다는 것만 요구합니다.

## 9.5.8. `extern` "C" 함수: 맹글링 미적용

[`extern` 블록](../005-modules/509-extern-block.md)에서 선언한 외부 함수는 해당 ABI의 표준 명명 규칙을 그대로 씁니다. Rok의 맹글링은 적용하지 않습니다.

```rok
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;     // 심볼: "malloc" (그대로)
}
```

반대 방향으로, `#[export_name("name")]`은 Rok 함수를 외부에 노출할 때 이름을 직접 지정합니다 ([`extern` 블록 ([FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 선언)](../005-modules/509-extern-block.md) 참고). 이런 항목은 도달 불가 함수 제거의 root이므로 항상 emit됩니다 ([9.2.8](902-aot-pipeline.md#928-specialize-단형화)).

```rok
#[export_name("rok_compute")]
pub fn compute(x: i32, y: i32) -> i32 { x + y }
// 심볼: "rok_compute" (맹글링 안 됨)
```

이 두 메커니즘으로 Rok과 C를 함께 쓸 수 있습니다.

## 9.5.9. main 함수의 심볼

[소스 단위](../000-meta/003-source-units.md)의 진입점 함수는 플랫폼의 일반 main 심볼을 그대로 씁니다.

```
main                                (Linux / macOS)
mainCRTStartup                      (Windows console)
WinMainCRTStartup + WinMain         (Windows GUI, -C subsystem=windows)
_start                              (no-OS)
__main_void                         (wasm32-wasi*; wasi-libc 의 _start 가 부름)
_initialize, then main              (wasm32-unknown-unknown; host 가 부르는 reactor export)
```

`wasm32-unknown-unknown`에서 모듈은 reactor입니다: host가 `_initialize`를, 그다음 export `main`을 부르며, 런타임은 그 뒤에도 살아 있습니다 (shutdown 없음). `#[export_name("x")]` 함수는 이름이 `x`인 wasm export입니다. 모든 wasm32 타깃에서 런타임은 `__main_void`, `main`, export 호출 가운데 처음 것에서 시작합니다.

Windows에서는 Rok 자신의 startup object가 `mainCRTStartup` (console)이나 `WinMainCRTStartup`과 `WinMain` (GUI)을 제공하며, 이 startup이 rokc가 생성한 `main` wrapper를 부릅니다.

플랫폼별 진입점은 rokc가 생성한 wrapper 안에서 사용자가 정의한 main 함수를 호출합니다. wrapper는 `rok_runtime_init`과 타입 등록 함수 `rok.register_types` ([8.2.13](../008-abi/802-scan-table.md#8213-등록-시점))를 차례로 호출한 뒤 진입 함수를 호출하고, [4.16.4](../004-functions/416-entry-point.md#4164-종료-코드와-미처리-예외)의 규칙으로 종료 코드를 정한 다음 `rok_runtime_shutdown`을 거쳐 반환합니다.

`main.rok`이 top-level 문으로 이루어진 경우 ([4.16.2](../004-functions/416-entry-point.md#4162-top-level-문의-의미-암묵-main)), 컴파일러가 합성한 암묵 `main`이 진입 함수가 됩니다. 암묵 `main`은 일반 함수와 같은 규칙으로 맹글링되지만 사용자 코드에서 이름으로 참조할 수 없으며, wrapper만 이 함수를 호출합니다.

## 9.5.10. 디버그 정보

DWARF와 PDB 디버그 정보에는 mangled 심볼마다 원본 이름이 함께 들어갑니다.

```
debugger 출력.
  fn distance<f64> at geom.rok:42
    (mangled: _Y3f8a1c2b_geom_distance_a7c91d8e44b2f190)
```

그래서 gdb, lldb, VS Code 같은 도구가 Rok 함수를 원본 이름으로 표시할 수 있습니다. 코루틴의 resume 함수 이름은 `<creator>::{coroutine#k}`이며, `k`는 그 creator의 클로저와 코루틴을 함께 셉니다.

## 9.5.11. 안정성

심볼 맹글링 스킴은 ABI의 일부이기도 합니다. 따라서 스킴을 바꿀 때는 [개요](../008-abi/README.md)의 안정성 정책을 따라야 합니다.

- 같은 major version 안에서는 호환성이 유지됩니다.
- 다른 major version 사이에서는 호환이 깨지며, ABI 해시도 달라집니다.
