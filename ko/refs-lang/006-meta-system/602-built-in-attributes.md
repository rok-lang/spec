# 6.2. 내장 속성

이 절에서는 컴파일러가 직접 처리하는 내장 속성을 정의합니다. `#[cfg]` 는 [조건부 컴파일](603-cfg.md) 에서 따로 다루며, 매직 속성과 메타 블록에 관한 내용은 [`#[lang(...)]` 매직](604-lang-magic.md) 이후의 절에서 이어 설명합니다.

```
derive_attr     ::= "#" "[" "derive" "(" trait_name ("," trait_name)* ","? ")" "]"
trait_name      ::= identifier
repr_attr       ::= "#" "[" "repr" "(" repr_arg ("," repr_arg)* ")" "]"
repr_arg        ::= "C" | "transparent" | "packed" | "align" "(" int_lit ")"
                  | identifier                          // primitive int repr
inline_attr     ::= "#" "[" "inline" ("(" ("always" | "never") ")")? "]"
deprecated_attr ::= "#" "[" "deprecated" ("(" string_lit ")" | "=" string_lit)? "]"
```

## 6.2.1. `#[derive(...)]` 자동 트레잇 구현

```rok
#[derive(Clone, PartialEq, Eq, Hash, Debug)]
struct Point { x: i64; y: i64; }
```

`#[derive(...)]` 의 인자에는 트레잇 이름들을 콤마로 구분하여 나열합니다. 그러면 나열된 각 트레잇에 대해 해당 타입에 어울리는 자동 구현이 생성됩니다.

### 내장 derive 가능 트레잇

`Clone`, `Copy`, `Debug`, `Default`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash` 만 derive 할 수 있습니다. 각 트레잇의 생성 규칙과 요구 조건은 [stdlib 2.13.11](../../refs-standard-libraries/200-core/213-marker.md#21311-derive-가능한-트레잇) 에서 정의합니다.

- `Display` 는 derive 할 수 없습니다. 표시 형식은 타입 작성자가 정합니다 ([stdlib 2.10](../../refs-standard-libraries/200-core/210-fmt.md)).
- `Eq` / `Ord` 를 상위 트레잇 (`PartialEq` / `PartialOrd`, `Eq`) 없이 derive 하면 컴파일 에러입니다. `Copy` 는 `Clone` 이 필요하고 `Drop` 과 함께 쓸 수 없습니다.
- `union` 에 `#[derive]` 를 붙이면 컴파일 에러입니다.

### 사용자 정의 derive 와의 결합

`#[derive(MyTrait)]` 처럼 사용자 정의 트레잇도 derive 할 수 있습니다. 다만 이렇게 사용하려면 `MyTrait` 가 `#[impl_derive(MyTrait)]` 메타 블록 ([6.8](608-meta-blocks.md)) 으로 정의되어 있어야 합니다. 위의 내장 derive 이름은 메타 블록이 다시 정의할 수 없습니다.

