# 5.9. `extern` 블록 ([FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 선언)

`extern` 키워드 ([키워드](../001-lexical/105-keywords.md)) 와 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 명시 (`"C"` 등) 를 결합하여 외부 함수와 전역에 대한 선언 블록을 정의합니다. 외부 라이브러리, 시스템 콜, 다른 언어와의 인터페이스를 다룰 때 쓰이는 형태입니다.

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
> `extern "C"` 블록 안의 함수 선언은 **묵시적으로 unsafe** 입니다. 즉 호출하는 쪽이 항상 `unsafe { ... }` 블록 안에서만 호출할 수 있으며, 시그니처에 `unsafe` 를 따로 적지 않아도 같은 효과를 가집니다. 자세한 정의는 [4.15 unsafe](../004-functions/415-unsafe.md) 를 참조해 주세요.

## 5.9.2. ABI 명시

지원되는 ABI 의 종류는 다음 표와 같습니다.

| ABI          | 의미                                                                       |
|--------------|----------------------------------------------------------------------------|
| `"C"`        | C ABI. 가장 일반적이며, 플랫폼별 표준 호출 규약을 그대로 따릅니다.         |
| `"C-unwind"` | `"C"` 와 동일하되, 호출 경계를 가로지르는 unwinding 을 허용.               |
| `"system"`   | OS 시스템 콜 ABI. 일부 플랫폼에서 `"C"` 와 다릅니다 (예: Win32 `stdcall`). |
| ...          | 추가 ABI 는 표준 라이브러리 / 컴파일러 사양에서 정의.                      |

> [!NOTE]
> `"Rust"` 같은 다른 언어의 ABI 는 일반적으로 unstable 하므로 `extern` 의 대상이 되지 않습니다. 다른 언어와의 인터페이스는 양쪽이 합의한 안정적인 ABI (보통 `"C"`) 위에서만 가능합니다.

ABI 표시가 없는 단순한 `extern { ... }` 형태는 컴파일 에러가 되므로, 반드시 ABI 를 명시해 주셔야 합니다.

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
- 함수 포인터: `extern "C" fn(...) -> ...` (별도 형식).

호환되지 않는 타입을 적으시면 컴파일 경고나 에러가 발생합니다. 정확한 검사 규칙은 [Calling Convention](../008-abi/806-calling-convention.md) 에서 정의됩니다.

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

`extern` 블록에 선언된 함수는 일반 rok 함수와 똑같이 호출하면 됩니다.

```rok
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;
    fn free(p: Ptr<u8>);
}

fn use_malloc() {
    let p = malloc(1024usize);
    if !p.is_null() {
        // ... 사용 ...
        free(p);
    }
}
```

호출은 그저 직접 함수 호출일 뿐이며, `extern` 함수도 외부 심볼이라는 점을 빼면 다를 바가 없습니다. 결과 타입에는 rok 의 일반적인 의미론이 그대로 적용됩니다. `extern static` 역시 일반 변수처럼 읽기와 (mut 인 경우) 쓰기가 모두 가능합니다.

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
    extern "C" { ... }                     // ERROR — 함수 안에서는 안 됨
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

`#[link("...")]` 는 링크할 라이브러리의 이름을 지정하며, 빌드 시점에 해당 라이브러리가 링킹 명령에 추가됩니다. `#[link]` 가 없는 경우에는 링킹 시점에 시스템 기본값 (`libc` 등) 이나 다른 메커니즘을 통해 해소됩니다. 자세한 `#[link]` 의 동작은 [내장 속성](../006-meta-system/602-built-in-attributes.md) 에서 정의됩니다.

## 5.9.10. `extern fn` 정의 (외부에 노출)

앞 절들에서 다룬 형태는 외부 함수를 rok 안에서 호출하는 쪽이었습니다. 반대로 rok 함수를 외부 ABI 로 노출하는 형태도 있습니다.

```rok
#[export_name("rok_compute")]
pub fn compute(x: i32, y: i32) -> i32 {
    x + y
}

extern "C" fn callback(arg: i32) -> i32 {
    arg * 2
}
```

일반 함수 선언에 `extern "ABI"` 수식어를 붙이시면 그 함수를 해당 ABI 호환으로 컴파일하라는 표시가 됩니다. 그리고 `#[export_name("...")]` 으로는 외부에 노출되는 심볼의 이름을 명시할 수 있습니다. 외부 함수 호출과 외부 노출, 이 두 메커니즘이 함께 FFI 의 양방향을 이룹니다.

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

이러한 부분들은 사용자께서 직접 보장해 주셔야 합니다.
