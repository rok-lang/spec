# 6.2. 내장 속성

이 절은 컴파일러가 직접 처리하는 내장 속성을 정의합니다. `#[cfg]`는 [조건부 컴파일](603-cfg.md)에서 따로 다룹니다. 매직 속성과 메타 블록은 [`#[lang(...)]` 매직](604-lang-magic.md)부터 이어지는 절에서 설명합니다.

```
derive_attr     ::= "#" "[" "derive" "(" trait_name ("," trait_name)* ","? ")" "]"
trait_name      ::= identifier
repr_attr       ::= "#" "[" "repr" "(" repr_arg ("," repr_arg)* ")" "]"
repr_arg        ::= "C" | "transparent" | "packed" | "align" "(" int_lit ")"
                  | identifier                          // primitive int repr
inline_attr     ::= "#" "[" "inline" ("(" ("always" | "never") ")")? "]"
deprecated_attr ::= "#" "[" "deprecated" ("(" deprecated_args ")" | "=" string_lit)? "]"
deprecated_args ::= string_lit | deprecated_kv ("," deprecated_kv)*
deprecated_kv   ::= ("since" | "note") "=" string_lit
deprecated_safe ::= "#" "[" "deprecated_safe" ("(" string_lit ")" | "=" string_lit)? "]"
sealed_attr     ::= "#" "[" "sealed" "]"
internal_attr   ::= "#" "[" "internal" "]"
discardable_attr ::= "#" "[" "discardable" "]"
```

## 6.2.1. `#[derive(...)]` 자동 트레잇 구현

```rok
#[derive(Clone, PartialEq, Eq, Hash, Debug)]
struct Point { x: i64; y: i64; }
```

`#[derive(...)]`의 인자로 트레잇 이름을 쉼표로 구분해 나열합니다. 그러면 컴파일러가 나열한 트레잇마다 그 타입에 맞는 구현을 자동으로 생성합니다.

### 내장 derive 가능 트레잇

derive할 수 있는 트레잇은 `Clone`, `Copy`, `Debug`, `Default`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`뿐입니다. 각 트레잇의 생성 규칙과 요구 조건은 [stdlib 2.13.11](../../refs-standard-libraries/200-core/213-marker.md#21311-derive-가능한-트레잇)에서 정의합니다.

- `Display`는 derive로 구현할 수 없습니다. 표시 형식은 타입 작성자가 정합니다([stdlib 2.10](../../refs-standard-libraries/200-core/210-fmt.md)).
- `Eq` / `Ord`를 상위 트레잇(`PartialEq` / `PartialOrd`, `Eq`) 없이 derive로 구현하면 컴파일 에러입니다. `Copy`는 `Clone`이 필요하고, `Drop`과 함께 쓸 수 없습니다.
- `union`에 `#[derive]`를 붙이면 컴파일 에러입니다.

### 사용자 정의 derive와 결합

`#[derive(MyTrait)]`처럼 사용자 정의 트레잇도 derive로 구현할 수 있습니다. 단, `MyTrait`를 정의하는 `#[impl_derive(MyTrait)]` 메타 블록([6.8](608-meta-blocks.md))이 있어야 합니다. 위의 내장 derive 이름은 메타 블록으로 다시 정의할 수 없습니다.

내장 derive와 사용자 정의 derive는 한 `#[derive(...)]` 안에 섞어 쓸 수 있습니다.

```rok
#[derive(PartialEq, Eq, Clone, MyCustomTrait)]
struct Foo { ... }
```

## 6.2.2. `#[repr(...)]` 메모리 표현 명시

```rok
#[repr(C)]
struct CCompatible { x: i32; y: i32; }

#[repr(transparent)]
struct Wrapped(InnerType);

#[repr(packed)]
struct Tight { a: u8; b: u32; }

#[repr(align(16))]
struct Aligned { ... }

#[repr(u32)]
enum Status { ... }
```

