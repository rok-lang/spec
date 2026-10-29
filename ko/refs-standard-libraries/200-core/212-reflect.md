# 2.12. reflect - `Reflect` / `Inspect`

`rok::core::reflect` 모듈은 런타임 타입 정보와 **필드 introspection** 의 권위 시그니처를 제공합니다. 자세한 의미는 [refs-lang 6.10 reflection](../../refs-lang/006-meta-system/610-reflection.md) 이 권위를 가지며, 이 장에서는 표준 라이브러리 측 트레잇 형태만 정리합니다.

## 2.12.1. 두 단계 트레잇

reflection 은 두 단계 트레잇으로 분리됩니다.

| 트레잇    | 자동 구현 / opt-in            | 제공 항목                         | 사용 자리                                        |
|-----------|-------------------------------|-----------------------------------|--------------------------------------------------|
| `Reflect` | **자동** (모든 타입)          | type_name / size / align          | 묵시 `rok::**` 으로 사용 가능 (드물게 사용)      |
| `Inspect` | opt-in (`#[derive(Inspect)]`) | field_names / fields / variant_name / get_field / methods | 묵시 `rok::**` 으로 trait 사용 가능, derive 명시 |

자동과 opt-in 으로 분리한 이유는 바이너리 크기를 통제하기 위함입니다. 모든 struct 가 field reflection 메타데이터를 포함하면 binary 가 커지므로, opt-in 방식을 통해 필요한 타입에만 메타데이터를 생성하도록 하였습니다.

## 2.12.2. `Reflect` - 정적 + 인스턴스 모두

```rok
trait Reflect {
    fn type_name() -> str;        // 정적: T::type_name()
    fn size() -> usize;
    fn align() -> usize;

    fn type_name_of(self) -> str; // 인스턴스: x.type_name_of() (any / sum 도 OK)
    fn size_of(self) -> usize;
}
```

이 트레잇은 모든 타입에 자동으로 구현되며, 컴파일러가 type info 를 binary 에 자동으로 부착합니다.

`Reflect` 는 컴파일러가 모든 타입에 제공하며, 소스에 impl 이 없습니다.

