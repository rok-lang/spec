# 2.14. error: 에러 트레잇 / 타입

`rok::core::error` / `rok::core::panic` 모듈은 에러 처리의 권위입니다(`ZeroError`는 `NonZero` 옆, `rok::core::num::niche`에 있습니다). Rok의 에러 모델은 하나입니다. 에러를 던질 수 있는 함수는 시그니처에 `throws`를 적고, 호출하는 쪽은 그 호출을 `try`로 표시합니다([refs-lang 4.14](../../refs-lang/004-functions/414-throws.md)). `Result<T, E>`는 에러를 **값으로 담아 두는** 평범한 enum입니다. 에러를 전파하는 수단이 아닙니다.

이 장은 lang slot 48개 가운데 `panic_fn` / `oom_fn` / `unreachable_fn` 세 개를 정의합니다.

## 2.14.1. 모델 개요: 던지면 `throws`, 담아 두면 `Result`

```rok
// (a) 던지기: throws 와 try
fn parse_int(s: str) throws (ParseError) -> i32 {
    if !s.is_digit() { throw ParseError { input: s, pos: 0, reason: "not a digit" }; }
    s.to_i32()
}

let n = try parse_int("42") catch e: ParseError {
    log(f"parse failed: {e:?}");
    0
};

// (b) 담아 두기: Result
let results: List<Result<i32, ParseError>> =
    inputs.map(|s| Result::catching(|| try parse_int(s)));

for r in results {
    match r {
        Ok(v) => total = total + v,
        Err(e) => failures.push(e),
    }
}
```

어느 쪽을 쓸지는 한 줄 규칙으로 정합니다. **에러가 호출자에게 흘러가야 하면 `throws`를, 에러를 나중에 다루려고 보관해야 하면 `Result`를 씁니다.**

| 상황                                      | 사용할 도구             | 이유                                 |
|-------------------------------------------|-------------------------|--------------------------------------|
| 실패할 수 있는 연산 (I/O, 파싱, 조회)     | `throws`                | 호출 지점마다 `try`로 흐름이 드러남 |
| API 경계 / 공개 API                       | `throws (E)` / `throws` | 시그니처에 에러 타입 노출            |
| 병렬 실행 결과 수집, 콜백으로 결과 전달   | `Result<T, E>`          | 에러를 값으로 보관                   |
| 여러 에러를 모아 한꺼번에 보고, 결과 캐시 | `Result<T, E>`          | 값으로 저장하고 나중에 처리          |
| 실패를 무시하고 값만 필요                 | `try?`                  | `T \| none`으로 받음                |
| 실패가 곧 프로그램 버그                   | `try!`                  | 실패하면 panic                       |

## 2.14.2. `Result<T, E>`: 보관용 enum

`Result<T, E>`는 `rok::core::error`의 평범한 enum입니다.

```rok
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

> [!NOTE]
> Rok에는 **`Some` / `None` enum 변형이 없습니다**. `Option<T>` 자체가 **존재하지 않으며**,
> 그 자리에 합 타입 `T | none`을 씁니다. `Result`는 **두 변형이 모두 의미 있는 데이터를 나르므로** 별도 enum으로 정의합니다.

### 핵심 메서드

`Result`의 핵심 메서드는 다음과 같습니다.

```rok
impl<T, E> Result<T, E> {
    fn is_ok(self) -> bool;
    fn is_err(self) -> bool;
    fn ok(self) -> T | none;
    fn err(self) -> E | none;

    fn unwrap(self) -> T;                               // Err 시 panic
    fn unwrap_or(self, default: T) -> T;
    fn unwrap_or_else(self, f: fn(E) -> T) -> T;

    fn map<U>(self, f: fn(T) -> U) -> Result<U, E>;
    fn map_err<F>(self, g: fn(E) -> F) -> Result<T, F>;
    fn and_then<U>(self, f: fn(T) -> Result<U, E>) -> Result<U, E>;
    fn or_else<F>(self, g: fn(E) -> Result<T, F>) -> Result<T, F>;

