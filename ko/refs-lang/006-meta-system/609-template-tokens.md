# 6.9. 템플릿 토큰과 확장

[메타 블록](608-meta-blocks.md) 의 본체 안에서 사용할 수 있는 템플릿 토큰의 전체 목록과, 메타 블록의 확장
시점과 메커니즘을 정의한다.

## 6.9.1. 템플릿 토큰의 형태

```
template_token  ::= "@" identifier
                  | "@" identifier "." identifier
                  | "@" identifier "{" <template body> "}"
                  | "@" identifier "(" <args> ")" "{" <body> "}"
```

모두 `@` 토큰 ([구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)) 으로 시작한다.

```
@Self                   대상 타입 이름
@field                  현재 필드의 자리 표시
@field.name             현재 필드의 이름
@for_fields { ... }     필드 반복 블록
```

## 6.9.2. 타입 컨텍스트

### `@Self` 대상 타입 이름

```nok
#[impl_derive(MyTrait)] pub @derive {
    impl MyTrait for @Self {                 // @Self → Point / 사용자 타입 이름
        ...
    }
}
```

`@Self` 는 메타 블록이 적용되는 대상 타입의 이름이다.

### `@if_struct { ... }` / `@if_enum { ... }` / `@if_union { ... }`

대상 타입이 `struct` / `enum` / `union` 일 때만 안의 코드를 생성한다.

```nok
@if_struct {
    fn struct_only_method() { ... }
}

@if_enum {
    fn enum_only_method() { ... }
}
```

대상 타입이 다른 종류면 그 블록은 무시된다 (코드가 생성되지 않는다).

## 6.9.3. 필드 반복

### `@for_fields { ... }`

`struct` / `union` 의 필드, 또는 `enum` named-variant 의 필드 각각에 대해 블록을 반복한다.

```nok
@for_fields {
    println(self.@field);
}
```

각 필드에 대해 한 번씩 위 코드가 생성된다.

### `@field`

현재 필드의 자리 표시다. 필드 접근 위치에서 사용한다.

```nok
self.@field                          // self.x, self.y, ...
@Self { @field: 0i32 }                // 필드 초기화 자리
```

### `@field.name` 식별자 토큰

```nok
let v = @field.name;                  // 식별자 자리에 그 이름이 들어감
fn @field.name() -> i32 { ... }       // 함수 이름으로도 사용 가능
```

식별자가 필요한 자리에서 필드 이름이 식별자 토큰으로 삽입된다.

문자열로 쓰려면 `@field.name.as_str()` 같은 변환을 사용한다 (정확한 형태는 표준 라이브러리에서 정의한다).

### `@field.type`

```nok
fn typed<T>() where T == @field.type { ... }
```

필드의 타입을 가리킨다.

### `@field.is_pub`

필드가 `pub` 이면 참, 아니면 거짓.

```nok
@apply_if(@field.is_pub) {
    // pub 필드만 처리
}
```

### `@field.has_attr(X)`

필드에 `#[X]` 가 부착되어 있으면 참.

```nok
@apply_if(@field.has_attr(skip)) {
    // #[skip] 부착된 필드만 처리
}
```

## 6.9.4. 변형 (variant) 반복

### `@for_variants { ... }`

`enum` 의 각 variant 에 대해 블록을 반복한다.

```nok
@for_variants {
    @variant.name => /* ... */
}
```

### `@variant.name` variant 이름

### `@variant.pattern` 매칭 패턴

variant 의 매칭 패턴 형태. 예: `Circle(r)`, `Pending`, `Click { x, y }`.

```nok
match self {
    @for_variants {
        @variant.pattern => /* ... */
    }
}
```

### `@variant.is_unit` / `@variant.is_tuple` / `@variant.is_named`

variant 의 종류를 검사한다.

```nok
@apply_if(@variant.is_tuple) {
    // tuple variant 만 처리
}
```

### `@for_variant_fields { ... }`

variant 안의 필드들에 대해 반복한다. 그 안에서 `@field.name` 등을 사용할 수 있다.

```nok
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

`cond` 가 참일 때만 안의 코드를 생성한다.

```nok
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

`cond` 는 템플릿 토큰의 부울 값 (예: `@field.is_pub`, `@field.has_attr(...)`) 과 일반
부울 연산자 (`!`, `&&`, `||`) 를 결합한 식이다. 컴파일 타임에 평가된다.

`else` 분기는 별도의 키워드 없이, 반대 조건의 `@apply_if` 를 사용한다.

### `@accept(...)`

derive 또는 attribute 의 대상 종류를 제한한다. 콤마나 `|` 로 구분된 항목 종류 목록을 받는다.

```nok
#[impl_attribute(my_wrap)] pub @attribute {
    @accept(struct, enum)                    // struct, enum 만 받음 — union 거부
    @item
}

#[impl_attribute(timed)] pub @attribute {
    @accept(fn)                              // fn 만 받음
    fn @item.name(...) { ... }
}
```

가능한 종류는 다음과 같다.

