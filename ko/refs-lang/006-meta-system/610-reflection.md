# 6.10. 리플렉션

표준 라이브러리에는 두 단계의 리플렉션 트레잇 `Reflect`와 `Inspect`가 있습니다.

```
Reflect    자동 구현 - 모든 타입이 공짜로 들고 있는 얕은 메타정보
Inspect    opt-in via #[derive(Inspect)] - 풍부한 필드/메서드 명세
```

두 단계로 나누면 바이너리 크기를 통제할 수 있습니다. 그러면서도 `any` 슬롯, 합 타입, REPL 자동 출력에 꼭 필요한 정보는 모든 타입에 비용 없이 들어갑니다.

## 6.10.1. `Reflect`: 자동 구현 트레잇

`Reflect`는 모든 타입에 자동으로 붙는 최소한의 타입 내부 조회(introspection)입니다. 사용자가 `#[derive(Reflect)]`를 쓰지 않아도 컴파일러가 모든 타입의 구현을 자동으로 합성합니다.

```rok
trait Reflect {
    // 정적 - 타입에 직접 호출
    fn type_name() -> str;
    fn size() -> usize;
    fn align() -> usize;

    // 인스턴스 - any 슬롯 / 합 타입에서 런타임 dispatch
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
v.type_name_of()                  // "i32" 또는 "f64" - 합 타입 variant 검사
```

자동으로 구현되므로 사용자 코드는 필요 없습니다. 컴파일러는 모든 타입의 `type_name`, `size`, `align`을 이미 알고 있으므로 비용 없이 합성합니다.

`Reflect`는 컴파일러가 모든 타입에 붙입니다. 소스에는 구현이 없습니다.

