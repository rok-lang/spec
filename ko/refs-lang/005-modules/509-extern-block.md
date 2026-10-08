# 5.9. `extern` 블록 ([FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 선언)

`extern` 키워드 ([키워드](../001-lexical/105-keywords.md)) 와 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 명시 (`"C"` 등) 를 결합한 외부 함수 / 전역 선언 블록을 정의한다. 외부 라이브러리, 시스템 콜, 다른 언어와의 인터페이스에 쓴다.

```
extern_block    ::= "extern" string_lit "{" extern_item* "}"
extern_item     ::= "fn" identifier "(" param_list? ")" return_type? ";"
                  | "static" "mut"? identifier ":" type_expr ";"
```

## 5.9.1. 기본 형태

```nok
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;
    fn free(p: Ptr<u8>);
    fn strlen(s: ConstPtr<c_char>) -> usize;

    static errno: i32;
    static mut atexit_count: u64;
}
```

- `extern` + ABI 문자열 (`"C"`, `"Rust"` 등. ABI 별 의미는 [Calling Convention](../008-abi/806-calling-convention.md) 에서 정의) + `{ ... }`.
- 본체에는 함수 선언 (본체 없이 시그니처만) 과 `static` 선언만 등장할 수 있다.

## 5.9.2. ABI 명시

| ABI        | 의미                                                  |
| ---------- | ----------------------------------------------------- |
| `"C"`      | C ABI. 가장 일반적.                                   |
| `"Rust"`   | Rust ABI.                                             |
| `"system"` | OS 시스템 콜 ABI.                                     |
| ...        | 추가 ABI 는 표준 라이브러리 / 컴파일러 사양에서 정의. |

ABI 표시가 없는 단순 `extern { ... }` 은 컴파일 에러다. ABI 를 명시해야 한다.

## 5.9.3. 본체에 등장 가능한 항목

### 외부 함수 선언

```nok
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;
}
```

- 일반 함수 선언과 같은 시그니처 형태.
- 본체 없음 (`;` 으로 종료).
- 매개변수와 반환 타입은 FFI 호환 타입만 쓸 수 있다 (원시 정수, 부동소수, 포인터 등).

### 외부 `static`

```nok
extern "C" {
    static errno: i32;             // read-only
    static mut atexit_count: u64;  // mutable
}
```

- 외부에서 정의된 전역 변수에 대한 참조.
- `static mut` 는 `extern` 블록 안에서만 허용한다 ([`static`](503-static.md)).

### 등장 불가

`extern` 블록 본체에는 `struct`, `enum`, `impl`, `fn` 본체, `const` 등이 등장하지 않는다. 외부 ABI 와의 경계만 선언한다.

## 5.9.4. FFI 호환 타입

매개변수, 반환, `static` 의 타입은 FFI 호환이어야 한다. 안전한 후보는 다음과 같다.

- 원시 정수: `i8` ~ `i64`, `u8` ~ `u64`, `usize`, `isize`.
- 원시 부동소수: `f32`, `f64`.
- 원시 문자형: `c_char` (1 바이트).
- 포인터: `Ptr<T>`, `ConstPtr<T>` (표준 라이브러리, [표준 라이브러리 계층](../000-meta/005-stdlib-tiers.md) 의 `unsafe` 영역).
- `#[repr(C)]` `struct` / `union`: C ABI 호환 표시가 붙은 사용자 정의 타입.
- 함수 포인터: `extern "C" fn(...) -> ...` (별도 형식).

호환되지 않는 타입을 적으면 컴파일 경고 또는 에러가 난다 (정확한 검사는 [Calling Convention](../008-abi/806-calling-convention.md) 에서 정의).

## 5.9.5. `Ptr<T>` / `ConstPtr<T>`

[표준 라이브러리 계층](../000-meta/005-stdlib-tiers.md) 의 `unsafe` 계층에서 정의하는 원시 포인터 타입이다.

```nok
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;
    fn free(p: Ptr<u8>);
    fn read_byte(p: ConstPtr<u8>) -> u8;
}
```