| 인자                        | 의미                                                                                                 |
|-----------------------------|------------------------------------------------------------------------------------------------------|
| `C`                         | C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 레이아웃: 선언 순서, 자연 정렬. 기본 `struct` 레이아웃과 같음 |
| `transparent`               | 필드 정확히 하나, 다른 `repr` 인자 없음. 그 필드와 같은 표현이고 C ABI에서도 그 필드처럼 건넘             |
| `packed`                    | 정렬을 무시하고 빽빽이 묶음 ([FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 한정). GC 참조를 담을 수 없고 `align(N)`과 함께 쓸 수 없음 |
| `align(N)`                  | 최소 정렬을 N바이트로 강제                                                                           |
| `u8` ~ `u64` / `i8` ~ `i64` | `enum`의 태그 타입. 32비트가 아닌 폭은 필드 없는 enum에만 씀. 데이터가 있는 enum은 `i32` / `u32` |

`#[repr]`가 없으면 [`struct`](../002-types/203-struct.md)는 선언 순서와 자연 정렬을 따릅니다. `repr(C)`와 같습니다. [`enum`](../002-types/204-enum.md)과 합 타입의 레이아웃은 컴파일러가 정합니다([2.7](../002-types/207-niche-optimization.md)).

## 6.2.3. `#[link(...)]` 외부 라이브러리 명시

```rok
#[link("m")]
extern "C" {
    fn sin(x: f64) -> f64;
    fn cos(x: f64) -> f64;
}

#[link(name = "openssl", kind = "static")]
extern "C" { ... }
```

`#[link(...)]`는 [`extern` 블록](../005-modules/509-extern-block.md)이 링크할 외부 라이브러리를 지정합니다. `extern` 블록에 붙여 씁니다.

| 인자 형식                   | 의미                        |
|-----------------------------|-----------------------------|
| `"libname"`                 | 단순 라이브러리 이름        |
| `name = "..."`              | 라이브러리 이름 (key-value) |
| `kind = "static" / "dylib"` | 링크 종류                   |
| `wasm_import_module = "m"` | wasm32: 블록의 함수를 core module `m`에서 가져옴. 그 밖에서는 무시 |

### `#[link_name = "sym"]`

extern 함수에 붙인 `#[link_name = "sym"]`은 그 함수가 링크할 심볼을 지정합니다. `wasm_import_module` 블록 안에서는 가져올 이름을 지정합니다.

```rok
#[link(wasm_import_module = "env")]
extern "C" {
    #[link_name = "log_line"]
    fn host_log(ptr: ConstPtr<u8>, len: usize);
}
```

## 6.2.4. `#[inline]` 인라이닝 힌트

```rok
#[inline]
fn small_helper(x: i32) -> i32 { x + 1 }

#[inline(always)]
fn very_hot() { ... }

#[inline(never)]
fn never_inline() { ... }
```

| 인자     | 의미                            |
|----------|---------------------------------|
| (없음)   | 인라이닝 권장 (컴파일러가 결정) |
| `always` | 인라이닝 강제                   |
| `never`  | 인라이닝 금지                   |

`#[inline]`은 최적화 힌트일 뿐입니다. 프로그램의 의미에는 영향을 주지 않습니다.

## 6.2.5. `#[test]` 테스트 함수 마커

```rok
#[test]
fn test_addition() {
    assert_eq(1i32 + 1i32, 2i32);
}
```

`assert_eq`는 [stdlib 2.14.5](../../refs-standard-libraries/200-core/214-error.md#assert--assert_eq--assert_ne)에 있습니다. 함수에 `#[test]`를 붙이면 테스트 함수가 됩니다. `rokc test`가 이런 함수를 모아 하네스를 만들고, 테스트 러너(`rok test` 등)가 자동으로 실행합니다. 일반 빌드에서는 `#[cfg(test)]`가 함께 붙은 것처럼 빠집니다. 테스트 함수는 자신과 바깥 모듈의 가시성과 상관없이 실행됩니다. `#[test]`가 다른 코드에서 보이는 가시성을 바꾸지는 않습니다.

`#[bench]`는 벤치마크 함수를 표시합니다. `#[test]`처럼 `#[cfg(test)]`를 함의합니다. `rokc bench`가 이런 함수를 모아 실행합니다([9.1.2](../009-tools/901-rokc-cli.md#rokc-bench-벤치마크)). 벤치마크 함수는 자신과 바깥 모듈의 가시성과 상관없이 실행됩니다. `#[bench]`가 다른 코드에서 보이는 가시성을 바꾸지는 않습니다.

## 6.2.6. `#[cold]` 드물게 호출되는 함수

```rok
#[cold]
fn handle_error() { ... }
```

`#[cold]`는 그 함수가 드물게 호출된다고 컴파일러에 알립니다. 코드 생성기는 이 정보를 보고 함수를 자주 실행되는 코드 경로 밖에 배치해 캐시 효율을 높입니다.

## 6.2.7. `#[deprecated]` 사용 자제 표시

```rok
#[deprecated]
fn old_api() { ... }

#[deprecated = "use new_api instead"]
fn old_api2() { ... }

impl Config {
    #[deprecated(since = "1.2.0", note = "use `Config::load`")]
    pub fn new() -> Config { ... }
}
```

`#[deprecated]`가 붙은 항목을 쓰면 경고(린트 `deprecated`)가 납니다. 에러는 아닙니다. `= "..."`, `("...")`, `note = "..."` 형태로 메시지를 적으면 경고에 그 메시지도 나옵니다. 메시지에는 대신 쓸 것을 적습니다. `since = "x.y.z"`는 그 항목을 사용 자제로 바꾼 릴리스입니다.

경고는 다음 항목에 납니다. 항목이 선언된 곳(그 패키지, 의존 패키지, 표준 라이브러리)은 가리지 않습니다.

- 함수와 메서드: 호출하거나 값으로 쓸 때. 트레잇 메서드에 붙은 표시는 그 메서드의 모든 impl에 적용됩니다.
- 필드: 읽거나, 대입하거나, struct 리터럴에서 값을 주거나, 패턴에서 이름을 쓸 때.
- struct, enum, union, 타입 별칭, 트레잇: 타입이나 경로의 앞부분(`Old::new`)에서 이름을 쓸 때.
- 상수와 static.

빌드하는 패키지 안의 사용에만 경고가 납니다([6.2.9](#629-allow--deny--warn-린트-제어)). 타입 자신의 `impl`은 그 타입에 대해 경고하지 않고, struct 자신의 메서드는 그 필드에 대해 경고하지 않습니다. 타입은 전체 경로로, 또는 다른 타입과 겹치지 않는 마지막 이름으로 알아봅니다. 이 속성은 어느 항목에나 붙일 수 있습니다. 위에 없는 항목(변형, impl, 모듈)에서는 아무 효과가 없습니다.

`since`는 참고용입니다. 어느 릴리스의 사용 자제든 `deprecated`와 `warnings`에 들어갑니다. 언어 서버는 그 사용에 사용 자제 태그를 달아서 편집기가 취소선으로 보여 줍니다.

## 6.2.8. `#[doc = "..."]` 문서 주석 형태

```rok
#[doc = "이 함수는 두 정수의 합을 반환한다."]
fn add(x: i32, y: i32) -> i32 { x + y }
```

[어휘 구조](../001-lexical/102-lexical-structure.md#문서-주석)의 `///` 문서 주석은 이 속성과 같습니다.

```rok
/// 이 함수는 ...
fn f() { ... }

// 같은 의미
#[doc = "이 함수는 ..."]
fn f() { ... }
```

내부 형태인 `#![doc]`도 `//!`와 효과가 같습니다.

### `#[doc(hidden)]`

```rok
/// dict spread 가 검사하는 것 (lowering 된 코드가 호출).
#[doc(hidden)]
pub const fn spread_misfit<V>(d: Dict<str, V>, key: str, why: str) { .. }
```

`#[doc(hidden)]`은 공개 항목을 API 문서(`rokc doc`)와 언어 서버의 자동 완성에서 뺍니다. 컴파일러가 생성한 코드에서 부를 수 있도록 `pub`으로 둔 항목에 씁니다. 항목은 여전히 공개입니다. 컴파일과 이름 해석도 달라지지 않습니다. 모듈에 붙이면 그 안의 모든 것을 숨깁니다. 항목의 문서 주석과 함께 둘 수 있고, 문서 주석은 호버 정보에 계속 나옵니다. 그 밖의 `#[doc(..)]` 형태는 에러입니다. `#[doc = ".."]`나 `#[doc(hidden)]`으로 쓰세요. toolchain 패키지끼리 공유하지만 사용자가 부르면 안 되는 항목에는 `#[doc(hidden)]`이 아니라 `#[internal]`([6.2.16](#6216-internal-toolchain-패키지끼리-공유))을 붙입니다.

## 6.2.9. `#[allow(...)]` / `#[deny(...)]` / `#[warn(...)]` 린트 제어

```rok
#[allow(unused)]
fn _placeholder() { }

#[deny(deprecated)]
fn strict_zone() { ... }
```

이 속성은 붙은 항목의 범위 안에서 린트 진단의 수준을 바꿉니다. 린트는 패키지를 컴파일하는 모든 명령에서 실행됩니다. `rokc build`, `run`, `test`, `bench`, `emit`, `check`, `lint`([9.1.2](../009-tools/901-rokc-cli.md#rokc-lint-린터))와 언어 서버가 해당합니다. `doc`은 해당하지 않습니다. 빌드하는 패키지만 검사하고 의존 패키지는 검사하지 않습니다. `warn` 린트는 출력만 하고 빌드를 실패로 만들지 않습니다. `deny` 린트는 에러이고, 빌드가 종료 코드 1로 실패합니다. `allow`는 진단을 무시하고, `warn`은 진단을 경고로, `deny`는 진단을 에러로 냅니다. `#![allow(..)]` 같은 내부 형태는 둘러싼 모듈에 적용됩니다. 파일 머리에서는 파일 전체에 적용됩니다. 인자는 린트 이름이나 그룹 이름입니다. 항목에 붙인 `#[allow]`는 그 항목의 변형, 필드, 메서드에도 적용됩니다.

수준은 다음 순서로 정해지고, 뒤에 오는 것이 앞의 것보다 우선합니다. 각 린트의 기본 수준 < `rok.rokon`의 `lints` 필드(`lints_from` 사슬을 적용한 뒤)([9.10.3](../009-tools/910-rok-frontend.md#9103-rokrokon--roklock)) < CLI의 `-A` / `-W` / `-D` < 소스의 속성(가장 안쪽 것이 우선). `lints`는 CLI 플래그 앞에 붙은 것처럼 처리되므로 뒤에 쓴 CLI 플래그가 이 설정을 덮어씁니다. 린트 진단의 코드는 `E0132`입니다.

모든 린트의 기본 수준은 `warn`입니다. 다만 늘 실수인 `nan_comparison`과 `self_assignment`는 `deny`이고, `missing_docs`는 `allow`입니다. 도입 열은 그 린트를 추가한 릴리스입니다.

| 린트                     | 도입    | 검사                                                               |
|--------------------------|---------|--------------------------------------------------------------------|
| `unused_variables`       | 0.1.0   | 읽지 않는 지역 변수나 매개변수. 대입하거나 `ref`로 캡처한 (`[ref n]`) `ref` 매개변수는 사용한 것으로 봄. 그 쓰기가 호출한 쪽에 닿기 때문 |
| `unused_imports`         | 0.1.0   | 모듈이 이름을 쓰지 않는 `use`                                      |
| `dead_code`              | 0.1.0   | 아무도 부르지 않는 비 `pub` 함수                                   |
| `unreachable_code`       | 0.1.0   | `return` / `break` / `continue` / `throw` / `panic` 뒤의 문장      |
| `unused_must_use`        | 0.1.0   | `#[must_use]` 함수의 결과, 또는 `#[must_use]` 타입의 값을 버림 ([6.2.11](#6211-쓰지-않은-결과-unused_result-discardable-must_use)) |
| `unused_result`          | 0.2.0   | 값이 `()`, `none`, `Never`가 아닌 식 문장. `#[discardable]` 함수를 부르는 문장은 제외. 값을 일부러 버리려면 `_ = expr;`로 씀 ([6.2.11](#6211-쓰지-않은-결과-unused_result-discardable-must_use)) |
| `unused_doc_comments`    | 0.1.0   | 문서화 대상이 아닌 것(문, match 가지, struct 리터럴이나 튜플 필드, 식)에 붙은 문서 주석. 일반 주석은 `//`를 씀 |
| `ignored_error`          | 0.1.0   | `try? f();` 문장이 에러를 조용히 버림. `let _ = try? f();`는 실패해도 괜찮은 호출이라는 표시이므로 보고하지 않음 |
| `nan_comparison`         | 0.1.0   | NaN과 비교 (늘 거짓, `!=`는 늘 참). `.is_nan()`을 씁니다 (deny) |
| `bool_comparison`        | 0.1.0   | `bool`을 `true` / `false`와 비교                                 |
| `self_assignment`        | 0.1.0   | `x = x;` (deny)                                                    |
| `while_true`             | 0.1.0   | `loop` 대신 `while true`                                           |
| `deprecated`             | 0.1.0   | `#[deprecated]` 함수, 메서드, 필드, 타입, 상수, static 사용 ([6.2.7](#627-deprecated-사용-자제-표시)) |
| `deprecated_safe`        | 0.1.6   | `unsafe` 밖에서 `#[deprecated_safe]` 함수를 호출하거나 그런 static을 씀 ([6.2.14](#6214-deprecated_safe-지금은-안전-나중에-unsafe)) |
| `redundant_cast`         | 0.1.0   | `x`가 이미 `T`인 `x as T`                                         |
| `redundant_clone`        | 0.1.0   | 원시 타입 값의 `.clone()`                                          |
| `non_snake_case`         | 0.1.0   | 함수, 지역 변수, 매개변수, 모듈 이름이 snake_case가 아님           |
| `non_camel_case_types`   | 0.1.0   | 타입, 트레잇, 변형 이름이 UpperCamelCase가 아님. 첫 글자가 소문자가 아니고, `__`가 없으며, `_`는 대소문자가 없는 문자 사이에만 (`V1_0`) |
| `non_upper_case_globals` | 0.1.0   | `const` / `static` 이름이 UPPER_SNAKE_CASE가 아님                  |
| `confusable_idents`      | 0.1.0   | 서로 닮은 식별자 ([1.4](../001-lexical/104-identifiers.md))        |
| `missing_f_prefix`       | 0.1.0   | 범위 안의 지역 변수 `name`을 가리키는 `{name}`을 담은 보통 문자열 리터럴: `f` 접두어가 빠짐 |
| `literal_spread_mismatch` | 0.1.6   | 고정 매개변수에 펼친 `List`나 Dict 리터럴이 자리를 채울 수 없음: 길이가 다르거나 남은 매개변수 어느 것의 이름도 아닌 키가 있음([4.2](../004-functions/402-parameters.md)). 그 호출은 런타임에 panic이 일어남 |
| `assoc_type_shadowing`   | 0.2.0   | 연관 타입이 바깥 타입과 이름이 같아 트레이트나 `impl` 안에서 그 타입을 가림 |
| `missing_docs`           | 0.2.0   | `///` 문서 주석이 없는 `pub` 항목, 고유 impl의 `pub` 메서드, `pub` 트레잇의 메서드, 인라인 `pub mod`. `#[doc(hidden)]`은 문서가 있는 것으로 보고, 트레잇 impl의 메서드는 검사하지 않음 (allow) |

0.2.0에는 0.1.6이 0.2.0에 앞서 더한 경고가 하나도 남아 있지 않습니다. 모두 에러가 되었거나 더는 해당하지 않습니다. 에러가 된 것: `keyword_idents`([1.5.6](../001-lexical/105-keywords.md)), `zero_prefixed_literal`([1.6.1](../001-lexical/106-numeric-literals.md)), `try_coalesce_precedence`([3.14.3](../003-expressions/314-error-control-flow.md)), `trailing_closure_return`([4.3.11](../004-functions/403-return.md)), `capture_field_write`([4.6.7](../004-functions/406-capture-semantics.md)), `const_item_mutation`([5.2](../005-modules/502-const.md)), `pattern_binds_const`([3.11](../003-expressions/311-patterns.md)), `ambiguous_int_literal`([1.6.3](../001-lexical/106-numeric-literals.md#163-접미사-없는-리터럴)), `finally_jump`([3.14.5](../003-expressions/314-error-control-flow.md)), `float_to_bool`([2.14](../002-types/214-cast-and-conversion.md)), `unclosed_sender`와 `using_twice`([stdlib 2.13.4](../../refs-standard-libraries/200-core/213-marker.md)), `guard_escape`와 `thread_local_across_await`([stdlib 2.16](../../refs-standard-libraries/200-core/216-sync.md)), `extern_static_unsafe`([5.9.6](../005-modules/509-extern-block.md#596-호출--사용)), `improper_ctypes`([5.9.4](../005-modules/509-extern-block.md#594-ffi-호환-타입)), `trait_not_in_scope`([3.5](../003-expressions/305-member-access-and-method-call.md)), `unqualified_meta`([6.8.11](608-meta-blocks.md#6811-등록가시성과-결합)). 더는 해당하지 않는 것: `trailing_closure_operator`(`}` 뒤의 연산자는 새 문장을 시작함, [1.3.4](../001-lexical/103-statement-terminator.md)), `implicit_core_fn`(prelude, [stdlib 0.2](../../refs-standard-libraries/000-meta/002-import.md)), `discarded_ref_write`(원소에 되씀, [4.7](../004-functions/407-ref-keyword.md)), `fstring_g_precision`(`g`는 일반 형식, [1.7.3](../001-lexical/107-string-literals.md#173-형식-문자열)). 이 이름들과 `unclosed_resource`, `shadowed_loop_var`(`let`이 이름을 다시 선언할 수 있음, [5.1.4](../005-modules/501-variable-declaration.md#514-같은-스코프-안의-재선언))는 제거된 린트 이름이라 `#[allow]`, `-A` / `-W` / `-D`, `lints`에 써도 받아들이고 무시합니다. `deprecated_safe`와 `literal_spread_mismatch`(0.1.6)는 린트로 남습니다. 0.1.0의 린트 `write_to_copy`도 0.2.0에서 에러가 됩니다. 아무도 나중에 읽지 않는 사본에 필드를 쓰는 코드입니다([2.3.8](../002-types/203-struct.md#238-값-의미)). 이 이름도 제거된 린트 이름입니다.

| 그룹                | 포함                                                                                         |
|---------------------|----------------------------------------------------------------------------------------------|
| `unused`            | `unused_variables`, `unused_imports`, `dead_code`, `unreachable_code`, `unused_must_use`, `unused_result`, `unused_doc_comments` |
| `nonstandard_style` | `non_snake_case`, `non_camel_case_types`, `non_upper_case_globals`                           |
| `warnings`          | 수준이 warn인 모든 린트. 이후 릴리스가 추가하는 린트도 포함                                  |
| `all`               | 이후 릴리스가 추가하는 린트까지 포함한 모든 린트                                             |

`warnings`는 rustc와 같은 뜻입니다. 이 그룹을 deny하면(`lints`의 `"warnings": "deny"`, `-D warnings`, `#![deny(warnings)]`) 모든 경고가 에러가 되고, 이후 릴리스가 추가하는 린트와 사용 자제도 마찬가지입니다. 릴리스가 바뀌어도 계속 빌드되어야 하는 프로젝트는 툴체인을 고정합니다(`rok.rokon`의 `toolchain`, [9.10.6](../009-tools/910-rok-frontend.md)). 린트마다 적힌 릴리스 열은 참고용입니다.

어떤 경고는 린트 단계가 아니라 컴파일러가 직접 내지만, 그래도 린트에 속합니다. 닮은 식별자는 `confusable_idents`이고, 잘못 놓인 문서 주석은 `unused_doc_comments`입니다. 이 경고의 수준도 다른 린트처럼 린트 이름으로 정합니다.

린트 진단은 수준이 어디서 왔는지 알려 줍니다. `` `#[warn(unused_variables)]` on by default ``, `` `-D unused` on the command line ``, 매니페스트에서 온 수준이면 `` `"unused_variables": "deny"` in `lints` of rok.rokon ``, 속성이 정했으면 그 속성에 붙은 레이블입니다. 모르는 린트나 그룹 이름은 경고이며, 가장 가까운 이름을 알려 주고 `rokc lint --list`를 가리킵니다. 제거된 린트(대부분 오류가 됨)의 이름은 속성, `-A` / `-W` / `-D`, `lints` 어디에 써도 받아들여지고 아무 일도 하지 않습니다. `-D warnings`에서도 보고하지 않습니다.

`rokc lint --list`는 린트를 기본 수준, 추가된 릴리스와 함께 출력하고, 이어서 그룹을 출력합니다.

`rokc check --check-library`와 `rokc lint --check-library`([9.1.2](../009-tools/901-rokc-cli.md))는 표준 라이브러리 자체의 코드도 그 수준으로 검사합니다. 모든 린트는 기본 수준이고, `missing_docs`만 `deny`입니다. 표준 라이브러리의 모든 공개 항목에는 문서 주석이 있습니다([stdlib 0.5.14](../../refs-standard-libraries/000-meta/005-notation.md#0514-문서와-속성)).

## 6.2.10. `#[non_exhaustive]` 외부 완전성 검사 회피

```rok
#[non_exhaustive]
pub enum ApiError {
    NotFound,
    Unauthorized,
}
```

`#[non_exhaustive]`는 `enum`이나 `struct`에 붙일 수 있습니다. `enum`에 붙이면 외부 패키지에서 그 타입을 `match`할 때 모든 변형을 처리하더라도 나머지 가지(`_ => ...`)를 더 써야 합니다. 덕분에 라이브러리 작성자는 SemVer 호환성을 깨지 않고 나중에 새 변형을 추가할 수 있습니다.

```rok
#[non_exhaustive]
pub struct ClientOptions {
    pub retries: i32 = 3;
    pub verbose: bool = false;
}

let o = ClientOptions { retries: 5 };           // 다른 패키지에서도 OK
let p = ClientOptions { verbose: true, ..o };   // OK
let ClientOptions { retries, .. } = p;          // 다른 패키지에서는 `..` 필요
```

`struct`에 붙이면 그 패키지가 나중에 필드를 더할 수 있는 레코드라는 뜻입니다.

- 외부 패키지에서 쓰는 struct 패턴(`let`, `match`, `if let`)은 `..`로 끝나야 합니다. `..`가 없으면 에러이고, 에러 메시지는 `..`를 붙인 패턴을 보여 줍니다. 정의한 패키지 안에서는 모든 필드를 나열해도 됩니다.
- struct를 처음 릴리스한 뒤에 더하는 필드는 모두 기본값([2.3.3](../002-types/203-struct.md#233-필드-기본값))을 가집니다. 그래서 그 필드를 생략한 리터럴은 계속 컴파일됩니다. 컴파일러는 struct의 이력을 볼 수 없으므로 이 규칙은 작성자가 지킵니다. 나중에 semver 검사가 이를 확인합니다.
- Rust와 달리 다른 패키지에서도 struct 리터럴과 `..base` spread([2.3.4](../002-types/203-struct.md#234-필드-spread))를 쓸 수 있습니다. 리터럴은 기본값이 있는 필드를 생략하고 나머지를 적습니다. 마땅한 기본값이 없는 필드는 모든 리터럴에 적어야 합니다.

## 6.2.11. 쓰지 않은 결과: `unused_result`, `#[discardable]`, `#[must_use]`

```rok
fn calculate() -> i32 { ... }

#[discardable]
fn insert(self, k: K, v: V) -> V | none { ... }

#[must_use("thread is never joined")]
pub struct JoinHandle<T> { ... }
```

값이 `()`, `none`, `Never`가 아닌 식 문장은 `unused_result` 린트로 경고합니다. `calculate();`는 `i32`를 버립니다. `try`, `try!`, `await`를 거친 값은 그 호출의 값으로 봅니다. `try? f();` 문장은 대신 `ignored_error`가 보고합니다. 블록 같은 문장(`if`, `match`, 블록, 루프)은 검사하지 않습니다. 그 값은 각 가지의 값이기 때문입니다. 값을 일부러 버리려면 `_ = expr;`([4.2](../004-functions/402-parameters.md))나 `let _ = expr;`로 씁니다.

함수나 메서드에 붙인 `#[discardable]`은 그 결과를 버려도 된다는 뜻입니다. 이 함수를 부르는 문장은 경고하지 않습니다. 인자는 받지 않습니다. 트레잇 메서드에 붙이면 모든 impl에 적용됩니다. 표준 라이브러리는 밀어낸 값이나 부수 정보를 돌려주는 메서드에 이 속성을 붙입니다. `Dict::insert` / `remove`, `Set::insert` / `remove`, `List::pop` / `remove` / `drain`, `Deque::pop_front` / `pop_back`, `BinaryHeap::pop`, `Atomic::swap` / `fetch_*`, `Cell::replace` / `take`, `marker::replace` / `take`, `mem::replace`, `Dir::copy`, `fs::copy`, `io::copy`, `io::copy_async`가 해당합니다. 짧은 쓰기(`Write::write -> usize`), `checked_*`, `str::trim` 같은 순수 메서드는 버릴 수 있는 결과가 아닙니다.

`#[must_use]`는 함수나 메서드, `struct`, `enum`에 붙일 수 있고, 메시지를 붙일 수 있습니다(`#[must_use("..")]`). 함수에 붙이면 메시지를 더합니다. 그 결과를 버리면 `unused_result` 대신 `unused_must_use` 경고가 납니다. 타입에 붙이면 더 강합니다. 그 타입의 값을 내는 식 문장과, 그 타입이면서 한 번도 읽지 않는 지역 변수에 `unused_must_use` 경고가 납니다. `#[discardable]` 함수가 돌려준 값이어도 마찬가지입니다. 경고에는 속성의 메시지가 함께 나옵니다. `async fn` 호출이나 `async` 블록의 값과 `thread::JoinHandle`([stdlib 3.4.2](../../refs-standard-libraries/300-std/304-thread.md#342-api))은 `#[must_use]`입니다. 기다리지 않은 future는 실행되지 않습니다. 합류하지 않은 스레드는 프로그램이 끝나기 전에 끝나지 않을 수 있습니다. `detach`는 스레드를 따로 실행한다는 뜻을 밝힙니다. 나머지는 [stdlib 0.5.14](../../refs-standard-libraries/000-meta/005-notation.md#0514-문서와-속성)에 있습니다. `#[must_use]` 타입은 속성이 붙은 바로 그 타입입니다. 이름이 같은 다른 패키지의 타입은 해당하지 않습니다.

타입의 메시지는 "this" 뒤에 이어지도록 씁니다. 경고는 `this <메시지>`로 나옵니다. 처음 나오는 `": "` 뒤의 글은 `help:` 줄이 되고, 그 안의 `{}`는 버려진 식의 소스(지역 변수면 그 이름)로 바뀝니다. `#[must_use("future is never awaited, so its body never runs: write `await {}`")]`는 ``this future is never awaited, so its body never runs`` 경고와 ``help: write `await fetch()` ``를 냅니다. 메시지가 없으면 경고가 타입을 밝힙니다. ``this value of type `T` is never used``.

## 6.2.12. `#[track_caller]` 호출 위치 전달

```rok
#[track_caller]
fn expect_positive(n: i32) -> i32 {
    if n <= 0 { panic("not positive") }      // 위치: expect_positive 를 부른 곳
    n
}
```

- 함수는 호출 위치를 숨겨진 마지막 `str` 인자로 받습니다. 함수 안의 panic은 그 위치를 보고합니다([4.14.13](../004-functions/414-throws.md#41413-panic과-fatal-에러)).
- 추적 대상 함수가 다른 추적 대상 함수를 부르면 받은 위치를 그대로 넘깁니다.
- 추적 대상 함수를 함수 값으로 쓰면 그 값을 만든 위치를 보고합니다.
- `async fn`에 붙이면 호출만 추적하고 코루틴 본체는 추적하지 않습니다.
- `extern`(런타임) 선언에 붙이면 그 심볼이 위치 인자를 하나 더 받습니다([8.5.7](../008-abi/805-extern-symbols.md#857-진단-영역)).
- `main`, `extern "C" fn`, `#[export_name]` 함수에는 붙일 수 없습니다.

잘못된 입력에 panic을 일으키는 표준 라이브러리 함수(`panic`, `unreachable`, `assert*`, 인덱싱, 슬라이싱, `unwrap`, `List::insert` / `remove` 등)는 `#[track_caller]`입니다.

## 6.2.13. 정리

위의 속성은 컴파일러와 도구가 직접 인식하는 내장 속성입니다. 메타 블록으로 따로 정의하지 않아도 됩니다.

`#[lang]`, `#[literal]` 같은 매직 속성은 언어의 의미와 결합되어 있으므로 다른 절에서 정의합니다.

### 그 밖의 내장 속성

컴파일러는 다음 속성도 인식합니다. 각 속성은 해당 기능을 다루는 곳에서 정의합니다.

| 속성                           | 붙는 곳                       | 의미 |
|--------------------------------|-------------------------------|------|
| `#[path("file.rok")]`          | `mod name;`                   | `name.rok` 대신 그 파일에서 모듈을 읽음 ([5.4.6](../005-modules/504-modules.md#546-사용자-정의-경로-path)) |
| `#![no_implicit_rok]`          | 파일 (내부)                   | 그 파일의 묵시적 `use rok::core::prelude::*;`를 끔 ([stdlib 0.2.6](../../refs-standard-libraries/000-meta/002-import.md#026-적용-해제-no_implicit_rok)) |
| `#[export_name("sym")]`        | 함수                          | 함수를 맹글링하지 않은 심볼 `sym`으로 내보냄 (wasm32에서는 그 이름의 wasm export). 도달 불가 함수 제거의 루트 ([9.5.8](../009-tools/905-symbol-mangling.md)) |
| `#[no_mangle]`                 | 함수                          | 함수를 이름 린트와 미사용 함수 린트에서 뺌. 심볼은 바꾸지 않음. 심볼 이름은 `#[export_name]`으로 정함 |
| `#[wit::export]`               | 함수, `pub struct`            | WIT export를 구현함. 함수 또는 resource ([stdlib 1.8](../../refs-standard-libraries/100-bare/108-wit.md)) |
| `#[niche(..)]`                 | newtype struct                | 타입에 없는 값을 선언함. 그 값이 `T \| none`의 niche가 됨 ([2.7.4](../002-types/207-niche-optimization.md#274-opt-in-niche-niche)) |
| `#[intrinsic("rok.OP.TYPE")]`  | 표준 라이브러리 전용          | 함수나 메서드를 컴파일러 연산에 묶음. 사용자 코드에서는 에러 ([stdlib 0.3](../../refs-standard-libraries/000-meta/003-intrinsics.md)) |
| `#[no_safepoint]`              | `extern "rok"` 블록           | toolchain 내부용. 블록의 함수는 할당도 블로킹도 하지 않으므로 그 호출은 safepoint가 아님. 다른 블록에서는 에러 ([5.9.2](../005-modules/509-extern-block.md#592-abi-명시)) |
| `#[sealed]`                    | `trait`                       | 정의한 패키지만 그 트레잇을 구현함 ([6.2.15](#6215-sealed-정의한-패키지에서만-구현)) |
| `#[internal]`                  | 함수, 메서드, `const`, `static`, 필드 | `rok::bare`, `rok::core`, `rok::std`끼리만 공유함 ([6.2.16](#6216-internal-toolchain-패키지끼리-공유)) |

## 6.2.14. `#[deprecated_safe]` 지금은 안전, 나중에 `unsafe`

```rok
#[deprecated_safe("`ptr` must point at a NUL-terminated string")]
pub fn from_ptr(ptr: Ptr<u8>) -> CStr { ... }
```

`#[deprecated_safe("note")]`는 나중 릴리스에서 `unsafe`가 될 라이브러리의 함수, 메서드, extern 함수, static을 표시합니다. `unsafe` 블록이나 `unsafe fn` 밖에서 그 함수를 호출하거나 그 static을 쓰면 린트 `deprecated_safe` 경고가 메시지와 함께 납니다. 호출을 `unsafe { .. }`로 감싸면 됩니다. 이 코드는 지금도 컴파일되고, 항목이 `unsafe`가 된 뒤에도 그대로 맞습니다. `unsafe` 안에서는 경고가 나지 않습니다.

이 속성은 표준 라이브러리와 라이브러리 패키지용입니다. 빌드하는 프로그램의 함수에 붙이면 에러입니다. 그 함수는 `unsafe fn`으로 만듭니다. 함수, 메서드, extern 함수, static이 아닌 항목에 붙여도 에러입니다.

## 6.2.15. `#[sealed]` 정의한 패키지에서만 구현

```rok
/// `Atomic<T>`가 지원하는 원소 타입.
#[sealed]
pub trait AtomicPrimitive {}
```

트레잇에 `#[sealed]`를 붙이면 그 트레잇을 정의한 패키지만 구현할 수 있습니다. 다른 패키지에서 그 트레잇을 `impl`하면 에러입니다. 에러는 트레잇이 sealed라는 것과 그 트레잇을 가진 패키지를 알려 줍니다. 그 밖에는 다른 트레잇과 똑같이 동작합니다. 다른 패키지도 바운드(`T: AtomicPrimitive`)에 쓰고, 메서드를 부르고, `use`로 가져올 수 있습니다.

라이브러리가 그 트레잇을 구현한 타입을 모두 알아야 할 때 트레잇을 sealed로 만듭니다. 건전성 때문일 수도 있고(원자 연산은 그 타입에만 있습니다), 다른 impl은 무시되기 때문일 수도 있습니다. 표준 라이브러리는 `AtomicPrimitive`를 sealed로 둡니다([stdlib 2.16](../../refs-standard-libraries/200-core/216-sync.md)).

toolchain의 각 계층은 저마다 별개의 패키지입니다([5.6.4](../005-modules/506-visibility.md#564-pubpkg-패키지-내부)). `rok::core`가 sealed로 둔 트레잇의 impl은 `rok::core`에 있습니다. `#[sealed]`는 인자를 받지 않으며, 트레잇이 아닌 곳에 붙이면 에러입니다.

## 6.2.16. `#[internal]` toolchain 패키지끼리 공유

```rok
/// `fd`의 준비를 기다리는 task를 모두 깨움.
#[internal]
pub fn wake_io(fd: i32) { .. }
```

`#[internal]`은 toolchain의 패키지인 `rok::bare`, `rok::core`, `rok::std`가 서로 공유하지만 API는 아닌 `pub` 항목을 표시합니다. 함수, 메서드, `const`, `static`, 필드에 붙일 수 있습니다.

- 다른 패키지에서 쓰면(호출, 함수 값, 읽기) "internal to the Rok toolchain" 에러입니다. 항목은 그대로 해소되므로 에러가 항목 이름을 알려 줍니다.
- `#[internal("..")]`은 대신 쓸 것을 적고, 에러가 그것을 보여 줍니다(`Int::from_literal`은 `s.parse::<Int>()`를 알려 줍니다). 설명이 없으면 에러는 그 항목이 컴파일러 전용이라고 알려 줍니다.
- `#[doc(hidden)]` 항목처럼 API 문서(`rokc doc`), 언어 서버의 자동 완성과 호버 목록, "did you mean" 제안에서 빠집니다.
- 새 가시성 단계가 아닙니다([5.6.1](../005-modules/506-visibility.md#561-네-단계)). 항목은 `pub`이고, 이 검사는 가시성과 따로 이루어집니다.
- toolchain의 패키지만 이 속성을 쓸 수 있습니다. 다른 패키지의 항목에 붙이면 에러입니다. 항목을 패키지 안에만 두려면 `pub(pkg)`로 만드세요.

이 속성이 붙은 항목은 [stdlib 0.4.4](../../refs-standard-libraries/000-meta/004-lang-slot-table.md)에 있습니다. 한 계층 안에서만 쓰는 항목은 `pub(pkg)`이고, 컴파일러가 lowering한 코드에서 부르는 함수는 lang 슬롯으로 찾으므로 둘 다 `#[internal]`이 필요 없습니다.
