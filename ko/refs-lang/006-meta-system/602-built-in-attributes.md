# 6.2. 내장 속성

이 절은 컴파일러가 직접 처리하는 내장 속성을 정의합니다. `#[cfg]`는 [조건부 컴파일](603-cfg.md)에서 따로 다룹니다. 매직 속성과 메타 블록은 [`#[lang(...)]` 매직](604-lang-magic.md)부터 이어지는 절에서 설명합니다.

```
derive_attr     ::= "#" "[" "derive" "(" trait_name ("," trait_name)* ","? ")" "]"
trait_name      ::= identifier
repr_attr       ::= "#" "[" "repr" "(" repr_arg ("," repr_arg)* ")" "]"
repr_arg        ::= "C" | "transparent" | "packed" | "align" "(" int_lit ")"
                  | identifier                          // primitive int repr
inline_attr     ::= "#" "[" "inline" ("(" ("always" | "never") ")")? "]"
deprecated_attr ::= "#" "[" "deprecated" ("(" string_lit ")")? "]"
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
| `transparent`               | 크기가 0이 아닌 필드 정확히 하나. 그 필드와 같은 표현이고 C ABI에서도 그 필드처럼 건넘                  |
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
```

`#[deprecated]`가 붙은 항목을 쓰면 경고가 납니다. 에러는 아닙니다. `= "..."` 형태로 메시지를 적으면 경고에 그 메시지도 나옵니다.

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

## 6.2.9. `#[allow(...)]` / `#[deny(...)]` / `#[warn(...)]` 린트 제어

```rok
#[allow(unused)]
fn _placeholder() { }

#[deny(deprecated)]
fn strict_zone() { ... }
```