- `Ptr<T>`: 변경 가능한 raw 포인터 (C 의 `T*`).
- `ConstPtr<T>`: read-only raw 포인터 (C 의 `const T*`).
- nullable: `null` 이 자동 niche 된다 ([niche 최적화](../002-types/207-niche-optimization.md)).

## 5.9.6. 호출 / 사용

`extern` 블록의 함수는 일반 nok 함수처럼 호출한다.

```nok
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

- 호출은 직접 함수 호출이다. `extern` 함수는 그냥 외부 심볼이다.
- 결과 타입에 따라 nok 의 일반 의미론이 적용된다.

`extern static` 도 일반 변수처럼 읽기 / (mut 인 경우) 쓰기가 된다.

## 5.9.7. 가시성

`extern` 블록의 각 항목도 가시성 키워드를 가진다.

```nok
extern "C" {
    pub fn public_api(x: i32) -> i32;       // 외부 공개
        fn _internal(x: i32) -> i32;        // 모듈 비공개
    pub static VERSION: i32;
}
```

`extern "C"` 블록 자체에는 가시성 키워드를 붙이지 않는다. 항목별로 부여한다.

## 5.9.8. `extern` 블록 자체의 위치

`extern "C" { ... }` 블록은 모듈 수준에만 등장한다. 함수 안이나 `impl` 안에는 등장하지 않는다.

```nok
fn f() {
    extern "C" { ... }                     // ERROR — 함수 안에서는 안 됨
}
```

## 5.9.9. `#[link("libname")]`: 라이브러리 명시

외부 라이브러리 이름을 명시하려면 `#[link]` 속성을 사용한다.

```nok
#[link("m")]
extern "C" {
    fn sin(x: f64) -> f64;
    fn cos(x: f64) -> f64;
    fn sqrt(x: f64) -> f64;
}
```

- `#[link("...")]`: 링크할 라이브러리 이름.
- 빌드 시 그 라이브러리가 링킹 명령에 추가된다.
- `#[link]` 가 없으면 링킹 시 시스템 기본 (`libc` 등) 이나 다른 메커니즘으로 해소된다.

자세한 `#[link]` 동작은 [내장 속성](../006-meta-system/602-built-in-attributes.md) 에서 정의한다.

## 5.9.10. `extern fn` 정의 (외부에 노출)

앞 절은 외부 함수를 nok 에서 호출하는 형태다. 반대로 nok 함수를 외부 ABI 로 노출하는 형태도 있다.

```nok
#[export_name("nok_compute")]
pub fn compute(x: i32, y: i32) -> i32 {
    x + y
}

extern "C" fn callback(arg: i32) -> i32 {
    arg * 2
}
```

- 일반 함수 선언에 `extern "ABI"` 수식어를 붙이면 그 함수를 ABI 호환으로 컴파일하라는 표시다.
- `#[export_name("...")]`: 외부에 노출되는 심볼 이름을 명시한다.

외부 함수 호출과 외부 노출, 이 두 메커니즘이 FFI 의 양방향을 이룬다.

## 5.9.11. `extern fn` 의 호출 규칙

`extern "C"` 함수는 C 의 calling convention 을 따른다. [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md) 의 일반 함수 호출 규칙과는 일부 다를 수 있다.

자세한 calling convention 차이는 런타임 ABI 사양 (7장 / 8장) 에서 다룬다.

## 5.9.12. FFI 의 안전성 책임

`extern` 블록의 모든 항목은 사용자 책임 영역이다 ([표준 라이브러리 계층](../000-meta/005-stdlib-tiers.md) 의 `unsafe` 정신과 일관). 컴파일러가 검증하는 것은 다음과 같다.

- 시그니처 형식.
- ABI 호환 타입 (가능한 만큼).

검증하지 않는 것은 다음과 같다.

- 외부 함수의 실제 동작 (null 반환, 에러 반환, 메모리 누수 등).
- 외부 `static` 의 동시성 안전.
- 외부 라이브러리의 진정한 ABI 호환.

이들은 사용자가 보장해야 한다.
