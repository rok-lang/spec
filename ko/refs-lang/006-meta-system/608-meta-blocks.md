# 6.8. 사용자 정의 메타 블록

`#[impl_derive(...)]` / `#[impl_attribute(...)]` / `#[impl_derive_attribute(...)]` 세 형태는
사용자가 자신의 derive 또는 attribute 를 정의하는 코드 생성기다. Rust 의 proc macro 를 대체한다.

```
meta_block      ::= meta_attr visibility? meta_kind block
meta_attr       ::= "#" "[" meta_kind_attr "(" meta_args ")" "]"
meta_kind_attr  ::= "impl_derive" | "impl_attribute" | "impl_derive_attribute"
meta_kind       ::= "@derive" | "@attribute"
meta_args       ::= identifier ("," identifier)*
```

## 6.8.1. 세 종류

```nok
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

| 형태                                    | 용도                                 |
| --------------------------------------- | ------------------------------------ |
| `#[impl_derive(TraitName)]`             | 사용자 정의 derive                   |
| `#[impl_attribute(name)]`               | 사용자 정의 attribute (item wrapper) |
| `#[impl_derive_attribute(Trait, attr)]` | 필드별 derive attribute              |

## 6.8.2. `#[impl_derive(TraitName)]` 사용자 정의 derive

```nok
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
// → 자동 생성됨:
//   impl MyHash for Point {
//       fn my_hash(self) -> u64 {
//           let h: u64 = 0u64;
//           h = combine_hash(h, self.x.my_hash());
//           h = combine_hash(h, self.y.my_hash());
//           h
//       }
//   }
```

- `#[impl_derive(이름)]` 와 본체 (템플릿 블록) 로 구성된다.
- 사용처에서 `#[derive(이름)]` 가 등장하면 템플릿이 그 타입에 대해 확장되어 코드가 생성된다.

## 6.8.3. `#[impl_attribute(name)]` 사용자 정의 attribute

```nok
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
// → 자동 생성됨:
//   fn process(x: i32) -> i32 {
//       let start = now();
//       let result = { expensive(x) };
//       log(f"process took {now() - start}");
//       result
//   }
```

- 항목을 감싸는 wrapper 다.
- 원본 항목 (`@item`) 의 내용과 시그니처를 재사용해 변환된 형태를 생성한다.
- 함수뿐 아니라 `struct`, `enum`, `impl` 에도 부착할 수 있다 (대상 제한은 `@accept` 로 표시).

## 6.8.4. `#[impl_derive_attribute(Trait, attr)]` 필드별 derive attribute

```nok
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
// → 자동 생성:
//   impl Serialize for User {
//       fn serialize(self) -> Json {
//           let obj = JsonObj::new();
//           obj.set("user_name", self.name.serialize());
//           obj.set("age", self.age.serialize());
//           obj.into()
//       }
//   }
```

- 두 인자를 받는다: Trait (derive 의 대상) 와 attribute (필드별 정보 표시).
- derive 의 본체 안에서 필드의 attribute 정보를 읽어 분기한다.

## 6.8.5. 본체 (템플릿 블록)

세 형태 모두 본체가 템플릿 블록이다. 본체 안에 일반 nok 코드와 템플릿 토큰 (`@Self`,
`@field`, `@for_fields` 등) 이 함께 들어간다.

자세한 템플릿 토큰 목록은 [템플릿 토큰과 확장](609-template-tokens.md) 에서 정의한다.

## 6.8.6. `pub` / 가시성

```nok
pub @derive { ... }
pub(pkg) @attribute { ... }
@derive { ... }                  // 비공개
```

가시성 키워드는 템플릿 블록 자체에 부착한다. 비공개면 같은 모듈 안에서만 사용할 수 있다.

## 6.8.7. 본체 시작자 (`@derive` / `@attribute`)

본체 시작자가 템플릿의 종류를 표시한다.

| 시작자       | 용도                                                 |
| ------------ | ---------------------------------------------------- |
| `@derive`    | derive 정의 (`impl_derive`, `impl_derive_attribute`) |
| `@attribute` | attribute 정의 (`impl_attribute`)                    |

이 시작자가 어떤 템플릿 토큰이 본체 안에서 의미를 갖는지 결정한다 (예: `@item` 은 attribute
전용).

## 6.8.8. fixed-point 확장

템플릿 확장 결과가 또 다른 derive/attribute 가 부착된 항목을 만들면 재귀적으로 확장된다.

```nok
#[impl_attribute(adds_helper)] pub @attribute {
    @item                                       // 원본 항목 그대로

    #[derive(MyHelper)]                          // 새 항목 — 다시 derive 처리됨
    struct Helper { ... }
}
```

위 attribute 가 적용되면 본체의 `#[derive(MyHelper)]` 가 다음 확장 패스에서 다시 처리된다.

확장이 고정점 (fixed-point) 에 도달하면 종료된다. 더 이상 확장할 derive 나 attribute 가 없을 때다.

순환 확장 (A 가 B 를 호출하고 B 가 A 를 호출) 은 컴파일러가 검출하여 ERROR 를 낸다.

## 6.8.9. 토큰 스트림 조작 없음

위 세 메타 블록은 템플릿 토큰만으로 코드를 생성한다. Rust 의 `syn` / `quote` 같은 임의 토큰
변환은 nok 에 없다.

이 제약은 메타 블록의 출력이 컴파일러가 이해할 수 있는 nok 코드임을 보장하기 위함이다. 도구 친화성과 디버깅 가능성을 동시에 확보한다.

## 6.8.10. proc-macro 와의 비교

| 측면   | Rust proc macro           | nok 메타 블록                    |
| ------ | ------------------------- | -------------------------------- |
| 입력   | 임의 토큰 스트림          | 항목의 구조 정보 (필드, 변형 등) |
| 출력   | 임의 토큰 스트림          | nok 코드와 템플릿 토큰           |
| 위생   | proc macro 의 결정에 따름 | 자동 (hygienic)                  |
| 디버깅 | 출력 토큰 확인            | 확장된 AST 확인                  |
| 표현력 | 매우 높음                 | 제한적                           |

nok 의 선택은 표현력을 제한하는 대신 예측 가능성과 디버깅 용이성을 얻는 것이다.

## 6.8.11. 등록/가시성과의 결합

`#[impl_derive(TraitName)]` 가 부착된 메타 블록은 `TraitName` 이라는 derive 매핑을 현재
모듈의 가시 범위에 등록한다.

다른 모듈에서 그 derive 를 쓰려면 메타 블록이 보이는 위치에 있거나 import 되어 있어야 한다.

```nok
// crate::derives
pub @derive {
    #[impl_derive(MyHash)]
    impl MyHash for @Self { ... }
}

// 다른 모듈
use crate::derives::*;     // MyHash derive 사용 가능
#[derive(MyHash)]
struct Foo { ... }
```

## 6.8.12. 정리

세 메타 블록 형태가 nok 의 전체 메타프로그래밍 표면이다. proc macro 만큼의 표현력은 없지만,
derive, attribute, 필드별 attribute 의 핵심 사용 사례를 모두 다룬다.

[템플릿 토큰과 확장](609-template-tokens.md) 에서 템플릿 토큰의 정확한 목록과 확장 메커니즘을 정의한다.