내장 derive 와 사용자 정의 derive 는 하나의 `#[derive(...)]` 안에 자유롭게 섞어 사용할 수 있습니다.

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
| `transparent`               | 필드 정확히 하나, 다른 `repr` 인자 없음. 그 필드와 같은 표현이며 C ABI 를 그 필드로 건넘             |
| `packed`                    | 정렬을 무시하고 빽빽이 묶음 ([FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 한정). GC 참조를 담을 수 없고 `align(N)` 과 함께 쓸 수 없음 |
| `align(N)`                  | 최소 정렬을 N 바이트로 강제                                                                          |
| `u8` ~ `u64` / `i8` ~ `i64` | `enum` 의 tag 타입. 32 비트가 아닌 폭은 필드 없는 enum 만; 데이터 enum 은 `i32` / `u32` |

`#[repr]` 가 없으면 [`struct`](../002-types/203-struct.md) 는 선언 순서와 자연 정렬 (`repr(C)` 와 같음) 이고, [`enum`](../002-types/204-enum.md) 과 합 타입의 레이아웃은 컴파일러가 정합니다 ([2.7](../002-types/207-niche-optimization.md)).

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

`#[link(...)]` 는 [`extern` 블록](../005-modules/509-extern-block.md) 이 연결될 외부 라이브러리를 명시하는 속성으로, `extern` 블록에 부착해 사용합니다.

| 인자 형식                   | 의미                        |
|-----------------------------|-----------------------------|
| `"libname"`                 | 단순 라이브러리 이름        |
| `name = "..."`              | 라이브러리 이름 (key-value) |
| `kind = "static" / "dylib"` | 링크 종류                   |
| `wasm_import_module = "m"` | wasm32: 블록의 함수를 core module `m` 에서 import. 그 밖에서는 무시 |

### `#[link_name = "sym"]`

extern 함수의 `#[link_name = "sym"]` 은 그 함수가 링크하는 심볼을, `wasm_import_module` 블록 안에서는 import 이름을 줍니다.

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

`#[inline]` 은 어디까지나 최적화에 대한 힌트일 뿐이며, 프로그램의 의미 자체에는 영향을 주지 않습니다.

## 6.2.5. `#[test]` 테스트 함수 마커

```rok
#[test]
fn test_addition() {
    assert_eq(1i32 + 1i32, 2i32);
}
```

`assert_eq` 는 [stdlib 2.14.5](../../refs-standard-libraries/200-core/214-error.md#assert--assert_eq--assert_ne) 에 있습니다. `#[test]` 를 함수에 부착하면 그 함수가 테스트 함수임을 표시합니다. `rokc test` 가 이러한 함수들을 하네스로 수집하고, 테스트 러너 (`rok test` 등) 가 자동으로 실행합니다. 일반 빌드에서는 마치 `#[cfg(test)]` 가 함께 붙어 있는 것처럼 제외되니 참고합니다. 테스트 함수는 자신과 바깥 모듈의 가시성과 관계없이 실행되며, `#[test]` 가 다른 코드에 대한 가시성을 바꾸지는 않습니다.

`#[bench]` 는 벤치마크 함수를 표시하며, `#[test]` 와 같이 `#[cfg(test)]` 를 함의합니다. `rokc bench` 가 모아 실행합니다 ([9.1.2](../009-tools/901-rokc-cli.md#rokc-bench-벤치마크)). 벤치마크 함수는 자신과 바깥 모듈의 가시성과 관계없이 실행되며, `#[bench]` 가 다른 코드에 대한 가시성을 바꾸지는 않습니다.

## 6.2.6. `#[cold]` 드물게 호출되는 함수

```rok
#[cold]
fn handle_error() { ... }
```

`#[cold]` 는 해당 함수가 드물게 호출된다는 사실을 컴파일러에게 알려 주는 속성입니다. 이 정보를 받은 코드 생성기는 해당 함수를 핵심 코드 경로 바깥으로 배치하여 캐시 친화성을 한층 높여 줍니다.

## 6.2.7. `#[deprecated]` 사용 자제 표시

```rok
#[deprecated]
fn old_api() { ... }

#[deprecated = "use new_api instead"]
fn old_api2() { ... }
```

`#[deprecated]` 가 부착된 함수가 사용되면 경고 (lint `deprecated`) 가 발생하지만, 에러로 이어지지는 않습니다. `= "..."` 형태로 메시지를 함께 적어 두면 그 내용이 경고에 포함되어 표시됩니다. 이 속성은 어느 항목에나 붙일 수 있지만, 경고는 함수에서만 발생합니다.

## 6.2.8. `#[doc = "..."]` 문서 주석 형태

```rok
#[doc = "이 함수는 두 정수의 합을 반환한다."]
fn add(x: i32, y: i32) -> i32 { x + y }
```

[어휘 구조](../001-lexical/102-lexical-structure.md#문서-주석) 의 `///` 문서 주석은 이 속성입니다.

```rok
/// 이 함수는 ...
fn f() { ... }

// 같은 의미
#[doc = "이 함수는 ..."]
fn f() { ... }
```

내부 형태인 `#![doc]` 또한 `//!` 와 동일한 효과를 냅니다.

### `#[doc(hidden)]`

```rok
/// dict spread 가 검사하는 것 (lowering 된 코드가 호출).
#[doc(hidden)]
pub const fn spread_misfit<V>(d: Dict<str, V>, key: str, why: str) { .. }
```

`#[doc(hidden)]` 은 공개 항목을 API 문서 (`rokc doc`) 와 language server 의 자동 완성에서 뺍니다. 컴파일러가 만든 코드가 부를 수 있도록 `pub` 일 뿐인 항목을 위한 것이며, 항목은 여전히 공개이고 컴파일과 이름 해석도 그대로입니다. 모듈에 붙이면 그 안의 모든 것을 숨깁니다. 항목의 문서 주석과 함께 둘 수 있고, 문서 주석은 hover 에 계속 쓰입니다. 그 밖의 `#[doc(..)]` 형태는 에러입니다: `#[doc = ".."]` 또는 `#[doc(hidden)]` 로 씁니다.

## 6.2.9. `#[allow(...)]` / `#[deny(...)]` / `#[warn(...)]` 린트 제어

```rok
#[allow(unused)]
fn _placeholder() { }

#[deny(deprecated)]
fn strict_zone() { ... }
```

이 속성들은 부착된 항목의 범위 안에서 lint 진단의 수준을 변경합니다. lint 는 패키지를 컴파일하는 모든 명령에서 돕니다: `rokc build`, `run`, `test`, `bench`, `emit`, `check`, `lint` ([9.1.2](../009-tools/901-rokc-cli.md#rokc-lint-린터)), 그리고 language server (`doc` 은 아님). 빌드하는 패키지만 검사하며 의존 패키지는 검사하지 않습니다. `warn` lint 는 출력되며 빌드를 실패시키지 않고, `deny` lint 는 에러이며 빌드는 종료 코드 1 로 실패합니다. `allow` 는 진단을 무시하고, `warn` 은 진단을 경고로 발생시키며, `deny` 는 진단을 에러로 발생시킵니다. 내부 형태 `#![allow(..)]` 등은 둘러싼 모듈 (파일 머리에서는 파일 전체) 에 적용됩니다. 인자는 lint 이름이나 그룹 이름입니다. 항목의 `#[allow]` 는 그 변형, 필드, 메서드에도 적용됩니다.

수준은 다음 순서로 정해지며, 뒤의 것이 앞의 것을 이깁니다: 각 lint 의 기본 수준 < `rok.rokon` 의 `lints` 필드 (그 `lints_from` 사슬 뒤) ([9.10.3](../009-tools/910-rok-frontend.md#9103-rokrokon--roklock)) < CLI 의 `-A` / `-W` / `-D` < 소스의 속성 (가장 안쪽 것이 이김). `lints` 는 CLI 플래그 앞에 붙은 것처럼 다뤄지므로, 뒤에 적은 CLI 플래그가 이를 덮습니다. lint 진단의 코드는 `E503` 입니다.

모든 lint 의 기본 수준은 `warn` 입니다.

| lint                     | 검사                                                               |
|--------------------------|--------------------------------------------------------------------|
| `unused_variables`       | 읽지 않는 지역 변수나 매개변수                                     |
| `unused_imports`         | 모듈이 이름을 쓰지 않는 `use`                                      |
| `dead_code`              | 아무도 부르지 않는 비 `pub` 함수                                   |
| `unreachable_code`       | `return` / `break` / `continue` / `throw` / `panic` 뒤의 문장      |
| `unused_must_use`        | `#[must_use]` 함수의 결과, 또는 `#[must_use]` 타입의 값을 버림 ([6.2.11](#6211-must_use-결과-무시-경고)) |
| `unused_doc_comments`    | 설명하지 않는 것 (문, match arm, 구조체 리터럴이나 tuple 필드, 식) 에 붙은 문서 주석. 일반 주석은 `//` 를 씀 |
| `write_to_copy`          | 아무도 나중에 읽지 않는 사본에 필드를 대입함: 값으로 받은 매개변수나 `self`, `for` 바인딩, 원소의 `let` (`xs[i]`, [2.3.8](../002-types/203-struct.md#238-값-의미)) 이며, 이후의 읽기, 루프 반복, 반환, 호출 어느 것도 그 변경을 볼 수 없을 때. 같은 변수에 대한 다른 쓰기에만 쓰이는 읽기는 세지 않으며, 클로저나 코루틴 블록이 언급하는 변수는 검사하지 않음 |
| `ignored_error`          | `try? f();` 문장이 에러를 조용히 버림. `let _ = try? f();` 는 의도한 best-effort 호출의 표시이며 보고하지 않음 |
| `shadowed_loop_var`      | `for` 본체의 `let` 이 루프 변수를 다시 묶음                        |
| `nan_comparison`         | NaN 과의 비교 (늘 거짓, `!=` 는 늘 참)                             |
| `none_comparison`        | `x is none` 대신 `x == none`                                       |
| `bool_comparison`        | `bool` 을 `true` / `false` 와 비교                                 |
| `self_assignment`        | `x = x;`                                                           |
| `unclosed_resource`      | `using` 으로 해제하지 않은 `File` / `Dir` / `ReadDir` / `TcpStream` / `TcpListener` / `UdpSocket` / `Child` (warn) |
| `while_true`             | `loop` 대신 `while true`                                           |
| `deprecated`             | `#[deprecated]` 함수 사용                                          |
| `redundant_cast`         | `x` 가 이미 `T` 인 `x as T`                                        |
| `redundant_clone`        | 원시 타입 값의 `.clone()`                                          |
| `non_snake_case`         | 함수, 지역 변수, 매개변수, 모듈 이름이 snake_case 가 아님          |
| `non_camel_case_types`   | 타입, 트레잇, 변형 이름이 UpperCamelCase 가 아님: 첫 글자가 소문자가 아니고, `__` 가 없으며, `_` 는 대소문자가 없는 문자 사이에만 (`V1_0`) |
| `non_upper_case_globals` | `const` / `static` 이름이 UPPER_SNAKE_CASE 가 아님                 |
| `confusable_idents`      | 서로 닮은 식별자 ([1.4](../001-lexical/104-identifiers.md))        |
| `missing_f_prefix`       | 범위 안의 지역 변수 `name` 에 대한 `{name}` 을 담은 보통 문자열 리터럴: `f` 접두어가 빠짐 (warn) |

| 그룹                | 포함                                                                                         |
|---------------------|----------------------------------------------------------------------------------------------|
| `unused`            | `unused_variables`, `unused_imports`, `dead_code`, `unreachable_code`, `unused_must_use`, `unused_doc_comments` |
| `nonstandard_style` | `non_snake_case`, `non_camel_case_types`, `non_upper_case_globals`                           |
| `all`               | 모든 lint                                                                                    |

`write_to_copy` 의 help 는 `ref self` / `ref p`, 인덱스 루프 (`for i in 0..xs.len() { xs[i].f = .. }`), 또는 원소를 다시 써 넣기를 제안합니다.

`rokc lint --list` 는 lint 와 그룹, 기본 수준을 출력합니다.

## 6.2.10. `#[non_exhaustive]` 외부에서의 완전성 검사 회피

```rok
#[non_exhaustive]
pub enum ApiError {
    NotFound,
    Unauthorized,
}
```

`#[non_exhaustive]` 는 `enum` 또는 `struct` 에 부착할 수 있습니다. 이 속성이 붙어 있으면 외부 패키지에서 해당 타입을 매칭할 때, `match` 가 모든 변형을 다 처리하더라도 catch-all (`_ => ...`) 을 추가로 요구합니다. 이러한 제약 덕분에 라이브러리 작성자는 SemVer 호환을 깨뜨리지 않으면서도 향후 새로운 변형을 자유롭게 추가할 수 있습니다.

## 6.2.11. `#[must_use]` 결과 무시 경고

```rok
#[must_use]
fn calculate() -> i32 { ... }
```

`#[must_use]` 가 붙은 함수의 반환값이 사용되지 않은 채 버려지면 경고가 발생합니다. 결과를 의도적으로 무시하고자 할 때에는 [매개변수](../004-functions/402-parameters.md) 에서 설명한 `_ = ...` 를 명시하면 됩니다.

`#[must_use]` 는 `struct` 나 `enum` 에도 붙일 수 있습니다. 그러면 그 타입의 값을 가진 식 문장이나, 한 번도 읽지 않는 그 타입의 지역 변수가 속성의 메시지와 함께 `unused_must_use` 경고를 냅니다. `async fn` 호출이나 `async` 블록의 값 (await 하지 않은 future 는 실행되지 않음) 과 `thread::JoinHandle` ([stdlib 3.4.2](../../refs-standard-libraries/300-std/304-thread.md#342-api); join 하지 않은 스레드는 프로그램이 끝나기 전에 끝나지 않을 수 있으며, `detach` 는 그것이 스스로 실행됨을 밝힘) 은 `#[must_use]` 입니다.

## 6.2.12. `#[track_caller]` 호출 위치 전달

```rok
#[track_caller]
fn expect_positive(n: i32) -> i32 {
    if n <= 0 { panic("not positive") }      // 위치: expect_positive 를 부른 곳
    n
}
```

- 함수는 호출 위치를 숨은 마지막 `str` 인자로 받습니다. 안의 panic 은 그 위치를 보고합니다 ([4.14.13](../004-functions/414-throws.md#41413-panic-과-fatal-에러)).
- tracked 함수가 다른 tracked 함수를 부르면 받은 위치를 그대로 넘깁니다.
- tracked 함수의 함수 값은 그 값을 만든 위치를 보고합니다.
- `async fn` 에 붙이면 호출만 추적하며 코루틴 본체는 추적하지 않습니다.
- `extern` (런타임) 선언에 붙이면 그 심볼이 위치 인자를 더 받습니다 ([8.5.7](../008-abi/805-extern-symbols.md#857-diagnostic-영역)).
- `main`, `extern "C" fn`, `#[export_name]` 함수에는 붙일 수 없습니다.

잘못된 입력에 panic 하는 표준 라이브러리 함수 (`panic`, `unreachable`, `assert*`, 인덱싱, 슬라이싱, `unwrap`, `List::insert` / `remove` 등) 는 `#[track_caller]` 입니다.

## 6.2.13. 정리

지금까지 살펴본 속성들은 컴파일러와 도구가 직접 인식하는 내장 속성이며, 메타 블록을 통한 사용자 정의는 따로 필요하지 않습니다.

`#[lang]`, `#[literal]` 같은 매직 속성은 언어의 의미 자체와 결합되어 있으므로, 별도의 문서에서 정의해 두었습니다.

### 그 밖의 내장 속성

컴파일러는 다음 속성도 인식합니다. 각각은 해당 기능을 다루는 곳에서 정의합니다.

| 속성                           | 붙는 곳                       | 의미 |
|--------------------------------|-------------------------------|------|
| `#[path("file.rok")]`          | `mod name;`                   | `name.rok` 대신 그 파일에서 모듈을 읽음 ([5.4.6](../005-modules/504-modules.md#546-사용자-정의-경로-path)) |
| `#![no_implicit_rok]`          | 파일 (내부)                   | 그 파일의 묵시적 `use rok::core::**;` 를 끔 ([stdlib 0.2.5](../../refs-standard-libraries/000-meta/002-import.md#025-묵시-무시---no_implicit_rok)) |
| `#[export_name("sym")]`        | 함수                          | 함수를 맹글링하지 않은 심볼 `sym` 으로 export 함 (wasm32 에서는 그 이름의 wasm export). 도달 불가 함수 제거의 root ([9.5.8](../009-tools/905-symbol-mangling.md)) |
| `#[no_mangle]`                 | 함수                          | 함수를 이름 lint 와 미사용 함수 lint 에서 뺌. 심볼은 바꾸지 않으며, 심볼 이름은 `#[export_name]` 으로 정함 |
| `#[wit::export]`               | 함수, `pub struct`            | WIT export 를 구현함: 함수 또는 resource ([stdlib 1.8](../../refs-standard-libraries/100-bare/108-wit.md)) |
| `#[niche(..)]`                 | newtype struct                | 타입이 갖지 않는 값을 선언하며, 그 값이 `T \| none` 의 niche 가 됨 ([2.7.4](../002-types/207-niche-optimization.md#274-opt-in-niche-niche)) |
| `#[intrinsic("rok.OP.TYPE")]`  | 표준 라이브러리 전용          | 함수나 메서드를 컴파일러 연산에 묶음. 사용자 코드에서는 에러 ([stdlib 0.3](../../refs-standard-libraries/000-meta/003-intrinsics.md)) |
| `#[no_safepoint]`              | `extern "rok"` 블록           | toolchain 내부용: 블록의 함수는 할당도 block 도 하지 않으므로 그 호출은 safepoint 가 아님. 다른 블록에서는 에러 ([5.9.2](../005-modules/509-extern-block.md#592-abi-명시)) |
