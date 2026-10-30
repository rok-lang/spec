# 6.10. 리플렉션

표준 라이브러리에는 두 단계의 리플렉션 트레잇 `Reflect`와 `Inspect`가 있습니다.

```
Reflect    자동 구현: 모든 타입이 공짜로 들고 있는 얕은 메타정보
Inspect    opt-in via #[derive(Inspect)]: 풍부한 필드/메서드 명세
```

두 단계로 나눈 이유는 binary 크기를 통제하면서도, `any` 슬롯, 합 타입, REPL 자동 출력에 꼭 필요한 정보는 모든 타입이 비용 없이 갖게 하기 위해서입니다.

## 6.10.1. `Reflect` - 자동 구현 트레잇

`Reflect`는 모든 타입에 자동으로 붙는 최소한의 type introspection입니다. 사용자가 `#[derive(Reflect)]`를 쓰지 않아도 컴파일러가 모든 타입에 대해 impl을 자동으로 합성합니다.

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

자동으로 구현되므로 사용자 코드는 필요 없습니다. 컴파일러는 모든 타입의 `type_name`, `size`, `align`을 이미 알고 있으므로 비용 없이 합성합니다.

`Reflect`는 컴파일러가 모든 타입에 제공하며, 소스에는 impl이 없습니다.

