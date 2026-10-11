# 5.3. `static`

`static` 키워드 (항목 시작에서만 키워드인 문맥 키워드이며, 뒤의 `mut` 도 `static mut` 자리에서만 키워드, [1.5.4](../001-lexical/105-keywords.md#154-문맥-키워드)) 는 프로그램이 살아 있는 동안 내내 유지되는 전역 메모리를 선언할 때 쓰입니다.

```
static_decl     ::= "static" "mut"? identifier ":" type_expr "=" const_expr ";"
extern_static   ::= "static" "mut"? identifier ":" type_expr ";"   // in extern block
```

## 5.3.1. 기본 형태

```rok
static SHARED_COUNTER: Atomic<i32> = Atomic::new(0i32);
static DEFAULT_CONFIG: Config = Config::default();
static APP_NAME: str = "rok";
```

기본 형태는 `static` 뒤에 이름을 적고, `:` 으로 타입을 붙이고, `=` 으로 초기값을 이어 붙이는 모양입니다. 타입은 반드시 적어야 하며, 초기값은 [`const`](502-const.md) 의 `const` 표현식이어야 하므로 컴파일 타임에 평가할 수 있어야 합니다.

## 5.3.2. `const` 와의 차이

```
const X: i32 = 42i32;              // 사용처마다 inline (주소 없음)
static X: i32 = 42i32;             // 안정 메모리 주소 (program 수명)
```

| 측면        | `const`     | `static`                           |
|-------------|-------------|------------------------------------|
| 메모리 주소 | 없음        | 안정된 주소                        |
| 사용처      | 인라인      | 메모리 참조                        |
| 수명        | 컴파일 타임 | 프로그램 수명                      |
| 가변성      | 항상 불변   | 기본 불변 (mutable 표현은 wrapper) |

## 5.3.3. 가능한 위치

`static` 은 오직 모듈 수준에서만 선언할 수 있습니다.

```rok
// 모듈 수준 — OK
static APP_NAME: str = "rok";

fn f() {
    static COUNTER: Atomic<i32> = Atomic::new(0i32);   // ERROR — 함수 안에서는 안 됨
}
```

함수 내부에서 지역적인 전역값이 필요하다면, 모듈 수준에 `static` 을 두고 적절한 가시성을 부여해서 그러한 상황을 표현하면 됩니다.

## 5.3.4. 일반 사용자 코드의 mutable 전역: wrapper 사용

```rok
static COUNTER: Atomic<i32> = Atomic::new(0i32);
static CACHE: Mutex<Dict<str, Data>> = Mutex::new(Dict::new());
```

내부적으로 변경 가능한 전역이 필요하다면, 동시성에 안전한 wrapper 로 감싸서 선언하면 됩니다.

- `Atomic<T>`: lock-free 원자 연산 (정수, 포인터) 에 쓰입니다.
- `Mutex<T>`: 잠금 기반 보호를 제공합니다.
- `RwLock<T>`: 읽기-쓰기 잠금을 제공합니다.
- `Cell<T>`: 단일 스레드에서의 내부 가변성을 제공합니다.

이들은 모두 표준 라이브러리 타입이며, 정확한 시그니처는 라이브러리 사양에서 정의합니다.

## 5.3.5. `static mut`: [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 한정

C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 와 직접 인터페이스해야 하는 상황에 한해, `extern "C"` 블록 안에서만 `static mut` 를 사용할 수 있습니다.

```rok
extern "C" {
    static errno: i32;             // FFI 전역 (read-only)
    static mut atexit_count: u64;  // FFI mutable 전역
}
```

- `static mut` 는 오직 `extern "C"` 블록 안에서만 허용합니다.
- 일반 rok 코드에서의 `static mut` 는 ERROR 로 처리합니다.
- extern `static mut` 가 아닌 `static` 에 대입하면 컴파일 에러입니다 (`COUNTER = ...`). 내부 변경은 위 wrapper 의 메서드로 합니다.

공유되는 mutable 상태는 데이터 경쟁의 직접적인 원인이 되기 때문에 이러한 제약을 두었습니다. 일반 코드에서는 wrapper 를 통해 안전성을 보장하고, FFI 영역에서만 raw access 를 허용합니다.

## 5.3.6. 초기화 순서

여러 `static` 선언 사이에서 초기화가 어떤 순서로 일어나는지 정리하면 다음과 같습니다.

- 같은 모듈 안에서는 선언된 순서를 따릅니다.
- 다른 모듈 사이에서는 의존 순서를 따르며, 한 `static` 의 초기값이 다른 `static` 을 참조하는 경우 후자가 먼저 초기화됩니다.
- 순환 의존이 있다면 컴파일 에러로 처리합니다.

`const` 초기값의 순서 검사는 컴파일 타임에 수행됩니다. `const` 가 아닌 초기값은 첫 접근 시점에 초기화됩니다. 초기화는 스레드 안전하며 초기화식은 정확히 한 번 실행되고, 동시에 접근한 다른 스레드는 끝날 때까지 기다립니다. 초기화식이 자기 자신의 `static` 을 읽으면 panic 입니다.

## 5.3.7. 가시성

`static` 도 다른 항목과 동일하게 가시성 키워드를 가질 수 있습니다.

```rok
pub static VERSION: str = "1.0.0";
pub(pkg) static INTERNAL: i32 = 42i32;
static _local: bool = false;
```

자세한 가시성 규칙은 [가시성](506-visibility.md) 에서 정의합니다.

## 5.3.8. 메서드 / 트레잇과의 결합

`static` 은 값을 담는 슬롯이지 항목 정의자가 아닙니다. 그래서 트레잇이나 `impl` 블록의 정의 위치에는 등장하지 않으며, 그 자리에는 `const` 나 연관 함수가 들어가게 됩니다.

```rok
impl Counter {
    static GLOBAL: i32 = 0i32;     // ERROR — impl 안에 static 없음
    const ZERO: i32 = 0i32;        // OK — 연관 const
}
```

## 5.3.9. 초기값의 `const` 한정

`static` 의 초기값이 `const` 표현식이면 컴파일 타임에 평가되어 바이너리에 들어갑니다. `const` 가 아닌 초기값도 허용되며, 그 경우 `static` 은 실제 저장소를 갖고 처음 접근할 때 한 번 초기화됩니다. 이 지연 초기화는 스레드 안전합니다 (동시에 접근한 스레드는 초기화가 끝날 때까지 기다림).

```rok
static PI: f64 = 3.14159f64;             // const — 컴파일 타임
static START: Instant = Instant::now();   // 첫 접근 때 한 번 초기화
static CACHE: Mutex<Dict<str, Data>> = Mutex::new(Dict::new());
```

## 5.3.10. `extern static`: FFI

위에서 본 `extern "C"` 블록의 `static` 은 외부에서 정의된 전역을 rok 측에서 참조하는 형태이며, 초기값은 적지 않아도 됩니다.

```rok
extern "C" {
    static errno: i32;             // 외부 정의 — rok 에서 읽기만
}
```

자세한 규칙은 [`extern` 블록](509-extern-block.md) 에서 정의합니다.

## 5.3.11. 평가 / 메모리 모델 측면

`static` 의 메모리는 종료 시점까지 유지됩니다 (`const` 초기값은 시작 시, 그 밖은 첫 접근 시 초기화). 모든 `static` 은 프로그램 시작 시 GC root 로 등록되므로 (`rok_gc_add_root`, [8.5](../008-abi/805-extern-symbols.md)) ([GC](../000-meta/007-glossary.md#gc-garbage-collector) 는 `static` 이 가리키는 객체를 살아 있는 상태로 유지합니다).

자세한 GC 와 런타임 동작에 관한 내용은 [7. 런타임 모델](../007-runtime/README.md) 과 [8. 런타임 ABI](../008-abi/README.md) 에서 다룹니다.
