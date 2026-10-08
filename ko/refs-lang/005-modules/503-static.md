# 5.3. `static`

`static` 키워드 ([키워드](../001-lexical/105-keywords.md)) 는 프로그램 수명 동안 살아있는 전역 메모리를 선언한다.

```
static_decl     ::= "static" "mut"? identifier ":" type_expr "=" const_expr ";"
extern_static   ::= "static" "mut"? identifier ":" type_expr ";"   // in extern block
```

## 5.3.1. 기본 형태

```nok
static SHARED_COUNTER: Atomic<i32> = Atomic::new(0i32);
static DEFAULT_CONFIG: Config = Config::default();
static APP_NAME: str = "nok";
```

- `static` + 이름 + `:` + 타입 + `=` + 초기값.
- 타입은 필수다.
- 초기값은 [`const`](502-const.md) 의 `const` 표현식이어야 한다. 컴파일 타임에 평가할 수 있어야 한다.

## 5.3.2. `const` 와의 차이

```
const X: i32 = 42i32;              // 사용처마다 inline (주소 없음)
static X: i32 = 42i32;             // 안정 메모리 주소 (program 수명)
```

| 측면        | `const`     | `static`                           |
| ----------- | ----------- | ---------------------------------- |
| 메모리 주소 | 없음        | 안정된 주소                        |
| 사용처      | 인라인      | 메모리 참조                        |
| 수명        | 컴파일 타임 | 프로그램 수명                      |
| 가변성      | 항상 불변   | 기본 불변 (mutable 표현은 wrapper) |

## 5.3.3. 가능한 위치

`static` 은 모듈 수준에서만 선언할 수 있다.

```nok
// 모듈 수준 — OK
static APP_NAME: str = "nok";

fn f() {
    static COUNTER: Atomic<i32> = Atomic::new(0i32);   // ERROR — 함수 안에서는 안 됨
}
```

함수 안에서 지역 전역이 필요하면 모듈 수준 `static` 에 적절한 가시성을 부여하여 표현한다.

## 5.3.4. 일반 사용자 코드의 mutable 전역: wrapper 사용

```nok
static COUNTER: Atomic<i32> = Atomic::new(0i32);
static CACHE: Mutex<Dict<str, Data>> = Mutex::new(Dict::new());
```

내부적으로 변경 가능한 전역이 필요하면 동시성 안전 wrapper 로 감싸서 선언한다.

- `Atomic<T>`: lock-free 원자 연산 (정수, 포인터).
- `Mutex<T>`: 잠금 기반 보호.
- `RwLock<T>`: 읽기-쓰기 잠금.
- `Cell<T>`: 단일 스레드 내부 가변성.

이들은 표준 라이브러리 타입이다. 정확한 시그니처는 라이브러리 사양에서 정의한다.

## 5.3.5. `static mut`: [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 한정

C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 와 직접 인터페이스해야 할 때만 `extern "C"` 블록 안에서 `static mut` 를 사용할 수 있다.

```nok
extern "C" {
    static errno: i32;             // FFI 전역 (read-only)
    static mut atexit_count: u64;  // FFI mutable 전역
}
```

- `extern "C"` 블록 안에서만 `static mut` 를 허용한다.
- 일반 nok 코드의 `static mut` 는 ERROR.

공유 mutable 상태는 데이터 경쟁의 직접 원인이므로 둔 제약이다. 일반 코드에서는 wrapper 로 안전성을 보장하고, FFI 영역에서만 raw access 를 허용한다.

## 5.3.6. 초기화 순서

여러 `static` 선언 사이의 초기화 순서.

- 같은 모듈 안: 선언된 순서.
- 다른 모듈 사이: 의존 순서. 한 `static` 의 초기값이 다른 `static` 을 참조하면 후자를 먼저 초기화한다.
- 순환 의존: 컴파일 에러.

`const fn` 의 평가는 결정론적이므로 초기화 순서 검사가 컴파일 타임에 완전하다.

## 5.3.7. 가시성

`static` 도 다른 항목과 똑같이 가시성 키워드를 가질 수 있다.

```nok
pub static VERSION: str = "1.0.0";
pub(pkg) static INTERNAL: i32 = 42i32;
static _local: bool = false;
```

자세한 가시성 규칙은 [가시성](506-visibility.md) 에서 정의한다.

## 5.3.8. 메서드 / 트레잇과의 결합

`static` 은 값 슬롯이지 항목 정의자가 아니다. 트레잇과 `impl` 블록의 정의 위치에는 등장하지 않는다 (그 자리는 `const` 나 연관 함수다).

```nok
impl Counter {
    static GLOBAL: i32 = 0i32;     // ERROR — impl 안에 static 없음
    const ZERO: i32 = 0i32;        // OK — 연관 const
}
```

## 5.3.9. 초기값의 `const` 한정

`static` 의 초기값은 `const` 표현식이어야 한다. 런타임 함수 호출은 쓸 수 없다. 런타임 전역이지만 초기화는 결정론적이라는 원칙이다.

```nok
static NOW: Time = current_time();      // ERROR — 런타임 함수
static PI: f64 = 3.14159f64;             // OK — 리터럴
static LIMIT: i32 = compute_limit();     // OK if compute_limit 가 const fn
```

런타임에 결정되는 전역이 필요하면 지연 초기화 패턴을 라이브러리 (`Lazy<T>` 같은 타입) 로 표현한다. 표준 라이브러리에서 정의한다.

## 5.3.10. `extern static`: FFI

위 `extern "C"` 블록의 `static` 은 외부에서 정의된 전역을 nok 에서 참조하는 형태다. 초기값을 적지 않는다.

```nok
extern "C" {
    static errno: i32;             // 외부 정의 — nok 에서 읽기만
}
```

자세한 규칙은 [`extern` 블록](509-extern-block.md) 에서 정의한다.

## 5.3.11. 평가 / 메모리 모델 측면

`static` 의 메모리는 프로그램 시작 시 초기화되며 종료까지 유지된다. 가비지 컬렉터가 루트로 추적한다 ([GC](../000-meta/007-glossary.md#gc-garbage-collector) 는 `static` 이 가리키는 객체를 살아 있게 유지한다).

자세한 GC / 런타임 동작은 [7. 런타임 모델](../007-runtime/README.md) 과 [8. 런타임 ABI](../008-abi/README.md) 에서 다룬다.
