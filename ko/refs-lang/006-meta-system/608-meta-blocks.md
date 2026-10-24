# 6.8. 사용자 정의 메타 정의

**메타 정의 (meta definition)** 는 사용자 정의 코드 생성기입니다. 모두 하나의 메커니즘입니다 — 입력의 일부를 바인딩하고, rok 코드와 템플릿 토큰 ([6.9](609-template-tokens.md)) 으로 된 템플릿을 위생적으로 ([6.8.9](#689-토큰-스트림-조작-없음)) 그리고 고정점까지 ([6.8.8](#688-fixed-point-확장)) 전사합니다. 차이는 오직 **무엇으로 트리거되고 어떤 입력을 바인딩하는가** 뿐입니다. 네 가지 형태가 있으며, 앞의 셋은 항목에 붙인 attribute 로, 넷째는 호출로 트리거됩니다.

| 형태                                    | 정의                                 | 트리거                 | 바인딩                                  |
|-----------------------------------------|--------------------------------------|------------------------|-----------------------------------------|
| derive                                  | `#[impl_derive(T)] @derive {}`       | 타입에 `#[derive(T)]`  | 타입의 shape (`@Self`, `@for_fields`)   |
| attribute                               | `#[impl_attribute(n)] @attribute {}` | 항목에 `#[n]`          | 항목 (`@item`)                          |
| 필드별 derive attribute                 | `#[impl_derive_attribute(T, a)] @derive {}` | `#[derive(T)]` + 필드 `#[a]` | shape + 필드 attribute          |
| **macro**                               | `#[impl_macro(n)] @macro { (m) => {t} }` | `n!(..)` 호출      | 호출의 토큰 (`@x: expr`, `@rep`)        |

앞의 셋은 *붙인 타입/항목으로부터* 코드를 생성하고, macro 는 *호출자가 쓴 것으로부터* 생성합니다 — 나머지가 할 수 없는 바로 그 수요입니다 ([6.8.12](#6812-macro-형태)). 넷은 템플릿 엔진, `@` 토큰, 위생, 아래의 도구 보장을 공유하며, 바인딩 입력이 다른 지점만 이 장과 [6.9](609-template-tokens.md) 에서 짚습니다. 이 넷이 rok 의 전체 메타프로그래밍 표면입니다 — 함께 Rust 의 derive·attribute·함수형 매크로를 대체하되, 절차적 코드나 원시 토큰 조작은 없습니다.

```
meta_block      ::= meta_attr visibility? meta_kind block
meta_attr       ::= "#" "[" meta_kind_attr "(" meta_args ")" "]"
meta_kind_attr  ::= "impl_derive" | "impl_attribute" | "impl_derive_attribute" | "impl_macro"
meta_kind       ::= "@derive" | "@attribute" | "@macro"
meta_args       ::= identifier ("," identifier)*

# @macro 블록은 평평한 템플릿 대신 규칙들을 담습니다:
macro_body      ::= macro_rule+
macro_rule      ::= "(" matcher ")" "=>" block
```

## 6.8.1. attribute 로 트리거되는 세 형태

attribute 로 트리거되는 세 형태는 다음과 같습니다.

```rok
#[impl_derive(TraitName)] pub @derive {
    impl TraitName for @Self { ... }
}

#[impl_attribute(name)] pub @attribute {
    @item                                    // 원본 항목 재사용
    fn additional_helper() { ... }            // 추가 항목 생성
}

#[impl_derive_attribute(Trait, attr)] pub @derive {
    @for_fields {
        @apply_if(@has_derive_attr(attr)) {
            // 그 attr 가 부착된 필드만 처리
        }
    }
}
```

각 형태가 어떤 용도로 쓰이는지는 아래 표에 정리됩니다.

| 형태                                    | 용도                                 |
|-----------------------------------------|--------------------------------------|
| `#[impl_derive(TraitName)]`             | 사용자 정의 derive                   |
| `#[impl_attribute(name)]`               | 사용자 정의 attribute (item wrapper) |
| `#[impl_derive_attribute(Trait, attr)]` | 필드별 derive attribute              |

호출로 트리거되는 넷째 형태 `macro` 는 [6.8.12](#6812-macro-형태) 에서 정의합니다.

## 6.8.2. `#[impl_derive(TraitName)]` 사용자 정의 derive

다음은 사용자 정의 derive 의 예시입니다.

```rok
#[impl_derive(MyHash)] pub @derive {
    impl MyHash for @Self {
        fn my_hash(self) -> u64 {
            let h: u64 = 0u64;
            @for_fields {
                h = combine_hash(h, self.@field.my_hash());
            }
            h
        }
    }
}

// 사용처
#[derive(MyHash)]
struct Point { x: f64; y: f64; }
// → 자동 생성됨.
//   impl MyHash for Point {
//       fn my_hash(self) -> u64 {
//           let h: u64 = 0u64;
//           h = combine_hash(h, self.x.my_hash());
//           h = combine_hash(h, self.y.my_hash());
//           h
//       }
//   }
```

이 형태는 `#[impl_derive(이름)]` 와 본체 (템플릿 블록) 로 구성되며, 사용처에서 `#[derive(이름)]` 가 등장하면 템플릿이 그 타입에 대해 확장되어 코드가 생성됩니다.

## 6.8.3. `#[impl_attribute(name)]` 사용자 정의 attribute

다음은 사용자 정의 attribute 의 예시입니다.

```rok
#[impl_attribute(timed)] pub @attribute {
    @accept(fn)                                  // fn 항목에만 부착 가능

    fn @item.name(@item.params) -> @item.return_type {
        let start = now();
        let result = {
            @item.body
        };
        log(f"@item.name took {now() - start}");
        result
    }
}

// 사용처
#[timed]
fn process(x: i32) -> i32 {
    expensive(x)
}
// → 자동 생성됨.
//   fn process(x: i32) -> i32 {
//       let start = now();
//       let result = { expensive(x) };
//       log(f"process took {now() - start}");
//       result
//   }
```

이는 항목을 감싸는 wrapper 에 해당하며, 원본 항목 (`@item`) 의 내용과 시그니처를 재사용해 변환된 형태를 생성합니다. 함수뿐 아니라 `struct`, `enum`, `impl` 에도 부착할 수 있고, 대상 제한은 `@accept` 로 표시합니다.

## 6.8.4. `#[impl_derive_attribute(Trait, attr)]` 필드별 derive attribute

필드별 derive attribute 의 예시는 다음과 같습니다.

```rok
#[impl_derive_attribute(Serialize, rename)] pub @derive {
    impl Serialize for @Self {
        fn serialize(self) -> Json {
            let obj = JsonObj::new();
            @for_fields {
                @apply_if(@has_derive_attr(rename)) {
                    obj.set(@derive_attr(rename), self.@field.serialize());
                }
                @apply_if(!@has_derive_attr(rename)) {
                    obj.set(@field.name.as_str(), self.@field.serialize());
                }
            }
            obj.into()
        }
    }
}

// 사용처
#[derive(Serialize)]
struct User {
    #[rename("user_name")]
    name: str;
    age: i32;
}
// → 자동 생성.
//   impl Serialize for User {
//       fn serialize(self) -> Json {
//           let obj = JsonObj::new();
//           obj.set("user_name", self.name.serialize());
//           obj.set("age", self.age.serialize());
//           obj.into()
//       }
//   }
```

이 형태는 두 인자, 즉 derive 의 대상이 되는 Trait 와 필드별 정보를 표시하는 attribute 를 받으며, derive 의 본체 안에서 필드의 attribute 정보를 읽어 분기합니다.

## 6.8.5. 본체 (템플릿 블록)

모든 형태에서 본체는 템플릿 블록이며, 일반 rok 코드와 템플릿 토큰 (attribute 로 트리거되는 형태는 `@Self`, `@field`, `@for_fields`; macro 는 `@x`, `@rep`, `@for`, `@concat`, [6.8.12](#6812-macro-형태)) 이 함께 들어갑니다.

자세한 템플릿 토큰 목록은 [템플릿 토큰과 확장](609-template-tokens.md) 에서 정의합니다.

## 6.8.6. `pub` / 가시성

가시성은 다음과 같이 표기합니다.

```rok
pub @derive { ... }
pub(pkg) @attribute { ... }
@derive { ... }                  // 비공개
```

가시성 키워드는 템플릿 블록 자체에 부착합니다. 효과는 다음과 같습니다.

| 가시성                  | 쓸 수 있는 곳                                     |
|-------------------------|---------------------------------------------------|
| `pub` (`pub(pkg)` 포함) | 빌드 전체. `use` 없이 `#[derive(X)]` / `#[x]` 로 씀 |
| 비공개                  | 메타 블록을 정의한 파일 안                        |

derive 와 attribute 이름은 모듈 경로를 갖지 않는 빌드 전역 이름입니다 (6.8.11). 메타 블록이나 키워드 처리기의 `#[cfg]` 는 그것을 정의한 패키지의 cfg (그 패키지 자신의 feature, [9.10.4c](../009-tools/910-rok-frontend.md#9104c-features)) 로 평가합니다.

## 6.8.7. 본체 시작자 (`@derive` / `@attribute`)

본체 시작자는 템플릿의 종류를 표시하는 역할을 합니다.

| 시작자       | 용도                                                 |
|--------------|------------------------------------------------------|
| `@derive`    | derive 정의 (`impl_derive`, `impl_derive_attribute`) |
| `@attribute` | attribute 정의 (`impl_attribute`)                    |
| `@macro`     | macro 정의 (`impl_macro`); 블록이 규칙들을 담음 ([6.8.12](#6812-macro-형태)) |

이 시작자가 어떤 템플릿 토큰이 본체 안에서 의미를 갖는지 결정합니다. 예를 들어 `@item` 은 attribute 전용이고, 매처 토큰 (`@x: expr`, `@rep`) 은 `@macro` 전용입니다.

## 6.8.8. fixed-point 확장

템플릿 확장 결과가 또 다른 derive 또는 attribute 가 부착된 항목을 만들어 내면, 그 항목은 재귀적으로 확장됩니다.

```rok
#[impl_attribute(adds_helper)] pub @attribute {
    @item                                       // 원본 항목 그대로

    #[derive(MyHelper)]                          // 새 항목 - 다시 derive 처리됨
    struct Helper { ... }
}
```

위 attribute 가 적용되면 본체의 `#[derive(MyHelper)]` 가 다음 확장 패스에서 다시 처리됩니다.

확장은 더 이상 확장할 derive 나 attribute 가 남아 있지 않은 고정점 (fixed-point) 에 도달하면 종료됩니다.

순환 확장 (A 가 B 를 호출하고 B 가 다시 A 를 호출하는 경우) 은 컴파일러가 검출하여 ERROR 를 발생시킵니다. 메시지는 순환 경로 (`A → B → A`) 를 보여 줍니다.

## 6.8.9. 토큰 스트림 조작 없음

모든 메타 정의 — attribute 로 트리거되는 세 형태와 macro — 는 오직 템플릿 토큰만으로 코드를 생성합니다. Rust 의 `syn` 이나 `quote` 같은 임의 토큰 변환 기능도, 컴파일타임에 코드를 실행하는 절차적 형태도 rok 에 존재하지 않습니다.

이러한 제약은 메타 정의의 출력이 컴파일러가 이해할 수 있는 rok 코드임을 보장하기 위한 것이며, 도구 친화성과 디버깅 가능성을 동시에 확보합니다.

## 6.8.10. proc-macro 와의 비교

Rust 의 proc macro 와 rok 메타 블록을 비교하면 다음과 같습니다.

| 측면   | Rust proc macro           | rok 메타 블록                    |
|--------|---------------------------|----------------------------------|
| 입력   | 임의 토큰 스트림          | 항목의 구조 정보 (필드, 변형 등) |
| 출력   | 임의 토큰 스트림          | rok 코드와 템플릿 토큰           |
| 위생   | proc macro 의 결정에 따름 | 자동 (hygienic)                  |
| 디버깅 | 출력 토큰 확인            | 확장된 AST 확인                  |
| 표현력 | 매우 높음                 | 제한적                           |

rok 의 선택은 표현력을 제한하는 대신 예측 가능성과 디버깅 용이성을 얻는 쪽에 무게를 두고 있습니다.

## 6.8.11. 등록/가시성과의 결합

`#[impl_derive(TraitName)]` 가 부착된 메타 블록은 `TraitName` 이라는 derive 를, `#[impl_attribute(name)]` 는 `#[name]` attribute 를, `#[impl_macro(name)]` 는 `name!` 매크로를 등록합니다. 컴파일러는 parse 전에 라이브러리와 사용자 패키지의 모든 파일에서 메타 블록을 모으므로, 정의 위치나 파일 순서와 무관하게 쓸 수 있습니다. 메타 블록에 붙은 `#[cfg]` 가 거짓이면 그 블록은 등록되지 않습니다 ([6.9.9](609-template-tokens.md#cfg-와의-관계)).

`pub` 메타 블록은 `use` 없이 빌드 전체에서 보입니다 (6.8.6). 메타 블록은 이름 공간의 항목이 아니므로 `use` 로 가져오는 대상도 아닙니다.

```rok
// pkg::derives
#[impl_derive(MyHash)] pub @derive {
    impl MyHash for @Self { ... }
}

// 다른 모듈 - import 없이 사용
#[derive(MyHash)]
struct Foo { ... }
```

다음은 에러입니다.

- 같은 derive·attribute·macro 이름을 두 메타 블록이 등록하는 것 (하나의 macro 는 모든 규칙을 담은 하나의 `@macro` 블록).
- 내장 derive ([6.2.1](602-built-in-attributes.md): `Clone`, `Copy`, `Default`, `Debug`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`) 나 내장 attribute (`cfg`, `derive`, `test`, `inline`, `repr`, `keyword` 등 6.2 ~ 6.7 의 속성) 의 이름을 메타 블록이 다시 정의하는 것.

## 6.8.12. macro 형태

**macro** 는 호출로 트리거되는 메타 정의입니다. derive 나 attribute 가 붙인 타입/항목으로부터 코드를 생성하는 반면, macro 는 `name!(..)` 처럼 *호출* 되어 그 자리에 코드로 확장됩니다 — 호출자가 쓴 것으로부터 코드를 생성합니다. attribute 로 트리거되는 형태들이 할 수 없는 바로 그 수요, 즉 타입의 필드가 아니라 호출이 주도하는 반복·생성을 담당합니다.

### 정의

다른 형태와 똑같이, macro 는 `#[impl_macro(name)]`, 가시성, `@macro` 시작자, 그리고 블록입니다. 블록은 평평한 템플릿 대신 하나 이상의 **규칙** 을 담습니다. 각 규칙은 괄호 안의 매처, `=>` ([`match`](../003-expressions/312-match-expression.md) 의 arm 과 같은 화살표), 중괄호 안의 템플릿입니다.

```rok
#[impl_macro(name)] pub @macro {
    ( <matcher> ) => { <template> }
    ( <matcher> ) => { <template> }
    ..
}
```

`#[impl_macro(name)]` 는 `#[impl_derive(T)]` 가 derive `T` 를 등록하는 것과 똑같이 매크로 `name!` 을 등록합니다. 호출 시 규칙을 위에서 아래로 시도해 매처가 처음 맞는 규칙이 확장됩니다. 하나도 안 맞으면 그 호출 위치에서 컴파일 에러입니다. macro 의 이름은 derive·attribute 처럼 `use` 없이 쓰는 빌드 전역 이름이며 ([6.8.6](#686-pub--가시성)), 모든 호출의 끝 `!` 가 함수가 아니라 매크로임을 표시합니다.

```rok
#[impl_macro(twice)] pub @macro {
    ( @e: expr ) => { { let v = @e; (v, v) } }
}

let p = twice!(compute());      // (그 값, 같은 값)
```

### 호출

macro 는 `name!( .. )`, `name![ .. ]`, `name!{ .. }` 로 호출합니다. 구분자는 서로 바꿔 써도 의미가 같습니다. 인자는 임의의 **토큰 트리** 입니다 — `()` `[]` `{}` 가 균형 잡힌 모든 토큰이라, 렉서와 포매터는 매크로를 몰라도 호출을 토큰 단위로 읽습니다. macro 는 다음 위치에서, 템플릿이 내는 것에 따라 확장됩니다.

| 위치   | 예                                        | 템플릿이 내는 것      |
|--------|-------------------------------------------|-----------------------|
| 식     | `let xs = grid![1, 2; 3, 4];`             | 식                    |
| 문     | `debug!("here");`                         | 0개 이상의 문         |
| 항목   | `int_impl!(i8, i16, i32);` (모듈 수준)    | 0개 이상의 항목       |
| 타입   | `let f: fn_ptr!(i32, i32) = ..;`          | 타입                  |
| 패턴   | `match x { pair!(a, b) => .. }`           | 패턴                  |

### 매처와 fragment

매처는 토큰 패턴입니다. 매처 안의 리터럴 토큰은 호출에 그대로 나타나야 하고, **메타변수** `@name: <fragment>` 는 호출의 일부를 템플릿용으로 캡처합니다. fragment 는 얼마만큼을 무엇으로 파싱할지 정합니다 — `expr`, `stmt`, `block`, `ty`, `pat`, `path`, `ident`, `literal`, `item`, 또는 `tt` (원시 토큰 트리 하나). `tt` 를 뺀 fragment 는 그 문법의 실제 구문 노드로 파싱되므로 `@e: expr` 의 인자는 타입을 갖고 검사되며 언어 서버가 그 안의 이름을 해석합니다 ([6.8.12](#도구-지원과-위생)). `tt` 만은 rok 문법이 아닌 DSL 을 위해 파싱 없이 캡처되며, 그래도 균형은 유지합니다.

```rok
#[impl_macro(clamp)] pub @macro {
    ( @x: expr , @lo: expr , @hi: expr ) => {
        { let v = @x; if v < @lo { @lo } else if v > @hi { @hi } else { v } }
    }
}
```

### 반복

`@rep` 은 하위 패턴을 0회 이상 매칭하고, `@for` ([6.9.3](609-template-tokens.md#693-필드-반복) 의 `@for_fields` 와 같은 반복자, 여기서는 캡처한 시퀀스에 적용) 는 원소마다 한 번씩 전사합니다.

```rok
#[impl_macro(list)] pub @macro {
    ( @rep[sep = ","] { @x: expr } ) => {
        { let acc = List::new(); @for @x { acc.push(@x); } acc }
    }
}

let xs = list![a, b, c];        // acc.push(a); acc.push(b); acc.push(c);
```

`@rep[sep = ","] { <하위 패턴> }` 은 하위 패턴을 `sep` 토큰으로 구분해 반복 매칭합니다 (`sep` 은 선택, `trailing` 이면 끝에 구분자 하나 허용). `@rep` 안에서 캡처한 메타변수는 시퀀스를 바인딩합니다. `@for @x { <본체> }` 는 원소마다 본체를 냅니다. 한 `@rep` 의 여러 메타변수는 lockstep 으로 돌며 길이가 같아야 합니다. `@rep` 과 `@for` 는 중첩됩니다. `@apply_if` ([6.9.5](609-template-tokens.md)) 는 macro 템플릿에서도 똑같이 동작합니다.

### 이름 합성

`@concat(..)` 은 이름 토큰과 문자열 리터럴을 하나의 식별자로 이어, macro 가 서로 다른 이름의 항목을 생성하게 합니다.

```rok
#[impl_macro(accessors)] pub @macro {
    ( @rep[sep = ","] { @f: ident : @t: ty } ) => {
        @for @f { fn @concat(get_, @f)(self) -> @t { self.@f } }
    }
}
// accessors!{ x: i32, y: i32 } -> fn get_x(self) -> i32 { self.x }  (그리고 get_y)
```

`@concat` 이 만든 이름은 **기록** 됩니다 — 컴파일러가 생성된 항목을 그 생성 이름으로 인덱싱하므로 심볼 검색·정의로 이동·참조 찾기가 도달합니다. `@concat` 은 정의나 경로 위치에서만 식별자를 만들며, 조각은 이름 토큰과 문자열 리터럴이고 임의 구문을 조립할 수는 없습니다.

### 라이브러리의 보일러플레이트

macro 의 전형적 이유는 작성자가 적은 고정 목록에 정의를 반복하는 것 — attribute 로 트리거되는 형태로는 할 수 없는 일입니다. [`rok::core::num`](../../refs-standard-libraries/200-core/README.md) 의 손으로 복사한 타입별 impl 을 대체합니다.

```rok
#[impl_macro(nonzero_div)] pub @macro {
    ( @rep[sep = ","] { @t: ty } ) => {
        @for @t {
            impl Div<NonZero<@t>> for @t {
                type Output = @t;
                const fn div(self, rhs: NonZero<@t>) -> @t { self / rhs.0 }
            }
        }
    }
}

nonzero_div!(i8, i16, i32, i64, u8, u16, u32, u64);
```

### 재귀

템플릿은 자신을 포함해 매크로를 호출할 수 있고, 매크로 호출이 남지 않을 때까지 확장이 반복됩니다. 한계가 있습니다 — 깊이가 한도 (256) 를 넘으면 그 호출을 가리키는 컴파일 에러라, 끝나지 않는 매크로는 멈추지 않고 실패합니다. 상호 재귀도 같은 깊이에 셈하며, attribute 형태와 마찬가지로 순환은 경로와 함께 보고됩니다 ([6.8.8](#688-fixed-point-확장)).

### 내장 `!`-폼은 예약

`cfg!` ([6.3](603-cfg.md)) 와 `include_str!` / `include_bytes!` ([6.11](611-include.md)) 은 `!` 로 끝나지만 고정된 컴파일러 내장폼이지 매크로가 아니며, 재정의·가림·매크로 이름으로 쓰기가 불가합니다. 컴파일러가 제공하는 `!`-폼은 이들뿐이고, 그 밖의 모든 `name!` 은 rok 으로 정의한 매크로입니다.

### <a name="도구-지원과-위생"></a>도구 지원과 위생

macro 는 모든 메타 정의의 보장 ([6.8.9](#689-토큰-스트림-조작-없음)) 을 지키며, 호출 위치가 더하는 세부는 다음과 같습니다.

- **위생** 은 자동입니다. 템플릿이 들여온 이름은 새 이름이라 호출 위치를 포획하거나 포획되지 않고, 메타변수로 넘어온 이름은 호출 위치에서 해석됩니다.
- **span**: 확장된 모든 토큰이 호출의 span 을 지녀, 진단은 확장을 note 로 보이며 호출을 가리킵니다.
- **해석**: fragment 가 실제 노드로 파싱되고 합성 이름이 인덱싱되므로, 호버·정의로 이동·참조 찾기가 매크로를 관통합니다 — 인자 안으로, 그리고 생성한 항목 위로.
- **요청 시 확장**: 언어 서버가 호출을 그 자리에서 펼칠 수 있습니다 ([9.9.13](../009-tools/909-debug-tools.md)).
- **포매팅**: 포매터는 호출 인자가 rok fragment 이면 포맷하고, 원시 `tt` DSL 은 균형 구분자 안에서 쓴 그대로 둡니다.

## 6.8.13. 정리

이 네 형태 — derive, attribute, 필드별 derive attribute, 그리고 호출로 트리거되는 macro — 가 하나의 템플릿 엔진 위에서 rok 의 전체 메타프로그래밍 표면을 구성합니다. 절차적 매크로만큼의 표현력은 없지만, derive·attribute·필드별 attribute·호출부 생성이라는 핵심 사용 사례를 모두, 각각 위생적이고 도구가 따라갈 수 있는 형태로 다룹니다.

[템플릿 토큰과 확장](609-template-tokens.md) 에서는 템플릿 토큰의 정확한 목록 — 타입 주도 입력과 호출 주도 입력 둘 다 — 과 확장 메커니즘을 정의합니다.
