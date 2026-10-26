# 3.17. `using` 표현식 - 자원의 결정론적 해제

`using` 키워드 ([키워드](../001-lexical/105-keywords.md)) 는 자원을 한 블록의 수명에 묶어 두기 위한 표현식 형태입니다. 블록이 어떤 경로로 끝나든 (정상 종료, 조기 `return`, `?` 의 `none` 반환, 에러 전파) 묶인 자원의 `Drop::drop` 이 반드시 호출됩니다.

GC 가 메모리는 자동으로 회수해 주지만 외부 자원 (file, socket, mutex, FFI 핸들, GC pin 등) 은 결정론적으로 해제되어야 합니다. `using` 은 이 결정론을 문법으로 보장하며, `Drop::drop` 이 호출되는 유일한 경로입니다. 일반 바인딩은 scope 를 벗어나도 `drop` 되지 않습니다 ([stdlib 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop---using-해제-시-정리)).

## 3.17.1. 형태

```
using_expr     ::= "using" using_binding ("," using_binding)* block
using_binding  ::= pattern "=" expr
```

각 구성 요소의 의미는 다음과 같이 정리됩니다.

- `pattern` 은 [패턴](311-patterns.md) 에서 정의한 반박 불가능한 (irrefutable) 패턴이면 무엇이든 됩니다 (식별자, 튜플, struct 분해 등). 대부분의 경우 단일 식별자를 씁니다. 해제되는 것은 `expr` 의 값 전체입니다.
- `expr` 은 자원을 만들거나 받아오는 식입니다. 이 식의 결과 타입은 [`Drop`](../../refs-standard-libraries/200-core/213-marker.md) 트레잇을 구현해야 합니다.
- 자원 획득이 실패할 수 있으면 `expr` 을 `try` 로 표시합니다 (`using f = try fs::open(path) { ... }`). 획득이 에러를 내면 자원이 아예 묶이지 않은 상태에서 빠져나가므로 `Drop` 호출도 일어나지 않습니다. 자세한 의미는 [에러 제어 흐름](314-error-control-flow.md) 을 따릅니다.
- `,` 로 여러 binding 을 이어 적으면 다중 자원이 되며, 자세한 동작은 [3.17.4 다중 자원](#3174-다중-자원) 에서 정의합니다.
- `block` 은 [블록 표현식](309-block-expressions.md) 의 규칙을 그대로 따르며, 그 마지막 식의 값이 `using` 식 전체의 값이 됩니다.

## 3.17.2. 가장 간단한 예

```rok
using f = fs::open(path) {
    f.read_to_end()                   // 마지막 식이 using 의 값
}                                  // ← 여기서 f.drop() 자동 호출
```

`fs::open(path)` 의 반환 타입은 `File` 이고 `File: Drop` 이므로 `using` 의 대상이 됩니다. 블록이 끝나면 `File::drop` 이 호출되어 OS 핸들이 close 됩니다.

## 3.17.3. `try` 와의 결합

자원 획득 자체가 실패할 수 있는 경우 획득 식을 `try` 로 표시합니다.

```rok
fn read_first_line(path: str) throws (IoError) -> str {
    using f = try fs::open(path) {     // 실패하면 IoError 가 전파됨
        try f.read_line()              // 본체 안에서도 try 로 전파 가능
    }                                  // f 가 묶였다면 정상/에러 모두 drop
}
```

이 형태는 의미상 다음 풀어쓴 코드와 동등합니다.

```rok
let f = try fs::open(path);
try {
    f.read_line()
} finally {
    f.drop();          // 컴파일러가 넣는 호출 - 사용자가 직접 쓰면 에러
}
```

본체 안에서 에러가 전파되더라도 `finally` 에 해당하는 자리에서 `drop` 이 호출됩니다.

## 3.17.4. 다중 자원

여러 자원을 한 블록에 묶어 두려면 `using` 을 중첩하거나 콤마로 이어 적습니다. 본 사양은 두 형태를 모두 허용하며 의미는 동등합니다.

```rok
using a = AcquireA() {
    using b = AcquireB(a) {
        body
    }
}

// 또는 단축 형태
using a = AcquireA(), b = AcquireB(a) {
    body
}
```

위 두 형태 모두 `b` 가 먼저, `a` 가 다음에 drop 되는 **역순 해제** 를 따릅니다. Java try-with-resources 와 같은 규칙입니다. 역순으로 해제해야 의존 관계 (`b` 가 `a` 를 참조) 가 안전하게 풀립니다.

## 3.17.5. 식으로서의 `using`

`using` 은 문이 아니라 표현식이며, 그 값은 본체 블록의 값이 됩니다.

```rok
let content = using f = try fs::open(path) {
    try f.read_to_end()
};
```

이 점이 함수 단위 `defer` (Go) 와의 가장 큰 구조적 차이입니다. `using` 표현식은 [블록 표현식](309-block-expressions.md) 의 일부 규칙을 그대로 따르며, 다른 표현식 자리 어디에서도 사용할 수 있습니다.

## 3.17.6. drop 호출 시점

`using` 은 다음 다섯 경로 모두에서 묶인 자원의 `drop` 을 보장합니다.

- 블록의 정상 종료 (마지막 식 평가 완료)
- 본체 안의 `return`
- 본체 안의 `break` / `continue`
- 본체 안의 `?` 에 의한 `none` 반환
- 본체 안의 `throw` 및 `try` 로 표시한 호출의 에러 전파

panic 과 fatal 에러는 프로세스를 곧바로 종료하므로 `drop` 을 호출하지 않습니다 ([3.14.8](314-error-control-flow.md#3148-panic-과-fatal-에러)). 이는 panic 의 일반 결과이며 `using` 한정 동작이 아닙니다.

## 3.17.7. 묶을 수 있는 타입

`using` 의 우변 식의 타입은 [`Drop`](../../refs-standard-libraries/200-core/213-marker.md) 을 구현하거나, `Drop` 을 가진 필드를 담아야 합니다 (drop glue, [stdlib 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop---using-해제-시-정리)). 둘 다 아니면 컴파일 에러입니다.

```rok
using x = 42 { ... }   // ERROR: using 에는 Drop 구현 타입이 필요 - i32 는 Drop 을 구현하지 않음
```

이 제약은 의미가 있는 자원만 `using` 으로 묶이도록 강제합니다. 단순 값에 `using` 이 등장하는 일을 사양 차원에서 차단합니다.

## 3.17.8. 다른 언어와의 비교

| 언어   | 형태                          | 스코프 단위  | 비고                            |
|--------|-------------------------------|--------------|---------------------------------|
| Rok    | `using x = try e { ... }`     | block        | 표현식, 다중 자원, `try` 통합   |
| C#     | `using (var x = e) { ... }`   | block        | `IDisposable` 필요              |
| Java   | `try (var x = e) { ... }`     | try block    | `AutoCloseable` 필요, 다중 자원 |
| Python | `with x as f: ...`            | block        | `__enter__` / `__exit__`        |
| Swift  | `defer { x.close() }`         | function     | 명시적 작성 필요                |
| Go     | `defer x.Close()`             | **function** | 블록 단위 X                     |
| Zig    | `defer x.close();`            | block        | `errdefer` 변형                 |
| Rust   | (RAII 자동, 별도 키워드 없음) | block        | GC 없음 - drop 시점 자동 결정   |

Rok 의 `using` 은 GC 언어 (C#, Java, Python) 의 명시적 블록 패턴과 식 (expression) 으로서의 합성성, `try` 통합, 다중 자원 역순 해제를 결합한 형태입니다.