| 종류     | 의미             |
| -------- | ---------------- |
| `struct` | `struct` 항목    |
| `enum`   | `enum` 항목      |
| `union`  | `union` 항목     |
| `fn`     | 함수 또는 메서드 |
| `impl`   | `impl` 블록      |
| `trait`  | `trait` 항목     |

여러 종류를 콤마나 `|` 로 구분하면 그중 하나만 맞아도 OK 다.

대상이 허용 종류가 아니면 사용처에서 컴파일 에러가 발생한다.

### `@where_all(Trait)`

모든 필드 타입이 `Trait` 를 충족해야 한다는 조건을 자동 where 절로 추가한다.

```nok
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

생성되는 `impl` 에는 `where T: Eq` 같은 절이 자동으로 추가된다.

### `@where_for_fields(pred)`

특정 조건의 필드에만 where 절을 적용한다 (정확한 형태는 표준 라이브러리에서 정의한다).

## 6.9.6. derive_attribute 토큰

`#[impl_derive_attribute(Trait, attr)]` 안에서 추가로 사용할 수 있는 토큰들이다.

### `@has_derive_attr(name)`

필드에 `derive_attribute` 의 대상 attr 가 부착되어 있는지를 나타낸다.

### `@derive_attr(name)` 그 attr 의 인자

```nok
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

`#[impl_attribute(name)]` 안에서 사용한다.

### `@item` 원본 항목 재사용

부착 대상 항목 전체를 그대로 재사용한다.

```nok
#[impl_attribute(deprecated_warning)] pub @attribute {
    @item                                    // 원본 그대로
}
// 위는 attribute 가 사실상 no-op — 원본 그대로 통과
```

### `@item.body`

함수의 본체다. 함수 항목에만 의미가 있다.

### `@item.name`, `@item.params`, `@item.return_type`

함수 시그니처의 각 부분을 가리킨다.

```nok
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

부착 시점에 적힌 인자다. `#[impl_attribute(name)]` 가 부착된 항목을 사용처가
`#[name(args)]` 로 사용했다면 `@attribute` 가 그 args 가 된다.

```nok
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
// → 생성:
//   fn run() {
//       let p = "INFO";
//       log(f"{p}: run");
//       ...
//   }
```

## 6.9.8. 확장 메커니즘

### parse 직후의 expand 패스

[컴파일 파이프라인](../000-meta/002-pipeline.md) 의 parse 직후에 메타 확장 패스가 실행된다.

1. parse 결과 AST 에서 `#[derive]`, `#[attribute_name]`, 메타 블록의 적용을 찾는다.
2. 각 적용을 해당 메타 블록의 본체로 확장한다 (템플릿 토큰을 실제 값으로 치환한다).
3. 확장 결과 AST 도 다시 메타 처리될 수 있다 (재귀).
4. 고정점에 도달할 때까지 반복한다.

### `cfg` 와의 순서

[`#[cfg]`](603-cfg.md) 의 cfg 처리는 메타 확장 전에 일어난다. cfg 가 거짓인 항목은 처음부터
사라지므로, 그 항목에 적용된 메타 블록은 처리되지 않는다.

순서는 다음과 같다.

```
parse → cfg eval → meta expansion (fixed-point) → 이름 해소 → 타입 검사 → ...
```

### 위생 (hygiene)

메타 블록이 생성하는 코드 안의 식별자는 생성 컨텍스트의 이름 공간만 본다. 사용처의
이름 공간을 오염시키지 않는다.

예를 들어 메타 블록이 임시 변수를 도입하는 경우를 보자.

```nok
#[impl_derive(Foo)] pub @derive {
    impl Foo for @Self {
        fn foo(self) -> i32 {
            let temp = compute();             // 'temp' 는 메타 블록 컨텍스트
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

생성된 메서드 안의 `temp` 는 메타 블록 안의 임시 변수일 뿐이며, 사용처의 `S::temp` 필드와는 무관하다.

### 자유 식별자의 해소

메타 블록 안에서 해소되지 않은 식별자 (예: `compute()`, `now()`, `log()`) 는 메타 블록의
모듈 컨텍스트에서 보이는 항목으로 해소된다.

따라서 메타 블록 작성자가 필요한 도구 (`compute`, `log` 등) 를 그 모듈 안에
import 해두면 사용처의 환경과 무관하게 동작한다.

## 6.9.9. 디버깅

`nokc --emit=expanded` 같은 명령으로 확장된 결과 AST 를 출력할 수 있다 (정확한 명령은 컴파일러
사양에 따른다). 메타 블록이 어떤 코드를 생성했는지 시각적으로 확인할 수 있다.

## 6.9.10. 정리

템플릿 토큰은 nok 메타 시스템의 코드 생성 어휘다. 단순한 형태이지만 실용적인 derive 와
attribute 의 핵심 사례를 모두 다룬다.

토큰 변환의 유연성을 제한하는 대신 디버깅 가능성, 도구 친화성, 위생 보장을 얻는 것이 nok 의 선택이다.

이로써 6장 (메타 시스템) 을 마무리한다.
