# 5.3. `static`

`static` 키워드는 프로그램이 실행되는 동안 유지되는 전역 메모리를 선언합니다. 이 키워드는 항목 시작에서만 키워드인 문맥 키워드이고, 뒤의 `mut`도 `static mut` 자리에서만 키워드입니다([1.5.4](../001-lexical/105-keywords.md#154-문맥-키워드)).

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

기본 형태는 `static`, 이름, `:`과 타입, `=`과 초기값 순서입니다. 타입은 적어야 합니다. 초기값은 [`const`](502-const.md)에서 말하는 `const` 표현식이어야 합니다. 곧 컴파일 타임에 평가할 수 있어야 합니다.

`static`은 모든 스레드에서 닿을 수 있으므로 그 타입은 `Sync`여야 합니다([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync-스레드-경계-마커)). 지연 초기화되는 static도 마찬가지입니다. `static XS: List<i32> = [];`는 컴파일 에러입니다(컬렉션, `StrBuf`, `Cell<T>`는 `!Sync`). 대신 `static XS: Mutex<List<i32>> = Mutex::new([]);`로 적습니다. 스레드별 상태에는 `thread::local`([stdlib 3.4.4](../../refs-standard-libraries/300-std/304-thread.md#344-스레드-로컬-저장소))을 씁니다. 이것이 돌려주는 `ThreadLocal<T>`는 `Sync`입니다. `extern` 블록의 static([5.3.10](#5310-extern-static-ffi))은 예외입니다.

## 5.3.2. `const`와 다른 점

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

`static`은 모듈 수준이나 블록([3.9.1](../003-expressions/309-block-expressions.md#391-형태))에 선언할 수 있습니다. 트레잇이나 `impl`에는 둘 수 없습니다([5.3.8](#538-메서드--트레잇과-결합)).

```rok
static APP_NAME: str = "rok";                          // 모듈 수준 - OK

fn f() -> i32 {
    static CALLS: Atomic<i32> = Atomic::new(0i32);     // OK - slot 하나, 이 블록에서 보임
    CALLS.fetch_add(1i32)
}
```

블록의 `static`은 그 함수의 어느 인스턴스가 실행되든 프로그램 전체에 하나뿐인 슬롯입니다. 초기화도 한 번만 일어납니다.

## 5.3.4. 일반 사용자 코드의 가변 전역: 래퍼 사용

```rok
static COUNTER: Atomic<i32> = Atomic::new(0i32);
static CACHE: Mutex<Dict<str, Data>> = Mutex::new(Dict::new());
```

내부 가변성이 필요한 전역은 동시성에 안전한 래퍼로 감싸서 선언합니다.

- `Atomic<T>`: lock-free 원자 연산(정수, `bool`, 포인터).
- `Mutex<T>`: 잠금 기반 보호.
- `RwLock<T>`: 읽기-쓰기 잠금. `T` 자체가 `Sync`일 때 씁니다([stdlib 2.16.4](../../refs-standard-libraries/200-core/216-sync.md#2164-rwlockt-읽기와-쓰기-분리)).

`Cell<T>`는 단일 스레드용이고 `!Sync`이므로 `static`이 될 수 없습니다. 스레드별 가변 전역은 `thread::local(..)`로 만듭니다.

모두 표준 라이브러리 타입입니다. 정확한 시그니처는 라이브러리 사양에서 정의합니다.

## 5.3.5. `static mut`: [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 한정

`static mut`는 C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface)와 직접 연동할 때만, `extern "C"` 블록 안에서만 쓸 수 있습니다.

```rok
extern "C" {
    static errno: i32;             // FFI 전역 (read-only)
    static mut atexit_count: u64;  // FFI mutable 전역
}
```

- `static mut`는 `extern "C"` 블록 안에서만 쓸 수 있습니다.
- 일반 Rok 코드의 `static mut`는 에러입니다.
- extern `static mut`가 아닌 `static`에 대입하면 컴파일 에러입니다(`COUNTER = ...`). 내부 변경은 위 래퍼의 메서드로 합니다.

공유되는 가변 상태는 데이터 경쟁의 직접 원인이므로 이렇게 제한합니다. 일반 코드는 래퍼로 안전하게 다루고, 원시 접근은 FFI 영역에서만 허용합니다.

## 5.3.6. 초기화 순서

여러 `static` 선언 사이의 초기화 순서는 다음과 같습니다.

- 같은 모듈 안에서는 선언 순서를 따릅니다.
- 모듈 사이에서는 의존 순서를 따릅니다. 한 `static`의 초기값이 다른 `static`을 참조하면 참조되는 쪽을 먼저 초기화합니다.
- 순환 의존이 있으면 컴파일 에러입니다.

`const` 초기값의 순서는 컴파일 타임에 검사합니다. `const`가 아닌 초기값은 첫 접근 때 초기화합니다. 초기화는 스레드 안전합니다. 초기화식은 정확히 한 번 실행되고, 동시에 접근한 다른 스레드는 초기화가 끝날 때까지 기다립니다. 초기화식이 자기 `static`을 읽으면 panic이 일어납니다.

## 5.3.7. 가시성

`static`에도 다른 항목처럼 가시성 키워드를 붙일 수 있습니다.

```rok
pub static VERSION: str = "1.0.0";
pub(pkg) static INTERNAL: i32 = 42i32;
static _local: bool = false;
```

자세한 가시성 규칙은 [가시성](506-visibility.md)에서 정의합니다.

## 5.3.8. 메서드 / 트레잇과 결합

`static`은 값을 담는 슬롯이지 항목 정의자가 아닙니다. 그래서 트레잇이나 `impl` 블록 정의에는 올 수 없습니다. 그 자리에는 `const`나 연관 함수를 씁니다.

```rok
impl Counter {
    static GLOBAL: i32 = 0i32;     // ERROR - impl 안에 static 없음
    const ZERO: i32 = 0i32;        // OK - 연관 const
}
```

## 5.3.9. 초기값과 `const`

초기값이 상수 식([5.2.4](502-const.md#524-상수-식과-평가기))이면 컴파일 타임에 초기화합니다. 그 밖의 초기값은 처음 쓸 때 한 번 초기화합니다(지연 초기화). 지연 초기화는 스레드 안전합니다. 동시에 접근한 스레드는 초기화가 끝날 때까지 기다립니다. 초기값 안의 함수가 `const fn`일 필요는 없습니다(`Mutex::new` 등).

```rok
static PI: f64 = 3.14159f64;             // const - 컴파일 타임
static START: Instant = Instant::now();   // 첫 접근 때 한 번 초기화
static CACHE: Mutex<Dict<str, Data>> = Mutex::new(Dict::new());
```

## 5.3.10. `extern static`: FFI

위에서 본 `extern "C"` 블록의 `static`은 외부에서 정의한 전역을 Rok에서 참조합니다. 초기값은 적지 않습니다.

```rok
extern "C" {
    static errno: i32;             // 외부 정의 - rok 에서 읽기만
}
```

자세한 규칙은 [`extern` 블록](509-extern-block.md)에서 정의합니다.

## 5.3.11. 평가 / 메모리 모델 측면

`static`의 메모리는 프로그램이 끝날 때까지 유지됩니다(`const` 초기값은 시작할 때, 그 밖은 첫 접근 때 초기화). 런타임은 프로그램을 시작할 때 모든 `static`을 GC root로 등록합니다(`rok_gc_add_root`, [8.5](../008-abi/805-extern-symbols.md)). 그래서 [GC](../000-meta/007-glossary.md#gc-garbage-collector)는 `static`이 가리키는 객체를 살려 둡니다.

GC와 런타임 동작은 [7. 런타임 모델](../007-runtime/README.md)과 [8. 런타임 ABI](../008-abi/README.md)에서 다룹니다.
