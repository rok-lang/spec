# 6.10. 리플렉션

표준 라이브러리는 두 단계의 리플렉션 트레잇을 제공합니다. 바로 `Reflect` 와 `Inspect` 입니다.

```
Reflect    자동 구현 — 모든 타입이 공짜로 들고 있는 얕은 메타정보
Inspect    opt-in via #[derive(Inspect)] — 풍부한 필드/메서드 명세
```

이렇게 두 단계로 나눈 이유는 binary 크기를 컨트롤하면서도 `any` 슬롯이나 합 타입, REPL 자동 출력처럼 꼭 필요한 기능은 모든 타입에 공짜로 제공하고 싶었기 때문입니다.

## 6.10.1. `Reflect` — 자동 구현 트레잇

`Reflect` 는 모든 타입에 자동으로 부착되는 minimal 한 type introspection 입니다. 사용자가 `#[derive(Reflect)]` 를 따로 적지 않아도, 컴파일러가 모든 타입에 대해 자동으로 impl 을 합성해 줍니다.

```rok
trait Reflect {
    // 정적 — 타입에 직접 호출
    fn type_name() -> str;
    fn size() -> usize;
    fn align() -> usize;

    // 인스턴스 — any 슬롯 / 합 타입에서 런타임 dispatch
    fn type_name_of(self) -> str;
    fn size_of(self) -> usize;
}
```

```rok
i32::type_name() == "i32"
i32::size()      == 4
i32::align()     == 4

let x: any = 42i32;
x.type_name_of() == "i32"        // 런타임 type tag 검사

let v: i32 | f64 = pick();
v.type_name_of()                  // "i32" 또는 "f64" — 합 타입 variant 검사
```

자동 구현이라 사용자 코드를 따로 작성할 필요는 없습니다. 컴파일러가 모든 타입의 `type_name`, `size`, `align` 정보를 이미 알고 있어서, 비용 0 으로 합성해 줍니다.

## 6.10.2. `Inspect` — opt-in 트레잇

`Inspect` 는 `#[derive(Inspect)]` 로 opt-in 하는 풍부한 리플렉션입니다. 필드 이름, 동적 필드 접근과 수정, 그리고 메서드 명세까지 노출해 줍니다.

```rok
trait Inspect: Reflect {
    fn field_names() -> List<str>;
    fn get_field(self, name: str) -> Any | none;
    fn set_field(ref self, name: str, value: Any) -> bool;
    fn methods() -> List<MethodInfo>;
}

struct MethodInfo {
    name: str;
    arity: usize;
    is_static: bool;
}
```

사용 예시는 다음과 같습니다.

```rok
#[derive(Inspect)]
struct Config { host: str; port: i32; }

let c = Config { host: "localhost", port: 8080 };
Config::field_names()              // ["host", "port"]
c.get_field("port")                // Any : 8080
c.set_field("port", 9090)          // true (성공)
c.set_field("xxx", 1)              // false (없는 필드)
```

`Inspect` 는 메타데이터를 binary 에 저장해야 하기 때문에 opt-in 으로 두었습니다. 모든 타입에 자동으로 부착하면 binary 크기 부담이 너무 커지기 때문입니다.

`Inspect` 의 자동 구현은 [메타 블록](608-meta-blocks.md) 의 `#[impl_derive(Inspect)]` 로 작성되며, 메타 시스템 자체로 표준 라이브러리 트레잇을 구현하는 패턴입니다.

## 6.10.3. REPL / Jupyter 자동 출력

REPL 에서 마지막 식 결과를 출력할 때는 `Reflect.type_name_of` 와 `Display.display` 를 조합해서 보여 줍니다.

```
> let x = 42i32;
> x
i32: 42

> [1, 2, 3]
List<i64>: [1, 2, 3]

> Config { host: "h", port: 80 }
Config: Config { host: "h", port: 80 }
```

출력 포맷은 다음과 같습니다.

```
{type_name_of(value)}: {display(value)}
```

이 동작은 [JIT 엔진](../009-tools/906-jit-engine.md) 의 셀 평가 루프가 사용합니다. 마지막 식의 값을 `Reflect` 와 `Display` 두 트레잇으로 가공해서 표면에 출력해 주는 방식입니다.

## 6.10.4. 글로벌 `typeof` / `sizeof` 함수 없음

다른 언어들이 `typeof` 나 `sizeof` 같은 키워드로 노출하는 introspection 을, rok 에서는 트레잇 메서드로만 제공합니다.

```rok
// rok — 메서드 호출
T::size()
x.type_name_of()

// 다른 언어 — 키워드
sizeof(T)        // C / C++
typeof(x)        // JS / TS
```

이렇게 하는 이유는 언어 syntax 추가가 0 이기 때문입니다. `Reflect` 트레잇 하나만으로 모든 introspection 표면을 커버할 수 있습니다.

## 6.10.5. `any` 슬롯과 합 타입에서의 dispatch

`any` 슬롯과 `T1 | T2` 형태의 합 타입은 런타임에 실제 타입을 결정하기 때문에, `type_name_of` 호출이 매우 중요합니다.

```rok
fn handle(x: any) {
    if x.type_name_of() == "i32" { handle_int(x); }
    else if x.type_name_of() == "str" { handle_str(x); }
}

fn classify(v: i32 | f64) -> str {
    match v {
        n: i32 => "integer",
        f: f64 => "float",
    }
}
```

타입 검사 패턴 (`v: T => ...`) 은 컴파일러가 `Reflect` 의 tag 검사로 desugar 해 줍니다.

## 6.10.6. 단계 간 관계

`Inspect: Reflect` 라는 표기에서 알 수 있듯이, `Inspect` 는 `Reflect` 를 super-trait 으로 가집니다. 따라서 `Inspect` 가 구현된 타입은 항상 `Reflect` 도 구현하게 되며, 어차피 자동으로 구현되니 당연한 이야기입니다.

```
모든 타입            Reflect          (자동, 비용 0)
+ #[derive(Inspect)] Inspect+Reflect  (opt-in, 메타데이터 박힘)
```
