# 6.10. 리플렉션

표준 라이브러리는 두 단계의 리플렉션 트레잇을 제공합니다. 바로 `Reflect` 와 `Inspect` 입니다.

```
Reflect    자동 구현: 모든 타입이 공짜로 들고 있는 얕은 메타정보
Inspect    opt-in via #[derive(Inspect)]: 풍부한 필드/메서드 명세
```

이렇게 두 단계로 나눈 이유는 binary 크기를 컨트롤하면서도 `any` 슬롯이나 합 타입, REPL 자동 출력처럼 꼭 필요한 기능은 모든 타입에 공짜로 제공하고 싶었기 때문입니다.

## 6.10.1. `Reflect` - 자동 구현 트레잇

`Reflect` 는 모든 타입에 자동으로 부착되는 minimal 한 type introspection 입니다. 사용자가 `#[derive(Reflect)]` 를 따로 적지 않아도, 컴파일러가 모든 타입에 대해 자동으로 impl 을 합성합니다.

```rok
trait Reflect {
    // 정적: 타입에 직접 호출
    fn type_name() -> str;
    fn size() -> usize;
    fn align() -> usize;

    // 인스턴스: any 슬롯 / 합 타입에서 런타임 dispatch
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
v.type_name_of()                  // "i32" 또는 "f64" (합 타입 variant 검사)
```

자동 구현이라 사용자 코드를 따로 작성할 필요는 없습니다. 컴파일러가 모든 타입의 `type_name`, `size`, `align` 정보를 이미 알고 있어서, 비용 0 으로 합성합니다.

`Reflect` 는 컴파일러가 모든 타입에 제공하며, 소스에 impl 이 없습니다.

