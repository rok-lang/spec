# 6.8. 사용자 정의 메타 정의

**메타 정의(meta definition)**는 사용자 정의 코드 생성기입니다. 형태는 여럿이지만 메커니즘은 하나입니다. 정의는 입력의 일부를 바인딩하고, Rok 코드와 템플릿 토큰([6.9](609-template-tokens.md))으로 된 템플릿을 위생적으로([6.8.9](#689-토큰-스트림-조작-없음)), 고정점에 이를 때까지([6.8.8](#688-fixed-point-확장)) 전사합니다. 형태끼리 다른 점은 **무엇으로 트리거되고 어떤 입력을 바인딩하는가**뿐입니다. 형태는 네 가지이며, 앞의 셋은 항목에 붙인 attribute로, 넷째는 호출로 트리거됩니다.

| 형태                                    | 정의                                 | 트리거                 | 바인딩                                  |
|-----------------------------------------|--------------------------------------|------------------------|-----------------------------------------|
| derive                                  | `#[impl_derive(T)] @derive {}`       | 타입에 `#[derive(T)]`  | 타입의 shape (`@Self`, `@for_fields`)   |
| attribute                               | `#[impl_attribute(n)] @attribute {}` | 항목에 `#[n]`          | 항목 (`@item`)                          |
| 필드별 derive attribute                 | `#[impl_derive_attribute(T, a)] @derive {}` | `#[derive(T)]` + 필드 `#[a]` | shape + 필드 attribute          |
| **macro**                               | `#[impl_macro(n)] @macro { (m) => {t} }` | `n!(..)` 호출      | 호출의 토큰 (`@x: expr`, `@rep`)        |

앞의 셋은 *작성자가 붙인 타입이나 항목에서* 코드를 생성하고, macro는 *호출자가 쓴 내용에서* 코드를 생성합니다. 다른 형태로는 채울 수 없는 수요입니다([6.8.12](#6812-macro-형태)). 네 형태는 템플릿 엔진, `@` 토큰, 위생, 아래에서 설명할 도구 보장을 공유합니다. 바인딩하는 입력이 다른 부분은 이 장과 [6.9](609-template-tokens.md)에서 따로 짚습니다. 이 넷이 Rok 메타프로그래밍의 전부입니다. 넷이 함께 Rust의 derive·attribute·함수형 매크로를 대신하며, 절차적 코드나 원시 토큰 조작은 없습니다.

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

attribute로 트리거되는 세 형태는 다음과 같습니다.

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

각 형태의 용도는 다음과 같습니다.

| 형태                                    | 용도                                 |
|-----------------------------------------|--------------------------------------|
| `#[impl_derive(TraitName)]`             | 사용자 정의 derive                   |
| `#[impl_attribute(name)]`               | 사용자 정의 attribute (item wrapper) |
| `#[impl_derive_attribute(Trait, attr)]` | 필드별 derive attribute              |

호출로 트리거되는 넷째 형태 `macro`는 [6.8.12](#6812-macro-형태)에서 정의합니다.

## 6.8.2. `#[impl_derive(TraitName)]` 사용자 정의 derive

사용자 정의 derive의 예입니다.

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

이 형태는 `#[impl_derive(이름)]`와 본체(템플릿 블록)로 이루어집니다. 사용처에 `#[derive(이름)]`가 있으면 템플릿을 그 타입에 대해 확장해 코드를 생성합니다.

## 6.8.3. `#[impl_attribute(name)]` 사용자 정의 attribute

사용자 정의 attribute의 예입니다.

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

이 형태는 항목을 감싸는 wrapper입니다. 원본 항목(`@item`)의 내용과 시그니처를 재사용해 변환된 항목을 생성합니다. 함수뿐 아니라 `struct`, `enum`, `impl`에도 붙일 수 있으며, 붙일 수 있는 대상은 `@accept`로 제한합니다.

## 6.8.4. `#[impl_derive_attribute(Trait, attr)]` 필드별 derive attribute

필드별 derive attribute의 예입니다.

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

이 형태는 인자 두 개를 받습니다. 하나는 derive할 Trait이고, 다른 하나는 필드별 정보를 표시하는 attribute입니다. derive 본체 안에서 필드의 attribute 정보를 읽어 분기합니다.

## 6.8.5. 본체 (템플릿 블록)

모든 형태에서 본체는 템플릿 블록입니다. 본체에는 일반 Rok 코드와 템플릿 토큰(attribute로 트리거되는 형태는 `@Self`, `@field`, `@for_fields`, macro는 `@x`, `@rep`, `@for`, `@concat`, [6.8.12](#6812-macro-형태))이 함께 들어갑니다.

템플릿 토큰의 전체 목록은 [템플릿 토큰과 확장](609-template-tokens.md)에서 정의합니다.

## 6.8.6. `pub` / 가시성

가시성은 다음과 같이 표기합니다.

```rok
pub @derive { ... }
pub(pkg) @attribute { ... }
@derive { ... }                  // 비공개
```

가시성 키워드는 템플릿 블록에 붙입니다. 효과는 다음과 같습니다.

| 가시성                  | 쓸 수 있는 곳                                     |
|-------------------------|---------------------------------------------------|
| `pub` (`pub(pkg)` 포함) | 빌드 전체. `use` 없이 `#[derive(X)]` / `#[x]`로 씀 |
| 비공개                  | 메타 블록을 정의한 파일 안                        |

derive와 attribute 이름은 모듈 경로가 없는 빌드 전역 이름입니다(6.8.11). 메타 블록이나 키워드 핸들러의 `#[cfg]`는 그것을 정의한 패키지의 cfg(그 패키지 자신의 feature, [9.10.4c](../009-tools/910-rok-frontend.md#9104c-features))로 평가합니다.

## 6.8.7. 본체 시작자 (`@derive` / `@attribute`)

본체 시작자는 템플릿의 종류를 나타냅니다.

| 시작자       | 용도                                                 |
|--------------|------------------------------------------------------|
| `@derive`    | derive 정의 (`impl_derive`, `impl_derive_attribute`) |
| `@attribute` | attribute 정의 (`impl_attribute`)                    |
| `@macro`     | macro 정의 (`impl_macro`); 블록이 규칙들을 담음 ([6.8.12](#6812-macro-형태)) |

본체 안에서 어떤 템플릿 토큰이 의미를 갖는지는 시작자가 정합니다. 예를 들어 `@item`은 attribute 전용이고, 매처 토큰(`@x: expr`, `@rep`)은 `@macro` 전용입니다.

## 6.8.8. fixed-point 확장

템플릿 확장 결과로 derive나 attribute가 붙은 항목이 또 생기면, 그 항목도 재귀적으로 확장합니다.

```rok
#[impl_attribute(adds_helper)] pub @attribute {
    @item                                       // 원본 항목 그대로

    #[derive(MyHelper)]                          // 새 항목: 다시 derive 처리됨
    struct Helper { ... }
}
```

위 attribute를 적용하면 본체의 `#[derive(MyHelper)]`를 다음 확장 패스에서 다시 처리합니다.

확장은 더 확장할 derive나 attribute가 남지 않은 고정점(fixed-point)에 이르면 끝납니다.

순환 확장(A가 B를 호출하고 B가 다시 A를 호출하는 경우)은 컴파일러가 찾아내 에러로 보고합니다. 메시지에는 순환 경로(`A → B → A`)가 나옵니다.

## 6.8.9. 토큰 스트림 조작 없음

모든 메타 정의(attribute로 트리거되는 세 형태와 macro)는 템플릿 토큰만으로 코드를 생성합니다. Rust의 `syn`이나 `quote` 같은 임의의 토큰 변환 기능도, 컴파일 타임에 코드를 실행하는 절차적 형태도 Rok에는 없습니다.

이 제약은 메타 정의의 출력이 컴파일러가 이해할 수 있는 Rok 코드임을 보장합니다. 그래서 도구가 따라가기 쉽고 디버깅도 쉽습니다.

## 6.8.10. proc-macro 와의 비교

Rust의 proc macro와 Rok 메타 블록을 비교하면 다음과 같습니다.

| 측면   | Rust proc macro           | Rok 메타 블록                    |
|--------|---------------------------|----------------------------------|
| 입력   | 임의 토큰 스트림          | 항목의 구조 정보 (필드, 변형 등) |
| 출력   | 임의 토큰 스트림          | Rok 코드와 템플릿 토큰           |
| 위생   | proc macro 의 결정에 따름 | 자동 (hygienic)                  |
| 디버깅 | 출력 토큰 확인            | 확장된 AST 확인                  |
| 표현력 | 매우 높음                 | 제한적                           |

Rok은 표현력을 제한하는 대신 예측 가능성과 디버깅 용이성을 택했습니다.

## 6.8.11. 등록/가시성과의 결합

`#[impl_derive(TraitName)]`가 붙은 메타 블록은 `TraitName`이라는 derive를, `#[impl_attribute(name)]`는 `#[name]` attribute를, `#[impl_macro(name)]`는 `name!` 매크로를 등록합니다. 컴파일러는 파싱 전에 라이브러리와 사용자 패키지의 모든 파일에서 메타 블록을 모으므로, 정의 위치나 파일 순서와 상관없이 쓸 수 있습니다. 메타 블록에 붙은 `#[cfg]`가 거짓이면 그 블록은 등록되지 않습니다([6.9.9](609-template-tokens.md#cfg-와의-관계)).

`pub` 메타 블록은 `use` 없이 빌드 전체에서 보입니다(6.8.6). 메타 블록은 이름 공간의 항목이 아니므로 `use`로 가져올 수도 없습니다.

```rok
// pkg::derives
#[impl_derive(MyHash)] pub @derive {
    impl MyHash for @Self { ... }
}

// 다른 모듈: import 없이 사용
#[derive(MyHash)]
struct Foo { ... }
```

다음은 에러입니다.

- 두 메타 블록이 같은 derive·attribute·macro 이름을 등록하는 것(macro 하나는 모든 규칙을 담은 `@macro` 블록 하나입니다).
- 메타 블록이 내장 derive([6.2.1](602-built-in-attributes.md): `Clone`, `Copy`, `Default`, `Debug`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`)나 내장 attribute(`cfg`, `derive`, `test`, `inline`, `repr`, `keyword` 등 6.2 ~ 6.7의 속성)의 이름을 다시 정의하는 것.

## 6.8.12. macro 형태

**macro**는 호출로 트리거되는 메타 정의입니다. derive나 attribute는 자신이 붙은 타입이나 항목에서 코드를 생성하지만, macro는 `name!(..)`처럼 *호출*되어 그 자리에서 코드로 확장됩니다. 코드는 호출자가 쓴 내용에서 생성합니다. attribute로 트리거되는 형태로는 할 수 없는 일, 곧 타입의 필드가 아니라 호출이 주도하는 반복과 생성을 맡습니다.

### 정의

다른 형태와 마찬가지로 macro는 `#[impl_macro(name)]`, 가시성, `@macro` 시작자, 블록으로 이루어집니다. 블록에는 평평한 템플릿 대신 **규칙**이 하나 이상 들어갑니다. 각 규칙은 괄호 안의 매처, `=>`([`match`](../003-expressions/312-match-expression.md)의 arm과 같은 화살표), 중괄호 안의 템플릿으로 이루어집니다.

```rok
#[impl_macro(name)] pub @macro {
    ( <matcher> ) => { <template> }
    ( <matcher> ) => { <template> }
    ..
}
```

`#[impl_macro(name)]`는 `#[impl_derive(T)]`가 derive `T`를 등록하는 것과 똑같이 매크로 `name!`을 등록합니다. 호출하면 규칙을 위에서 아래로 시도하고, 매처가 처음으로 맞는 규칙을 확장합니다. 맞는 규칙이 없으면 그 호출 위치에서 컴파일 에러입니다. macro 이름은 derive·attribute처럼 `use` 없이 쓰는 빌드 전역 이름입니다([6.8.6](#686-pub--가시성)). 모든 호출 끝의 `!`는 그것이 함수가 아니라 매크로임을 표시합니다.

```rok
#[impl_macro(twice)] pub @macro {
    ( @e: expr ) => { { let v = @e; (v, v) } }
}

let p = twice!(compute());      // (그 값, 같은 값)
```

### 호출

macro는 `name!( .. )`, `name![ .. ]`, `name!{ .. }`로 호출합니다. 어느 구분자를 써도 의미가 같습니다. 인자는 임의의 **토큰 트리**, 곧 `()` `[]` `{}`의 짝이 맞는 토큰 열이므로, 렉서와 포매터는 매크로를 몰라도 호출을 토큰 단위로 읽을 수 있습니다. macro는 다음 위치에서 확장되며, 템플릿이 내놓아야 하는 것은 위치에 따라 다릅니다.

| 위치   | 예                                        | 템플릿이 내는 것      |
|--------|-------------------------------------------|-----------------------|
| 식     | `let xs = grid![1, 2; 3, 4];`             | 식                    |
| 문     | `debug!("here");`                         | 0개 이상의 문         |
| 항목   | `int_impl!(i8, i16, i32);` (모듈 수준)    | 0개 이상의 항목       |
| 타입   | `let f: fn_ptr!(i32, i32) = ..;`          | 타입                  |
| 패턴   | `match x { pair!(a, b) => .. }`           | 패턴                  |

### 매처와 fragment

매처는 토큰 패턴입니다. 매처 안의 리터럴 토큰은 호출에 그대로 나와야 하고, **메타변수** `@name: <fragment>`는 호출의 일부를 템플릿에서 쓰도록 캡처합니다. fragment는 토큰을 얼마나 캡처할지 정합니다. `ident`는 식별자 하나, `literal`은 리터럴 토큰 하나, `tt`는 토큰 트리 하나(토큰 하나 또는 짝이 맞는 묶음)를 받습니다. `expr`, `stmt`, `block`, `ty`, `pat`, `path`, `item`은 괄호 깊이 0에서 매처의 다음 리터럴 토큰까지(또는 호출 끝까지)의 비어 있지 않은 토큰 열을 받습니다. 모든 fragment는 원시 토큰으로 캡처하며, 매칭할 때 파싱하지 않습니다. 전사를 마친 확장 결과 전체를 파싱하므로, 템플릿이 놓은 자리에 맞지 않는 인자는 확장 결과의 구문 에러가 되어 호출 위치에 보고됩니다.

```rok
#[impl_macro(clamp)] pub @macro {
    ( @x: expr , @lo: expr , @hi: expr ) => {
        { let v = @x; if v < @lo { @lo } else if v > @hi { @hi } else { v } }
    }
}
```

### 반복

`@rep`은 하위 패턴을 0회 이상 매칭하고, `@for`는 원소마다 한 번씩 전사합니다. 이 반복자는 [6.9.3](609-template-tokens.md#693-필드-반복)의 `@for_fields`와 같은 반복자를 캡처한 시퀀스에 적용한 것입니다.

```rok
#[impl_macro(list)] pub @macro {
    ( @rep[sep = ","] { @x: expr } ) => {
        { let acc = List::new(); @for @x { acc.push(@x); } acc }
    }
}

let xs = list![a, b, c];        // acc.push(a); acc.push(b); acc.push(c);
```

`@rep[sep = ","] { <하위 패턴> }`은 하위 패턴을 `sep` 토큰으로 구분해 반복 매칭합니다(`sep`은 선택이며, `trailing`이면 끝에 구분자 하나를 허용). `@rep` 안에서 캡처한 메타변수는 시퀀스를 바인딩합니다. `@for @x { <본체> }`는 원소마다 본체를 하나씩 냅니다. 한 `@rep`의 메타변수 여러 개는 lockstep으로 함께 돌며, 길이가 같아야 합니다. `@rep`과 `@for`는 중첩할 수 있습니다. shape 토큰(`@Self`, `@field`, `@apply_if` 등)은 타입이 주도하는 형태의 토큰이며 macro의 토큰이 아닙니다. macro에는 검사할 shape가 없으므로, 쓰면 에러입니다.

### 이름 합성

`@concat(..)`은 이름 토큰과 문자열 리터럴을 이어 식별자 하나를 만듭니다. macro는 이를 써서 이름이 서로 다른 항목을 생성할 수 있습니다.

```rok
#[impl_macro(accessors)] pub @macro {
    ( @rep[sep = ","] { @f: ident : @t: ty } ) => {
        @for @f { fn @concat(get_, @f)(self) -> @t { self.@f } }
    }
}
// accessors!{ x: i32, y: i32 } -> fn get_x(self) -> i32 { self.x }  (그리고 get_y)
```

`@concat`이 만든 이름은 **기록**됩니다. 컴파일러가 생성된 항목을 생성된 이름으로 인덱싱하므로, 심볼 검색, 정의로 이동, 참조 찾기로 그 항목에 닿을 수 있습니다. `@concat`은 정의 위치나 경로 위치에서만 식별자를 만듭니다. 조각은 이름 토큰과 문자열 리터럴뿐이며, 임의의 구문을 조립할 수는 없습니다.

### 라이브러리의 보일러플레이트

macro를 쓰는 대표적인 이유는 작성자가 적은 고정 목록에 대해 정의를 반복하는 것입니다. attribute로 트리거되는 형태로는 할 수 없는 일입니다. [`rok::core::num`](../../refs-standard-libraries/200-core/README.md)에서 손으로 복사하던 타입별 impl을 이것으로 대신합니다.

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

템플릿은 다른 매크로를 호출할 수 있고, 그 호출도 이어서 확장됩니다. 매크로는 자기 자신을 호출할 수 없습니다. 자기 확장 안에서 자신을 호출하면 직접이든 다른 매크로를 거치든 바로 거부되며, 그 호출을 가리키는 컴파일 에러(`` `name!` expands too deeply or cyclically ``)가 됩니다. 서로 다른 매크로의 중첩 확장은 256단계까지이며, 넘으면 같은 에러입니다. 목록을 반복하려면 재귀 대신 `@rep` / `@for`를 쓰세요.

### 내장 `!`-폼은 예약

`cfg!`([6.3](603-cfg.md))와 `include_str!` / `include_bytes!`([6.11](611-include.md))는 `!`로 끝나지만 매크로가 아니라 고정된 컴파일러 내장 형태입니다. 다시 정의하거나, 가리거나, 매크로 이름으로 쓸 수 없습니다. 컴파일러가 제공하는 `!`-폼은 이것들뿐이며, 그 밖의 모든 `name!`은 Rok으로 정의한 매크로입니다.

### 도구 지원과 위생

macro는 모든 메타 정의가 지키는 보장([6.8.9](#689-토큰-스트림-조작-없음))을 지킵니다. 호출 위치 때문에 더해지는 세부 사항은 다음과 같습니다.

- **위생**은 자동입니다. 템플릿이 들여온 이름은 새 이름이므로 호출 위치의 이름을 포획하지도, 호출 위치에 포획되지도 않습니다. 메타변수로 넘어온 이름은 호출 위치에서 해석됩니다.
- **span**: 확장된 모든 토큰이 호출의 span을 가지므로, 진단은 호출을 가리키고 확장 내용은 note로 보여 줍니다.
- **해석**: 확장 결과를 보통 코드처럼 파싱하고 합성된 이름을 인덱싱하므로, 호버, 정의로 이동, 참조 찾기가 매크로가 생성한 항목에 닿습니다.
- **요청 시 확장**: 언어 서버는 호출을 그 자리에서 펼쳐 보여 줄 수 있습니다([9.9.13](../009-tools/909-debug-tools.md)).
- **포매팅**: 포매터는 호출 인자가 Rok fragment이면 포맷하고, 원시 `tt` DSL은 짝이 맞는 구분자 안에 쓴 그대로 둡니다.

## 6.8.13. 정리

네 형태(derive, attribute, 필드별 derive attribute, 호출로 트리거되는 macro)는 하나의 템플릿 엔진 위에서 Rok 메타프로그래밍의 전부를 이룹니다. 절차적 매크로만큼 표현력이 크지는 않지만, derive, attribute, 필드별 attribute, 호출부 생성이라는 주요 사용 사례를 모두 다룹니다. 어느 것이든 위생적이고 도구가 따라갈 수 있습니다.

[템플릿 토큰과 확장](609-template-tokens.md)은 템플릿 토큰의 정확한 목록(타입이 주도하는 입력과 호출이 주도하는 입력 모두)과 확장 메커니즘을 정의합니다.
