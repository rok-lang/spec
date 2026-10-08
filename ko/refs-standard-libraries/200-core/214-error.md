# 2.14. error — 에러 트레잇 / 타입

`rok::core::error` / `rok::core::panic` 모듈은 에러 처리 권위를 제공합니다. rok 은 두 갈래의 에러 모델을 채택하고 있습니다. 첫째는 제어 흐름 예외(control flow exception) 를 다루는 `throw` / `try` / `catch` 모델이며, 둘째는 값으로서의 에러를 함수형으로 표현하는 `Result<T, E>` 모델입니다. 두 모델은 서로 변환할 수 있으며, 사용자가 제어 흐름의 시각적 분리를 직접 선택할 수 있도록 해 줍니다.

이 장은 48 개의 lang slot 가운데 `panic_fn` / `oom_fn` / `unreachable_fn` 세 개를 정의합니다.

## 2.14.1. 모델 개요 — 두 갈래 에러

먼저 두 모델이 실제 코드에서 어떻게 사용되는지 살펴보겠습니다.

```rok
// (a) throw 모델 — control flow exception
fn parse_int(s: str) throws (ParseError) -> i32 {
    if !s.is_digit() { throw ParseError::Invalid(s); }
    s.to_i32()
}

let n = try parse_int("42") catch e: ParseError {
    log("parse failed: {e:?}");
    return 0;
};

// (b) Result 모델 — value
fn parse_int_r(s: str) -> Result<i32, ParseError> {
    if !s.is_digit() { Err(ParseError::Invalid(s)) }
    else { Ok(s.to_i32()) }
}

let n = parse_int_r("42")?;     // ? 가 Err → 호출자에 전파
```

그렇다면 어느 모델을 언제 사용해야 할까요? 다음 표가 권장 기준을 정리해 줍니다.

| 상황                                       | 권장 모델         | 이유                         |
|--------------------------------------------|-------------------|------------------------------|
| 거의 항상 성공 / 드물게 실패 (I/O, parse)  | `throws`          | 흐름 단순 + 호출자 부담 적음 |
| 자주 실패 가능 / 분기 처리 일상적 (lookup) | `Result<T,E>`     | match / `?` 로 분기 명시     |
| API 경계 / public                          | `throws (E)`      | 시그니처에 에러 타입 노출    |
| 함수형 / monadic 합성                      | `Result<T,E>`     | `.map` / `.and_then` 체이닝  |
| panic / unrecoverable                      | `throw` (untyped) | catch 안 하면 process abort  |

## 2.14.2. `Result<T, E>` — 합 타입

`Result<T, E>` 는 합 타입으로 정의되며, 다음과 같이 두 가지 형태로 표현할 수 있습니다.

```rok
type Result<T, E> = Ok(T) | Err(E);

// 또는 enum 형태로.
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

> [!NOTE]
> rok 은 **`Some` / `None` enum variant 가 없습니다**. `Option<T>` 자체가 **존재하지 않으며**
> 그 자리에 sum type `T | none` 을 사용합니다. `Result` 는 **두 갈래 모두 의미 있는 데이터를 나르는** 경우라서 별도 sum type 으로 정의됩니다.

### 핵심 메서드

`Result` 가 제공하는 핵심 메서드는 다음과 같이 구성되어 있습니다.

```rok
impl<T, E> Result<T, E> {
    fn is_ok(self) -> bool;
    fn is_err(self) -> bool;
    fn ok(self) -> T | none;
    fn err(self) -> E | none;

    fn unwrap(self) -> T;                   // Err 시 panic
    fn unwrap_or(self, default: T) -> T;
    fn unwrap_or_else<F: Fn(E) -> T>(self, f: F) -> T;

