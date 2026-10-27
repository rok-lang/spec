# 5.9. `extern` 블록 ([FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 선언)

`extern` 키워드 (항목 시작에서만 키워드인 문맥 키워드, [1.5.4](../001-lexical/105-keywords.md#154-문맥-키워드)) 와 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 명시 (`"C"` 등) 를 결합하여 외부 함수와 전역에 대한 선언 블록을 정의합니다. 외부 라이브러리, 시스템 콜, 다른 언어와의 인터페이스를 다룰 때 쓰이는 형태입니다.

```
extern_block    ::= "extern" string_lit "{" extern_item* "}"
extern_item     ::= visibility? "fn" identifier "(" param_list? ")" return_type? ";"
                  | visibility? "static" "mut"? identifier ":" type_expr ";"
```

## 5.9.1. 기본 형태

기본적인 사용 모습은 다음과 같습니다.

```rok
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;
    fn free(p: Ptr<u8>);
    fn strlen(s: ConstPtr<c_char>) -> usize;

    static errno: i32;
    static mut atexit_count: u64;
}
```

블록은 `extern` 과 ABI 문자열 (`"C"`, `"system"` 등이며, ABI 별 의미는 [Calling Convention](../008-abi/806-calling-convention.md) 에서 정의됩니다), 그리고 `{ ... }` 의 조합으로 구성됩니다. 본체에는 본체 없이 시그니처만을 가진 함수 선언과 `static` 선언만이 등장할 수 있습니다.

> [!IMPORTANT]
> `extern "C"` 블록 안의 함수 선언은 **묵시적으로 unsafe** 입니다. 즉 호출하는 쪽이 항상 `unsafe { ... }` 블록 안에서만 호출할 수 있으며, 시그니처에 `unsafe` 를 따로 적지 않아도 같은 효과를 가집니다. 자세한 정의는 [4.15 unsafe](../004-functions/415-unsafe.md) 를 참조합니다.

## 5.9.2. ABI 명시

지원되는 ABI 의 종류는 다음 표와 같습니다.

| ABI        | 의미                                                                       |
|------------|----------------------------------------------------------------------------|
| `"C"`      | 플랫폼 C ABI. 심볼 이름과 무관하게 항상 같습니다.                          |
| `"system"` | OS 시스템 콜 ABI. 일부 플랫폼에서 `"C"` 와 다릅니다 (예: Win32 `stdcall`). |
| `"rok"`    | 런타임 (`rokr`) 의 ABI. 아래 참고.                                          |

`"system"` 과 `"C"` 가 같은 target (32-bit x86 Windows 를 뺀 모든 Tier 1 target) 에서는 `"system"` 이 `"C"` 입니다. 32-bit x86 Windows 에서는 stdcall 입니다 ([5.9.10](#5910-extern-fn-정의-외부에-노출)).

`extern "rok"` 은 toolchain 이 선언하는 런타임 함수의 ABI 이며, 그 선언은 `rok::core::rt` 에 `pub(pkg)` 로 있습니다. `extern "rok"` 블록의 `#[no_safepoint]` 는 그 함수들이 할당도 block 도 하지 않음을 선언하며, 그 호출은 safepoint 가 아닙니다. toolchain 내부용이며 `extern "C"` 블록에서는 에러입니다.

| 값                          | `extern "rok"` 의 전달                   |
|-----------------------------|------------------------------------------|
| `str`                       | `(ptr, len)` 두 word                     |
| `fn` 값                     | `{code, env}` ([8.5.0a](../008-abi/805-extern-symbols.md#850a-rokfn---함수-값의-c-abi-형태)) |
| `extern "C" fn(..)` 매개변수 | env 없는 C 함수 포인터                   |
| 그 밖                       | 값 그대로 (C ABI 와 같음)                |

> [!NOTE]
> `"Rust"` 같은 다른 언어의 ABI 는 일반적으로 unstable 하므로 `extern` 의 대상이 되지 않습니다. 다른 언어와의 인터페이스는 양쪽이 합의한 안정적인 ABI (보통 `"C"`) 위에서만 가능합니다.

> [!NOTE]
> Rok 은 에러를 반환값으로 전달하며 스택 unwinding 을 사용하지 않습니다 ([8.6.5](../008-abi/806-calling-convention.md#865-throwing-함수의-반환-값-기반-에러-전달)). 따라서 외부 언어의 예외나 unwinding 이 Rok frame 을 가로지르는 것은 정의되지 않은 동작이며, 외부 함수는 호출 경계 안에서 예외를 처리해야 합니다.

ABI 표시가 없는 단순한 `extern { ... }` 형태는 컴파일 에러가 되므로, 반드시 ABI 를 명시해야 합니다.

## 5.9.3. 본체에 등장 가능한 항목

### 외부 함수 선언

외부 함수는 다음과 같이 선언합니다.

```rok
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;
}
```

시그니처는 일반 함수 선언과 동일한 형태를 사용하지만, 본체는 두지 않고 `;` 으로 종료합니다. 매개변수와 반환 타입에는 원시 정수, 부동소수, 포인터 등 FFI 호환 타입만을 쓸 수 있습니다.

### 외부 `static`

외부에서 정의된 전역 변수는 다음과 같이 참조합니다.

```rok
extern "C" {
    static errno: i32;             // read-only
    static mut atexit_count: u64;  // mutable
}
```

이 형태는 외부에서 정의된 전역 변수에 대한 참조이며, `static mut` 는 오로지 `extern` 블록 안에서만 허용됩니다 ([`static`](503-static.md)).

### 등장 불가

`extern` 블록의 본체에는 `struct`, `enum`, `impl`, `fn` 의 본체, `const` 등은 등장하지 않습니다. 이 블록은 어디까지나 외부 ABI 와의 경계만을 선언하는 자리이기 때문입니다.

## 5.9.4. FFI 호환 타입

매개변수와 반환, 그리고 `static` 의 타입은 모두 FFI 호환이어야 합니다. 안전하게 사용할 수 있는 후보는 다음과 같습니다.

- 원시 정수: `i8` ~ `i64`, `u8` ~ `u64`, `usize`, `isize`.
- 원시 부동소수: `f32`, `f64`.
- 원시 문자형: `c_char` (1 바이트).
- 포인터: `Ptr<T>`, `ConstPtr<T>` (표준 라이브러리, [표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md) 의 `unsafe` 영역).
- `#[repr(C)]` `struct` / `union`: C ABI 호환 표시가 붙은 사용자 정의 타입.
- 함수 포인터: `extern "C" fn(...) -> ...` (아래 "C 함수 포인터 타입").
- 함수 타입 매개변수: `extern "C"` 선언의 `fn(..) -> R` 매개변수는 `extern "C" fn(..) -> R` 과 같습니다.
- `str`: 아래 규칙으로 변환됩니다.

`extern "C"` 시그니처의 `str` 매개변수는 NUL 로 끝나는 `char*` 하나입니다. 바이트 뒤에 NUL 이 있으면 그 바이트를 그대로 넘기고, 없으면 (slice 등) 호출 동안 고정한 사본을 넘깁니다. C 는 호출이 끝난 뒤 그 포인터를 보관하면 안 됩니다. `str` 반환은 `char*` 이며 길이는 `strlen` 으로 잽니다. 내부에 NUL 이 있으면 C 쪽에서는 첫 NUL 까지만 보입니다. 반환된 `char*` 가 null 이면 동작은 정의되지 않으므로, null 일 수 있는 C 문자열은 `CPtr<c_char>` 로 받습니다 ([stdlib 1.7.2](../../refs-standard-libraries/100-bare/107-ffi.md#172-cstr--cstring---null-terminated-string)).

모듈 `m` 의 `extern` 블록에 있는 `pub` 함수는 `m::f` 로도 닿을 수 있으며, 그 심볼에는 접두어가 붙지 않습니다. `extern "C"` 블록의 `rok_*` 이름은 `str` 을 받거나 돌려줄 수 없습니다 ([8.5](../008-abi/805-extern-symbols.md)).

호환되지 않는 타입을 적으시면 컴파일 경고나 에러가 발생합니다. 정확한 검사 규칙은 [Calling Convention](../008-abi/806-calling-convention.md) 에서 정의됩니다.

### C 함수 포인터 타입

```
c_fn_type ::= "unsafe"? "extern" ("\"C\"" | "\"system\"") "fn" "(" (type_expr ("," type_expr)*)? ")" ("->" unary_type)?
```

- ABI 문자열은 `"C"` 나 `"system"` 만 됩니다. 그 밖은 에러입니다.
- `extern "system" fn(..) -> R` 은 `extern "C" fn(..) -> R` 과 같은 규칙을 따르므로 Win32 callback 을 표현할 수 있습니다. 별개의 타입인 것은 stdcall 인 32-bit x86 Windows 에서뿐이며, 그 밖에서는 두 타입이 같습니다.
- 이 타입의 값은 이름 있는 C-ABI 함수로만 만듭니다. `extern` 블록의 C 선언의 값은 `unsafe extern "C" fn(..)` 타입이라 안전한 (`unsafe` 없는) 자리에 넣을 수 없고, Rok 의 `extern "C" fn` 정의 (그 C 진입점, 5.9.10) 는 안전한 `extern "C" fn(..)` 값을 줍니다.
- Rok `fn` 값 (클로저, 함수 타입 변수 포함) 과 C 함수 포인터는 서로 변환되지 않습니다.
- `unsafe` 가 없는 타입으로 부르는 것은 안전하고, `unsafe extern "C" fn` 으로 부를 때는 `unsafe` 가 필요합니다.
- `none` 이 아닌 C 함수 포인터 값은 null 이 아닙니다. 그래서 `| none` 없는 `extern "C" fn` 자리 (매개변수, 반환, 필드) 로 NULL 을 받는 것은 정의되지 않은 동작이며, NULL 을 돌려줄 수 있는 C API 는 `| none` 으로 선언해야 합니다.

#### nullable C 함수 포인터: `extern "C" fn(..) -> R | none`

C 함수 포인터 타입 하나와 `none` 만으로 된 합 (순서 무관, 다른 항 없음) 은 null niche 를 씁니다 ([2.7.2](../002-types/207-niche-optimization.md#272-자동-niche)).

- 포인터 크기이며 `none` 은 null 입니다.
- C 의 nullable 함수 포인터와 ABI 가 같습니다. extern 매개변수 / 반환, C 와 주고받는 struct 필드, extern `static` 에 그대로 쓸 수 있습니다.
- 0 으로 채운 저장소는 `none` 으로 읽힙니다.
- 이것을 포함하는 더 큰 합은 일반 태그 배치를 쓰며, 그 사이의 변환은 일반 합 타입 변환입니다.

```rok
extern "C" {
    fn cmp_i32(a: ConstPtr<u8>, b: ConstPtr<u8>) -> i32;
    fn qsort(base: Ptr<u8>, n: usize, size: usize, cmp: unsafe extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32);
}
let f: unsafe extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32 = cmp_i32;   // C 선언의 값은 unsafe
let g: extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32 = cmp_i32;          // ERROR: 안전한 자리
```

### extern 함수의 이름 공간

`extern` 블록에 선언한 함수는 하나의 전역 이름 공간을 이루며, 어느 모듈에서든 접두 없는 이름으로 부를 수 있습니다. C 심볼은 링크 시점에 하나의 전역 공간에 있으므로 이를 그대로 반영한 것입니다. 이는 일반 항목에 대한 "조상 모듈 암묵 탐색 없음" 규칙 ([5.7.1](507-paths.md#571-경로의-시작-식별자)) 의 예외이며, extern 함수에만 적용됩니다. 같은 이름의 extern 함수를 시그니처가 다르게 두 번 선언하면 컴파일 에러입니다. `rok::core::rt` 의 toolchain 런타임 선언도 여기에 셉니다: `rok_*` 런타임 함수를 다시 선언하는 사용자 `extern` 은 그 시그니처와 같아야 합니다.

## 5.9.5. `Ptr<T>` / `ConstPtr<T>`

[표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md) 의 `unsafe` 계층에서 정의되는 원시 포인터 타입입니다.

```rok
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;
    fn free(p: Ptr<u8>);
    fn read_byte(p: ConstPtr<u8>) -> u8;
}
```

`Ptr<T>` 는 변경 가능한 raw 포인터로 C 의 `T*` 에 대응하며, `ConstPtr<T>` 는 read-only raw 포인터로 C 의 `const T*` 에 대응합니다. 두 타입 모두 nullable 이며, `null` 이 자동으로 niche 처리됩니다 ([niche 최적화](../002-types/207-niche-optimization.md)).

## 5.9.6. 호출 / 사용

`extern` 블록에 선언된 함수는 일반 Rok 함수와 똑같이 호출하면 됩니다.

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

사용자가 선언한 `extern "C"` 함수는 unsafe 이므로 `unsafe { ... }` 안에서 부릅니다 ([4.15](../004-functions/415-unsafe.md)). toolchain 이 선언하는 런타임 함수 (`rok_print_*`, `rok_panic_str` 등) 는 안전하게 부를 수 있습니다. 그 밖에는 `extern` 함수도 외부 심볼이라는 점을 빼면 일반 호출과 다를 바가 없습니다. 결과 타입에는 Rok 의 일반적인 의미론이 그대로 적용됩니다. `extern static` 역시 일반 변수처럼 읽기와 (mut 인 경우) 쓰기가 모두 가능합니다.

## 5.9.7. 가시성

`extern` 블록의 각 항목에도 가시성 키워드를 붙일 수 있습니다.

```rok
extern "C" {
    pub fn public_api(x: i32) -> i32;       // 외부 공개
        fn _internal(x: i32) -> i32;        // 모듈 비공개
    pub static VERSION: i32;
}
```

다만 `extern "C"` 블록 자체에는 가시성 키워드를 붙이지 않으며, 가시성은 항목별로 부여해야 합니다.

## 5.9.8. `extern` 블록 자체의 위치

`extern "C" { ... }` 블록은 모듈 수준에서만 등장할 수 있습니다. 함수 안이나 `impl` 안에는 둘 수 없습니다.

```rok
fn f() {
    extern "C" { ... }                     // ERROR: 함수 안에서는 안 됨
}
```

## 5.9.9. `#[link("libname")]`: 라이브러리 명시

외부 라이브러리의 이름을 명시하고 싶으실 때에는 `#[link]` 속성을 사용합니다.

```rok
#[link("m")]
extern "C" {
    fn sin(x: f64) -> f64;
    fn cos(x: f64) -> f64;
    fn sqrt(x: f64) -> f64;
}
```

extern 블록의 `#[link("x")]` 나 `#[link(name = "x", kind = "static" | "dylib")]` 는 링커에 `-lx` 를 넘깁니다. 라이브러리 검색 경로는 `-L <dir>` 디렉토리, 그 다음 `LIBRARY_PATH` 입니다. Apple 대상에서 `kind = "static"` 은 거기서 찾은 `lib<name>.a` 를 경로로 링크합니다. rokc 의 `-l` 과 native 패키지의 `links` 도 같은 방식으로 라이브러리를 더합니다. wasm32 에서 `#[link(wasm_import_module = "m")]` 는 블록의 함수를 core module `m` 에서 import 하며, 함수의 `#[link_name = "sym"]` 은 링크하는 심볼이나 import 이름을 정합니다 ([6.2.3](../006-meta-system/602-built-in-attributes.md#623-link-외부-라이브러리-명시)). `#[link]` 가 없는 경우에는 링킹 시점에 시스템 기본값 (`libc` 등) 이나 다른 메커니즘을 통해 해소됩니다. 자세한 `#[link]` 의 동작은 [내장 속성](../006-meta-system/602-built-in-attributes.md) 에서 정의됩니다.

## 5.9.10. `extern fn` 정의 (외부에 노출)

앞 절들에서 다룬 형태는 외부 함수를 Rok 안에서 호출하는 쪽이었습니다. 반대로 Rok 함수를 외부 ABI 로 노출하는 형태도 있습니다.

```rok
#[export_name("rok_compute")]
pub fn compute(x: i32, y: i32) -> i32 {
    x + y
}

extern "C" fn callback(arg: i32) -> i32 {
    arg * 2
}
```

일반 함수 선언에 `extern "ABI"` 수식어를 붙이시면 그 함수를 해당 ABI 호환으로 컴파일하라는 표시가 됩니다. `extern "C" fn` 정의도 심볼 이름은 맹글링된 이름 ([9.5](../009-tools/905-symbol-mangling.md)) 그대로이며, C 에서 보이는 이름이 필요하면 `#[export_name("...")]` 을 함께 붙입니다.

`extern "ABI" fn` 정의는 `"C"` 나 `"system"` 을 받으며, 그 밖의 ABI 문자열은 에러입니다. 32-bit x86 Windows 에서 `extern "system"` 선언과 정의는 stdcall 입니다: 피호출자가 인자를 pop 하며, 심볼은 MSVC 처럼 `_name@N` 입니다.

`extern "C"` 와 `extern "system"` fn 정의에는 다음 제약이 있습니다.

- 제네릭, `async`, `throws` 일 수 없습니다.
- 매개변수와 반환 타입은 5.9.4 의 FFI 호환 타입이어야 하며, `str` 같은 비 FFI 타입은 쓸 수 없습니다. (5.9.4 의 `str` 변환은 C 함수를 *호출* 하는 선언에만 적용됩니다.)

```rok
#[export_name("on_event")]
extern "C" fn on_event(code: i32) -> i32 { code }     // C 에서 on_event 로 보임
extern "C" fn bad(s: str) { }                          // ERROR: str 은 쓸 수 없음
```

외부 함수 호출과 외부 노출, 이 두 메커니즘이 함께 FFI 의 양방향을 이룹니다.

## 5.9.11. `extern fn` 의 호출 규칙

`extern "C"` 함수는 C 의 calling convention 을 따르며, [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 에서 다루는 일반 함수 호출 규칙과는 일부 다를 수 있습니다. 자세한 calling convention 의 차이점은 런타임 ABI 사양 (7장 / 8장) 에서 다루어집니다.

## 5.9.12. FFI 의 안전성 책임

`extern` 블록의 모든 항목은 사용자가 책임지셔야 할 영역입니다 ([표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md) 의 `unsafe` 정신과 일관됩니다). 컴파일러가 검증하는 것은 다음과 같습니다.

- 시그니처 형식.
- ABI 호환 타입 (가능한 만큼).

반면에 컴파일러가 검증하지 않는 것은 다음과 같습니다.

- 외부 함수의 실제 동작 (null 반환, 에러 반환, 메모리 누수 등).
- 외부 `static` 의 동시성 안전.
- 외부 라이브러리의 진정한 ABI 호환.

이러한 부분들은 사용자께서 직접 보장해야 합니다.
