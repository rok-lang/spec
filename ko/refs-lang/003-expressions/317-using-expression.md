# 3.17. `using` 표현식: 자원의 결정론적 해제

`using` 키워드([키워드](../001-lexical/105-keywords.md))는 자원을 블록 하나의 수명에 묶는 표현식 형태입니다. 블록이 어떻게 끝나든(정상 종료, 조기 `return`, `?`의 `none` 반환, 에러 전파) 묶인 자원의 `Drop::drop`이 항상 호출됩니다.

GC가 메모리는 자동으로 회수하지만, 외부 자원(파일, 소켓, 뮤텍스, FFI 핸들, GC pin 등)은 결정론적으로 해제해야 합니다. `using`은 이 결정론을 문법으로 보장합니다. `Drop::drop`이 호출되는 경로는 이것뿐입니다. 일반 바인딩은 스코프를 벗어나도 `drop`이 호출되지 않습니다([stdlib 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop-using-해제-시-정리)).

## 3.17.1. 형태

```
using_expr     ::= "using" using_binding ("," using_binding)* block
using_binding  ::= pattern "=" expr
```

각 구성 요소의 의미는 다음과 같습니다.

- `pattern`은 [패턴](311-patterns.md)에서 정의한 반증 불가능(irrefutable) 패턴이면 무엇이든 됩니다(식별자, 튜플, struct 분해 등). 대부분은 식별자 하나를 씁니다. 해제되는 것은 `expr`의 값 전체입니다.
- `expr`은 자원을 만들거나 얻어 오는 식입니다. 이 식의 결과 타입은 [`Drop`](../../refs-standard-libraries/200-core/213-marker.md) 트레잇을 구현해야 합니다.
- 자원 획득이 실패할 수 있으면 `expr`을 `try`로 표시합니다(`using f = try fs::open(path) { ... }`). 획득에서 에러가 나면 자원이 묶이지 않은 채로 빠져나가므로 `Drop` 호출도 일어나지 않습니다. 자세한 의미는 [에러 제어 흐름](314-error-control-flow.md)을 따릅니다.
- `,`로 여러 바인딩을 이어 쓰면 다중 자원이 됩니다. 자세한 동작은 [3.17.4 다중 자원](#3174-다중-자원)에서 정의합니다.
- `block`은 [블록 표현식](309-block-expressions.md)의 규칙을 그대로 따르며, 마지막 식의 값이 `using` 식 전체의 값입니다.

## 3.17.2. 가장 간단한 예

```rok
using f = fs::open(path) {
    f.read_to_end()                   // 마지막 식이 using 의 값
}                                  // ← 여기서 f.drop() 자동 호출
```

`fs::open(path)`의 반환 타입은 `File`이고 `File: Drop`이므로 `using`의 대상이 될 수 있습니다. 블록이 끝나면 `File::drop`이 호출되어 OS 핸들을 닫습니다.

## 3.17.3. `try`와 결합

자원 획득 자체가 실패할 수 있으면 획득 식을 `try`로 표시합니다.

```rok
fn read_first_line(path: str) throws (IoError) -> str {
    using f = try fs::open(path) {     // 실패하면 IoError 가 전파됨
        try f.read_line()              // 본체 안에서도 try 로 전파 가능
    }                                  // f 가 묶였다면 정상/에러 모두 drop
}
```

이 형태는 의미상 다음 코드와 같습니다.

```rok
let f = try fs::open(path);
try {
    f.read_line()
} finally {
    f.drop();          // 컴파일러가 넣는 호출. 사용자가 직접 쓰면 에러
}
```

본체 안에서 에러가 전파되어도 `finally`에 해당하는 자리에서 `drop`이 호출됩니다.

## 3.17.4. 다중 자원

자원 여러 개를 한 블록에 묶으려면 `using`을 중첩하거나 콤마로 이어 씁니다. 두 형태 모두 쓸 수 있고, 의미는 같습니다.

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

두 형태 모두 `b`를 먼저, `a`를 나중에 해제하는 **역순 해제**를 따릅니다. Java try-with-resources와 같은 규칙입니다. 역순으로 해제해야 의존 관계(`b`가 `a`를 참조)가 안전하게 풀립니다.

## 3.17.5. 식으로 쓰는 `using`

`using`은 문장이 아니라 표현식이며, 그 값은 본체 블록의 값입니다.

```rok
let content = using f = try fs::open(path) {
    try f.read_to_end()
};
```

이 점이 함수 단위로 동작하는 `defer`(Go)와 구조적으로 가장 크게 다른 점입니다. `using` 표현식은 [블록 표현식](309-block-expressions.md)의 해당 규칙을 따르며, 표현식이 올 수 있는 자리라면 어디에나 쓸 수 있습니다.

## 3.17.6. drop 호출 시점

`using`은 다음 다섯 경로 모두에서 묶인 자원의 `drop`을 보장합니다.

- 블록의 정상 종료 (마지막 식 평가 완료)
- 본체 안의 `return`
- 본체 안의 `break` / `continue`
- 본체 안의 `?`가 일으킨 `none` 반환
- 본체 안의 `throw`와 `try`로 표시한 호출의 에러 전파

panic과 fatal 에러는 프로세스를 바로 종료하므로 `drop`을 호출하지 않습니다([3.14.8](314-error-control-flow.md#3148-panic과-fatal-에러)). panic의 일반적인 동작이며, `using`에만 해당하지 않습니다.

## 3.17.7. 묶을 수 있는 타입

`using` 우변 식의 타입은 [`Drop`](../../refs-standard-libraries/200-core/213-marker.md)을 구현하거나, `Drop`이 있는 필드를 담아야 합니다(drop glue, [stdlib 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop-using-해제-시-정리)). 둘 다 아니면 컴파일 에러입니다.

```rok
using x = 42 { ... }   // ERROR: using 에는 Drop 구현 타입이 필요. i32 는 Drop 을 구현하지 않음
```

이 제약은 의미 있는 자원만 `using`으로 묶이게 합니다. 단순한 값에 `using`을 쓰는 일을 사양 차원에서 막습니다.

## 3.17.8. 다른 언어와 비교

| 언어   | 형태                          | 스코프 단위  | 비고                            |
|--------|-------------------------------|--------------|---------------------------------|
| Rok    | `using x = try e { ... }`     | block        | 표현식, 다중 자원, `try` 통합   |
| C#     | `using (var x = e) { ... }`   | block        | `IDisposable` 필요              |
| Java   | `try (var x = e) { ... }`     | try block    | `AutoCloseable` 필요, 다중 자원 |
| Python | `with x as f: ...`            | block        | `__enter__` / `__exit__`        |
| Swift  | `defer { x.close() }`         | function     | 명시적 작성 필요                |
| Go     | `defer x.Close()`             | **함수**     | 블록 단위 아님                  |
| Zig    | `defer x.close();`            | block        | `errdefer` 변형                 |
| Rust   | (RAII 자동, 별도 키워드 없음) | block        | GC 없음, drop 시점 자동 결정    |

Rok의 `using`은 GC 언어(C#, Java, Python)의 명시적 블록 패턴에 세 가지를 더했습니다. 식으로 합성할 수 있고, `try`와 통합되며, 다중 자원을 역순으로 해제합니다.