이 속성은 붙은 항목의 범위 안에서 린트 진단의 수준을 바꿉니다. 린트는 패키지를 컴파일하는 모든 명령에서 실행됩니다. `rokc build`, `run`, `test`, `bench`, `emit`, `check`, `lint`([9.1.2](../009-tools/901-rokc-cli.md#rokc-lint-린터))와 언어 서버가 해당합니다. `doc`은 해당하지 않습니다. 빌드하는 패키지만 검사하고 의존 패키지는 검사하지 않습니다. `warn` 린트는 출력만 하고 빌드를 실패로 만들지 않습니다. `deny` 린트는 에러이고, 빌드가 종료 코드 1로 실패합니다. `allow`는 진단을 무시하고, `warn`은 진단을 경고로, `deny`는 진단을 에러로 냅니다. `#![allow(..)]` 같은 내부 형태는 둘러싼 모듈에 적용됩니다. 파일 머리에서는 파일 전체에 적용됩니다. 인자는 린트 이름이나 그룹 이름입니다. 항목에 붙인 `#[allow]`는 그 항목의 변형, 필드, 메서드에도 적용됩니다.

수준은 다음 순서로 정해지고, 뒤에 오는 것이 앞의 것보다 우선합니다. 각 린트의 기본 수준 < `rok.rokon`의 `lints` 필드(`lints_from` 사슬을 적용한 뒤)([9.10.3](../009-tools/910-rok-frontend.md#9103-rokrokon--roklock)) < CLI의 `-A` / `-W` / `-D` < 소스의 속성(가장 안쪽 것이 우선). `lints`는 CLI 플래그 앞에 붙은 것처럼 처리되므로 뒤에 쓴 CLI 플래그가 이 설정을 덮어씁니다. 린트 진단의 코드는 `E503`입니다.

모든 린트의 기본 수준은 `warn`입니다.

| 린트                     | 검사                                                               |
|--------------------------|--------------------------------------------------------------------|
| `unused_variables`       | 읽지 않는 지역 변수나 매개변수                                     |
| `unused_imports`         | 모듈이 이름을 쓰지 않는 `use`                                      |
| `dead_code`              | 아무도 부르지 않는 비 `pub` 함수                                   |
| `unreachable_code`       | `return` / `break` / `continue` / `throw` / `panic` 뒤의 문장      |
| `unused_must_use`        | `#[must_use]` 함수의 결과, 또는 `#[must_use]` 타입의 값을 버림 ([6.2.11](#6211-must_use-결과-무시-경고)) |
| `unused_doc_comments`    | 문서화 대상이 아닌 것(문, match 가지, struct 리터럴이나 튜플 필드, 식)에 붙은 문서 주석. 일반 주석은 `//`를 씀 |
| `write_to_copy`          | 아무도 나중에 읽지 않는 사본에 필드를 대입함. 사본은 값으로 받은 매개변수나 `self`, `for` 바인딩, 원소의 `let`(`xs[i]`, [2.3.8](../002-types/203-struct.md#238-값-의미))이고, 이후의 읽기, 루프 반복, 반환, 호출 어느 것도 그 변경을 볼 수 없을 때. 같은 변수에 다시 쓰는 데에만 쓰이는 읽기는 세지 않음. 클로저나 코루틴 블록이 언급하는 변수는 검사하지 않음 |
| `ignored_error`          | `try? f();` 문장이 에러를 조용히 버림. `let _ = try? f();`는 실패해도 괜찮은 호출이라는 표시이므로 보고하지 않음 |
| `shadowed_loop_var`      | `for` 본체의 `let`이 루프 변수를 다시 묶음                        |
| `nan_comparison`         | NaN과 비교 (늘 거짓, `!=`는 늘 참)                               |
| `none_comparison`        | `x is none` 대신 `x == none`                                       |
| `bool_comparison`        | `bool`을 `true` / `false`와 비교                                 |
| `self_assignment`        | `x = x;`                                                           |
| `unclosed_resource`      | `using`으로 해제하지 않은 `File` / `Dir` / `ReadDir` / `TcpStream` / `TcpListener` / `UdpSocket` / `Child` (warn) |
| `while_true`             | `loop` 대신 `while true`                                           |
| `deprecated`             | `#[deprecated]` 함수 사용                                          |
| `redundant_cast`         | `x`가 이미 `T`인 `x as T`                                         |
| `redundant_clone`        | 원시 타입 값의 `.clone()`                                          |
| `non_snake_case`         | 함수, 지역 변수, 매개변수, 모듈 이름이 snake_case가 아님           |
| `non_camel_case_types`   | 타입, 트레잇, 변형 이름이 UpperCamelCase가 아님. 첫 글자가 소문자가 아니고, `__`가 없으며, `_`는 대소문자가 없는 문자 사이에만 (`V1_0`) |
| `non_upper_case_globals` | `const` / `static` 이름이 UPPER_SNAKE_CASE가 아님                  |
| `confusable_idents`      | 서로 닮은 식별자 ([1.4](../001-lexical/104-identifiers.md))        |
| `missing_f_prefix`       | 범위 안의 지역 변수 `name`을 가리키는 `{name}`을 담은 보통 문자열 리터럴: `f` 접두어가 빠짐 (warn) |

| 그룹                | 포함                                                                                         |
|---------------------|----------------------------------------------------------------------------------------------|
| `unused`            | `unused_variables`, `unused_imports`, `dead_code`, `unreachable_code`, `unused_must_use`, `unused_doc_comments` |
| `nonstandard_style` | `non_snake_case`, `non_camel_case_types`, `non_upper_case_globals`                           |
| `all`               | 모든 린트                                                                                    |

`write_to_copy`의 help는 `ref self` / `ref p`, 인덱스 루프(`for i in 0..xs.len() { xs[i].f = .. }`), 또는 원소를 다시 써 넣는 방법을 제안합니다.

`rokc lint --list`는 린트, 그룹, 기본 수준을 출력합니다.

## 6.2.10. `#[non_exhaustive]` 외부 완전성 검사 회피

```rok
#[non_exhaustive]
pub enum ApiError {
    NotFound,
    Unauthorized,
}
```

`#[non_exhaustive]`는 `enum`이나 `struct`에 붙일 수 있습니다. 이 속성이 있으면 외부 패키지에서 그 타입을 `match`할 때 모든 변형을 처리하더라도 나머지 가지(`_ => ...`)를 더 써야 합니다. 덕분에 라이브러리 작성자는 SemVer 호환성을 깨지 않고 나중에 새 변형을 추가할 수 있습니다.

## 6.2.11. `#[must_use]` 결과 무시 경고

```rok
#[must_use]
fn calculate() -> i32 { ... }
```

`#[must_use]`가 붙은 함수의 반환값을 쓰지 않고 버리면 경고가 납니다. 결과를 일부러 무시하려면 [매개변수](../004-functions/402-parameters.md)에서 설명한 `_ = ...`를 쓰세요.

`#[must_use]`는 `struct`나 `enum`에도 붙일 수 있습니다. 그러면 그 타입의 값을 내는 식 문장과, 그 타입이면서 한 번도 읽지 않는 지역 변수에 `unused_must_use` 경고가 납니다. 경고에는 속성의 메시지가 함께 나옵니다. `async fn` 호출이나 `async` 블록의 값과 `thread::JoinHandle`([stdlib 3.4.2](../../refs-standard-libraries/300-std/304-thread.md#342-api))은 `#[must_use]`입니다. 기다리지 않은 future는 실행되지 않습니다. 합류하지 않은 스레드는 프로그램이 끝나기 전에 끝나지 않을 수 있습니다. `detach`는 스레드를 따로 실행한다는 뜻을 밝힙니다.

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
