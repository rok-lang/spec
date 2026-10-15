# 6.9. 템플릿 토큰과 확장

이 절에서는 [메타 블록](608-meta-blocks.md) 의 본체 안에서 사용할 수 있는 템플릿 토큰의 전체 목록과 함께, 메타 블록이 언제 어떤 메커니즘으로 확장되는지를 정의합니다.

## 6.9.1. 템플릿 토큰의 형태

템플릿 토큰은 다음과 같은 문법 형태를 가집니다.

```
template_token  ::= "@" identifier
                  | "@" identifier "." identifier
                  | "@" identifier "{" <template body> "}"
                  | "@" identifier "(" <args> ")" "{" <body> "}"
```

모든 템플릿 토큰은 `@` 토큰 ([구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)) 으로 시작합니다. 대표적인 예는 다음과 같습니다.

```
@Self                   대상 타입 이름
@field                  현재 필드의 자리 표시
@field.name             현재 필드의 이름
@for_fields { ... }     필드 반복 블록
```

### 토큰 목록

| 토큰                                                     | 값                                          | 절    |
|----------------------------------------------------------|---------------------------------------------|-------|
| `@Self`                                                  | 대상 타입 (`Name<T, ..>`)                   | 6.9.2 |
| `@Self.name`                                             | 타입 인자 없는 대상 타입 이름               | 6.9.2 |
| `@Self.is_unit` / `.is_tuple` / `.is_named`              | struct 모양 (부울)                          | 6.9.2 |
| `@if_struct` / `@if_enum` / `@if_union { .. }`           | 대상 종류가 맞을 때만 본문                  | 6.9.2 |
| `@for_fields { .. }`                                     | 필드마다 본문                               | 6.9.3 |
| `@field`, `@field.name`, `@field.type`                   | 현재 필드 (접근 / 이름 / 타입)              | 6.9.3 |
| `@field.is_pub`, `@field.has_attr(x)`                    | 부울                                        | 6.9.3 |
| `@field.has_default`, `@field.default`                   | 필드의 `= expr` 유무 (부울) / 그 식         | 6.9.3 |
| `@for_variants { .. }`                                   | `enum` 변형마다 본문                        | 6.9.4 |
| `@variant.name`, `@variant.pattern`                      | 현재 변형 (이름 / 매칭 패턴)                | 6.9.4 |
| `@variant.is_unit` / `.is_tuple` / `.is_named`           | 부울                                        | 6.9.4 |
| `@for_variant_fields { .. }`                             | 현재 변형의 필드마다 본문                   | 6.9.4 |
| `@apply_if(cond) { .. }`                                 | `cond` 가 참일 때만 본문                    | 6.9.5 |
| `@accept(kind, ..)`                                      | 대상 종류 제한 (attribute 전용)             | 6.9.5 |
| `@where_all(Trait)`                                      | 타입 파라미터마다 `T: Trait`                | 6.9.5 |
| `@has_derive_attr(a)`, `@derive_attr(a)`                 | 필드의 derive attribute (부울 / 인자)       | 6.9.6 |
| `@item`, `@item.name` / `.params` / `.return_type` / `.body` | 부착 대상 항목과 그 부분 (attribute 전용) | 6.9.7 |
| `@attribute`                                             | 사용처의 attribute 인자 (attribute 전용)    | 6.9.7 |
| `<이름 토큰>.as_str()`                                   | 이름의 문자열 리터럴                        | 6.9.3 |

목록에 없는 `@이름` 은 에러입니다. `@where_for_fields` 는 예약된 이름이며, 써도 에러입니다.

## 6.9.2. 타입 컨텍스트

### `@Self` 대상 타입 이름

`@Self` 는 메타 블록이 적용되는 대상 타입의 이름을 나타냅니다.

```rok
#[impl_derive(MyTrait)] pub @derive {
    impl MyTrait for @Self {                 // @Self → Point / 사용자 타입 이름
        ...
    }
}
```

