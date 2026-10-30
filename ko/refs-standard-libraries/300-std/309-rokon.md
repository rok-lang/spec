# 3.9. `rokon::`: ROKON 데이터 형식

ROKON(Rok Object Notation)은 Rok의 리터럴 문법으로 값을 적는 데이터 형식입니다. 확장자는 `.rokon`입니다. Rust의 RON, Zig의 ZON과 같은 역할을 합니다. 패키지 매니페스트 `rok.rokon`과 lockfile `rok.lock`([refs-lang 9.10.3](../../refs-lang/009-tools/910-rok-frontend.md#9103-rokrokon--roklock))이 이 형식을 씁니다.

## 3.9.1. 형식

ROKON은 Rok 식 문법의 부분집합입니다. 라이브러리는 이 부분집합을 자체 렉서와 재귀 하강 파서로 읽습니다. 실행 중에 컴파일러의 파서를 쓰지 않습니다. 파일 하나에 값이 하나 들어갑니다.

| 형태        | 예                                                    | 비고                                                  |
|-------------|-------------------------------------------------------|-------------------------------------------------------|
| none        | `none`                                                |                                                       |
| bool        | `true`, `false`                                       |                                                       |
| 정수        | `42`, `-7`, `0xff`, `0b1010`, `0o17`, `1_000_000`      | 접미사 없음                                           |
| 실수        | `3.14`, `-1.5e-3`, `2E10`                              | 접미사 없음                                           |
| char        | `'a'`, `'\n'`                                         | `c'..'`는 안 됨                                      |
| 문자열      | `"text"`, 원시 문자열, 여러 줄 문자열                 | f-string은 안 됨                                     |
| 리스트      | `[a, b, c]`                                           |                                                       |
| dict        | `{ "k": v, 1: w }`                                    | 키는 값 (문자열, 수 등)                               |
| 튜플        | `(a, b)`, `(a,)`                                      | `(x)`는 묶음이지 튜플이 아님                         |
| struct      | `Point { x: 1, y: 2 }`, `{ x: 1, y: 2 }`, `Pair(1, 2)`, `Unit {}` | 이름은 선택. 필드 이름은 식별자. 튜플 struct는 `Name(a, b)` |
| enum 변형   | `Shape::Unit`, `Shape::Circle(1.0, at: p)`, `Shape::Rect { w: 2, h: 3 }` | 전체 경로로 적음                      |

- 쉼표는 뒤에 하나 더 붙여도 됩니다. 주석은 `//`와 `/* */`입니다.
- `{}`는 빈 struct이자 빈 dict입니다(대상 타입이 정함). `{ .. }` 하나에는 필드(식별자 키)나 키(값 키) 중 한쪽만 올 수 있습니다. 섞으면 에러입니다.
- 다음은 쓸 수 없습니다. 맨 식별자, 연산자(수 앞의 단항 `-`만 허용), 메서드 호출, `..`, 블록, f-string, Set 리터럴(`{1, 2}`, Set은 리스트로 적음), `c'..'` 리터럴, 제네릭 경로(`List<i32>::..`), 값 뒤에 남은 토큰입니다. 모두 `line:col`을 보여 주는 에러입니다.
- NaN과 무한대는 ROKON 표기가 없습니다. 입력의 `nan` / `inf`는 맨 식별자로 거부됩니다.

> [!NOTE]
> 알려진 한계: 첫 키가 `none`인 dict(`{ none: 1 }`)는 지원하지 않습니다. 읽는 쪽이 `none:`을 필드 이름으로 받아 중괄호를 struct로 읽기 때문입니다.

```rokon
// config.rokon
Config {
    name: "geom",
    retries: 3,
    ratio: 0.75,
    tags: ["a", "b"],
    limits: { "cpu": 2, "mem": 512 },
    mode: Mode::Fast,
    proxy: none,
}
```

## 3.9.2. Rok 타입과 대응

| ROKON          | Rok 타입                                                                     |
|----------------|------------------------------------------------------------------------------|
| `none`         | `T \| none`의 `none`                                                        |
| `true` / `false` | `bool`                                                                     |
| 정수           | 모든 정수 타입, `int`. 실수를 기대하는 자리 (`f32` / `f64` / `float`)에도 됨 |
| 실수           | `f32`, `f64`, `float`                                                         |
| char           | `char`                                                                        |
| 문자열         | `str`                                                                         |
| 리스트         | `List<T>`, `[T; N]`, `Set<T>`, `OrderedSet<T>`                                |
| dict           | `Dict<K, V>`, `OrderedDict<K, V>`                                             |
| 튜플           | 원소 1~6개의 튜플                                                             |
| struct         | `struct` (`#[derive(Rokon)]`)                                                 |
| enum 변형      | `enum` (`#[derive(Rokon)]`)                                                   |
| 아무 값        | `Value`                                                                       |

## 3.9.3. 읽기

`parse`는 텍스트를 `Value`로 읽습니다.

| 입력                         | `Value`                                                        |
|------------------------------|----------------------------------------------------------------|
| `{ ident: v, .. }`           | 이름 없는 `Struct`                                             |
| `{ "k": v, .. }`, `{}`       | `Dict`                                                         |
| `Name { f: v }`              | 이름 있는 `Struct`                                             |
| `A::B { f: v }`              | `Variant` (`named`에 값)                                      |
| `A::B(a, k: b)`, `A::B`      | `Variant` (`args`, `named`)                                    |
| `Name(a, b)`                 | 경로가 한 마디인 `Variant` (튜플 struct)                        |
| 정수                         | `Int` (정확한 `int`)                                           |

타입이 있는 읽기(`from_str`, `decode`)에서는 대상 타입이 해석을 정합니다.

- struct 대상은 `Struct`, `{}`, `A::B { .. }`를 받습니다. 그래서 모듈 경로를 붙인 struct 이름(`geom::Point { .. }`)도 받습니다.
- 적은 이름은 대상과 같아야 합니다. struct는 경로의 마지막 마디, enum 변형은 마지막 두 마디를 비교합니다.
- 모르는 필드는 그 필드 이름 위치의 에러입니다.
- 빠진 필드는 그 필드에 `= 기본값`이 있으면 기본값을 받습니다. 타입이 `T | none`이면 `none`을 받습니다. 그 밖에는 에러입니다(`FromRokon::from_missing`).
- 정수 값이 대상 정수 타입의 범위를 넘으면 에러입니다.
- 이름 있는 변형은 `E::V(a, k: b)`로도 읽습니다. 위치 인자가 필드를 순서대로 채우고, 이어서 이름 있는 인자가 채웁니다.

## 3.9.4. 정규 형식

`rokon::to_string`과 `rok`(매니페스트를 고치거나 `rok.lock`을 쓸 때)이 내는 정규 형식은 다음과 같습니다. 정규 형식이 곧 읽기 좋은 형식이므로 보기 좋게 출력하는 함수는 따로 없습니다.

- 들여쓰기는 공백 4칸입니다.
- 100칸 안에 들어가면 한 줄로, 아니면 여러 줄로 쓰고 각 원소 뒤에 쉼표를 붙입니다.
- 실수는 다시 읽었을 때 같은 값이 되는 가장 짧은 표기로 씁니다(`1.0`, `1.0e300`). NaN / 무한대를 쓰려 하면 panic이 일어납니다("NaN/infinity has no ROKON form").
- 이름 있는 값만 있는 변형의 정규 형식은 `E::V { .. }` 하나입니다(`Value::Variant`에는 형태 표시가 없음).
- 문자열과 문자의 양방향(bidi) 제어 문자(U+202A-U+202E, U+2066-U+2069)는 이스케이프해서 씁니다(`\u{202e}`).
- 정규 형식을 다시 읽으면 같은 값이 됩니다(왕복 변환). 읽는 쪽은 범위 밖의 실수 리터럴(`1e999`)을 거부하고, 범위를 알려 줍니다. 메시지는 ``float `1e999` is out of range (ROKON floats are f64, up to about 1.8e308)``입니다.

`rokc fmt`는 `.rokon` 파일이 ROKON인지 검사하고 들여쓰기만 다시 맞춥니다. 정규 형식으로 다시 쓰지는 않으므로 주석과 배치가 그대로 남습니다.

## 3.9.5. 라이브러리: `rok::std::rokon`

```rok
use rok::std::rokon;

pub enum Value {
    None,
    Bool(bool),
    Int(int),
    Float(float),
    Char(char),
    Str(str),
    List(List<Value>),
    Dict(OrderedDict<Value, Value>),
    Tuple(List<Value>),
    Struct { name: str | none; fields: OrderedDict<str, Value>; },
    Variant { path: str; args: List<Value>; named: OrderedDict<str, Value>; },
}
impl Value {
    fn to_string(self) -> str;       // 정규 형식
    fn kind(self) -> str;            // 에러 메시지용 종류 이름 (아래)
}                                    // + Display, Debug, PartialEq, Eq, Hash

pub struct RokonError { pub msg: str; pub line: u32; pub col: u32; pub path: str; }

pub trait ToRokon { fn to_rokon(self) -> Value; }
pub trait FromRokon {
    fn from_rokon(v: Value) throws (RokonError) -> Self;
    fn from_missing() throws (RokonError) -> Self;   // 기본: 에러. T | none 은 none
}

pub fn parse(s: str) throws (RokonError) -> Value;
pub fn to_string<T: ToRokon>(v: T) -> str;
pub fn from_str<T: FromRokon>(s: str) throws (RokonError) -> T;
pub fn encode<T: ToRokon>(x: T) -> Value;
pub fn decode<T: FromRokon>(v: Value, seg: str) throws (RokonError) -> T;
```

- `RokonError`의 `Display`는 위치가 있으면 `line:col: msg`, 없으면 `at <path>: msg`입니다. 타입이 있는 읽기의 에러는 `.servers[1].port`, `.limits["cpu"]` 같은 경로를 담습니다. `from_str`은 그 경로를 입력의 `line:col`로 바꿔 줍니다. `decode`의 `seg`는 경로의 시작 마디입니다.
- `Value::kind`는 에러 메시지에 쓰는 종류 이름입니다. 차례로 `"none"`, `"bool"`, `"integer"`, `"float"`, `"char"`, `"string"`, `"list"`, `"dict"`, `"tuple"`, `"struct"`이고 `Variant`는 ``"variant `<path>`"``입니다.
- `Value`의 `==`와 `hash`는 구조적입니다. dict의 항목과 struct의 필드는 순서와 무관하게 비교합니다. 부동소수는 비트 정체성으로 비교합니다(NaN == NaN, -0.0 != 0.0). 이름 없는 빈 struct와 빈 dict는 같습니다.
- `Value`의 `Debug`, `==`, `hash`에서는 panic이 일어나지 않습니다.
- 표준 구현은 다음과 같습니다. 모든 정수 타입, `f32` / `f64`, `int` / `float`, `bool` / `char` / `str`, `Value`, `T | none`, `List`, `Set`, `Dict`, `OrderedDict`, `OrderedSet`, 원소 1~6개의 튜플, `[T; N]`.

모듈은 `Fields`, `struct_args`, `variant_of`, `variant_args`, `unknown_variant`, `RokonError::new` / `within`, 그리고 하위 모듈 `read` / `write`도 내보냅니다. 이들은 `#[derive(Rokon)]`에 대해 `@derive`가 생성하는 코드에 쓰입니다. 안정 API가 아닙니다.

### `#[derive(Rokon)]`

struct와 enum(제네릭 포함)에 `ToRokon`과 `FromRokon`을 함께 구현합니다.

| 정의                 | 쓰는 모양              |
|----------------------|------------------------|
| 이름 있는 필드 struct | `Name { f: v }`       |
| 튜플 struct          | `Name(a, b)`           |
| unit struct          | `Name {}`              |
| unit 변형            | `E::V`                 |
| 튜플 변형            | `E::V(a)`              |
| 이름 있는 변형       | `E::V { f: v }`        |

필드의 `= 기본값`은 빠진 필드에 쓰입니다(3.9.3). derive는 메타 토큰 `@Self.name`, `@Self.is_unit` / `is_tuple` / `is_named`, `@field.has_default` / `@field.default`로 구현합니다([refs-lang 6.9](../../refs-lang/006-meta-system/609-template-tokens.md)).

```rok
#[derive(Rokon)]
struct Server { host: str; port: u16 = 80; tls: bool | none; }

#[derive(Rokon)]
struct Config { name: str; servers: List<Server>; limits: OrderedDict<str, i32>; }

let cfg: Config = try rokon::from_str(text);   // 에러는 "line:col: msg"
print(rokon::to_string(cfg));
```

### 한계

- `Dict` / `Set`은 해시 순서로 씁니다. 순서를 지키려면 `OrderedDict` / `OrderedSet`을 씁니다.
- `[T; 0]`은 읽을 수 없습니다.
- 튜플은 원소 6개까지입니다.
- 첫 키가 `none`인 dict는 읽을 수 없습니다(3.9.1).

## 3.9.6. 다른 형식과 비교

| 형식   | struct 표기                 | 비고                              |
|--------|-----------------------------|-----------------------------------|
| RON    | `Name(field: v)`            | Rust와 다른 전용 문법            |
| ZON    | `.{ .name = "x" }`          | Zig의 익명 struct 리터럴         |
| ROKON  | `Name { field: v }`         | Rok 자체의 리터럴 문법 그대로     |

ROKON은 별도 문법 없이 Rok 리터럴을 그대로 씁니다. 그래서 소스에 적은 값과 파일에 적은 값의 모양이 같습니다.