    fn map<U, F: Fn(T) -> U>(self, f: F) -> Result<U, E>;
    fn map_err<F, G: Fn(E) -> F>(self, g: G) -> Result<T, F>;
    fn and_then<U, F: Fn(T) -> Result<U, E>>(self, f: F) -> Result<U, E>;
    fn or_else<F, G: Fn(E) -> Result<T, F>>(self, g: G) -> Result<T, F>;
}
```

### `?` 연산자 — 조기 반환

`?` 연산자의 syntax 와 desugar 골격은 [refs-lang 3.14.9 `?` 연산자](../../refs-lang/003-expressions/314-error-control-flow.md#3149--연산자--에러--none-조기-전파) 가 권위를 가집니다. 본 절은 표준 라이브러리가 제공하는 `Try` / `FromResidual` 트레잇 정의와 builtin impl, 그리고 사용 패턴에 집중합니다.

#### `Try` / `FromResidual` 트레잇

```rok
trait Try {
    type Output;
    type Residual;

    #[lang("try_branch")]
    fn branch(self) -> Output | Residual;
}

trait FromResidual<R = <Self as Try>::Residual> {
    #[lang("from_residual")]
    fn from_residual(r: R) -> Self;
}
```

`Try::branch` 는 fallible 값을 정상 출력 (`Output`) 과 조기 종료 잔류값 (`Residual`) 으로 분기합니다. `FromResidual::from_residual` 은 잔류값을 호출자가 기대하는 반환 타입으로 재구성합니다.

`Residual` 을 별도 타입으로 둔 까닭은 서로 다른 fallible 패밀리의 혼용을 막기 위함입니다. 예를 들어 `Result<T, E>` 의 잔류값을 `T | none` 함수에서 `?` 로 풀 수 없으며, 그 역도 마찬가지입니다.

#### `Result<T, E>` 의 builtin impl

```rok
struct ResultResidual<E>(E);

impl<T, E> Try for Result<T, E> {
    type Output = T;
    type Residual = ResultResidual<E>;

    fn branch(self) -> T | ResultResidual<E> {
        match self {
            Ok(v)  => v,
            Err(e) => ResultResidual(e),
        }
    }
}

impl<T, E, F> FromResidual<ResultResidual<E>> for Result<T, F>
    where F: From<E>
{
    fn from_residual(r: ResultResidual<E>) -> Result<T, F> {
        Err(F::from(r.0))           // From<E> for F 자동 변환
    }
}
```

따라서 `outer() -> Result<T, F>` 안에서 `inner()? // -> Result<U, E>` 는 `F: From<E>` 가 충족될 때 자동으로 에러 타입 변환을 수행합니다.

#### `T | none` 의 builtin impl

```rok
struct NoneResidual;

impl<T> Try for T | none {
    type Output = T;
    type Residual = NoneResidual;

    fn branch(self) -> T | NoneResidual {
        match self {
            v: T => v,
            none => NoneResidual,
        }
    }
}

impl<T> FromResidual<NoneResidual> for T | none {
    fn from_residual(_: NoneResidual) -> T | none { none }
}
```

이 impl 은 `T | none` 끼리의 `?` 만 허용하며, `Result` 와 `T | none` 사이의 혼용을 자연스럽게 차단합니다.

#### `throws` 함수 컨텍스트

