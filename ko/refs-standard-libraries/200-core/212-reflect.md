# 2.12. reflect: `Reflect` / `Inspect`

`rok::core::reflect` 모듈은 런타임 타입 정보와 **필드 인트로스펙션(introspection)**의 권위 시그니처를 정의합니다. 자세한 의미는 [refs-lang 6.10 reflection](../../refs-lang/006-meta-system/610-reflection.md)이 정합니다. 이 장은 표준 라이브러리 쪽 트레잇 형태만 다룹니다.

## 2.12.1. 두 단계 트레잇

리플렉션은 트레잇 두 단계로 나뉩니다.

| 트레잇    | 자동 구현 / opt-in            | 제공 항목                         | 사용 자리                                        |
|-----------|-------------------------------|-----------------------------------|--------------------------------------------------|
| `Reflect` | **자동** (모든 타입)          | type_name / size / align          | 묵시 `rok::**`으로 쓸 수 있음(드물게 씀)        |
| `Inspect` | opt-in (`#[derive(Inspect)]`) | field_names / get_field / methods | 묵시 `rok::**`으로 트레잇을 쓸 수 있음, derive는 명시 |

자동과 opt-in으로 나눈 이유는 바이너리 크기입니다. 모든 struct에 필드 리플렉션 메타데이터가 붙으면 바이너리가 커집니다. 그래서 opt-in 방식으로 필요한 타입에만 메타데이터를 만듭니다.

## 2.12.2. `Reflect`: 정적 + 인스턴스 모두

```rok
trait Reflect {
    fn type_name() -> str;        // 정적: T::type_name()
    fn size() -> usize;
    fn align() -> usize;

    fn type_name_of(self) -> str; // 인스턴스: x.type_name_of() (any / sum 도 OK)
    fn size_of(self) -> usize;
}
```

이 트레잇은 모든 타입에 자동으로 구현됩니다. 컴파일러가 타입 정보를 바이너리에 자동으로 넣습니다.

컴파일러가 모든 타입에 `Reflect`를 구현하므로 소스에는 구현 코드가 없습니다.

- 손으로 쓴 `impl Reflect for T`는 에러입니다. `T: Reflect` 바운드는 늘 성립합니다.
- 타입에 `size()` / `type_name()`처럼 이름이 같은 메서드가 있으면 그 메서드가 리플렉션 메서드를 가립니다.
- 이름(`type_name`)에는 모듈 경로가 붙지 않습니다(`List<i64>`, `Config`). 합 타입의 정적 이름은 정규 순서([2.6.4](../../refs-lang/002-types/206-sum-types.md#264-집합-의미론))로 적습니다(`f64 | i32`, `none`은 마지막). `type_name_of(self)` 같은 인스턴스 형태는 런타임에 실제로 든 항(합 타입일 때)이나 `any`의 내용을 보고합니다.
- `rok::core::reflect::cast<T>(value: any) -> T | none`은 `set_field`와 같은 규칙으로 `any`를 `T`로 꺼냅니다. `rok::core::reflect::methods_of<T>() -> List<MethodInfo>`는 `T`의 고유 메서드 목록입니다.

### 정적과 인스턴스의 차이

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

`type_name_of`는 런타임 디스패치이므로 합 타입이나 any 슬롯에서는 지금 들어 있는 변형의 이름을 반환합니다.

### 전역 함수 없음

```rok
typeof(x);                 // ERROR: 글로벌 typeof 없음
sizeof(T);                 // ERROR: 글로벌 sizeof 없음
```

대신 모두 메서드입니다.

```rok
x.type_name_of()           // 인스턴스
T::type_name()             // 정적
T::size()
T::align()
```

이 규칙은 [refs-lang 3.5 멤버 접근과 메서드 호출](../../refs-lang/003-expressions/305-member-access-and-method-call.md)의 일부입니다. 모든 인트로스펙션은 메서드 형태입니다.

## 2.12.3. `Inspect`: opt-in 필드 / 메서드

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

- `get_field`는 그 필드의 값을 `any`로 돌려주고, 없는 이름이면 `none`을 돌려줍니다. 비공개 필드도 꺼낼 수 있습니다.
- `set_field`는 값의 타입이 필드 타입과 **정확히 같으면** 대입하고 `true`를 돌려줍니다. 정수는 타입이 달라도 **손실 없이 들어가면** 대입합니다(`u.set_field("id", 42)`가 `u64` 필드에). 그 밖에는 대입하지 않고 `false`를 돌려줍니다. 다른 변환(예: `i32` → `f64`)은 없습니다.
- `methods()`는 트레잇 메서드를 빼고 그 타입의 **고유 메서드만** 이름순으로 돌려줍니다. 제네릭 `impl`의 메서드는 타입을 적힌 그대로(`T`) 보여 줍니다.
- `enum`에 derive로 구현하면 필드는 없고(`field_names()`는 빈 목록, `get_field`는 `none`, `set_field`는 `false`), `methods()`만 의미가 있습니다.

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

`#[derive(Inspect)]`는 `rok::core::reflect`의 메타 블록으로 구현합니다([refs-lang 6.8](../../refs-lang/006-meta-system/608-meta-blocks.md), [6.9](../../refs-lang/006-meta-system/609-template-tokens.md)). 요약하면 다음과 같습니다.

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
                    if name == @field.name.as_str() { return self.@field; }
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
    }
}
```

메타 블록의 자세한 내용은 [refs-lang 6.8 메타 블록](../../refs-lang/006-meta-system/608-meta-blocks.md)에 있습니다.

### 동적 디스패치가 필요한 자리

`Inspect`는 설정 객체, 직렬화기, 디버거처럼 필드 이름으로 접근해야 하는 곳에서 씁니다. 일반 코드에서는 `Reflect`만으로 충분합니다.

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

REPL은 평범한 마지막 식을 `Display`로 출력합니다. 뒤에 `?`를 붙이면 `Debug`로 출력하고, `:type`은 이 장의 `Reflect::type_name_of`로 출력합니다([2.10 fmt](210-fmt.md), [refs-lang 6.10.3](../../refs-lang/006-meta-system/610-reflection.md#6103-repl--jupyter-자동-출력)).

```
> let x = 42i32;
> x
42

