# 5.9. `extern` 블록([FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 선언)

이 절은 외부 함수와 전역을 선언하는 블록을 정의합니다. 블록은 `extern` 키워드와 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 지정(`"C"` 등)으로 시작합니다. 이 키워드는 항목 시작에서만 키워드인 문맥 키워드입니다([1.5.4](../001-lexical/105-keywords.md#154-문맥-키워드)). 외부 라이브러리, 시스템 콜, 다른 언어와 연동할 때 씁니다.

```
extern_block    ::= "extern" string_lit "{" extern_item* "}"
extern_item     ::= visibility? "fn" identifier "(" param_list? ")" return_type? ";"
                  | visibility? "static" "mut"? identifier ":" type_expr ";"
```

## 5.9.1. 기본 형태

기본 사용법은 다음과 같습니다.

```rok
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;
    fn free(p: Ptr<u8>);
    fn strlen(s: ConstPtr<c_char>) -> usize;

    static errno: i32;
    static mut atexit_count: u64;
}
```

블록은 `extern`, ABI 문자열(`"C"`, `"system"` 등), `{ ... }`로 이루어집니다. ABI별 의미는 [호출 규약](../008-abi/806-calling-convention.md)에서 정의합니다. 본체에는 본체 없이 시그니처만 있는 함수 선언과 `static` 선언만 올 수 있습니다.

> [!IMPORTANT]
> `extern "C"` 블록 안의 함수 선언은 **묵시적으로 unsafe**입니다. 호출하는 쪽은 `unsafe { ... }` 블록 안에서만 호출할 수 있습니다. 시그니처에 `unsafe`를 따로 적지 않아도 효과는 같습니다. 자세한 정의는 [4.15 unsafe](../004-functions/415-unsafe.md)에 있습니다.

## 5.9.2. ABI 명시

지원하는 ABI는 다음과 같습니다.

| ABI        | 의미                                                                       |
|------------|----------------------------------------------------------------------------|
| `"C"`      | 플랫폼 C ABI. 심볼 이름과 무관하게 항상 같습니다.                          |
| `"system"` | OS 시스템 콜 ABI. 일부 플랫폼에서 `"C"`와 다릅니다(예: Win32 `stdcall`).   |
| `"rok"`    | 런타임(`rokr`)의 ABI. 아래 참고.                                            |

`"system"`과 `"C"`가 일치하는 타깃(32비트 x86 Windows를 뺀 모든 Tier 1 타깃)에서는 `"system"`이 `"C"`입니다. 32비트 x86 Windows에서는 stdcall입니다([5.9.10](#5910-extern-fn-정의외부에-노출)).

`extern "rok"`은 toolchain이 선언하는 런타임 함수의 ABI입니다. 그 선언은 `rok::core::rt`에 `pub(pkg)`로 있습니다. `extern "rok"` 블록의 `#[no_safepoint]`는 그 함수가 할당도 블로킹도 하지 않는다고 선언합니다. 그래서 그 호출은 safepoint가 아닙니다. 이 속성은 toolchain 내부용이며, `extern "C"` 블록에 쓰면 에러입니다.

| 값                          | `extern "rok"`의 전달                    |
|-----------------------------|------------------------------------------|
| `str`                       | `(ptr, len)` 두 word                     |
| `fn` 값                     | `{code, env}` ([8.5.0a](../008-abi/805-extern-symbols.md#850a-rokfn-함수-값의-c-abi-형태)) |
| `extern "C" fn(..)` 매개변수 | env 없는 C 함수 포인터                   |
| 그 밖                       | 값 그대로(C ABI와 같음)                  |

> [!NOTE]
> `"Rust"` 같은 다른 언어의 ABI는 대개 안정적이지 않으므로 `extern`의 대상이 아닙니다. 다른 언어와는 양쪽이 합의한 안정적인 ABI(보통 `"C"`)로만 연동할 수 있습니다.

> [!NOTE]
> Rok은 에러를 반환값으로 전달하고 스택 되감기(unwinding)를 쓰지 않습니다([8.6.5](../008-abi/806-calling-convention.md#865-던지는-함수의-반환-값-기반-에러-전달)). 따라서 외부 언어의 예외나 되감기가 Rok 프레임을 가로지르면 미정의 동작(UB)입니다. 외부 함수는 호출 경계 안에서 예외를 처리해야 합니다.

ABI가 없는 `extern { ... }` 형태는 컴파일 에러입니다. ABI를 명시해야 합니다.

## 5.9.3. 본체에 등장 가능한 항목

### 외부 함수 선언

외부 함수는 다음과 같이 선언합니다.

```rok
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;
}
```

시그니처는 일반 함수 선언과 형태가 같지만, 본체 없이 `;`으로 끝납니다. 매개변수와 반환 타입에는 원시 정수, 부동소수, 포인터 같은 FFI 호환 타입만 쓸 수 있습니다.

### 외부 `static`

외부에서 정의한 전역 변수는 다음과 같이 참조합니다.

```rok
extern "C" {
    static errno: i32;             // read-only
    static mut atexit_count: u64;  // mutable
}
```

이 형태는 외부에서 정의한 전역 변수를 참조합니다. `static mut`는 `extern` 블록 안에서만 쓸 수 있습니다([`static`](503-static.md)).

### 등장 불가

`extern` 블록의 본체에는 `struct`, `enum`, `impl`, `fn`의 본체, `const` 등이 올 수 없습니다. 이 블록은 외부 ABI와 맞닿는 경계만 선언합니다.

## 5.9.4. FFI 호환 타입

매개변수, 반환값, `static`의 타입은 모두 FFI 호환이어야 합니다. 안전하게 쓸 수 있는 후보는 다음과 같습니다.

- 원시 정수: `i8` ~ `i64`, `u8` ~ `u64`, `usize`, `isize`.
- 원시 부동소수: `f32`, `f64`.
- 원시 문자형: `c_char`(1바이트).
- 포인터: `Ptr<T>`, `ConstPtr<T>`(표준 라이브러리, [표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md)의 `unsafe` 영역).
- `#[repr(C)]` `struct` / `union`: C ABI 호환 표시가 붙은 사용자 정의 타입.
- 함수 포인터: `extern "C" fn(...) -> ...`(아래 "C 함수 포인터 타입" 참고).
- 함수 타입 매개변수: `extern "C"` 선언의 `fn(..) -> R` 매개변수는 `extern "C" fn(..) -> R`과 같습니다.
- `str`: 아래 규칙으로 변환합니다.

`extern "C"` 시그니처의 `str` 매개변수는 NUL로 끝나는 `char*` 하나입니다. 바이트 뒤에 NUL이 있으면 그 바이트를 그대로 넘기고, 없으면(슬라이스 등) 호출 동안 고정한 사본을 넘깁니다. C는 호출이 끝난 뒤 그 포인터를 보관하면 안 됩니다. `str` 반환은 `char*`이며 길이는 `strlen`으로 잽니다. 중간에 NUL이 있으면 C 쪽에서는 첫 NUL까지만 보입니다. 반환된 `char*`가 null이면 미정의 동작입니다. 그래서 null일 수 있는 C 문자열은 `CPtr<c_char>`로 받습니다([stdlib 1.7.2](../../refs-standard-libraries/100-bare/107-ffi.md#172-cstr--cstring-null-종료-문자열)).

모듈 `m`의 `extern` 블록에 있는 `pub` 함수는 `m::f`로도 닿을 수 있습니다. 그 심볼에는 접두어가 붙지 않습니다. `extern "C"` 블록의 `rok_*` 이름은 `str`을 받거나 돌려줄 수 없습니다([8.5](../008-abi/805-extern-symbols.md)).

호환되지 않는 타입을 쓰면 정의되지 않은 동작입니다. C는 GC 참조(`List`, `Dict`)를 그저 비트로 봅니다. 컴파일러는 아직 이를 검사하지 않습니다. `extern "C" fn` 정의에서 `str`을 거부하는 것뿐입니다([5.9.10](#5910-extern-fn-정의외부에-노출)). 따라서 이 요구 사항은 프로그램이 지켜야 합니다. 타입별 호출 규약은 [호출 규약](../008-abi/806-calling-convention.md)에서 정의합니다.

### C 함수 포인터 타입

```
c_fn_type ::= "unsafe"? "extern" ("\"C\"" | "\"system\"") "fn" "(" (type_expr ("," type_expr)*)? ")" ("->" unary_type)?
```

- ABI 문자열은 `"C"`나 `"system"`만 쓸 수 있습니다. 그 밖은 에러입니다.
- `extern "system" fn(..) -> R`은 `extern "C" fn(..) -> R`과 같은 규칙을 따르므로 Win32 콜백을 표현할 수 있습니다. 두 타입이 다른 타입이 되는 곳은 stdcall을 쓰는 32비트 x86 Windows뿐입니다. 그 밖에서는 같은 타입입니다.
- 이 타입의 값은 이름 있는 C-ABI 함수로만 만듭니다. `extern` 블록에 있는 C 선언의 값은 `unsafe extern "C" fn(..)` 타입이므로 안전한(`unsafe` 없는) 자리에 넣을 수 없습니다. Rok의 `extern "C" fn` 정의(그 C 진입점, 5.9.10)는 안전한 `extern "C" fn(..)` 값을 줍니다.
- Rok `fn` 값(클로저, 함수 타입 변수 포함)과 C 함수 포인터는 서로 변환되지 않습니다.
- `unsafe`가 없는 타입으로 하는 호출은 안전합니다. `unsafe extern "C" fn`으로 호출할 때는 `unsafe`가 필요합니다.
- `none`이 아닌 C 함수 포인터 값은 null이 아닙니다. 그래서 `| none`이 없는 `extern "C" fn` 자리(매개변수, 반환, 필드)로 NULL을 받으면 미정의 동작입니다. NULL을 돌려줄 수 있는 C API는 `| none`으로 선언해야 합니다.

#### null이 될 수 있는 C 함수 포인터: `extern "C" fn(..) -> R | none`

C 함수 포인터 타입 하나와 `none`만으로 된 합(순서 무관, 다른 항 없음)은 null niche를 씁니다([2.7.2](../002-types/207-niche-optimization.md#272-자동-niche)).

- 포인터 크기입니다. `none`은 null입니다.
- C의 nullable 함수 포인터와 ABI가 같습니다. extern 매개변수 / 반환, C와 주고받는 struct 필드, extern `static`에 그대로 쓸 수 있습니다.
- 0으로 채운 저장소는 `none`으로 읽힙니다.
- 이 합을 포함하는 더 큰 합은 일반 태그 배치를 씁니다. 둘 사이의 변환은 일반 합 타입 변환입니다.

```rok
extern "C" {
    fn cmp_i32(a: ConstPtr<u8>, b: ConstPtr<u8>) -> i32;
    fn qsort(base: Ptr<u8>, n: usize, size: usize, cmp: unsafe extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32);
}
let f: unsafe extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32 = cmp_i32;   // C 선언의 값은 unsafe
let g: extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32 = cmp_i32;          // ERROR: 안전한 자리
```

### extern 함수의 이름 공간

`extern` 블록에 선언한 함수는 전역 이름 공간 하나를 이룹니다. 어느 모듈에서든 접두 없는 이름으로 부를 수 있습니다. 이 규칙은 링크 시점에 C 심볼이 전역 공간 하나에 모여 있다는 점을 반영합니다. 일반 항목에 적용되는 "조상 모듈 암묵 탐색 없음" 규칙([5.7.1](507-paths.md#571-경로의-시작-식별자))의 예외입니다. extern 함수에만 적용됩니다. 같은 이름의 extern 함수를 다른 시그니처로 두 번 선언하면 컴파일 에러입니다. `rok::core::rt`에 있는 toolchain 런타임 선언도 여기에 포함됩니다. `rok_*` 런타임 함수를 다시 선언하는 사용자 `extern`은 그 시그니처와 같아야 합니다.

## 5.9.5. `Ptr<T>` / `ConstPtr<T>`

[표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md)의 `unsafe` 계층에서 정의하는 원시 포인터 타입입니다.

```rok
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;
    fn free(p: Ptr<u8>);
    fn read_byte(p: ConstPtr<u8>) -> u8;
}
```

`Ptr<T>`는 가변 원시 포인터로, C의 `T*`에 대응합니다. `ConstPtr<T>`는 읽기 전용 원시 포인터로, C의 `const T*`에 대응합니다. 두 타입 모두 null이 될 수 있고, `null`은 자동으로 niche로 처리됩니다([niche 최적화](../002-types/207-niche-optimization.md)).

## 5.9.6. 호출 / 사용

`extern` 블록에 선언한 함수는 일반 Rok 함수처럼 호출합니다.

```rok
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;
    fn free(p: Ptr<u8>);
}

fn use_malloc() {
    let p = unsafe { malloc(1024usize) };
    if !p.is_null() {
        // ... 사용 ...
        unsafe { free(p) };
    }
}
```

사용자가 선언한 `extern "C"` 함수는 unsafe이므로 `unsafe { ... }` 안에서 호출합니다([4.15](../004-functions/415-unsafe.md)). toolchain이 선언하는 런타임 함수(`rok_print_*`, `rok_panic_str` 등)는 안전하게 호출할 수 있습니다. 그 밖에는 `extern` 함수 호출도 외부 심볼이라는 점만 빼면 일반 호출과 같습니다. 결과 타입에는 Rok의 일반 의미론이 그대로 적용됩니다. `extern static`도 `unsafe` 없이 일반 변수처럼 읽을 수 있고, `mut`이면 쓸 수도 있습니다. 그래도 타입은 FFI 호환이어야 합니다(5.9.4). Rok은 외부 메모리를 그 타입으로 읽으므로 `static environ: List<str>;`는 정의되지 않은 동작입니다. 포인터로 선언합니다(`static environ: Ptr<Ptr<c_char>>;`).

## 5.9.7. 가시성

`extern` 블록의 각 항목에도 가시성 키워드를 붙일 수 있습니다.

```rok
extern "C" {
    pub fn public_api(x: i32) -> i32;       // 외부 공개
        fn _internal(x: i32) -> i32;        // 모듈 비공개
    pub static VERSION: i32;
}
```

`extern "C"` 블록 자체에는 가시성 키워드를 붙이지 않습니다. 가시성은 항목마다 붙입니다.

## 5.9.8. `extern` 블록 자체의 위치

`extern "C" { ... }` 블록은 모듈 수준에만 올 수 있습니다. 함수 안이나 `impl` 안에는 둘 수 없습니다.

```rok
fn f() {
    extern "C" { ... }                     // ERROR: 함수 안에서는 안 됨
}
```

## 5.9.9. `#[link("libname")]`: 라이브러리 명시

외부 라이브러리의 이름은 `#[link]` 속성으로 명시합니다.

```rok
#[link("m")]
extern "C" {
    fn sin(x: f64) -> f64;
    fn cos(x: f64) -> f64;
    fn sqrt(x: f64) -> f64;
}
```

extern 블록의 `#[link("x")]`나 `#[link(name = "x", kind = "static" | "dylib")]`는 링커에 `-lx`를 넘깁니다. 라이브러리 검색 경로는 `-L <dir>` 디렉터리, 그다음 `LIBRARY_PATH`입니다. Apple 타깃에서 `kind = "static"`은 거기서 찾은 `lib<name>.a`를 경로로 링크합니다. rokc의 `-l`과 네이티브 패키지의 `links`도 같은 방식으로 라이브러리를 추가합니다. wasm32에서 `#[link(wasm_import_module = "m")]`는 블록의 함수를 core module `m`에서 가져옵니다. 함수의 `#[link_name = "sym"]`은 링크할 심볼이나 import 이름을 정합니다([6.2.3](../006-meta-system/602-built-in-attributes.md#623-link-외부-라이브러리-명시)). `#[link]`가 없으면 링커가 시스템 기본값(`libc` 등)이나 다른 메커니즘으로 심볼을 해소합니다. `#[link]`의 자세한 동작은 [내장 속성](../006-meta-system/602-built-in-attributes.md)에서 정의합니다.

## 5.9.10. `extern fn` 정의(외부에 노출)

앞 절에서 다룬 형태는 Rok에서 외부 함수를 호출합니다. 반대로 Rok 함수를 외부 ABI로 노출하는 형태도 있습니다.

```rok
#[export_name("rok_compute")]
pub fn compute(x: i32, y: i32) -> i32 {
    x + y
}

extern "C" fn callback(arg: i32) -> i32 {
    arg * 2
}
```

일반 함수 선언에 `extern "ABI"` 수식어를 붙이면 그 함수를 해당 ABI와 호환되게 컴파일합니다. `extern "C" fn` 정의도 심볼 이름은 맹글링된 이름([9.5](../009-tools/905-symbol-mangling.md))을 그대로 씁니다. C에서 보이는 이름이 필요하면 `#[export_name("...")]`을 함께 붙입니다.

`extern "ABI" fn` 정의는 `"C"`나 `"system"`을 받습니다. 그 밖의 ABI 문자열은 에러입니다. 32비트 x86 Windows에서 `extern "system"` 선언과 정의는 stdcall입니다. 피호출자가 인자를 스택에서 꺼내고, 심볼은 MSVC처럼 `_name@N`입니다.

`extern "C"`와 `extern "system"` fn 정의에는 다음 제약이 있습니다.

- 제네릭, `async`, `throws`일 수 없습니다.
- 매개변수와 반환 타입은 5.9.4의 FFI 호환 타입이어야 합니다. `str` 같은 비 FFI 타입은 쓸 수 없습니다. 5.9.4의 `str` 변환은 C 함수를 '호출'하는 선언에만 적용됩니다.

```rok
#[export_name("on_event")]
extern "C" fn on_event(code: i32) -> i32 { code }     // C 에서 on_event 로 보임
extern "C" fn bad(s: str) { }                          // ERROR: str 은 쓸 수 없음
```

외부 함수 호출과 외부 노출, 두 메커니즘이 함께 FFI의 양방향을 이룹니다.

## 5.9.11. `extern fn`의 호출 규칙

`extern "C"` 함수는 C 호출 규약(calling convention)을 따릅니다. 그래서 [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md)에서 다루는 일반 함수 호출 규칙과 일부 다를 수 있습니다. 호출 규약의 차이는 런타임 ABI 사양(7장 / 8장)에서 다룹니다.

## 5.9.12. FFI의 안전성 책임

`extern` 블록의 모든 항목은 사용자가 책임집니다. [표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md)의 `unsafe` 정신과 같습니다. 컴파일러가 검증하는 것은 다음과 같습니다.

- 시그니처 형식.
- ABI 호환 타입(가능한 범위까지).

컴파일러가 검증하지 않는 것은 다음과 같습니다.

- 외부 함수의 실제 동작(null 반환, 에러 반환, 메모리 누수 등).
- 외부 `static`의 동시성 안전.
- 외부 라이브러리의 실제 ABI 호환.

이 부분은 사용자가 직접 보장해야 합니다.