### `@if_struct { ... }` / `@if_enum { ... }` / `@if_union { ... }`

이 토큰들은 대상 타입이 각각 `struct`, `enum`, `union` 일 때만 안의 코드를 생성하도록 해줍니다.

```rok
@if_struct {
    fn struct_only_method() { ... }
}

@if_enum {
    fn enum_only_method() { ... }
}
```

만약 대상 타입이 다른 종류라면 해당 블록은 무시되며, 코드가 생성되지 않습니다.

### `@Self.name` / `@Self.is_unit` / `.is_tuple` / `.is_named`

`@Self.name` 은 타입 인자 없는 대상 타입 이름 (`Pair<T>` 의 `Pair`) 입니다. `@Self.is_unit` / `@Self.is_tuple` / `@Self.is_named` 는 대상 struct 가 unit struct, 튜플 struct, 이름 있는 필드 struct 인지를 나타내는 부울이며, `@apply_if` 에 씁니다.

## 6.9.3. 필드 반복

### `@for_fields { ... }`

`@for_fields` 는 `struct` 나 `union` 의 필드 각각에 대해 블록을 반복합니다. `@for_variants` 안에서는 현재 변형의 필드를 반복하며 `@for_variant_fields` 와 같습니다. `enum` 에 `@for_variants` 없이 쓰면 에러입니다.

```rok
@for_fields {
    println(self.@field);
}
```

위 코드는 각 필드마다 한 번씩 생성됩니다.

### `@field`

`@field` 는 현재 필드의 자리 표시이며, 필드 접근 위치에서 사용합니다.

```rok
self.@field                          // self.x, self.y, ...
@Self { @field: 0i32 }                // 필드 초기화 자리
```

### `@field.name` 식별자 토큰

식별자가 필요한 자리에서는 필드 이름이 식별자 토큰으로 삽입됩니다.

```rok
let v = @field.name;                  // 식별자 자리에 그 이름이 들어감
fn @field.name() -> i32 { ... }       // 함수 이름으로도 사용 가능
```

문자열이 필요하면 `.as_str()` 을 붙입니다. `@field.name.as_str()`, `@variant.name.as_str()`, `@item.name.as_str()`, `@Self.as_str()` 은 확장 시점에 그 이름의 문자열 리터럴 (`"x"`) 로 바뀝니다. `@Self.as_str()` 은 타입 인자 없는 이름입니다. 문자열이나 f-string 안에 적은 `@item.name` 같은 이름 토큰은 글자 그대로 치환됩니다.

### `@field.type`

`@field.type` 은 필드의 타입을 가리킵니다.

```rok
fn typed<T>() where T == @field.type { ... }
```

### `@field.is_pub`

이 토큰은 필드가 `pub` 이면 참, 아니면 거짓의 값을 가집니다.

```rok
@apply_if(@field.is_pub) {
    // pub 필드만 처리
}
```

### `@field.has_attr(X)`

필드에 `#[X]` 가 부착되어 있으면 참이 됩니다.

```rok
@apply_if(@field.has_attr(skip)) {
    // #[skip] 부착된 필드만 처리
}
```

### `@field.has_default` / `@field.default`