- 손으로 쓴 `impl Reflect for T` 는 에러입니다. `T: Reflect` 바운드는 늘 성립합니다.
- 타입이 자기 `size()` / `type_name()` 같은 같은 이름의 메서드를 가지면 그 메서드가 리플렉션 메서드를 가립니다.
- 이름 (`type_name`) 에는 모듈 경로가 붙지 않습니다 (`List<i64>`, `Config`). 합 타입의 정적 이름은 정규 순서 ([2.6.4](../002-types/206-sum-types.md#264-집합-의미론)) 로 적습니다 (`f64 | i32`, `none` 은 마지막). `type_name_of(self)` 같은 인스턴스 형태는 런타임에 들어 있는 항 (합) 이나 `any` 의 내용을 보고합니다.
- `rok::core::reflect::cast<T>(value: any) -> T | none` 은 `set_field` 와 같은 규칙으로 `any` 를 `T` 로 꺼냅니다. `rok::core::reflect::methods_of<T>() -> List<MethodInfo>` 는 `T` 의 고유 메서드 목록입니다.

## 6.10.2. `Inspect` - opt-in 트레잇

`Inspect` 는 `#[derive(Inspect)]` 로 opt-in 하는 풍부한 리플렉션입니다. 필드 이름, 동적 필드 접근과 수정, 그리고 메서드 명세까지 노출합니다.

```rok
trait Inspect: Reflect {
    fn field_names() -> List<str>;
    fn get_field(self, name: str) -> any | none;            // 비공개 필드도
    fn set_field(ref self, name: str, value: any) -> bool;  // 성공 여부
    fn methods() -> List<MethodInfo>;

    // 기본 본문이 있으며, enum 에서는 derive 가 덮어씁니다.
    fn fields() -> List<FieldInfo>;          // fields_of::<Self>()
    fn variant_names() -> List<str>;         // struct 는 빈 목록
    fn variant_name(self) -> str;            // struct: self.type_name_of()
    fn field_names_of(self) -> List<str>;    // self 가 지닌 필드
}

struct FieldInfo {
    name: str;                 // 튜플 필드는 "0", "1", ..
    type_name: str;            // 인스턴스화된 타입: Box<f64> 의 `v: T` 는 f64
    is_pub: bool;              // 그냥 `pub` 인 필드만 true
}

struct MethodInfo {
    name: str;
    arity: usize;              // self 를 뺀 인자 수
    is_static: bool;           // self 를 받지 않는 메서드
    arg_types: List<str>;      // self 를 뺀 인자 타입 이름
    return_type: str;
}
```

- `get_field` 는 그 필드의 값을 `any` 로 돌려주며, 없는 이름이면 `none` 입니다. 비공개 필드도 꺼낼 수 있습니다.
- `set_field` 는 값이 그 필드의 타입과 **정확히 같을 때**, 또는 정수가 다른 정수 타입이어도 **손실 없이 들어갈 때** (`u.set_field("id", 42)` 가 `u64` 필드에) 대입하고 `true` 를 돌려줍니다. 그 밖에는 대입하지 않고 `false` 입니다. 다른 변환 (예: `i32` → `f64`) 은 없습니다.
- `methods()` 는 그 타입의 **고유 메서드만** (트레잇 메서드 제외) 이름순으로 돌려줍니다. 제네릭 `impl` 의 메서드는 타입을 적힌 그대로 (`T`) 보여 줍니다.
- `fields()` 는 모든 필드를 선언 순서대로, 타입 이름과 `pub` 여부와 함께 돌려줍니다. **비공개 필드도 나옵니다**: `get_field` 와 마찬가지로 `Inspect` 를 derive 하는 것이 타입 작성자의 동의입니다. 타입 이름은 인스턴스화된 것이며 (`Box<f64>` 는 `v: f64`, `items: List<f64>`), `type_name` 처럼 모듈 경로가 없습니다. `rok::core::reflect::fields_of<T: Inspect>() -> List<FieldInfo>` 는 같은 목록을 돌려주는 함수입니다.
- `enum` 은 타입 수준의 필드가 없습니다 (`field_names()` 와 `fields()` 는 빈 목록). `variant_names()` 는 variant 를 선언 순서대로 돌려줍니다. 값에서 `variant_name()` 은 지금 지닌 variant (`"Circle"`), `field_names_of()` 는 그 variant 의 필드 (튜플 variant 는 `"0"`, `"1"`, 이름 있는 variant 는 그 이름) 이며, `get_field` 로 읽을 수 있습니다. variant 의 payload 는 `Inspect` 로는 읽기 전용입니다: enum 의 `set_field` 는 `false` 입니다.
- struct 에서 `variant_names()` 는 빈 목록, `variant_name()` 은 그 타입 이름 (`type_name_of()`), `field_names_of()` 는 그 필드입니다.
- `fields()`, `variant_names()`, `variant_name()`, `field_names_of()` 는 기본 본문이 있으므로, 손으로 쓰는 `impl Inspect` 는 앞의 네 메서드만 쓰면 됩니다.
- derive 하지 않은 타입에서 `Inspect` 메서드를 부르면 오류이며, 도움말이 무엇을 쓸지 알려 줍니다: ``add `#[derive(Inspect)]` to `Point` ``.

사용 예시는 다음과 같습니다.

```rok
#[derive(Inspect)]
struct Config { host: str; port: i32; }

let c = Config { host: "localhost", port: 8080 };
Config::field_names()              // ["host", "port"]
c.get_field("port")                // any 안의 8080
c.set_field("port", 9090)          // true (성공)
c.set_field("xxx", 1)              // false (없는 필드)
Config::fields()[1].type_name      // "i32"
```

`Inspect` 는 메타데이터를 binary 에 저장해야 하기 때문에 opt-in 으로 두었습니다. 모든 타입에 자동으로 부착하면 binary 크기 부담이 너무 커지기 때문입니다.

`#[derive(Inspect)]` 의 구현은 `rok::core::reflect` 에 있는 [메타 블록](608-meta-blocks.md) `#[impl_derive(Inspect)]` 이며, 메타 시스템 자체로 표준 라이브러리 트레잇을 구현하는 패턴입니다 ([stdlib 2.12.3](../../refs-standard-libraries/200-core/212-reflect.md#2123-inspect---opt-in-필드--메서드)).

## 6.10.3. REPL / Jupyter 자동 출력

REPL 은 평범한 마지막 식을 `Display` 로 (없으면 `Debug` 로), 뒤에 `?` 를 붙이면 `Debug` 로, `:type` 은 식을 실행하지 않고 타입 검사로 얻은 정적 타입을 출력합니다 ([9.7.7](../009-tools/907-jit-cells.md#977-시각화--출력)).

```
>>> let x = 42i32;
>>> x
42

>>> [1, 2, 3]
[1, 2, 3]

>>> Config { host: "h", port: 80 }
Config { host: "h", port: 80 }

>>> Opaque { id: 1 }
Opaque
```

평범한 식은 `Display` 를, 없으면 `Debug` 를 쓰고, 뒤에 `?` 는 `Debug` 를 씁니다. 둘 다 없는 값 (뒤에 `?` 를 붙이면 `Debug` 가 없는 값) 은 타입 이름만 출력합니다. 이 동작은 REPL 과 Jupyter 의 셀 세션 ([9.7](../009-tools/907-jit-cells.md)) 이 사용합니다.

## 6.10.4. 글로벌 `typeof` / `sizeof` 함수 없음

다른 언어들이 `typeof` 나 `sizeof` 같은 키워드로 노출하는 introspection 을, Rok 에서는 트레잇 메서드로만 제공합니다.

```rok
// rok: 메서드 호출
T::size()
x.type_name_of()

// 다른 언어: 키워드
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

타입 검사 패턴 (`v: T => ...`) 은 컴파일러가 `Reflect` 의 tag 검사로 desugar 합니다.

## 6.10.6. 단계 간 관계

`Inspect: Reflect` 라는 표기에서 알 수 있듯이, `Inspect` 는 `Reflect` 를 super-trait 으로 가집니다. 따라서 `Inspect` 가 구현된 타입은 항상 `Reflect` 도 구현하게 되며, 어차피 자동으로 구현되니 당연한 이야기입니다.

```
모든 타입            Reflect          (자동, 비용 0)
+ #[derive(Inspect)] Inspect+Reflect  (opt-in, 메타데이터 박힘)
```
