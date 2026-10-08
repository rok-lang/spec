# 2.12. reflect — `Reflect` / `Inspect`

`rok::core::reflect` 모듈은 런타임 타입 정보와 **필드 introspection** 의 권위 시그니처를 제공합니다. 자세한 의미는 [refs-lang 6.10 reflection](../../refs-lang/006-meta-system/610-reflection.md) 이 권위를 가지며, 이 장에서는 표준 라이브러리 측 트레잇 형태만 정리합니다.

## 2.12.1. 두 단계 트레잇

reflection 은 두 단계 트레잇으로 분리되어 있습니다.

| 트레잇    | 자동 구현 / opt-in            | 제공 항목                         | 사용 자리                                        |
|-----------|-------------------------------|-----------------------------------|--------------------------------------------------|
| `Reflect` | **자동** (모든 타입)          | type_name / size / align          | 묵시 `rok::**` 으로 사용 가능 (드물게 사용)      |
| `Inspect` | opt-in (`#[derive(Inspect)]`) | field_names / get_field / methods | 묵시 `rok::**` 으로 trait 사용 가능, derive 명시 |

자동과 opt-in 으로 분리한 이유는 바이너리 크기를 통제하기 위함입니다. 모든 struct 가 field reflection 메타데이터를 포함하면 binary 가 커지므로, opt-in 방식을 통해 필요한 타입에만 메타데이터를 생성하도록 하였습니다.

## 2.12.2. `Reflect` — 정적 + 인스턴스 모두

```rok
trait Reflect {
    fn type_name() -> str;        // 정적 — T::type_name()
    fn size() -> usize;
    fn align() -> usize;

    fn type_name_of(self) -> str; // 인스턴스 — x.type_name_of() (any / sum 도 OK)
    fn size_of(self) -> usize;
}
```

이 트레잇은 모든 타입에 자동으로 구현되며, 컴파일러가 type info 를 binary 에 자동으로 부착합니다.

### 정적 vs 인스턴스 차이

```rok
i32::type_name();          //→ "i32"           (정적 — 컴파일 타임 결정)
i32::size();               //→ 4
i32::align();              //→ 4

let x: i32 = 42;
x.type_name_of();          //→ "i32"           (인스턴스 — 보통 정적과 같음)

let y: i32 | f64 = 42i32;
y.type_name_of();          //→ "i32"           (런타임 dispatch — sum tag 검사)

let z: any = "hello";
z.type_name_of();          //→ "str"           (런타임 — any slot dispatch)
```

`type_name_of` 는 런타임 dispatch 이므로, sum type 이나 any 의 경우 현재 들어 있는 변형의 이름을 반환합니다.

### 글로벌 함수 없음

```rok
typeof(x);                 // ERROR — 글로벌 typeof 없음
sizeof(T);                 // ERROR — 글로벌 sizeof 없음
```

대신 메서드로 일관되게 통일되어 있습니다.

```rok
x.type_name_of()           // 인스턴스
T::type_name()             // 정적
T::size()
T::align()
```

이는 [refs-lang 3.5 멤버 접근과 메서드 호출](../../refs-lang/003-expressions/305-member-access-and-method-call.md) 의 일부로, 모든 introspection 이 메서드 형태로 통합되어 있습니다.

## 2.12.3. `Inspect` — opt-in 필드 / 메서드

```rok
trait Inspect: Reflect {
    fn field_names() -> List<str>;
    fn get_field(self, name: str) -> any | none;
    fn set_field(ref self, name: str, value: any) -> bool;     // 성공 여부
    fn methods() -> List<MethodInfo>;
}

struct MethodInfo {
    name: str;
    arg_types: List<str>;
    return_type: str;
}
```

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
```

### 메타 블록 활용

`#[derive(Inspect)]` 본문은 메타 블록 메커니즘으로 작성됩니다.

```rok
#[impl_derive(Inspect)]
@accept(struct, enum)
@for_fields {
    // 각 필드별 코드 생성
    case @field.name => any(self.@field),
}
```

자세한 메타 블록 내용은 [refs-lang 6.8 메타 블록](../../refs-lang/006-meta-system/608-meta-blocks.md) 을 참고합니다.

### 동적 dispatch 가 필요한 자리

`Inspect` 는 config 객체, serializer, debugger 등 필드 이름으로 접근이 필요한 도메인에서 사용됩니다. 일반 코드에서는 `Reflect` 만으로 충분합니다.

```rok
fn dump_struct<T: Inspect>(x: T) {
    for name in T::field_names() {
        if let v: any = x.get_field(name) {
            println(f"{name} = {v.type_name_of()}: {v:?}");
        }
    }
}
```

## 2.12.4. REPL 출력

REPL 의 마지막 식 자동 출력은 이 장의 `Reflect::type_name_of` 와 [2.10 fmt](210-fmt.md) 의 `Display` / `Debug` 가 합성되어 작동합니다.

```
> let x = 42i32;
i32 = 42

> [1, 2, 3]
List<i64> = [1, 2, 3]

> User { id: 1, name: "Alice" }
User = User { id: 1, name: "Alice" }
```

표시 형태는 `{type_name_of} = {Display 또는 Debug}` 입니다. `Display` 가 정의되지 않은 경우에는 `Debug` 로 fallback 되며, `Debug` 도 없는 경우에는 `Reflect` 만 출력됩니다 (예: `<i32>` 처럼 type 만 표시).

## 2.12.5. 모듈 매핑

| 항목             | 위치                                             | 메커니즘                      |
|------------------|--------------------------------------------------|-------------------------------|
| `Reflect` 자동   | 컴파일러 + `rok::core::reflect`                   | type info table — binary 부착 |
| `Inspect` derive | `rok::core::reflect` + 메타 블록 `#[impl_derive]` | template token 기반 코드 생성 |
| `MethodInfo`     | `rok::core::reflect`                              | 단순 struct                   |
| `any` 타입       | 컴파일러 내장 + `rok::core::reflect::any`         | type tag + payload            |

## 2.12.6. 다른 언어와의 비교

| 측면               | Rust                         | Java             | Python              | rok                                         |
|--------------------|------------------------------|------------------|---------------------|---------------------------------------------|
| 타입 이름          | `std::any::type_name::<T>()` | `getClass()`     | `type(x).__name__`  | `x.type_name_of()` 메서드                   |
| 필드 introspection | **없음** (lib `bincode` 등)  | Reflection API   | `vars(x)` `getattr` | `x.get_field(name)` (opt-in)                |
| 런타임 dispatch    | `dyn Trait` (vtable)         | 자동             | 자동                | sum type tag + `any` / `any Trait` (opt-in) |
| 자동 구현          | **없음**                     | 모든 객체        | 모든 객체           | `Reflect` 자동, `Inspect` opt-in            |
| 메모리 비용        | 0                            | 모든 클래스 메타 | 모든 객체 dict      | `Reflect` 만 자동 (opt-in 분리)             |

rok 의 가장 큰 차이점은 두 가지입니다. 첫째, 두 단계로 분리되어 있어 가벼운 `Reflect` 는 자동으로 제공되고 무거운 `Inspect` 는 opt-in 으로 제공됩니다. 이는 Java 나 Python 의 전부 자동 모델보다 메모리 효율이 좋고, Rust 의 전부 수동 모델보다 ergonomic 합니다. 둘째, 메서드 일관성을 유지하여 `typeof(x)` 같은 글로벌 함수가 **없으며**, 모든 introspection 이 메서드 형태로 제공됩니다.

자세한 reflection 의미는 [refs-lang 6.10](../../refs-lang/006-meta-system/610-reflection.md) 이 권위입니다.
