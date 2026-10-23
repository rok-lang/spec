# 6.8. 사용자 정의 메타 블록

`#[impl_derive(...)]`, `#[impl_attribute(...)]`, `#[impl_derive_attribute(...)]` 세 형태는 사용자가 자신의 derive 또는 attribute 를 정의할 수 있도록 해주는 코드 생성기이며, Rust 의 proc macro 를 대체하는 역할을 합니다.

```
meta_block      ::= meta_attr visibility? meta_kind block
meta_attr       ::= "#" "[" meta_kind_attr "(" meta_args ")" "]"
meta_kind_attr  ::= "impl_derive" | "impl_attribute" | "impl_derive_attribute"
meta_kind       ::= "@derive" | "@attribute"
meta_args       ::= identifier ("," identifier)*
```

## 6.8.1. 세 종류

세 가지 형태는 다음과 같이 표현됩니다.

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

세 형태 모두 본체가 템플릿 블록이며, 본체 안에는 일반 rok 코드와 템플릿 토큰 (`@Self`, `@field`, `@for_fields` 등) 이 함께 들어갑니다.

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

이 시작자가 어떤 템플릿 토큰이 본체 안에서 의미를 갖는지 결정합니다. 예를 들어 `@item` 은 attribute 전용입니다.

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

위의 세 메타 블록은 오직 템플릿 토큰만으로 코드를 생성합니다. Rust 의 `syn` 이나 `quote` 같은 임의 토큰 변환 기능은 rok 에 존재하지 않습니다.

이러한 제약은 메타 블록의 출력이 컴파일러가 이해할 수 있는 rok 코드임을 보장하기 위한 것이며, 도구 친화성과 디버깅 가능성을 동시에 확보합니다.

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

`#[impl_derive(TraitName)]` 가 부착된 메타 블록은 `TraitName` 이라는 derive 를, `#[impl_attribute(name)]` 는 `#[name]` attribute 를 등록합니다. 컴파일러는 parse 전에 라이브러리와 사용자 패키지의 모든 파일에서 메타 블록을 모으므로, 정의 위치나 파일 순서와 무관하게 쓸 수 있습니다. 메타 블록에 붙은 `#[cfg]` 가 거짓이면 그 블록은 등록되지 않습니다 ([6.9.8](609-template-tokens.md#cfg-와의-관계)).

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

- 같은 derive 또는 attribute 이름을 두 메타 블록이 등록하는 것.
- 내장 derive ([6.2.1](602-built-in-attributes.md): `Clone`, `Copy`, `Default`, `Debug`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`) 나 내장 attribute (`cfg`, `derive`, `test`, `inline`, `repr`, `keyword` 등 6.2 ~ 6.7 의 속성) 의 이름을 메타 블록이 다시 정의하는 것.

## 6.8.12. 정리

이 세 가지 메타 블록 형태가 rok 의 전체 메타프로그래밍 표면을 구성합니다. proc macro 만큼의 표현력은 없지만, derive, attribute, 그리고 필드별 attribute 라는 핵심 사용 사례를 모두 다루고 있습니다.

[템플릿 토큰과 확장](609-template-tokens.md) 에서는 템플릿 토큰의 정확한 목록과 확장 메커니즘을 정의합니다.
