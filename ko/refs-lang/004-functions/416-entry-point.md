# 4.16. 진입점

이 절은 실행 패키지가 시작될 때 어떤 코드가 어떤 순서로 실행되는지 정의합니다. [소스 단위](../000-meta/003-source-units.md)에서 "진입점의 정확한 형태는 함수 사양에서 정의"한다고 남겨 둔 내용입니다.

## 4.16.1. 진입 모듈: `main.rok`

실행 패키지의 루트 파일 `src/main.rok`([0.3.1](../000-meta/003-source-units.md#실행-패키지-srcmainrok))이 **진입 모듈**입니다. 진입 모듈은 다음 두 형태 중 **정확히 하나**로 진입점을 둡니다.

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

`item`은 [모듈 안의 항목](../005-modules/504-modules.md#547-모듈-안의-항목)입니다(`fn`, `struct`, `enum`, `trait`, `impl`, `const`, `static`, `mod`, `use` 등). `let_stmt` / `let_else_stmt` / `expr ";"`은 [블록 표현식](../003-expressions/309-block-expressions.md)의 문장과 같습니다. 제어 흐름(`if`, `match`, `for`, `while`, `loop`, `try`)은 모두 식이므로 `expr ";"`에 들어갑니다. 함수 본체와 달리 종결자 없는 마지막 식(`tail_expr`)은 두지 않습니다. 암묵 `main`은 값을 반환하지 않기 때문입니다. 항목만 있고 문장도 `fn main`도 없는 `main.rok`의 암묵 `main`은 비어 있습니다. 다만 WIT를 내보내는 패키지는 예외입니다. 그 패키지는 진입점 없는 reactor입니다([stdlib 1.8.2](../../refs-standard-libraries/100-bare/108-wit.md#export)).

> [!IMPORTANT]
> 한 `main.rok` 안에 top-level 문과 `fn main` 선언이 **함께** 있으면 **컴파일 에러**입니다. 두 진입점 중 어느 쪽이 실행되는지 읽는 사람이 추측하는 일을 아예 막으려는 규칙입니다.

```rok
// main.rok - ERROR
fn main() { run(); }
run();                      // ERROR: fn main 이 있는 파일에 top-level 문
```

## 4.16.2. 최상위 문의 의미: 암묵 `main`

`fn main`이 없는 `main.rok`에서 컴파일러는 top-level 문을 **소스 순서대로** 함수 본체 하나로 모읍니다. 이 본체가 컴파일러가 합성하는 암묵 `main`의 본체입니다. 항목(item)은 실행하지 않으므로 본체에서 빠집니다. 항목과 문이 섞여 있어도 의미가 있는 것은 문 사이의 상대 순서뿐입니다.

```rok
// main.rok
let names = ["ada", "grace"];
for n in names {
    greet(n);               // greet 은 아래에서 선언 - 순서 무관하게 보임
}

fn greet(name: str) {
    println(f"hello, {name}");
}

let total = names.len();
println(f"{total} greeted");
```

위 파일은 다음과 같이 쓴 것과 의미가 같습니다.

```rok
fn greet(name: str) {
    println(f"hello, {name}");
}

fn main() throws {          // 암묵 main 의 형태 - 4.16.4 참고
    let names = ["ada", "grace"];
    for n in names {
        greet(n);
    }
    let total = names.len();
    println(f"{total} greeted");
}
```

이 등가 변환에서 다음 규칙이 나옵니다.

- **항목은 순서와 상관없이 보입니다.** `main.rok`의 항목은 일반 모듈 항목과 똑같이 모듈 스코프에 속합니다. 그래서 top-level 문은 자신보다 뒤에 선언된 `fn` / `struct` 등도 참조할 수 있습니다.
- **top-level `let`은 암묵 `main`의 지역 변수입니다.** 전역 변수가 아니며 선언 이후의 top-level 문에서만 보입니다. [변수 선언](../005-modules/501-variable-declaration.md)의 블록 스코프 규칙을 그대로 따릅니다.
- **top-level `let`은 항목 본체 안에서 보이지 않습니다.** 위 예에서 `greet`의 본체가 `names`나 `total`을 참조하면 이름 해소 에러입니다. 지역 변수를 클로저로 캡처할 수는 있습니다(`let shout = |s| println(f"{s}! ({total})");`). 클로저는 문이고, 암묵 `main` 본체 안에서 만들어지기 때문입니다.

> [!NOTE]
> **top-level `let`이 항목에서 보이지 않는 이유.** 항목은 순서와 상관없이 존재하지만 `let`은 실행 시점에야 값이 생깁니다. `fn greet`이 top-level `let`을 볼 수 있다면 `greet`을 그 `let`보다 **먼저** 호출할 때의 의미를 정해야 합니다(예: 첫 줄에서 `greet()` 호출). 그러면 "전역 초기화 순서" 의미론을 언어에 들여야 합니다. Rok은 전역 초기화 순서를 두지 않는다는 원칙이 있습니다(4.16.6). 그래서 top-level `let`을 암묵 `main`의 지역 변수로 한정합니다. 항목 사이에 공유해야 하는 상태는 매개변수로 넘기거나 `static`으로 선언합니다.

## 4.16.3. 명시 `main`의 시그니처

명시 `fn main`에는 매개변수와 제네릭 매개변수가 없습니다. 형태는 다음 중 하나여야 합니다. 그 밖의 형태는 컴파일 에러입니다.

| 시그니처                          | 정상 종료 시 종료 코드 | 비고                               |
|-----------------------------------|------------------------|------------------------------------|
| `fn main()`                       | `0`                    |                                    |
| `fn main() -> i32`                | 반환값                 | 반환값이 그대로 프로세스 종료 코드 |
| `fn main() throws` / `throws (E)` | `0`                    | 미처리 예외 → 4.16.4               |
| 위 세 형태에 `async`를 붙인 것    | 위와 같음              | 4.16.5                             |

`Result`를 반환하는 `main`도, `throws`와 `-> i32`를 함께 쓴 `main`도 없습니다. 에러가 났을 때 특정 종료 코드가 필요하면 `catch`에서 [`rok::std::env::exit(code)`](../../refs-standard-libraries/300-std/306-env.md)를 호출합니다.

명령행 인자와 환경 변수는 매개변수가 아니라 표준 라이브러리 [`rok::std::env`](../../refs-standard-libraries/300-std/306-env.md)로 얻습니다. `unsafe`, `yield` 수식어는 `main`에 붙일 수 없습니다. 가시성(`pub`)은 진입점인지 여부에 영향을 주지 않습니다.

`lib.rok`이나 일반 모듈에 선언한 `fn main`은 진입점이 아닙니다. 이름이 `main`인 평범한 함수일 뿐입니다.

## 4.16.4. 종료 코드와 미처리 예외

진입점이 끝나는 방식에 따라 프로세스 종료 코드가 정해집니다.

- **정상 종료**: 4.16.3의 표대로 `0` 또는 반환된 `i32`.
- **미처리 예외**: `throws` main에서 에러가 진입점 밖으로 전파되면 런타임은 예외 값을 표준 에러(stderr)에 출력합니다. 그리고 종료 코드 **`1`**로 끝냅니다. 값이 [`Display`](../../refs-standard-libraries/200-core/210-fmt.md)를 구현하면 `Display`로, 아니면 `Debug`로 출력합니다. 둘 다 없으면 타입 이름을 출력합니다. [`Error`](../../refs-standard-libraries/200-core/214-error.md) 트레잇은 `Display`를 요구하므로 일반적인 에러 타입은 `Display`로 출력됩니다. 다음 줄에는 에러가 `main`을 떠난 위치를 `  --> path:line:col` 형식으로 씁니다.
- **panic**: panic과 fatal 에러는 `catch`로 잡히지 않습니다. 런타임은 메시지를 표준 에러에 출력하고 종료 코드 **`101`**로 끝냅니다([4.14.13](414-throws.md#41413-panic과-fatal-에러)).
- **명시 종료**: [`rok::std::env::exit(code)`](../../refs-standard-libraries/300-std/306-env.md)는 어느 지점에서든 즉시 프로세스를 종료합니다. 이 경로는 `finally`와 `Drop`을 실행하지 않습니다.

**암묵 `main`**은 타입을 적지 않은 `fn main() throws`(에러 타입 `any Error`)로 봅니다. 본체에 `await`가 있으면 `async fn main() throws`로 봅니다(4.16.5). 그래서 top-level 문에서 `try`로 표시한 throwing 호출의 에러는 따로 `catch`하지 않아도 진입점 밖으로 전파될 수 있습니다. 미처리 예외는 위 규칙대로 종료 코드 `1`이 됩니다. 끝까지 정상 실행되면 종료 코드는 `0`입니다.

`wasm32-wasi`(preview 2 component)에서는 `wasi:cli/exit`가 성공과 실패만 전합니다. 그래서 0이 아닌 종료 코드(`1`, `101` 포함)는 모두 호스트에 실패(status 1)로 전해집니다. `wasm32-wasi-threads`(preview 1)는 코드를 그대로 전합니다. 다만 호스트가 126 이상의 `proc_exit`를 실패(status 1)로 바꿀 수 있습니다. `abort` 같은 trap은 134로 끝납니다.

> [!IMPORTANT]
> **top-level `return`은 쓸 수 없습니다.** `main.rok`의 top-level에 `return`이 있으면 컴파일 에러입니다. 암묵 `main`에서 0이 아닌 종료 코드가 필요하면 `env::exit(code)`를 호출합니다. 반환값으로 종료 코드를 나타내려면 명시 `fn main() -> i32` 형태를 씁니다. top-level `return`을 허용하면 암묵 `main`에만 따로 규칙이 생깁니다. "어떤 타입의 `return`을 허용하는가", "top-level 블록 안쪽의 `return`은 무엇을 반환하는가" 같은 규칙입니다. 그래서 진입점 형태를 둘로 나누는 편이 더 단순합니다.

## 4.16.5. 비동기 `main`

`async fn main()`은 런타임 [스케줄러](../007-runtime/706-scheduler.md) 위의 최상위 task로 실행됩니다. 플랫폼 진입 래퍼가 런타임을 초기화한 뒤 `main`이 만든 Future를 스케줄러에 넘깁니다. 래퍼는 그 Future가 완료될 때까지 기다린 다음 4.16.4의 규칙으로 종료 코드를 정합니다. [`task::block_on`](../../refs-standard-libraries/200-core/220-future.md)과 같은 동작입니다. `main`이 끝난 시점에 아직 실행 중인 다른 task는 기다리지 않습니다.

top-level 문에 `await`가 직접 나타나면 암묵 `main`은 `async fn main() throws`가 됩니다. '직접'은 다른 `async` 블록이나 클로저 안이 아니라는 뜻입니다. 그렇지 않으면 동기 `fn main() throws`입니다.

```rok
// main.rok - 암묵 async main
let body = await http::get("https://example.com");
println(body.len());
```

## 4.16.6. 모듈은 정의 전용

top-level 문은 **진입 모듈 `main.rok`에서만** 쓸 수 있습니다. `lib.rok`을 비롯해 `main.rok`이 아닌 모든 모듈 파일(인라인 `mod { ... }` 본체 포함)은 항목만 담을 수 있습니다(`module_file ::= item*`). 그런 파일에 top-level 문이 있으면 **컴파일 에러**입니다. 인라인 `mod x { stmt }` 본체의 문에는 컴파일러가 이 절을 가리키는 전용 진단을 냅니다.

그래서 **모듈을 가져와도(`mod`, `use`) 아무 코드도 실행되지 않습니다.** Python과 일부러 다르게 정한 부분입니다. Python에서는 `import`가 모듈 파일의 top-level 코드를 실행합니다. 그래서 라이브러리로도 스크립트로도 쓰이는 파일은 `if __name__ == "__main__":` 가드로 두 용도를 구분해야 합니다. Rok에는 이런 가드가 필요 없습니다. 스크립트로 실행되는 코드는 `main.rok`에만 있고, 나머지 파일은 가져올 때 아무 일도 일으키지 않습니다.

이렇게 정한 근거는 다음과 같습니다.

- **AOT가 의미의 기준**([0.1.4](../000-meta/001-execution-modes.md#014-의미의-기준)). AOT 산출물에서 가져오는 시점에 코드를 실행하려면 링크 단계에서 모듈별 초기화 함수와 그 호출 순서를 합성해야 합니다. 정의 전용 모듈에는 이 장치가 필요 없습니다.
- **초기화 순서 의미론이 없음.** 가져오는 시점에 실행되는 모듈 코드가 있으면 "어느 모듈이 먼저 초기화되는가", "순환 import에서 무엇이 보이는가"를 정의해야 합니다. Rok에서는 이런 질문이 아예 생기지 않습니다. 전역 상태의 초기값은 컴파일 타임에 정해집니다(4.16.7).
- **캐시와 증분 빌드.** 모듈의 의미가 정의만으로 정해집니다. 그래서 내용 해시([0.3.3](../000-meta/003-source-units.md#033-패키지-식별자)) 기반 캐시와 증분 빌드는 실행 부수 효과를 고려하지 않아도 됩니다.
- **패키지 종류는 파일이 있는지로 결정**([0.3.4](../000-meta/003-source-units.md#034-매니페스트rokrokon는-선택-사항)). 실행 코드가 있는 곳은 언제나 `src/main.rok`입니다. 그래서 "이 패키지는 실행 패키지인가"와 "실행 코드는 어디에 있는가"의 답이 항상 같습니다.

## 4.16.7. 전역 상태

모듈 수준의 전역 상태는 [`static`](../005-modules/503-static.md)으로만 선언합니다. 초기값은 컴파일 타임에 평가할 수 있는 `const` 식이어야 합니다([5.3.9](../005-modules/503-static.md#539-초기값과-const)). 실행 중 처음 접근할 때 계산하는 지연 초기화(lazy initialization)는 언어 기능이 아닙니다. 표준 라이브러리 래퍼가 맡습니다. 그래서 프로그램 실행은 언제나 진입점의 첫 문장에서 시작하고, 그보다 먼저 사용자 코드가 실행되는 일은 없습니다.

## 4.16.8. 단일 파일 실행

패키지에 속하지 않은 `.rok` 파일 하나도 바로 실행할 수 있습니다.

```
rokc run hello.rok
rokc build hello.rok
```

인자로 받은 파일이 패키지의 `src/` 안에 있지 않으면 `rokc`는 그 파일 하나로 이루어진 **익명 실행 패키지**를 만듭니다. 그리고 그 파일을 이 패키지의 `main.rok`으로 취급합니다. 파일 이름에는 의미가 없습니다. 진입점 규칙(4.16.1~4.16.5)은 `main.rok`과 똑같이 적용합니다. 패키지의 `src/` 옆에 있는 `.rok` 파일(패키지 루트의 스크립트 등)도 이렇게 실행합니다. 익명 패키지에도 파일 서브모듈이 있을 수 있습니다. `mod foo;`는 패키지의 `src/`에서와 똑같이([5.4.4](../005-modules/504-modules.md#544-패키지-루트)) 파일 옆의 `foo.rok`을 읽습니다(그 서브모듈은 `foo/`에 둡니다). 파일이 없는 소스(표준 입력, REPL 세션)에만 파일 서브모듈이 없고, 인라인 모듈을 씁니다. 의존성이 필요하면 그 파일과 서브모듈 파일 머리의 인라인 매니페스트 블록으로 선언합니다([9.10.7](../009-tools/910-rok-frontend.md#9107-단일-파일-스크립트의-인라인-매니페스트)). 명령의 자세한 형태는 [`rokc` CLI](../009-tools/901-rokc-cli.md)에서 정의합니다.

## 4.16.9. 진입점 속성은 없음

Rok에는 진입점을 지정하는 속성(예: `#[main]`)이 **없습니다**. 진입 모듈은 루트 파일 `src/main.rok`이 이미 정합니다. 진입 함수는 이름 `main`(또는 top-level 문)이 정합니다. 그래서 같은 정보를 나타내는 두 번째 수단을 두지 않습니다. 진입점을 정하는 메커니즘이 하나뿐이면 "어느 함수가 실행되는가"를 파일 구조만 보고 알 수 있습니다. `main`은 예약어가 아니며, `#[main]`이라는 이름도 언어가 예약하지 않습니다.

## 4.16.10. JIT / REPL과 관계

REPL의 입력과 Jupyter의 셀([JIT 셀](../009-tools/907-jit-cells.md))은 **암묵 `main.rok`의 끝에 top-level 문과 항목을 차례로 덧붙이는 것**과 의미가 같습니다. 셀의 `let`은 암묵 `main`의 지역 변수이므로 이후 셀에서 보입니다. 셀의 `fn` / `struct`는 모듈 항목이 됩니다. 4.16.2의 규칙도 그대로 적용됩니다. 그래서 셀에서 선언한 `fn`의 본체는 이전 셀의 top-level `let`을 참조할 수 없습니다(클로저 캡처는 됩니다).

이 등가성 덕분에 [0.1.1](../000-meta/001-execution-modes.md#의미론은-같습니다)의 "JIT와 AOT는 같은 의미론을 공유한다"는 원칙이 진입점에서도 성립합니다. 노트북의 셀을 순서대로 이어 붙인 파일은 같은 결과를 내는 `main.rok`이 됩니다. 다만 다음 세 가지는 대화형 세션에만 있는 **세션 조작**입니다. 언어 의미론을 확장한 것이 아닙니다.

- 셀 최상위에서 같은 이름을 `let`으로 다시 선언([9.7.3](../009-tools/907-jit-cells.md#같은-이름의-재선언)). 새 바인딩을 만듭니다.
- 셀 삭제와 재실행([9.7.4](../009-tools/907-jit-cells.md#974-undo--셀-재실행)).
- 종결자 없는 마지막 식의 값 자동 표시([9.7.7](../009-tools/907-jit-cells.md#977-시각화--출력)). 파일에서는 이 식에 `;`를 붙인 것과 같습니다.