    fn get(self) throws (E) -> T;                       // Err 면 throw, 에러 흐름으로 되돌리기
    fn catching(f: fn() throws (E) -> T) -> Result<T, E>;   // 에러 흐름을 값으로 담기
}
```

`get`과 `catching`은 두 세계를 잇는 다리입니다. `catching`은 에러를 던지는 함수를 호출해 성공이면 `Ok`로, 에러면 `Err`로 담습니다. `get`은 담아 둔 값을 꺼내고, `Err`면 그 에러를 다시 던집니다.

### `Result`에는 `?`가 없습니다

[refs-lang 3.14.10](../../refs-lang/003-expressions/314-error-control-flow.md#31410--연산자-none-조기-반환)의 후위 `?`는 `T | none` 전용입니다. `Result` 값에 `?`를 쓰면 컴파일러는 다음 진단을 냅니다.

```rok
fn load(r: Result<Config, IoError>) -> Result<Config, IoError> {
    let cfg = r?;
    // ERROR: `?` 는 `T | none` 전용입니다. 에러 전파는 `throws` 와 `try` 를 쓰세요.
    ...
}
```

담아 둔 에러를 호출자에게 전파하려면 함수를 `throws`로 선언하고 `try r.get()`을 씁니다.

```rok
fn load(r: Result<Config, IoError>) throws (IoError) -> Config {
    try r.get()
}
```

## 2.14.3. `Error` 트레잇: 에러의 표면

모든 에러 타입의 표면은 `Error` 트레잇이 정의합니다.

```rok
trait Error: Display + Debug {
    fn source(self) -> any Error | none {
        none                                  // default: chain 없음
    }
}
```

에러 타입은 이 트레잇을 구현합니다. 메시지 출력에는 `Display`를, 디버그 표시에는 `Debug`를 씁니다. 타입 없는 `throws`는 `throws (any Error)`의 축약이므로, 타입 없이 에러를 던지는 함수가 던지는 값은 `Error`를 구현해야 합니다. 타입에 상관없이 에러를 담거나 넘길 때는 [제약 붙은 `any`](../../refs-lang/002-types/202-any-slot.md)인 `any Error`를 씁니다(`any Error | none`은 `(any Error) | none`입니다).

### 체인: `source()`

에러를 감쌀 때는 원인 사슬을 다음과 같이 표현합니다.

```rok
struct ConfigError {
    msg: str;
    cause: any Error | none;
}

impl Error for ConfigError {
    fn source(self) -> any Error | none {
        self.cause
    }
}

impl Display for ConfigError {
    fn fmt(self, ref f: Formatter) throws (FmtError) {
        try f.write_str(self.msg)
    }
}
```

`cause`에 값을 넣으면 컴파일러가 그 값의 정적 타입이 `Error`를 구현하는지 컴파일 타임에 검사합니다. 이렇게 만든 사슬을 REPL이나 로거가 따라가며 원인 traceback을 출력할 수 있습니다.

```rok
let cfg = try parse_config(text) catch e: ParseError {
    throw ConfigError { msg: "bad config", cause: e };
};
```

## 2.14.4. 내장 에러 타입

표준 라이브러리의 기본 에러 타입은 다음과 같습니다. 필드는 모두 `pub`입니다.

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

struct ConversionError {
    msg: str;
}

struct ZeroError;      // rok::core::num::niche
```

각 타입은 `Error + Display + Debug`를 구현합니다. `Display` 본문은 사람이 읽기 좋은 메시지를 만듭니다. 각 에러를 던지는 자리는 다음과 같습니다.