- 손으로 쓴 `impl Reflect for T` 는 에러입니다. `T: Reflect` 바운드는 늘 성립합니다.
- 타입이 자기 `size()` / `type_name()` 같은 같은 이름의 메서드를 가지면 그 메서드가 리플렉션 메서드를 가립니다.
- 이름 (`type_name`) 에는 모듈 경로가 붙지 않습니다 (`List<i64>`, `Config`). 합 타입의 정적 이름은 정규 순서 ([2.6.4](../../refs-lang/002-types/206-sum-types.md#264-집합-의미론)) 로 적습니다 (`f64 | i32`, `none` 은 마지막). `type_name_of(self)` 같은 인스턴스 형태는 런타임에 들어 있는 항 (합) 이나 `any` 의 내용을 보고합니다.
- `rok::core::reflect::cast<T>(value: any) -> T | none` 은 `set_field` 와 같은 규칙으로 `any` 를 `T` 로 꺼냅니다. `rok::core::reflect::methods_of<T>() -> List<MethodInfo>` 는 `T` 의 고유 메서드 목록입니다.

### 정적 vs 인스턴스 차이

```rok
i32::type_name();          //→ "i32"           (정적, 컴파일 타임 결정)
i32::size();               //→ 4
i32::align();              //→ 4

let x: i32 = 42;
x.type_name_of();          //→ "i32"           (인스턴스, 보통 정적과 같음)

let y: i32 | f64 = 42i32;
y.type_name_of();          //→ "i32"           (런타임 dispatch, sum tag 검사)

let z: any = "hello";
z.type_name_of();          //→ "str"           (런타임, any slot dispatch)
```

`type_name_of` 는 런타임 dispatch 이므로, sum type 이나 any 의 경우 현재 들어 있는 변형의 이름을 반환합니다.

### 글로벌 함수 없음

```rok
typeof(x);                 // ERROR: 글로벌 typeof 없음
sizeof(T);                 // ERROR: 글로벌 sizeof 없음
```

대신 메서드로 일관되게 통일됩니다.

```rok
x.type_name_of()           // 인스턴스
T::type_name()             // 정적
T::size()
T::align()
```

이는 [refs-lang 3.5 멤버 접근과 메서드 호출](../../refs-lang/003-expressions/305-member-access-and-method-call.md) 의 일부로, 모든 introspection 이 메서드 형태로 통합됩니다.

## 2.12.3. `Inspect` - opt-in 필드 / 메서드

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

### derive 예

```rok
#[derive(Inspect)]
struct User {
    id: u64;
    name: str;
    email: str;
}

let u = User { id: 1, name: "Alice", email: "a@b" };

User::field_names();       //→ ["id", "name", "email"]
u.get_field("name");       //→ any("Alice")
u.set_field("id", 42);     //→ true
u.id;                      //→ 42
User::fields()[0].type_name; //→ "u64"

#[derive(Inspect)]
enum Shape { Circle(f64), Rect { w: f64; h: f64; } }

let s = Shape::Rect { w: 2.0, h: 3.0 };
Shape::variant_names();    //→ ["Circle", "Rect"]
s.variant_name();          //→ "Rect"
s.field_names_of();        //→ ["w", "h"]
s.get_field("w");          //→ any(2.0)
```

### 메타 블록 활용

`#[derive(Inspect)]` 의 구현은 `rok::core::reflect` 의 메타 블록입니다 ([refs-lang 6.8](../../refs-lang/006-meta-system/608-meta-blocks.md), [6.9](../../refs-lang/006-meta-system/609-template-tokens.md)). 요약하면 다음과 같습니다.

```rok
#[impl_derive(Inspect)]
pub @derive {
    impl rok::core::reflect::Inspect for @Self {
        fn field_names() -> List<str> {
            let names: List<str> = List::new();
            @if_struct {
                @for_fields { names.push(@field.name.as_str()); }
            }
            return names;
        }

        fn get_field(self, name: str) -> any | none {
            @if_struct {
                @for_fields {
                    if name == @field.name.as_str() { let held: any = self.@field; return held; }
                }
            }
            @if_enum {
                match self {
                    @for_variants {
                        @variant.pattern => {
                            @for_fields {
                                if name == @field.name.as_str() { let held: any = @field; return held; }
                            }
                        },
                    }
                }
            }
            return none;
        }

        fn set_field(ref self, name: str, value: any) -> bool {
            @if_struct {
                @for_fields {
                    if name == @field.name.as_str() {
                        match rok::core::reflect::cast::<@field.type>(value) {
                            v: @field.type => { self.@field = v; return true; },
                            none => { return false; },
                        }
                    }
                }
            }
            return false;
        }

        fn methods() -> List<rok::core::reflect::MethodInfo> {
            return rok::core::reflect::methods_of::<@Self>();
        }

        @if_enum {
            fn variant_name(self) -> str {
                return match self {
                    @for_variants { @variant.pattern => @variant.name.as_str(), }
                };
            }
            // variant_names() and field_names_of(self) likewise, per variant
        }
    }
}
```

자세한 메타 블록 내용은 [refs-lang 6.8 메타 블록](../../refs-lang/006-meta-system/608-meta-blocks.md) 을 참고합니다.

### 동적 dispatch 가 필요한 자리

`Inspect` 는 config 객체, serializer, debugger 등 필드 이름으로 접근이 필요한 도메인에서 사용됩니다. 일반 코드에서는 `Reflect` 만으로 충분합니다.

```rok
fn dump_struct<T: Inspect>(x: T) {
    for f in T::fields() {
        if let v: any = x.get_field(f.name) {
            println(f"{f.name}: {f.type_name} ({v.type_name_of()})");
        }
    }
}
```

## 2.12.4. REPL 출력

REPL 은 평범한 마지막 식을 `Display` 로 (없으면 `Debug` 로), 뒤에 `?` 를 붙이면 `Debug` 로, 그 트레이트가 없는 값은 이 장의 `Reflect::type_name_of` 로 출력합니다 ([2.10 fmt](210-fmt.md), [refs-lang 6.10.3](../../refs-lang/006-meta-system/610-reflection.md#6103-repl--jupyter-자동-출력)).

```
>>> let x = 42i32;
>>> x
42

>>> [1, 2, 3]
[1, 2, 3]

>>> [1, 2, 3]?
[1, 2, 3]

>>> User { id: 1, name: "Alice" }?
User { id: 1, name: "Alice" }
```

평범한 식은 `Display` 를, 없으면 `Debug` 를 쓰고, 뒤에 `?` 가 붙으면 `Debug` 를 씁니다. 둘 다 없는 값(뒤에 `?` 는 `Debug` 가 없는 값)은 타입 이름만 출력하며, `any` 값은 담은 타입의 이름을 출력합니다.

## 2.12.5. 모듈 매핑

| 항목             | 위치                                             | 메커니즘                      |
|------------------|--------------------------------------------------|-------------------------------|
| `Reflect` 자동   | 컴파일러 + `rok::core::reflect`                   | type info table, binary 부착  |
| `Inspect` derive | `rok::core::reflect` + 메타 블록 `#[impl_derive]` | template token 기반 코드 생성 |
| `MethodInfo`     | `rok::core::reflect`                              | 단순 struct                   |
| `FieldInfo`, `fields_of` | `rok::core::reflect`                      | 컴파일러 필드 테이블 (인스턴스화된 타입) |
| `any` 타입       | 컴파일러 내장                                     | type tag + payload            |

## 2.12.6. 다른 언어와의 비교

| 측면               | Rust                         | Java             | Python              | Rok                                         |
|--------------------|------------------------------|------------------|---------------------|---------------------------------------------|
| 타입 이름          | `std::any::type_name::<T>()` | `getClass()`     | `type(x).__name__`  | `x.type_name_of()` 메서드                   |
| 필드 introspection | **없음** (lib `bincode` 등)  | Reflection API   | `vars(x)` `getattr` | `x.get_field(name)` (opt-in)                |
| 런타임 dispatch    | `dyn Trait` (vtable)         | 자동             | 자동                | sum type tag + `any` / `any Trait` (opt-in) |
| 자동 구현          | **없음**                     | 모든 객체        | 모든 객체           | `Reflect` 자동, `Inspect` opt-in            |
| 메모리 비용        | 0                            | 모든 클래스 메타 | 모든 객체 dict      | `Reflect` 만 자동 (opt-in 분리)             |

Rok 의 가장 큰 차이점은 두 가지입니다. 첫째, 두 단계로 분리되어 있어 가벼운 `Reflect` 는 자동으로 제공되고 무거운 `Inspect` 는 opt-in 으로 제공됩니다. 이는 Java 나 Python 의 전부 자동 모델보다 메모리 효율이 좋고, Rust 의 전부 수동 모델보다 ergonomic 합니다. 둘째, 메서드 일관성을 유지하여 `typeof(x)` 같은 글로벌 함수가 **없으며**, 모든 introspection 이 메서드 형태로 제공됩니다.

자세한 reflection 의미는 [refs-lang 6.10](../../refs-lang/006-meta-system/610-reflection.md) 이 권위입니다.