- 손으로 쓴 `impl Reflect for T`는 에러입니다. `T: Reflect` 바운드는 항상 성립합니다.
- 타입에 `size()` / `type_name()`처럼 이름이 같은 메서드가 있으면 그 메서드가 리플렉션 메서드를 가립니다.
- 이름(`type_name`)에는 모듈 경로가 붙지 않습니다(`List<i64>`, `Config`). 합 타입의 정적 이름은 정규 순서([2.6.4](../002-types/206-sum-types.md#264-집합-의미론))로 씁니다(`f64 | i32`, `none`은 마지막). `type_name_of(self)` 같은 인스턴스 형태는 런타임에 실제로 들어 있는 항(합 타입)이나 `any`의 내용을 보고합니다.
- `rok::core::reflect::cast<T>(value: any) -> T | none`은 `set_field`와 같은 규칙으로 `any`에서 `T`를 꺼냅니다. `rok::core::reflect::methods_of<T>() -> List<MethodInfo>`는 `T`의 고유 메서드 목록입니다.

## 6.10.2. `Inspect` - opt-in 트레잇

`Inspect`는 `#[derive(Inspect)]`로 opt-in하는 상세한 리플렉션입니다. 필드 이름, 동적 필드 읽기와 수정, 메서드 명세까지 노출합니다.

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

- `get_field`는 그 필드의 값을 `any`로 돌려주며, 없는 이름이면 `none`입니다. 비공개 필드도 꺼낼 수 있습니다.
- `set_field`는 값의 타입이 필드의 타입과 **정확히 같을 때**, 또는 정수가 다른 정수 타입이더라도 **손실 없이 들어갈 때**(`u.set_field("id", 42)`를 `u64` 필드에) 대입하고 `true`를 돌려줍니다. 그 밖에는 대입하지 않고 `false`를 돌려줍니다. 그 외의 변환(예: `i32` → `f64`)은 없습니다.
- `methods()`는 그 타입의 **고유 메서드만**(트레잇 메서드 제외) 이름순으로 돌려줍니다. 제네릭 `impl`의 메서드는 타입을 쓰인 그대로(`T`) 보여 줍니다.
- `fields()`는 모든 필드를 선언 순서대로 타입 이름, `pub` 여부와 함께 돌려줍니다. **비공개 필드도 나옵니다.** `get_field`와 마찬가지로, `Inspect`를 derive하는 것이 곧 타입 작성자의 동의입니다. 타입 이름은 인스턴스화된 타입이며(`Box<f64>`는 `v: f64`, `items: List<f64>`), `type_name`처럼 모듈 경로가 없습니다. `rok::core::reflect::fields_of<T: Inspect>() -> List<FieldInfo>`는 같은 목록을 돌려주는 함수입니다.
- `enum`에는 타입 수준의 필드가 없습니다(`field_names()`와 `fields()`는 빈 목록). `variant_names()`는 variant를 선언 순서대로 돌려줍니다. 값에 대해 `variant_name()`은 지금 들고 있는 variant(`"Circle"`)를, `field_names_of()`는 그 variant의 필드(튜플 variant는 `"0"`, `"1"`, 이름 있는 variant는 필드 이름)를 돌려주며, 이 필드는 `get_field`로 읽을 수 있습니다. variant의 payload는 `Inspect`로는 읽기만 할 수 있습니다. enum의 `set_field`는 `false`를 돌려줍니다.
- struct에서 `variant_names()`는 빈 목록, `variant_name()`은 그 타입 이름(`type_name_of()`), `field_names_of()`는 그 필드입니다.
- `fields()`, `variant_names()`, `variant_name()`, `field_names_of()`에는 기본 본문이 있으므로, 손으로 쓰는 `impl Inspect`는 앞의 네 메서드만 구현하면 됩니다.
- derive하지 않은 타입에서 `Inspect` 메서드를 부르면 에러이며, 도움말이 무엇을 써야 하는지 알려 줍니다: ``add `#[derive(Inspect)]` to `Point` ``.

사용 예는 다음과 같습니다.

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

`Inspect`는 메타데이터를 binary에 저장해야 하므로 opt-in입니다. 모든 타입에 자동으로 붙이면 binary 크기 부담이 너무 커집니다.

`#[derive(Inspect)]`는 `rok::core::reflect`에 있는 [메타 블록](608-meta-blocks.md) `#[impl_derive(Inspect)]`로 구현합니다. 메타 시스템으로 표준 라이브러리 트레잇을 구현하는 예입니다([stdlib 2.12.3](../../refs-standard-libraries/200-core/212-reflect.md#2123-inspect---opt-in-필드--메서드)).

## 6.10.3. REPL / Jupyter 자동 출력

REPL은 평범한 마지막 식을 `Display`로(없으면 `Debug`로) 출력하고, 뒤에 `?`를 붙이면 `Debug`로 출력합니다. `:type`은 식을 실행하지 않고 타입 검사로 얻은 정적 타입을 출력합니다([9.7.7](../009-tools/907-jit-cells.md#977-시각화--출력)).

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

평범한 식은 `Display`를, 없으면 `Debug`를 쓰고, 뒤에 `?`를 붙이면 `Debug`를 씁니다. 둘 다 없는 값(뒤에 `?`를 붙였다면 `Debug`가 없는 값)은 타입 이름만 출력합니다. REPL과 Jupyter의 셀 세션([9.7](../009-tools/907-jit-cells.md))이 이 방식으로 출력합니다.

## 6.10.4. 글로벌 `typeof` / `sizeof` 함수 없음

다른 언어가 `typeof`나 `sizeof` 같은 키워드로 노출하는 introspection을 Rok은 트레잇 메서드로만 제공합니다.

```rok
// rok: 메서드 호출
T::size()
x.type_name_of()

// 다른 언어: 키워드
sizeof(T)        // C / C++
typeof(x)        // JS / TS
```

이렇게 하면 언어 구문을 하나도 늘리지 않아도 됩니다. introspection은 모두 `Reflect` 트레잇 하나로 해결됩니다.

## 6.10.5. `any` 슬롯과 합 타입에서의 dispatch

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

컴파일러는 타입 검사 패턴(`v: T => ...`)을 `Reflect`의 tag 검사로 desugar합니다.

## 6.10.6. 단계 간 관계

`Inspect: Reflect`라는 표기대로 `Inspect`의 super-trait은 `Reflect`입니다. 따라서 `Inspect`를 구현한 타입은 항상 `Reflect`도 구현합니다. 어차피 모든 타입에 자동으로 구현되므로 당연한 결과입니다.

```
모든 타입            Reflect          (자동, 비용 0)
+ #[derive(Inspect)] Inspect+Reflect  (opt-in, 메타데이터 박힘)
```