> [1, 2, 3]
List<i32>

> [1, 2, 3]?
[1, 2, 3]

> User { id: 1, name: "Alice" }?
User { id: 1, name: "Alice" }
```

평범한 식은 `Display`를, 뒤에 `?`가 붙으면 `Debug`를 씁니다. 요청한 트레잇(평범한 식은 `Display`, 뒤에 `?`가 붙으면 `Debug`)이 없는 값은 타입 이름만 출력합니다.

## 2.12.5. 모듈 매핑

| 항목             | 위치                                             | 메커니즘                      |
|------------------|--------------------------------------------------|-------------------------------|
| `Reflect` 자동   | 컴파일러 + `rok::core::reflect`                   | 타입 정보 테이블, 바이너리에 부착  |
| `Inspect` derive | `rok::core::reflect` + 메타 블록 `#[impl_derive]` | 템플릿 토큰 기반 코드 생성 |
| `MethodInfo`     | `rok::core::reflect`                              | 단순 struct                   |
| `any` 타입       | 컴파일러 내장                                     | 타입 태그 + 페이로드            |

## 2.12.6. 다른 언어와 비교

| 측면               | Rust                         | Java             | Python              | Rok                                         |
|--------------------|------------------------------|------------------|---------------------|---------------------------------------------|
| 타입 이름          | `std::any::type_name::<T>()` | `getClass()`     | `type(x).__name__`  | `x.type_name_of()` 메서드                   |
| 필드 introspection | **없음** (lib `bincode` 등)  | Reflection API   | `vars(x)` `getattr` | `x.get_field(name)` (opt-in)                |
| 런타임 디스패치    | `dyn Trait` (vtable)         | 자동             | 자동                | 합 타입 태그 + `any` / `any Trait` (opt-in) |
| 자동 구현          | **없음**                     | 모든 객체        | 모든 객체           | `Reflect` 자동, `Inspect` opt-in            |
| 메모리 비용        | 0                            | 모든 클래스 메타 | 모든 객체 dict      | `Reflect`만 자동 (opt-in 분리)             |

Rok은 크게 두 가지가 다릅니다. 첫째, 트레잇을 두 단계로 나눠 가벼운 `Reflect`는 자동으로, 무거운 `Inspect`는 opt-in으로 둡니다. 그래서 Java나 Python처럼 모두 자동인 모델보다 메모리를 덜 쓰고, Rust처럼 모두 수동인 모델보다 쓰기 편합니다. 둘째, 메서드 일관성을 위해 `typeof(x)` 같은 글로벌 함수가 **없습니다**. 모든 인트로스펙션은 메서드 형태입니다.

리플렉션의 자세한 의미는 [refs-lang 6.10](../../refs-lang/006-meta-system/610-reflection.md)이 정합니다.
