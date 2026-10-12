# 4.16. 진입점

실행 패키지가 시작될 때 어떤 코드가 어떤 순서로 실행되는지를 정의합니다. [소스 단위](../000-meta/003-source-units.md) 가 "진입점의 정확한 형태는 함수 사양에서 정의" 한다고 남겨 둔 자리가 바로 이 절입니다.

## 4.16.1. 진입 모듈: `main.rok`

실행 패키지의 루트 파일 `main.rok` ([0.3.1](../000-meta/003-source-units.md#실행-패키지-mainrok)) 가 **진입 모듈** 입니다. 진입 모듈은 다음 두 형태 중 **정확히 하나** 로 진입점을 제공합니다.

| 형태              | 진입점                                         | 용도                           |
|-------------------|------------------------------------------------|--------------------------------|
| 명시 `fn main`    | 사용자가 선언한 `fn main` 함수                 | 일반 애플리케이션              |
| top-level 문      | top-level 문들을 본체로 하는 **암묵 `main`**   | 스크립트, 예제, 짧은 프로그램  |

```ebnf
main_file   ::= item* fn_main item*            // 형태 1: 명시 main
              | (item | top_stmt)*             // 형태 2: top-level 문 (fn main 없음)
top_stmt    ::= let_stmt | let_else_stmt | expr ";" | block_like_expr ";"?   // 3.9.1
module_file ::= item*                          // lib.rok 및 main.rok 가 아닌 모든 모듈 파일
```

`item` 은 [모듈 안의 항목](../005-modules/504-modules.md#547-모듈-안의-항목) (`fn`, `struct`, `enum`, `trait`, `impl`, `const`, `static`, `mod`, `use` 등) 이고, `let_stmt` / `let_else_stmt` / `expr ";"` 은 [블록 표현식](../003-expressions/309-block-expressions.md) 의 문장과 같습니다. 제어 흐름 (`if`, `match`, `for`, `while`, `loop`, `try`) 은 모두 식이므로 `expr ";"` 에 포함됩니다. 함수 본체와 달리 종결자 없는 마지막 식 (`tail_expr`) 은 두지 않습니다 — 암묵 `main` 은 값을 반환하지 않기 때문입니다.

> [!IMPORTANT]
> 한 `main.rok` 안에 top-level 문과 `fn main` 선언이 **함께** 있으면 **컴파일 에러** 입니다. 두 진입점 가운데 어느 쪽이 실행되는지를 독자가 추측해야 하는 상황을 원천적으로 막기 위함입니다.

```rok
// main.rok — ERROR
fn main() { run(); }
run();                      // ERROR: fn main 이 있는 파일에 top-level 문
```

## 4.16.2. top-level 문의 의미: 암묵 `main`

`fn main` 이 없는 `main.rok` 의 top-level 문들은 **소스 순서대로** 하나의 함수 본체로 모여, 컴파일러가 합성하는 암묵 `main` 의 본체가 됩니다. 항목 (item) 은 실행되는 대상이 아니므로 본체에서 빠지며, 항목과 문이 서로 섞여 있어도 문 사이의 상대 순서만 의미를 가집니다.

```rok
// main.rok
let names = ["ada", "grace"];
for n in names {
    greet(n);               // greet 은 아래에서 선언 — 순서 무관하게 보임
}

fn greet(name: str) {
    println(f"hello, {name}");
}

let total = names.len();
println(f"{total} greeted");
```

위 파일은 다음과 같이 쓴 것과 같은 의미를 가집니다.

```rok
fn greet(name: str) {
    println(f"hello, {name}");
}

fn main() throws {          // 암묵 main 의 형태 — 4.16.4 참고
    let names = ["ada", "grace"];
    for n in names {
        greet(n);
    }
    let total = names.len();
    println(f"{total} greeted");
}
```

이 등가 변환에서 다음 규칙이 따라 나옵니다.

- **항목은 순서와 무관하게 보입니다.** `main.rok` 의 항목은 일반 모듈 항목과 똑같이 모듈 스코프에 속하므로, top-level 문은 자신보다 뒤에 선언된 `fn` / `struct` 등도 참조할 수 있습니다.
- **top-level `let` 은 암묵 `main` 의 지역 변수입니다.** 전역 변수가 아니며, 선언 이후의 top-level 문에서만 보입니다 ([변수 선언](../005-modules/501-variable-declaration.md) 의 블록 스코프 규칙 그대로).
- **top-level `let` 은 항목 본체 안에서 보이지 않습니다.** 위 예에서 `greet` 의 본체가 `names` 나 `total` 을 참조하면 이름 해소 에러입니다. 지역 변수를 클로저로 캡처하는 것은 가능합니다 (`let shout = |s| println(f"{s}! ({total})");`) — 클로저는 문이고, 암묵 `main` 본체 안에서 만들어지기 때문입니다.

> [!NOTE]
> **top-level `let` 이 항목에서 보이지 않는 이유.** 항목은 순서와 무관하게 존재하지만 `let` 은 실행 시점에 비로소 값이 생깁니다. 만약 `fn greet` 이 top-level `let` 을 볼 수 있다면, `greet` 이 그 `let` 보다 **먼저** 호출되는 경우 (예: 첫 줄에서 `greet()` 호출) 의 의미를 정해야 하고, 이는 곧 "전역 초기화 순서" 의미론을 언어에 들이는 일이 됩니다. rok 은 전역 초기화 순서를 갖지 않는 것을 원칙으로 삼으므로 (4.16.6), top-level `let` 을 암묵 `main` 의 지역 변수로 한정합니다. 항목 사이에 공유해야 하는 상태는 매개변수로 넘기거나 `static` 으로 선언합니다.

## 4.16.3. 명시 `main` 의 시그니처

명시 `fn main` 은 매개변수와 제네릭 매개변수를 갖지 않으며, 다음 형태 가운데 하나여야 합니다. 그 외의 형태는 컴파일 에러입니다.

| 시그니처                          | 정상 종료 시 종료 코드 | 비고                               |
|-----------------------------------|------------------------|------------------------------------|
| `fn main()`                       | `0`                    |                                    |
| `fn main() -> i32`                | 반환값                 | 반환값이 그대로 프로세스 종료 코드 |
| `fn main() throws` / `throws (E)` | `0`                    | 미처리 예외 → 4.16.4               |
| 위 세 형태에 `async` 를 붙인 것    | 위와 같음              | 4.16.5                             |

`Result` 를 반환하는 `main` 이나 `throws` 와 `-> i32` 를 함께 쓴 `main` 은 없습니다. 에러가 있을 때 특정 종료 코드가 필요하면 `catch` 에서 [`rok::std::env::exit(code)`](../../refs-standard-libraries/300-std/306-env.md) 를 호출합니다.

명령행 인자와 환경 변수는 매개변수가 아니라 표준 라이브러리 [`rok::std::env`](../../refs-standard-libraries/300-std/306-env.md) 로 얻습니다. `unsafe`, `yield` 수식어는 `main` 에 붙일 수 없습니다. 가시성 (`pub`) 은 진입점 여부에 영향을 주지 않습니다.

`lib.rok` 이나 일반 모듈에 선언된 `fn main` 은 진입점이 아니며, 이름이 `main` 인 평범한 함수일 뿐입니다.

## 4.16.4. 종료 코드와 미처리 예외

진입점이 끝나는 방식에 따라 프로세스 종료 코드가 정해집니다.

- **정상 종료**: 4.16.3 의 표대로 `0` 또는 반환된 `i32`.
- **미처리 예외**: `throws` main 에서 에러가 진입점 밖으로 전파되면, 런타임은 예외 값을 표준 에러 (stderr) 에 출력하고 종료 코드 **`1`** 로 종료합니다. 출력 형식은 값이 [`Display`](../../refs-standard-libraries/200-core/210-fmt.md) 를 구현하면 `Display`, 아니면 `Debug`, 둘 다 없으면 타입 이름입니다 ([`Error`](../../refs-standard-libraries/200-core/214-error.md) 트레잇은 `Display` 를 요구하므로 일반적인 에러 타입은 `Display` 로 출력됩니다).
- **panic**: panic 과 fatal 에러는 `catch` 로 잡히지 않으며, 런타임은 메시지를 표준 에러에 출력하고 종료 코드 **`101`** 로 종료합니다 ([4.14.13](414-throws.md#41413-panic-과-fatal-에러)).
- **명시 종료**: [`rok::std::env::exit(code)`](../../refs-standard-libraries/300-std/306-env.md) 는 어느 지점에서든 즉시 프로세스를 종료합니다. 이 경로는 `finally` 와 `Drop` 을 실행하지 않습니다.

**암묵 `main`** 은 `fn main() throws` 로 취급됩니다 (본체에 `await` 가 있으면 `async fn main() throws`, 4.16.5). 따라서 top-level 문에서 `try` 로 표시한 throwing 호출의 에러는 별도의 `catch` 없이 진입점 밖으로 전파될 수 있고, 미처리 예외는 위 규칙대로 종료 코드 `1` 이 됩니다. 정상적으로 끝까지 실행되면 종료 코드는 `0` 입니다.

> [!IMPORTANT]
> **top-level `return` 은 허용하지 않습니다.** `main.rok` 의 top-level 에 `return` 이 나타나면 컴파일 에러입니다. 암묵 `main` 에서 0 이 아닌 종료 코드가 필요하면 `env::exit(code)` 를 호출하고, 반환값으로 종료 코드를 표현하고 싶다면 명시 `fn main() -> i32` 형태를 씁니다. top-level `return` 을 허용하면 "어떤 타입의 `return` 이 허용되는가", "top-level 블록 안쪽의 `return` 은 무엇을 반환하는가" 같은 규칙이 암묵 `main` 에만 추가되므로, 진입점 형태를 둘로 나누는 쪽이 더 단순합니다.

## 4.16.5. 비동기 `main`

`async fn main()` 은 런타임 [스케줄러](../007-runtime/706-scheduler.md) 위의 최상위 task 로 실행됩니다. 플랫폼 진입 wrapper 가 런타임을 초기화한 뒤 `main` 이 만든 Future 를 스케줄러에 넘기고, 그 Future 가 완료될 때까지 기다린 다음 4.16.4 의 규칙으로 종료 코드를 정합니다 ([`task::block_on`](../../refs-standard-libraries/200-core/220-future.md) 과 같은 동작). 이때 `main` 이 끝난 시점에 아직 실행 중인 다른 task 는 기다리지 않습니다.

암묵 `main` 은 top-level 문에 `await` 가 (다른 `async` 블록이나 클로저 안이 아닌 위치에) 직접 나타날 때 `async fn main() throws` 가 되고, 그렇지 않으면 동기 `fn main() throws` 입니다.

```rok
// main.rok — 암묵 async main
let body = await http::get("https://example.com");
println(body.len());
```

## 4.16.6. 모듈은 정의 전용

top-level 문은 **진입 모듈 `main.rok` 에서만** 허용됩니다. `lib.rok` 과 `main.rok` 가 아닌 모든 모듈 파일 (그리고 인라인 `mod { ... }` 본체) 은 항목만 담을 수 있으며 (`module_file ::= item*`), top-level 문이 나타나면 **컴파일 에러** 입니다.

그 결과 **모듈을 가져오는 일 (`mod`, `use`) 은 어떤 코드도 실행하지 않습니다.** 이는 Python 과 의도적으로 다른 지점입니다. Python 에서는 `import` 가 모듈 파일의 top-level 코드를 실행하므로, 라이브러리로도 쓰이고 스크립트로도 쓰이는 파일은 `if __name__ == "__main__":` 가드로 두 용도를 구분해야 합니다. rok 에는 이런 가드가 필요하지 않습니다. 스크립트로 실행되는 코드는 오직 `main.rok` 에만 있고, 그 밖의 모든 파일은 import 시점에 아무 일도 일으키지 않습니다.

이 결정의 근거는 다음과 같습니다.

- **AOT 우선** ([0.1.4](../000-meta/001-execution-modes.md#014-구현-우선순위)). AOT 산출물에서 import 시점 실행을 지원하려면 모듈별 초기화 함수와 그 호출 순서를 링크 단계에서 합성해야 합니다. 정의 전용 모듈은 이 장치가 필요 없습니다.
- **초기화 순서 의미론이 없음.** 모듈 사이의 import 시점 코드는 "어느 모듈이 먼저 초기화되는가", "순환 import 시 무엇이 보이는가" 를 정의해야 합니다. rok 은 이 질문 자체가 생기지 않도록 합니다. 전역 상태의 초기값은 컴파일 타임에 결정됩니다 (4.16.7).
- **캐시와 증분 빌드.** 모듈의 의미가 그 정의만으로 결정되므로, 내용 해시 ([0.3.3](../000-meta/003-source-units.md#033-패키지-식별자)) 기반 캐시와 증분 빌드가 실행 부수효과를 고려할 필요가 없습니다.
- **패키지 종류는 파일 존재로 결정** ([0.3.4](../000-meta/003-source-units.md#034-매니페스트-roktoml-는-선택)). 실행 가능한 코드가 있는 곳이 곧 `main.rok` 이므로, "이 패키지는 실행 패키지인가" 라는 질문과 "실행 코드가 어디에 있는가" 라는 질문의 답이 항상 같습니다.

## 4.16.7. 전역 상태

모듈 수준의 전역 상태는 [`static`](../005-modules/503-static.md) 으로만 선언하며, 그 초기값은 컴파일 타임에 평가 가능한 `const` 식이어야 합니다 ([5.3.9](../005-modules/503-static.md#539-초기값의-const-한정)). 실행 시점에 처음 접근할 때 계산되는 지연 초기화 (lazy initialization) 는 언어 기능이 아니라 표준 라이브러리 wrapper 의 책임입니다. 따라서 프로그램의 실행은 언제나 진입점의 첫 문장에서 시작하며, 그보다 앞서 사용자 코드가 실행되는 일은 없습니다.

## 4.16.8. 단일 파일 실행

패키지에 속하지 않은 단일 `.rok` 파일도 바로 실행할 수 있습니다.

```
rokc run hello.rok
rokc build hello.rok
```

인자로 받은 파일이 어떤 패키지 디렉토리 안에도 있지 않으면, `rokc` 는 그 파일 하나만으로 이루어진 **익명 실행 패키지** 를 구성하고 그 파일을 해당 패키지의 `main.rok` 로 취급합니다. 파일 이름은 의미를 갖지 않으며, 진입점 규칙 (4.16.1 ~ 4.16.5) 은 `main.rok` 와 똑같이 적용됩니다. 익명 패키지는 파일 기반 서브모듈 (`mod foo;`) 을 가질 수 없지만, 인라인 모듈은 사용할 수 있습니다. 의존성이 필요하면 파일 머리의 inline 매니페스트 블록으로 선언합니다 (계획, [9.10.7](../009-tools/910-rok-frontend.md#9107-단일-파일-스크립트의-inline-매니페스트)). 명령의 자세한 형태는 [`rokc` CLI](../009-tools/901-rokc-cli.md) 에서 정의합니다.

## 4.16.9. 진입점 속성은 없음

rok 에는 진입점을 지정하는 속성 (예: `#[main]`) 이 **없습니다**. 진입 모듈은 파일 이름 `main.rok` 가, 진입 함수는 이름 `main` (또는 top-level 문) 이 이미 지정하므로, 같은 정보를 표현하는 두 번째 수단을 두지 않습니다. 진입점을 정하는 메커니즘을 하나로 유지하면 "어느 함수가 실행되는가" 를 파일 구조만 보고 판단할 수 있습니다. `main` 은 예약어가 아니며, `#[main]` 이라는 이름도 언어가 예약하지 않습니다.

## 4.16.10. JIT / REPL 과의 관계

REPL 의 입력과 Jupyter 의 셀 ([JIT 셀](../009-tools/907-jit-cells.md)) 은 **암묵 `main.rok` 의 끝에 top-level 문과 항목을 차례로 덧붙이는 것** 과 같은 의미를 가집니다. 셀의 `let` 은 암묵 `main` 의 지역 변수로서 이후 셀에서 보이고, 셀의 `fn` / `struct` 는 모듈 항목이 됩니다. 4.16.2 의 규칙도 그대로 적용되므로, 셀에서 선언한 `fn` 의 본체는 이전 셀의 top-level `let` 을 참조할 수 없습니다 (클로저 캡처는 가능합니다).

이 등가성 덕분에 [0.1.1](../000-meta/001-execution-modes.md#의미론은-같습니다) 의 "JIT 와 AOT 는 같은 의미론을 공유한다" 는 원칙이 진입점에서도 성립합니다. 노트북의 셀을 순서대로 이어 붙인 파일은 같은 결과를 내는 `main.rok` 가 됩니다. 다만 다음 세 가지는 대화형 세션에만 있는 **세션 조작** 이며, 언어 의미론의 확장이 아닙니다.

- 셀 최상위에서 같은 이름의 재 `let` 선언 ([9.7.3](../009-tools/907-jit-cells.md#같은-이름의-재선언)). 새 바인딩을 만듭니다.
- 셀의 삭제와 재실행 ([9.7.4](../009-tools/907-jit-cells.md#974-undo--셀-재실행)).
- 종결자 없는 마지막 식의 값 자동 표시 ([9.7.7](../009-tools/907-jit-cells.md#977-시각화--출력)). 파일에서는 이 식에 `;` 를 붙인 것과 같습니다.