`@field.has_default` 는 필드에 기본값 (`x: i32 = 0`, [2.3.3](../002-types/203-struct.md#233-필드-기본값)) 이 있으면 참이고, `@field.default` 는 그 `= expr` 의 식입니다. 기본값이 없는 필드에서 `@field.default` 를 쓰면 에러입니다.

```rok
@apply_if(@field.has_default) {
    let @field.name = @field.default;
}
```

## 6.9.4. 변형 (variant) 반복

### `@for_variants { ... }`

`@for_variants` 는 `enum` 의 각 variant 에 대해 블록을 반복합니다.

```rok
@for_variants {
    @variant.name => /* ... */
}
```

### `@variant.name` variant 이름

`@variant.name` 은 variant 의 이름입니다.

### `@variant.pattern` 매칭 패턴

`@variant.pattern` 은 variant 의 매칭 패턴 형태로, 예를 들면 `Shape::Circle(r)`, `Shape::Pending`, `Shape::Click { x: x1, y: y1 }` 와 같은 모양이 됩니다. 여기서 `r`, `x1`, `y1` 자리의 바인딩은 컴파일러가 만든 이름이며 (위생, 6.9.8), 그 값은 `@for_variant_fields` 안의 `@field` 로 꺼냅니다. 바인딩 이름을 템플릿이 직접 쓸 수는 없습니다.

```rok
match self {
    @for_variants {
        @variant.pattern => /* ... */
    }
}
```

### `@variant.is_unit` / `@variant.is_tuple` / `@variant.is_named`

이 토큰들은 variant 의 종류를 검사할 때 사용합니다.

```rok
@apply_if(@variant.is_tuple) {
    // tuple variant 만 처리
}
```

### `@for_variant_fields { ... }`

variant 안의 필드들에 대해 반복할 때는 `@for_variant_fields` 를 사용합니다. 그 안에서는 `@field.name` 등을 함께 사용할 수 있습니다.

```rok
@for_variants {
    @variant.pattern => {
        @for_variant_fields {
            process(@field);
        }
    }
}
```

## 6.9.5. 조건부와 경계

### `@apply_if(cond) { ... }`

`@apply_if` 는 `cond` 가 참일 때만 안의 코드를 생성합니다.

```rok
@apply_if(@field.is_pub) {
    // 코드
}

@apply_if(!@field.has_attr(skip)) {
    // skip 이 *없는* 필드만
}

@apply_if(@field.is_pub && @field.has_attr(serialize)) {
    // 두 조건 모두 참
}
```

여기서 `cond` 는 템플릿 토큰의 부울 값 (예: `@field.is_pub`, `@field.has_attr(...)`) 과 일반 부울 연산자 (`!`, `&&`, `||`) 를 결합한 식이며, 컴파일 타임에 평가됩니다.

`else` 분기는 별도의 키워드 없이 반대 조건의 `@apply_if` 를 사용해 표현합니다.

### `@accept(...)`

`@accept` 는 derive 또는 attribute 의 대상 종류를 제한하는 데 사용합니다. 콤마나 `|` 로 구분된 항목 종류 목록을 받습니다.

```rok
#[impl_attribute(my_wrap)] pub @attribute {
    @accept(struct, enum)                    // struct, enum 만 받음 — union 거부
    @item
}

#[impl_attribute(timed)] pub @attribute {
    @accept(fn)                              // fn 만 받음
    fn @item.name(...) { ... }
}
```

가능한 종류는 다음과 같습니다.

| 종류     | 의미             |
|----------|------------------|
| `struct` | `struct` 항목    |
| `enum`   | `enum` 항목      |
| `union`  | `union` 항목     |
| `fn`     | 함수 또는 메서드 |
| `impl`   | `impl` 블록      |
| `trait`  | `trait` 항목     |

여러 종류를 콤마나 `|` 로 구분하면 그중 하나만 맞아도 허용됩니다. 만약 대상이 허용 종류가 아니라면 사용처에서 컴파일 에러가 발생합니다.

### `@where_all(Trait)`

`@where_all` 은 모든 필드 타입이 `Trait` 를 충족해야 한다는 조건을 자동 where 절로 추가합니다.

```rok
#[impl_derive(Eq)] pub @derive {
    @where_all(Eq)                           // 모든 필드의 타입이 Eq 면 OK
    impl Eq for @Self {
        fn eq(self, other: Self) -> bool {
            @for_fields {
                if !self.@field.eq(other.@field) { return false; }
            }
            true
        }
    }
}
```

이렇게 하면 생성되는 `impl` 에는 대상 타입의 타입 파라미터마다 `T: Eq` 가 추가됩니다. 구체 필드 타입에 대한 절은 넣지 않으며, 그 필드들이 `Eq` 를 구현하는지는 생성된 본문의 타입 검사가 봅니다. 템플릿의 `impl` 뒤에는 대상 타입의 제네릭 파라미터가 들어가며, 바운드는 `@where_all` 이 붙인 것뿐입니다.

## 6.9.6. derive_attribute 토큰

다음은 `#[impl_derive_attribute(Trait, attr)]` 안에서 추가로 사용할 수 있는 토큰들입니다.

### `@has_derive_attr(name)`

이 토큰은 필드에 `derive_attribute` 의 대상 attr 가 부착되어 있는지를 나타냅니다.

### `@derive_attr(name)` 그 attr 의 인자

`@derive_attr(name)` 은 부착된 attr 의 인자를 가리킵니다.

```rok
#[impl_derive_attribute(Serialize, rename)] pub @derive {
    @for_fields {
        @apply_if(@has_derive_attr(rename)) {
            let key = @derive_attr(rename);  // #[rename("...")] 의 인자
            obj.set(key, ...);
        }
    }
}
```

## 6.9.7. attribute 전용 토큰

다음 토큰들은 `#[impl_attribute(name)]` 안에서 사용합니다.

### `@item` 원본 항목 재사용

`@item` 은 부착 대상 항목 전체를 그대로 재사용할 때 사용합니다.

```rok
#[impl_attribute(deprecated_warning)] pub @attribute {
    @item                                    // 원본 그대로
}
// 위는 attribute 가 사실상 no-op — 원본 그대로 통과
```

### `@item.body`

`@item.body` 는 함수의 본체이며, 함수 항목에 대해서만 의미가 있습니다.

### `@item.name`, `@item.params`, `@item.return_type`

이 토큰들은 함수 시그니처의 각 부분을 가리킵니다.

```rok
#[impl_attribute(timed)] pub @attribute {
    fn @item.name(@item.params) -> @item.return_type {
        let start = now();
        let r = { @item.body };
        log(f"@item.name took {now() - start}");
        r
    }
}
```

### `@attribute` `#[...]` 인자

`@attribute` 는 부착 시점에 적힌 인자를 의미합니다. 즉 `#[impl_attribute(name)]` 가 부착된 항목을 사용처가 `#[name(args)]` 형태로 사용했다면, `@attribute` 는 그 args 가 됩니다.

```rok
#[impl_attribute(prefix)] pub @attribute {
    fn @item.name() {
        let p = @attribute;                  // 사용처의 인자
        log(f"{p}: {@item.name}");
        @item.body
    }
}

// 사용처
#[prefix("INFO")]
fn run() { ... }
// → 생성.
//   fn run() {
//       let p = "INFO";
//       log(f"{p}: run");
//       ...
//   }
```

## 6.9.8. 확장 메커니즘

### 모으기와 확장

메타 확장은 다음 흐름으로 진행됩니다.

1. parse 전의 pre-pass 가 라이브러리와 사용자 패키지의 모든 파일에서 메타 블록을 모읍니다 ([6.8.11](608-meta-blocks.md#6811-등록가시성과의-결합)).
2. 파서가 `#[derive(X)]` 나 `#[x]` 가 붙은 항목을 만나면, 그 메타 블록의 본체에서 템플릿 토큰을 항목의 값으로 치환하고 결과를 다시 파싱합니다.
3. 생성된 항목도 다시 메타 처리됩니다. 고정점에 도달할 때까지 반복합니다.
4. 순환 (`A → B → A`) 은 에러입니다 ([6.8.8](608-meta-blocks.md#688-fixed-point-확장)).

### `cfg` 와의 관계

메타 블록 자체에 붙은 `#[cfg(..)]` 는 그 블록의 **등록** 을 정합니다. cfg 평가가 메타 블록 수집보다 먼저이므로, cfg 가 거짓인 블록의 derive / attribute 는 등록되지 않습니다. 그래서 서로 배타적인 cfg 아래에서 두 블록이 같은 이름을 정의할 수 있습니다. 블록을 둘러싼 모듈 (인라인 `mod` 와 파일 모듈) 의 `#[cfg]` 도 등록에 적용됩니다. 파일 안의 `#![cfg]` 는 아직 적용되지 않습니다.

항목에 붙은 [`#[cfg]`](603-cfg.md) 는 그 항목에서 생성된 모든 항목에 그대로 복사됩니다. 따라서 cfg 가 거짓인 항목은 생성물과 함께 사라지며, 결과는 다음 순서와 같습니다.

```
parse → cfg eval → meta expansion (fixed-point) → 이름 해소 → 타입 검사 → ...
```

### 위생 (hygiene)

템플릿이 묶는 지역 이름, 곧 `let` / `for` 의 바인딩과 클로저 매개변수 (f-string 의 `{..}` 안 포함) 는 확장할 때 겹치지 않는 이름으로 바뀝니다. 그래서 템플릿의 지역은 항목에서 온 이름 (`@item.body`, `@item.params`, `let @field.name = ..`) 을 가리거나 붙잡지 않습니다.

```rok
#[impl_derive(Foo)] pub @derive {
    impl Foo for @Self {
        fn foo(self) -> i32 {
            let temp = compute();             // 확장 시 이름이 바뀜
            temp * 2
        }
    }
}

// 사용처
#[derive(Foo)]
struct S { temp: str; }                       // 사용처의 'temp' 필드

let s = S { temp: "hi" };
let r = s.foo();                              // 메타가 만든 'temp' 와 충돌 안 함
```

`match` arm 패턴이 묶는 이름과 `S { x }` 축약은 바꾸지 않습니다. 템플릿은 그 자리에 항목과 겹칠 수 있는 이름을 쓰지 않습니다. `@variant.pattern` 의 바인딩은 컴파일러가 만든 이름이므로 겹치지 않습니다.

### 자유 식별자의 해소

생성된 항목은 **사용처의 모듈** 에 놓입니다. 따라서 템플릿 안에서 묶이지 않은 식별자 (예: `compute()`, `now()`, `log()`) 는 메타 블록의 모듈이 아니라 사용처에서 해소됩니다.

템플릿 작성자는 절대 경로 (`rok::core::..`, 라이브러리의 `pkg::..`) 를 쓰거나, 사용처에서 보이는 이름 (묵시 `use rok::core::**;` 의 항목 등) 만 씁니다.

```rok
#[impl_derive(MyHash)] pub @derive {
    impl MyHash for @Self {
        fn my_hash(self) -> u64 {
            let h: u64 = 0u64;
            @for_fields { h = mylib::hash::combine(h, self.@field.my_hash()); }   // 절대 경로
            h
        }
    }
}
```

### 진단 위치

확장 중의 에러와 확장 결과의 파싱 에러는 템플릿의 위치를 가리키고, 사용처 (`#[derive(X)]` / `#[x]`) 를 note 로 덧붙입니다. 템플릿 토큰을 거쳐 항목에서 온 토큰은 항목의 위치를 가리킵니다.

## 6.9.9. 디버깅

`rokc build --emit=expanded` ([9.9.10](../009-tools/909-debug-tools.md#9910-메타-블록-디버그)) 는 사용자 패키지의 확장 결과를 소스 텍스트로 출력합니다. 확장마다 `// expanded from this #[derive(X)] (file:line:col)` 주석과 생성된 소스가 이어지며, 생성물이 다시 확장되면 바깥 확장 뒤에 이어집니다.

## 6.9.10. 정리

템플릿 토큰은 rok 메타 시스템의 코드 생성 어휘입니다. 단순한 형태이지만 실용적인 derive 와 attribute 의 핵심 사례를 모두 다룰 수 있습니다.

토큰 변환의 유연성을 제한하는 대신 디버깅 가능성과 도구 친화성, 위생 보장을 얻는 것이 rok 의 선택입니다.

이로써 6장 (메타 시스템) 을 마무리합니다.