- 손으로 쓴 `impl Reflect for T`는 에러입니다. `T: Reflect` 바운드는 항상 성립합니다.
- 타입에 `size()` / `type_name()`처럼 이름이 같은 메서드가 있으면 그 메서드가 리플렉션 메서드를 가립니다.
- 이름(`type_name`)에는 모듈 경로가 붙지 않습니다(`List<i64>`, `Config`). 합 타입의 정적 이름은 정규 순서([2.6.4](../002-types/206-sum-types.md#264-집합-의미론))로 씁니다(`f64 | i32`, `none`은 마지막). `type_name_of(self)` 같은 인스턴스 형태는 런타임에 실제로 들어 있는 것을 보고합니다. 합 타입이면 그 항을, `any`면 그 내용을 보고합니다.
- `rok::core::reflect::cast<T>(value: any) -> T | none`은 `set_field`와 같은 규칙으로 `any`에서 `T`를 꺼냅니다. `rok::core::reflect::methods_of<T>() -> List<MethodInfo>`는 `T`의 고유 메서드 목록입니다.

## 6.10.2. `Inspect`: opt-in 트레잇

`Inspect`는 `#[derive(Inspect)]`로 직접 켜는(opt-in) 상세한 리플렉션입니다. 필드 이름, 동적 필드 읽기와 수정, 메서드 설명까지 드러냅니다.

```rok
trait Inspect: Reflect {
    fn field_names() -> List<str>;
    fn get_field(self, name: str) -> any | none;            // 비공개 필드도
    fn set_field(ref self, name: str, value: any) -> bool;  // 성공 여부
    fn methods() -> List<MethodInfo>;
}

struct MethodInfo {
    name: str;
    arity: usize;              // self 를 뺀 인자 수
    is_static: bool;           // self 를 받지 않는 메서드
    arg_types: List<str>;      // self 를 뺀 인자 타입 이름
    return_type: str;
}
```

- `get_field`는 그 필드의 값을 `any`로 돌려줍니다. 없는 이름이면 `none`입니다. 비공개 필드도 꺼낼 수 있습니다.
- `set_field`는 두 경우에 대입하고 `true`를 돌려줍니다. 값의 타입이 필드의 타입과 **정확히 같을 때**, 그리고 정수가 다른 정수 타입이더라도 **손실 없이 들어갈 때**(`u.set_field("id", 42)`를 `u64` 필드에)입니다. 그 밖에는 대입하지 않고 `false`를 돌려줍니다. 그 외의 변환(예: `i32` → `f64`)은 없습니다.
- `methods()`는 그 타입의 **고유 메서드만**(트레잇 메서드 제외) 이름순으로 돌려줍니다. 제네릭 `impl`의 메서드는 타입을 쓰인 그대로(`T`) 보여 줍니다.
- `enum`에 derive로 구현하면 필드는 없고(`field_names()`는 빈 목록, `get_field`는 `none`, `set_field`는 `false`), `methods()`만 의미가 있습니다.

사용 예는 다음과 같습니다.

```rok
#[derive(Inspect)]
struct Config { host: str; port: i32; }

let c = Config { host: "localhost", port: 8080 };
Config::field_names()              // ["host", "port"]
c.get_field("port")                // any 안의 8080
c.set_field("port", 9090)          // true (성공)
c.set_field("xxx", 1)              // false (없는 필드)
```

`Inspect`는 메타데이터를 바이너리에 저장해야 하므로 직접 켜야 합니다. 모든 타입에 자동으로 붙이면 바이너리 크기 부담이 너무 커집니다.

`#[derive(Inspect)]`는 `rok::core::reflect`에 있는 [메타 블록](608-meta-blocks.md) `#[impl_derive(Inspect)]`로 구현합니다. 메타 시스템으로 표준 라이브러리 트레잇을 구현하는 예입니다([stdlib 2.12.3](../../refs-standard-libraries/200-core/212-reflect.md#2123-inspect-opt-in-필드--메서드)).

## 6.10.3. REPL / Jupyter 자동 출력

REPL은 마지막 식의 결과를 출력할 때 `Reflect.type_name_of`와 `Debug`를 함께 써서 보여 줍니다([9.7.7](../009-tools/907-jit-cells.md#977-시각화--출력)).

```
> let x = 42i32;
> x
i32: 42

> [1, 2, 3]
List<i32>: [1, 2, 3]

> Config { host: "h", port: 80 }
Config: Config { host: "h", port: 80 }
```

출력 형식은 다음과 같습니다.

```
{type_name_of(value)}: {value:?}
```

`Debug`가 없는 타입이면 타입 이름만 출력합니다. REPL과 Jupyter의 셀 세션([9.7](../009-tools/907-jit-cells.md))이 이 방식으로 출력합니다.

## 6.10.4. 전역 `typeof` / `sizeof` 함수 없음

다른 언어는 `typeof`나 `sizeof` 같은 키워드로 내부 조회 기능을 드러냅니다. Rok에서는 트레잇 메서드로만 쓸 수 있습니다.

```rok
// rok - 메서드 호출
T::size()
x.type_name_of()

// 다른 언어 - 키워드
sizeof(T)        // C / C++
typeof(x)        // JS / TS
```

이렇게 하면 언어 구문을 하나도 늘리지 않아도 됩니다. 내부 조회는 모두 `Reflect` 트레잇 하나로 해결됩니다.

## 6.10.5. `any` 슬롯과 합 타입의 디스패치

`any` 슬롯과 `T1 | T2` 형태의 합 타입은 실제 타입이 런타임에 정해지므로 `type_name_of` 호출이 중요합니다.

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

컴파일러는 타입 검사 패턴(`v: T => ...`)을 `Reflect`의 태그 검사로 풉니다.

## 6.10.6. 단계 간 관계

`Inspect: Reflect`라는 표기대로 `Inspect`의 상위 트레잇은 `Reflect`입니다. 따라서 `Inspect`를 구현한 타입은 항상 `Reflect`도 구현합니다. 상위 트레잇은 어차피 모든 타입에 자동으로 구현되므로 당연한 결과입니다.

```
모든 타입            Reflect          (자동, 비용 0)
+ #[derive(Inspect)] Inspect+Reflect  (opt-in, 메타데이터 박힘)
```