함수 시그니처에 `throws` 가 적혀 있으면, `?` 의 잔류값은 `return` 대신 `throw` 로 lower 됩니다. 이 동작은 컴파일러가 컨텍스트별로 처리하는 부분이며 ([refs-lang 3.14.9](../../refs-lang/003-expressions/314-error-control-flow.md#3149--연산자--에러--none-조기-전파)), 표준 라이브러리는 별도의 impl 을 제공하지 않습니다.

```rok
fn parse(s: str) throws (ParseError) -> i32 {
    let v = inner()?;       // Err(e) → throw e (return 대신 throw 로 lower)
    v * 2
}
```

## 2.14.3. `Error` 트레잇 — 에러의 표면

모든 에러 타입의 표면은 `Error` 트레잇으로 정의됩니다.

```rok
trait Error: Display + Debug {
    fn source(self) -> &dyn Error | none {
        none                                  // default — chain 없음
    }
}
```

모든 에러 타입이 이 트레잇을 구현하며, 메시지 출력에는 `Display` 를, 디버그 표시에는 `Debug` 를 활용합니다.

### chain — `source()`

에러를 wrap 할 때는 원인 사슬을 다음과 같이 표현할 수 있습니다.

```rok
struct ConfigError {
    msg: str;
    cause: &dyn Error | none;
}

impl Error for ConfigError {
    fn source(self) -> &dyn Error | none {
        self.cause.as_ref()
    }
}

impl Display for ConfigError {
    fn fmt(self, ref f: Formatter) -> Result<(), FmtError> {
        f.write_str(self.msg.as_str())
    }
}
```

이렇게 chain 을 구성하면 REPL 이나 로거가 사슬을 따라가며 원인 traceback 을 출력할 수 있습니다.

## 2.14.4. 내장 에러 타입

표준 라이브러리가 기본으로 제공하는 에러 타입은 다음과 같이 정의됩니다.

```rok
struct TypeError {
    expected: str;
    actual: str;
}

struct ValueError {
    msg: str;
}

struct IndexError {
    index: i64;
    len: usize;
}

struct OverflowError {
    op: str;          // "add" / "mul" / ...
    ty: str;          // "i32" / "i64" / ...
}

struct ParseError {
    input: str;
    pos: usize;
    reason: str;
}
```

각 타입은 `Error + Display + Debug` 를 구현하며, `Display` 본문은 사람이 읽기 친화적인 메시지를 제공합니다. 각 에러가 던져지는 자리는 다음 표와 같이 정리할 수 있습니다.

| 타입            | 던지는 자리                                          |
|-----------------|------------------------------------------------------|
| `TypeError`     | `as` 캐스트 실패, runtime cast 실패, type check 실패 |
| `ValueError`    | 일반 invalid value (parser / API 등)                 |
| `IndexError`    | `xs[10]` 의 out-of-bounds                            |
| `OverflowError` | fixed primitive 산술 overflow (debug 모드 panic)     |
| `ParseError`    | `str → number` / TOML 등 파싱 실패                   |

### Fatal 에러

rok 의 에러 모델은 Fatal 과 Catchable 을 분리합니다. 다음과 같은 타입들이 Fatal 로 분류됩니다.

```rok
struct StackOverflowError;     // Fatal — catch 불가
struct OutOfMemoryError;       // Fatal — catch 불가
struct InternalError;          // Fatal — 컴파일러 내부 버그
```

Fatal 은 catch 를 그대로 통과하므로, `try { ... } catch e { ... }` 로는 잡을 수 없습니다. 대신 process abort 또는 panic handler 가 처리하도록 되어 있습니다.

## 2.14.5. `panic_fn` / `oom_fn` / `unreachable_fn` — 컴파일러 hook

에러 모델의 바닥에는 컴파일러가 직접 호출하는 hook 함수들이 자리잡고 있습니다.

```rok
mod rok::core::panic {
    #[lang("panic_fn")]
    fn panic(msg: str) -> ! {
        // unwinding 시작 또는 abort
    }

    #[lang("oom_fn")]
    fn oom() -> ! {
        // GC alloc 실패 시 호출
    }

    #[lang("unreachable_fn")]
    fn unreachable() -> ! {
        // match exhaustive arm 의 unreachable 자리
    }
}
```

각 함수는 `-> !` (never) 를 반환 타입으로 가지며, 절대 정상 return 하지 않습니다.

### `throw` 의 desugar

`throw` 표현식은 다음과 같이 lower 됩니다.

```rok
throw expr;
// ↓ lower
rok::core::panic::panic(format!("{:?}", expr));
```

정식 desugar 에서는 `throw expr` 가 unwinding 데이터 페이로드를 `expr` 로 설정하며, panic handler 가 그 페이로드를 받아서 처리합니다.

### OOM 처리

GC alloc 이 실패하면 다음과 같은 흐름으로 OOM 이 처리됩니다.

```rok
// GC 의 alloc fastpath 가 실패
@rok_gc_alloc_slow:
    if !try_collect_and_retry() {
        @rok::core::panic::oom();        // -> never return
    }
```

OOM 은 Fatal 에 해당하므로 catch 되지 않습니다.

### `unreachable` 사용

`unreachable` 은 도달할 수 없는 분기를 명시할 때 사용합니다.

```rok
match (x: i32 | f64) {
    v: i32 => process_int(v),
    v: f64 => process_float(v),
    // 컴파일러가 모든 variant 처리 확인 — unreachable 자리 없음
}

// 또는 명시.
fn require_positive(x: i32) -> u32 {
    if x < 0 { rok::core::panic::unreachable() }     // 호출자 보장
    x as u32
}
```

컴파일러는 다음과 같은 자리에 `unreachable` 을 자동으로 삽입합니다.
- `match` exhaustive 검사 후 dead arm
- `if let ... else` 의 else 가 divergent 임을 보장하는 자리

## 2.14.6. `try` / `catch` / `throw` / `throws` / `finally`

문법 권위는 [refs-lang 3.14 에러 제어 흐름](../../refs-lang/003-expressions/314-error-control-flow.md) 에 있으며, 이 절에서는 표준 라이브러리 측의 의미만 정의합니다.

### 시그니처 모델

함수 시그니처에서 throw 동작은 다음 네 가지 형태로 표현됩니다.

```rok
fn foo() -> T                      // non-throwing — 절대 throw 안 함
fn foo() throws -> T               // untyped — any Error 던질 수 있음 (기본)
fn foo() throws (E) -> T           // typed single — E 또는 그 변환만
fn foo() throws (A | B | C) -> T   // typed sum — A | B | C 만
```

기본적으로 권장되는 형태는 untyped 인 `throws` 이며, API 명세를 가볍게 유지할 수 있기 때문입니다. 한편 `throws (E)` 는 라이브러리 내부 API 나 임베디드 환경, 또는 catch 측에서 분기 처리가 필요한 경우에 적합합니다. typed sum 인 `throws (A | B | C)` 는 rok 고유의 형태로, Swift SE-0413 에서 영감을 받았습니다.

### `try` / `catch` 형태

`try` 는 표현식 형태를 취하며, `catch` 가 값을 반환하면 `try` 표현식 전체가 그 값을 가집니다.

```rok
let result = try compute() catch e: TypeError {
    log("type error: {e}");
    default_value
};
```

### `finally`

자원 정리 등을 위해서는 `finally` 블록을 활용할 수 있습니다.

```rok
let f = File::open(path)?;
try {
    process(f);
} catch e: IoError {
    log("io error: {e}");
} finally {
    f.close();      // 정상 / 예외 둘 다 실행
}
```

`finally` 의 실행 의미는 다음과 같이 정리할 수 있습니다.
- 정상 종료의 경우 실행됩니다.
- catch 후 종료의 경우에도 실행됩니다.
- catch 되지 않은 throw 가 rethrow 되는 경우 실행 후에 rethrow 됩니다.
- panic 또는 Fatal 의 경우 unwinding 중에 실행됩니다.

또한 `finally` 안에서 throw 가 발생하면 현재 throw 를 대체하게 됩니다.

### `throw expr` 의 type

`throw value` 는 `-> !` (never) 표현식이므로, 분기 합 타입 계산에서는 다른 분기의 타입만 반영됩니다. 예를 들어 `if c { x } else { throw e }` 의 결과는 `x` 의 타입이 됩니다.

```rok
throw value
```

## 2.14.7. `Result` ↔ `throws` 변환

두 에러 모델은 문법적으로 서로 변환할 수 있도록 설계되어 있습니다.

### `try expr` → `Result<T, E>`

`try { ... }` 블록은 내부에서 발생한 throw 를 Err 으로 wrap 하며, 성공한 경우 `Ok(v)` 를 반환합니다.

```rok
let r: Result<T, ParseError> = try { compute_or_throw() };
```

### `result?` → throw

반대로 `?` 연산자는 Err 을 throw 로 변환해 줍니다. 함수의 `throws` 시그니처가 이를 가능하게 만듭니다.

```rok
fn foo() throws (E) -> T {
    let v = some_result()?;     // result.is_err() → throw err
    use(v)
}
```

### `Result::ok()` / `err()`

또한 `Result` 의 두 변형은 `ok()` / `err()` 메서드를 통해 `T | none` 형태로도 추출할 수 있습니다.

```rok
let r: Result<i32, ParseError> = parse("42");
if let Ok(v) = r { use(v); }
if let n: i32 = r.ok() { use(n); }     // T | none 으로 추출
```

## 2.14.8. 모듈 매핑

각 에러 타입이 어떤 트레잇을 구현하는지 정리하면 다음 표와 같습니다.

| 타입                 | Display | Debug         | Error | 비고                      |
|----------------------|---------|---------------|-------|---------------------------|
| `TypeError`          | ✓       | ✓             | ✓     |                           |
| `ValueError`         | ✓       | ✓             | ✓     |                           |
| `IndexError`         | ✓       | ✓             | ✓     |                           |
| `OverflowError`      | ✓       | ✓             | ✓     | fixed primitive overflow  |
| `ParseError`         | ✓       | ✓             | ✓     |                           |
| `IoError`            | ✓       | ✓             | ✓     | std tier (target-dep)     |
| `StackOverflowError` | ✓       | ✓             | ✓     | Fatal — catch 불가        |
| `OutOfMemoryError`   | ✓       | ✓             | ✓     | Fatal — catch 불가        |
| `InternalError`      | ✓       | ✓             | ✓     | Fatal — 컴파일러 버그 시  |
| `Result<T, E>`       | ✗       | ✓ (T,E:Debug) | ✗     |                           |
| `&dyn Error`         | ✓       | ✓             | ✓     | trait object — chain 가능 |

`Display` 는 사용자 친화적인 메시지를 제공하며, 예를 들어 `IndexError { index: 10, len: 3 }` 은 `"index 10 out of bounds (len 3)"` 처럼 표현됩니다. 반면 `Debug` 는 구조 그대로를 보여 줍니다.

## 2.14.9. 다른 언어와의 비교

다른 주요 언어들과 비교했을 때 rok 의 에러 모델은 다음과 같은 위치에 자리잡고 있습니다.

| 측면         | Rust           | Java                   | Python          | rok                             |
|--------------|----------------|------------------------|-----------------|---------------------------------|
| 기본 모델    | `Result<T, E>` | `throws`               | `try/except`    | **둘 다** — 사용자 선택         |
| typed throws | **없음**       | `throws E1, E2`        | **없음**        | `throws (E)` / `throws (A\|B)`   |
| `?` 연산자   | `Result?`      | **없음**               | **없음**        | `Result?` + `T \| none ?`        |
| Option 타입  | `Option<T>`    | `Optional<T>`          | **없음** (None) | **없음** — `T \| none` 사용      |
| panic        | `panic!`       | `RuntimeException`     | `raise`         | `throw` (untyped 또는 panic_fn) |
| Fatal 분리   | **없음**       | `Error` vs `Exception` | **없음**        | `Fatal vs Catchable`            |

rok 의 가장 큰 차이점은 다음과 같이 세 가지로 정리됩니다. 첫째, `Option` 자체가 없으며 그 자리를 `T | none` sum type 이 대체하므로 타입을 한 개 줄일 수 있습니다. 둘째, `Result` 와 `throws` 두 모델이 모두 1급으로 다뤄지는데, 이는 Rust 의 Result-only 와 Swift 의 try-only 사이의 타협이라 할 수 있습니다. 셋째, typed sum throws 인 `throws (A | B | C)` 는 single typed throws 만 지원하는 Swift 보다 표현력이 더 높습니다.