| 타입            | 던지는 자리                                                                |
|-----------------|----------------------------------------------------------------------------|
| `TypeError`     | 런타임 캐스트 실패, 타입 검사 실패                                         |
| `ValueError`    | 일반적인 잘못된 값 (파서 / API 등)                                         |
| `IndexError`    | 인덱스를 받는, 에러를 던지는 API의 범위 밖 인자 (`xs[10]`의 경계 검사는 panic) |
| `OverflowError` | `try_into` 등 검사하는 변환의 범위 초과 (산술 오버플로는 panic)            |
| `ParseError`    | `str → number` 등 파싱 실패 (ROKON은 `RokonError`)                                         |
| `ConversionError` | `TryFrom::Error`의 기본값. 숫자가 아닌 변환의 실패 ([2.11.4](211-conversion.md#2114-tryfromu-트레잇-실패할-수-있는-변환)), `NonNan` / `Finite`의 `try_from` |
| `ZeroError`     | `NonZero::try_from`에 0 ([2.2.2](202-non-zero.md#222-nonzerot-0-금지))     |

### fatal 에러

다음 타입은 Fatal로 분류됩니다.

```rok
struct StackOverflowError;     // Fatal: catch 불가
struct OutOfMemoryError;       // Fatal: catch 불가
struct InternalError;          // Fatal: 컴파일러 내부 버그
```

Fatal은 `throws`의 에러가 아니므로 `catch`로 잡을 수 없습니다. panic과 마찬가지로 메시지를 stderr에 출력하고 프로세스를 종료 코드 `101`로 끝냅니다([refs-lang 3.14.8](../../refs-lang/003-expressions/314-error-control-flow.md#3148-panic과-fatal-에러)).

## 2.14.5. `panic_fn` / `oom_fn` / `unreachable_fn`: 컴파일러 훅

에러 모델의 가장 아래에는 컴파일러가 직접 호출하는 hook 함수가 있습니다.

```rok
mod panic {                         // rok::core::panic
    #[lang("panic_fn")]
    #[track_caller]
    fn panic(msg: str) -> ! {
        rok_panic_at(msg)            // 호출 위치와 함께 stderr 출력 후 종료 코드 101
    }

    #[lang("oom_fn")]
    fn oom() -> ! {
        rok_oom()                    // "fatal error: out of memory", 종료 코드 101
    }

    #[lang("unreachable_fn")]
    #[track_caller]
    fn unreachable() -> ! {
        rok_panic_at("internal error: entered unreachable code")
    }
}
```

각 함수의 반환 타입은 `-> !`(never)이며, 정상적으로 반환하는 일이 없습니다. `panic`과 `unreachable`은 `#[track_caller]`이므로, 보고되는 위치는 `#[track_caller]` 사슬 밖의 첫 호출자입니다.

### panic이 일어나는 자리

컴파일러는 다음 자리에서 `panic_fn`을 호출합니다. 어느 경우든 그 panic은 `catch`로 잡을 수 없으며, `finally`와 `Drop`은 실행되지 않습니다.

- `try!`의 에러 경로. 메시지는 에러 값의 `Display`(없으면 `Debug`)
- 0으로 나누기, 정수 오버플로, 경계 검사 실패
- `Result::unwrap`의 `Err` 같은 표준 라이브러리의 명시적 panic 호출

`throw`는 panic이 아닙니다. `throw`는 에러를 던지는 함수의 에러 결과를 만들 뿐이며([refs-lang 8.9.7](../../refs-lang/008-abi/809-lowering-control.md#897-throw)), `panic_fn`을 거치지 않습니다.

### OOM 처리

GC 할당이 실패하면 OOM은 다음과 같이 처리됩니다. 런타임은 Rok 함수를 이름으로 부를 수 없습니다. 그래서 `oom()` 대신 런타임 심볼 `rok_oom`을 직접 부릅니다([refs-lang 8.5.7](../../refs-lang/008-abi/805-extern-symbols.md#857-진단-영역)).

```
rok_gc_alloc_slow:
    if !try_collect_and_retry() {
        rok_oom();                       // "fatal error: out of memory", exit 101
    }
```

core의 `oom()`도 같은 `rok_oom`을 부르므로 두 경로의 출력과 종료 코드는 같습니다.

OOM은 Fatal이므로 잡을 수 없습니다.

### `unreachable` 사용

`unreachable`은 도달할 수 없는 분기를 명시할 때 씁니다.

```rok
match (x: i32 | f64) {
    v: i32 => process_int(v),
    v: f64 => process_float(v),
    // 컴파일러가 모든 variant 처리 확인, unreachable 자리 없음
}

// 또는 명시.
fn require_positive(x: i32) -> u32 {
    if x < 0 { rok::core::panic::unreachable() }     // 호출자 보장
    x as u32
}
```

컴파일러는 다음 자리에 `unreachable`을 자동으로 넣습니다.
- `match` 망라성 검사 뒤에 남은 죽은 가지
- `if let ... else`의 else가 발산함을 보장하는 자리

### `assert` / `assert_eq` / `assert_ne`

```rok
fn assert(cond: bool, msg: str = "") -> ();
fn assert_eq<T: PartialEq + Debug>(left: T, right: T, msg: str = "") -> ();
fn assert_ne<T: PartialEq + Debug>(left: T, right: T, msg: str = "") -> ();
```

core prelude로 어디서든 쓸 수 있습니다. 조건이 거짓이면 panic이 일어납니다(종료 코드 101). 메시지는 다음과 같으며, `msg`가 비어 있지 않으면 `: msg`를 붙입니다.

```
assertion failed[: msg]
assertion `left == right` failed[: msg]
  left: <left 의 Debug>
 right: <right 의 Debug>
```

`assert_ne`는 `left != right`이고 나머지는 같습니다.

## 2.14.6. `try` / `catch` / `throw` / `throws` / `finally`

문법의 권위는 [refs-lang 3.14 에러 제어 흐름](../../refs-lang/003-expressions/314-error-control-flow.md)과 [refs-lang 4.14 `throws` 시그니처](../../refs-lang/004-functions/414-throws.md)에 있습니다. 이 절은 표준 라이브러리 쪽 관례만 다룹니다.

### 시그니처 모델

함수 시그니처는 에러를 던지는 동작을 다음 다섯 형태로 표현합니다.

```rok
fn foo() -> T                      // non-throwing: 절대 throw 안 함
fn foo() throws -> T               // untyped: throws (any Error)
fn foo() throws (E) -> T           // typed single: E 만
fn foo() throws (A | B | C) -> T   // typed sum: A | B | C 만
fn foo(f: fn() throws -> T) rethrows -> T   // f 가 던질 때만 던짐
```

공개 API에는 타입 없는 `throws`를 기본으로 권장합니다. API 사양이 가벼워지기 때문입니다. `throws (E)`는 라이브러리 내부 API, 임베디드 환경, 그리고 에러를 잡는 쪽에서 에러에 따라 분기해야 할 때 알맞습니다. 타입 합인 `throws (A | B | C)`는 Swift SE-0413의 typed throws를 합 타입으로 넓힌 형태입니다. 함수형 매개변수를 받는 표준 라이브러리 고차 함수는 `rethrows`를 씁니다. 그래서 에러를 던지지 않는 클로저를 넘긴 호출은 `try` 없이 쓸 수 있습니다.

### `try` / `catch` 형태

`catch`는 후위 식입니다. 에러를 잡으면 핸들러 블록의 값이 식 전체의 값이 됩니다.

```rok
let result = try compute() catch e: TypeError {
    log(f"type error: {e}");
    default_value
};
```

### `finally`

자원 정리 같은 일에는 `finally` 블록을 씁니다.

```rok
let f = try File::open(path);
try {
    process(f);
} catch e: IoError {
    log(f"io error: {e}");
} finally {
    f.close();      // 정상 / 에러 둘 다 실행
}
```

`finally`의 실행 의미는 다음과 같습니다.
- 정상 종료하면 실행됩니다.
- 에러를 잡은 뒤 종료해도 실행됩니다.
- 잡히지 않은 에러는 이 블록을 실행한 뒤 전파됩니다.
- `return` / `break` / `continue`로 빠져나가도 실행됩니다.
- panic이나 Fatal이면 프로세스가 바로 끝나므로 실행되지 않습니다.

`finally` 안에서 던진 에러는 현재 에러를 대체합니다.

### `throw`와 분기 타입

`throw value;`는 문장이며, `throw`로 끝나는 블록은 발산합니다. 따라서 분기 결과 타입에는 다른 분기의 타입만 반영됩니다. 예를 들어 `if c { x } else { throw e; }`의 결과는 `x`의 타입입니다.

## 2.14.7. `Result` ↔ `throws` 변환

두 도구는 서로 명시적으로 변환할 수 있습니다.

### 던지는 호출 → `Result<T, E>`

`Result::catching`은 에러를 던지는 클로저를 호출해 결과를 담습니다. `catch` 식으로 직접 만들 수도 있습니다.

```rok
let r: Result<i32, ParseError> = Result::catching(|| try parse_int(s));
let r: Result<i32, ParseError> = try Ok(parse_int(s)) catch e { Err(e) };
```

### `Result<T, E>` → throw

`get`은 `Err`를 다시 던집니다. 둘러싼 함수의 `throws`가 그 에러 타입을 받아야 합니다.

```rok
fn first_ok(rs: List<Result<i32, ParseError>>) throws (ParseError) -> i32 {
    try rs[0].get()
}
```

### `T | none`으로 추출

에러를 버리고 값만 필요하면, 에러를 던지는 호출에는 `try?`를, 담아 둔 `Result`에는 `ok()` / `err()`를 씁니다.

```rok
let a: i32 | none = try? parse_int("42");
let r: Result<i32, ParseError> = Result::catching(|| try parse_int("42"));
if let Ok(v) = r { use(v); }
if let n: i32 = r.ok() { use(n); }     // T | none 으로 추출
```

## 2.14.8. 모듈 매핑

각 에러 타입이 구현하는 트레잇은 다음과 같습니다.

| 타입                 | Display | Debug         | Error | 비고                       |
|----------------------|---------|---------------|-------|----------------------------|
| `TypeError`          | ✓       | ✓             | ✓     |                            |
| `ValueError`         | ✓       | ✓             | ✓     |                            |
| `IndexError`         | ✓       | ✓             | ✓     |                            |
| `OverflowError`      | ✓       | ✓             | ✓     | 검사하는 변환의 범위 초과  |
| `ParseError`         | ✓       | ✓             | ✓     |                            |
| `ConversionError`    | ✓       | ✓             | ✓     | 기본 `TryFrom::Error`      |
| `ZeroError`          | ✓       | ✓             | ✓     | `rok::core::num::niche`에 있음 |
| `IoError`            | ✓       | ✓             | ✓     | std 계층 (타깃 의존)       |
| `StackOverflowError` | ✓       | ✓             | ✓     | Fatal: 잡을 수 없음        |
| `OutOfMemoryError`   | ✓       | ✓             | ✓     | Fatal: 잡을 수 없음        |
| `InternalError`      | ✓       | ✓             | ✓     | Fatal: 컴파일러 버그일 때  |
| `Result<T, E>`       | ✗       | ✓ (T,E:Debug) | ✗     | 보관용 enum                |
| `any Error`          | ✓       | ✓             | ✓     | 제약 붙은 any, 사슬 가능   |

`Display`는 사용자가 읽기 좋은 메시지를 냅니다. 예를 들어 `IndexError { index: 10, len: 3 }`은 `"index 10 out of bounds (len 3)"`입니다. `Debug`는 구조를 그대로 보여 줍니다. `any Error` 값은 런타임 태그로 실제 타입의 `Display` / `Debug` / `source`를 호출합니다.

## 2.14.9. 다른 언어와 비교

다른 주요 언어와 비교하면 Rok의 에러 모델은 다음과 같습니다.

| 측면           | Rust           | Swift                   | Java            | Rok                                    |
|----------------|----------------|-------------------------|-----------------|----------------------------------------|
| 기본 모델      | `Result<T, E>` | `throws`                | `throws`        | `throws`, `Result`는 보관용           |
| 호출 지점 표시 | `?`            | `try`                   | **없음**        | `try` / `try?` / `try!`                |
| typed throws   | **없음**       | `throws(E)` (단일 타입) | `throws E1, E2` | `throws (E)` / `throws (A \| B)`       |
| `?` 연산자     | `Result?`      | optional chaining       | **없음**        | `T \| none` 전용                       |
| Option 타입    | `Option<T>`    | `Optional<T>`           | `Optional<T>`   | **없음**: `T \| none` 사용             |
| 구현 방식      | 반환값         | 반환값 (에러 레지스터)  | 스택 unwinding  | 반환값 `{ is_err, payload }`           |
| panic          | `panic!`       | `fatalError`            | `Error`         | `panic_fn`: catch 불가, 종료 코드 101  |

Rok은 크게 세 가지가 다릅니다. 첫째, `Option`이 없고 그 자리를 합 타입 `T | none`이 대신하므로 타입이 하나 줄어듭니다. 둘째, 에러 전파는 `throws`와 `try` 하나로 통일하고, `Result`는 에러를 값으로 담아 두는 역할만 맡습니다. 셋째, 타입 합 throws인 `throws (A | B | C)`는 단일 타입만 받는 Swift의 typed throws보다 표현력이 높습니다.
