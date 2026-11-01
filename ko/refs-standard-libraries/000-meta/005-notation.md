# 0.5. 표기법

이 사양의 표기 규약은 [refs-lang 0.8 표기법](../../refs-lang/000-meta/008-notation.md)을 그대로 따릅니다. 이 장은 표준 라이브러리 사양에만 덧붙는 규약을 정의합니다.

## 0.5.1. 트레잇 시그니처 표기

각 트레잇 사양은 시그니처 블록으로 시작합니다. 이 블록은 모든 메서드, 연관 타입(associated type), 상위 트레잇(supertrait)을 한곳에 모아 보여 줍니다.

```rok
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}
```

이 형태는 표준 라이브러리를 실제로 작성하는 형태와 같습니다. 그래서 사용자도 자기 코드에서 같은 형식을 쓸 수 있습니다.

## 0.5.2. 메서드 시그니처 표

메서드가 여럿인 트레잇은 다음과 같이 표로도 요약합니다.

| 메서드 | 시그니처                       | 의미      |
|--------|--------------------------------|-----------|
| `add`  | `fn(self, rhs: Rhs) -> Output` | 산술 덧셈 |
| `sub`  | `fn(self, rhs: Rhs) -> Output` | 산술 뺄셈 |

## 0.5.3. 트레잇 구현 매트릭스

각 트레잇 사양의 끝에는 그 트레잇을 구현하는 표준 라이브러리 타입을 표로 적습니다. 열은 타입과 binding 메커니즘 두 가지뿐입니다. 구현 파일 경로는 사양에 적지 않습니다.

```markdown
## X.Y.Z. 트레잇 구현 매트릭스

| 타입  | binding                         | 비고                 |
|-------|---------------------------------|----------------------|
| `i32` | `#[intrinsic("rok.add.i32")]`   | 컴파일러 PrimOp 직결 |
| `i64` | `#[intrinsic("rok.add.i64")]`   |                      |
| `f32` | `#[intrinsic("rok.add.f32")]`   |                      |
| `f64` | `#[intrinsic("rok.add.f64")]`   |                      |
| `int` | (intrinsic 없음, auto-promote) | 표준 라이브러리 정의 |
```

## 0.5.4. 의미 정의(semantics)

각 메서드의 의미는 산문으로 풀어 씁니다. 형식과 예시뿐 아니라 한계와 에러 상황도 다룹니다.

```markdown
### `Add::add(self, rhs)`

산술 덧셈. 두 값을 더한 결과를 반환합니다.

- `i32 + i32`: 2's complement wrap (debug=panic, release=wrap)
- `f64 + f64`: IEEE 754 덧셈
- `int + Int`: 무한정 정밀도. 결과가 i64 범위 넘어가면 BigInt 로 promote

오버플로 핸들링은 [2.1 숫자 타입](../200-core/201-numbers.md) 의 fixed-width 산술
규칙을 따릅니다. saturate / checked / wrapping 변형은 별도 메서드 (`saturating_add` 등)
로 노출됩니다.
```

## 0.5.5. 예시

각 트레잇과 메서드에는 동작하는 예시를 하나 이상 붙입니다. 예시는 묵시적인 prelude import `use rok::core::prelude::*;`가 적용된 상태를 가정합니다. 그래서 아무것도 가져오지 않고 그대로 실행되는 코드여야 합니다.

```rok
let a: i32 = 5;
let b: i32 = 3;
let c = a + b;        //→ 8 (i32)

let x: f64 = 1.5;
let y: f64 = 2.5;
let z = x + y;        //→ 4.0 (f64)
```

## 0.5.6. 상호 참조

다른 문서를 참조할 때는 위치에 따라 두 형식 중 하나를 씁니다. 같은 사양 안에서는 `[2.7 iter](../200-core/207-iter.md)`처럼 쓰거나 약식으로 "refs-standard-libraries 2.7"이라고 부릅니다. refs-lang은 `[refs-lang 6.4](../../refs-lang/006-meta-system/604-lang-magic.md)` 형태로 참조합니다.

## 0.5.7. 사양과 구현의 분리

이 문서는 **사양**(specification)만 정의합니다. 컴파일러와 표준 라이브러리의 실제 구현은 별도 저장소에서 관리합니다. 구현이 이 사양과 충돌하면 이 사양이 우선합니다.

## 0.5.8. `try_` 접두사

`try_` 이름은 다음 두 형태 가운데 하나를 나타내며, 이 둘만 나타냅니다.

1. 막히지 않는 시도: 기본 형태가 기다리는 일을 지금 하고, 할 수 없었으면 그렇다고 알립니다(`Mutex::try_lock`, `Receiver::try_recv`, `Sender::try_send`).
2. 받은 클로저나 future가 던진 것을 그대로 넘기는 형태(`Iterator::try_fold`, `Iterator::try_for_each`, `par::try_scope`, `task::try_block_on`).

던질 수 없는 trait 메서드의 던지는 형태도 `try_` 이름을 씁니다. `TryFrom::try_from`, `TryInto::try_into`, `File::try_clone`이 그렇습니다. 그 밖의 함수는 이 접두사를 쓰지 않습니다. 실패하는 함수는 기본 이름 그대로 던지므로(0.5.9) `try_` 짝이 따로 없습니다. `try_`는 접두사로만 쓰며, `_try`로 끝나는 이름은 없습니다.

## 0.5.9. 오류 모델

함수는 실패를 한 통로로만 알리며, 통로는 실패의 성격으로 정합니다.

| 실패 | 통로 | 예 |
|---|---|---|
| 원인이 하나뿐인 평범한 "답 없음" | `-> T \| none` | `List::get`, `Iterator::find`, `Path::parent`, `Path::strip_prefix`, `checked_*`, `char::from_u32`, `NonZero::new` |
| 입력, 환경, I/O 때문이거나, 호출자가 구별할 수 있는 원인이 여럿인 실패 | `throws (E)` | `FromStr::from_str`, `str::from_utf8`, `Int::parse_radix`, `FormatSpec::parse`, `ClientConfig::new`, `fs`와 `net`의 연산 |
| 호출자가 먼저 쉽게 확인할 수 있는 호출자의 버그 | panic, `#[track_caller]` | 범위를 벗어난 연산자 인덱스, 2..=36 밖의 진법, 0인 chunk 크기 |

1. **짝을 두지 않습니다.** 실패 통로만 다른 두 함수는 없습니다. `try? f()`가 모든 던지는 호출의 `T | none` 형태이므로([refs-lang 3.14.3](../../refs-lang/003-expressions/314-error-control-flow.md)), 라이브러리에는 던지는 함수의 `_or_none` 짝도, panic하는 짝도 없습니다.
2. **sentinel 값을 쓰지 않습니다.** 결과는 실패를 자기 타입의 값(`0`, `-1`, `""`)으로 나타내지 않습니다. `T | none`을 돌려주거나 던집니다.
3. **닫기는 던집니다.** OS 자원(파일, 소켓, TLS 연결)의 `close()`는 OS가 알린 오류를 던집니다. `Drop`은 던지지 않으며, `Drop`이 놓아 주는 값의 오류는 버려집니다.
4. 연산자와 그 이름 붙은 메서드는 다를 수 있습니다. `xs[i]`는 panic하지만 `xs.get(i)`는 `none`을 돌려줍니다.

0.2.0에서 이 규칙에 따라 없앤 형태는 다음과 같습니다. `FormatSpec::try_parse`(`FormatSpec::parse`가 던집니다), `Float::parse_or_none`과 `Float::try_parse`(`try? s.parse()`), `task::try_id`(`task::current_id`), `task::block_on_try`(`task::try_block_on`으로 이름이 바뀌었습니다), `ClientConfig::try_new`(`ClientConfig::new`가 던집니다), 소켓의 `try_close`(`close`가 던집니다), `Path::strip_prefix_or_none`과 `StripPrefixError`(`Path::strip_prefix`가 `Path | none`을 돌려줍니다).

## 0.5.10. 에러 타입

표준 라이브러리의 에러 타입은 모두 다음 규칙을 따릅니다.

1. **필드는 비공개입니다.** 에러는 구조체 리터럴이 아니라 생성자(`new(..)`, 또는 `IoError::from_os_code` 같은 이름 붙은 생성자)로 만들고, 사실을 이름으로 삼은 getter(`kind()`, `position()`, `path()`, `os_code()`)로 읽습니다. 모든 에러 타입에는 `message() -> str`이 있습니다. 무엇이 잘못됐는지를 `Display`가 덧붙일 수 있는 문맥 없이 돌려줍니다.
2. **트레잇 묶음.** 모든 에러 타입은 `Display`, `Debug`, `Error`, `PartialEq`, `Eq`, `Clone`을 구현합니다. 타입 매개변수의 값을 담는 에러(`SendError<T>`, `TrySendError<T>`)는 앞의 셋을 구현합니다.
3. **종류.** 종류 getter는 `str`이나 정수가 아니라 enum(`IoErrorKind`, `TlsErrorKind`, `JoinErrorKind`)을 돌려주며, 그 enum은 `#[non_exhaustive]`입니다. OS 에러 코드는 `os_code() -> i32 | none`이고, OS에서 온 에러가 아니면 `none`입니다.
4. **unit 구조체.** 필드가 없는 에러는 unit 구조체(`struct TimeoutError;`)이며, `TimeoutError {}`가 아니라 `TimeoutError`로 씁니다.
5. **실패 영역마다 에러 하나.** core는 함께 쓰는 에러(`ValueError`, `ParseError`, `OverflowError`, `ConversionError`, `TypeError`, `Utf8Error`)를 둡니다. 자기 실패 영역이 있는 std 모듈은 에러 타입을 하나 둡니다(`IoError`, `TlsError`, `RokonError`). UTF-8 검사가 실패하면 모두 `Utf8Error`를 던집니다.
6. **메시지.** 메시지는 소문자로 시작하고(고유명사와 타입 이름은 예외), 끝에 마침표나 `error:` 접두사를 붙이지 않습니다. 연산자의 panic은 Rust의 문구(`index 5 out of bounds for length 3`)를 씁니다. 메서드의 panic은 `Type::method: `(자유 함수는 `module::fn: `)로 시작하고, 문제를 말한 뒤, 대신 쓸 것이 있으면 `; <대신 쓸 것>`을 붙입니다. 예: `i32::isqrt: the number is negative; use `checked_isqrt` to get `none` instead`. 잘못된 인자에 panic하는 공개 함수는 `#[track_caller]`입니다.

0.2.0에서 이 규칙에 따라 없앤 형태는 다음과 같습니다. 모든 에러 타입의 공개 필드(그리고 `Ipv4Addr` / `Ipv6Addr`의 공개 필드), `IoError::new(message)`, `IoError::with_kind`, `IoError::from_os`, `IoError::code`(`IoError::new(kind, message)`, `IoError::from_os_code`, `IoError::os_code`), `JOIN_LOST` / `JOIN_TAKEN`(`JoinError::kind`), `NonUtf8Error`(`Utf8Error`), `process::signal(i32, handler)`(`process::on_signal(Signal, handler)`).

## 0.5.11. 이름 짓기

표준 라이브러리의 모든 이름은 이 표를 따릅니다. 표에 어긋나는 이름은 라이브러리의 결함이지 선례가 아닙니다.

| 종류 | 규칙 |
|---|---|
| 생성자 | `new`(필수 인자), `with_<x>`(인자 하나 더), `from_<src>`(변환, `Formatter::from_sink`), 특별한 값은 `empty()` / `zeroed()`. `to_<x>`는 쓰지 않음 |
| 변환 | `as_`는 비용 없는 뷰, `to_`는 새로 소유하는 값(비용 있음), `into_`는 소비. 문자열 타입이 `str`이므로 `to_str`, `read_to_str`, `from_str`이고 `to_string`은 쓰지 않음 |
| 수 변환 | 손실 없는 변환과 검사하는 변환은 `From` / `TryFrom` / `SaturatingFrom` / `WrappingFrom`으로만. 고유 `to_<num>`은 손실 있는 반올림(`to_f64`)에만 |
| 게터 | `get_` 접두사 없음. `get`은 키나 인덱스로 찾을 때, 또는 래퍼의 값(`NonZero::get`)에만 |
| 술어 | `is_` / `has_`, 또는 Rust에서 굳은 단어(`success`, `readonly`) |
| 세터 | `set_<x>(ref self, ..)`는 unit을 돌려줌. 빌더는 필드 이름을 그대로 쓰고 `Self`를 돌려줌 |
| 크기 | `len`은 컨테이너의 원소 수나 바이트 수, `<x>_len`은 길이 속성, `count`는 센 것(`count_ones`), `size`는 메모리 크기 |
| 추가 / 제거 | 양 끝은 `push` / `pop`, 키나 인덱스는 `insert` / `remove`, 여럿은 `extend`, `clear`, `retain`, 꺼내 옮기기는 `take` |
| 순회 | `iter()`는 `<Type>Iter`를 돌려줌. 다른 순회 메서드는 정의한 모듈에서 메서드 이름의 CamelCase를 돌려줌(아래) |
| 리플렉션 | 접미사 없는 형태는 타입의 답(`T::type_name()`, `T::size()`), `_of_val` 형태는 실행 중 값의 답(`x.type_name_of_val()`, `x.size_of_val()`, `x.field_names_of_val()`) |
| async | `<name>_async`는 막히는 `<name>`의 짝. `async_` 접두사는 쓰지 않음 |
| 약어 | 풀어 씀. 정해진 집합만 예외: `len`, `fd`, `io`, `utf8`, `ns` / `us` / `ms` / `secs` / `mins`, `tls`, `der` / `pem`, `mmap`, `pid` |
| 상수 | `SCREAMING_SNAKE`. 상수와 짝을 이루는 함수는 두지 않음(`f64::INFINITY`) |
| 모듈 | 단수 명사나 영역 이름(`fs`, `net`, `iter`). `collections`와 `ops`는 복수 그대로 |

순회 타입. `str`: `chars()` -> `Chars`, `bytes()` -> `Bytes`, `char_indices()` -> `CharIndices`, `lines()` -> `Lines`, `split()` -> `Split`, `splitn()` -> `SplitN`, `rsplit()` -> `RSplit`, `rsplitn()` -> `RSplitN`, `split_whitespace()` -> `SplitWhitespace`, `match_indices()` -> `MatchIndices`. `Dict`: `keys()` -> `Keys`, `values()` -> `Values`. `List` / `Slice`: `windows()` -> `slice::Windows`, `chunks()` -> `slice::Chunks`(어댑터 `iter::Windows` / `iter::Chunks`와 나란히). `Receiver::iter()` -> `ReceiverIter`. 두 모듈이 같은 끝 이름을 쓰면 모듈과 함께 적습니다(`slice::Windows`, `str::Bytes`).

## 0.5.13. 트레잇 범위, `#[non_exhaustive]`, `#[sealed]`

표준 라이브러리의 모든 공개 타입은 자기 부류의 트레잇을 구현합니다.

| 타입 부류 | 구현하는 트레잇 |
|---|---|
| 모든 공개 타입 | `Debug` |
| 값 타입(주소, 경로, id, 시간, 종류, 알고리즘, 범위, 스펙) | `Clone`, `PartialEq`; 같음이 전체적이면 `Eq`; `Eq`이면 `Hash`; 값에 순서가 있으면 `PartialOrd` / `Ord`; 작으면 `Copy` |
| 빈 값이나 0이 분명한 타입 | `Default` |
| 텍스트 같은 타입 | `Display` |

부류별 `Debug` 출력은 다음과 같습니다.

- 값은 내용을 보여 줍니다: `0..3`, `0..=3`, `Start(3)`, `SocketAddrV4 { ip: 127.0.0.1, port: 80 }`, `Permissions { mode: 0o644, readonly: false }`. `StrBuf`는 담은 텍스트를 따옴표로 감싸 보여 줍니다.
- 핸들은 타입과 그 핸들이 가리키는 것을 보여 줍니다: `File { path: "a.txt" }`, `Child { pid: 42 }`, `Thread { id: ThreadId(3), name: "worker" }`. 싸게 보여 줄 것이 없는 핸들은 타입 이름만 보여 줍니다(`TcpStream`, `Mutex`).
- 반복자, 스트림, future는 타입 이름을 보여 줍니다(`Map`, `Chars`, `AsyncFuture`).
- 비밀을 담은 값은 그 비밀을 보여 주지 않습니다: `PrivateKey { .. }`, `Hmac { .. }`, `RsaPrivateKey { bits: 2048, .. }`.

0.2.0에서 더한 값 트레잇입니다(`Dict` 키와 `Set` 원소로 쓰고, 비교하고, 정렬합니다).

| 타입 | 트레잇 |
|---|---|
| `Path` | `Clone`, `Eq`, `Hash`, `PartialOrd`, `Ord`(텍스트 순), `Default`(빈 경로) |
| `Ipv4Addr`, `Ipv6Addr` | `Clone`, `Copy`, `Eq`, `Hash`, `PartialOrd`, `Ord` |
| `SocketAddr`, `SocketAddrV4`, `SocketAddrV6` | `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash`, `Debug` |
| `Ordering` | `Clone`, `Copy`, `Hash`, `PartialOrd`, `Ord`(`Less < Equal < Greater`) |
| `IoErrorKind` | `Clone`, `Copy`, `Eq`, `Hash`, `Display`(`not found`) |
| `TlsErrorKind`, `CipherSuite`, `AeadAlgorithm`, `EcCurve`, `HashAlgorithm` | `Hash` |
| `Signal`, `FileType`, `JoinErrorKind` | `Clone`, `Copy`, `Eq`, `Hash` |
| `Component`, `RawStr`, `RawPath` | `Clone`, `Eq`, `Hash` |
| `SeekFrom`, `Shutdown`, `Alignment`, `Sign`, `AtomicOrdering` | `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash`, `Debug` |
| `ThreadId` | `Clone`, `Copy`, `Eq`, `Hash`, `Debug` |
| `Permissions`, `Layout`, `ExitStatus` | `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash` |
| `CStr`, `CString`, `Certificate`(DER 바이트 기준), `FormatSpec` | `Clone`, `PartialEq`, `Eq`, `Hash` |
| `Range` | `Debug`, `Hash` |
| `RangeInclusive`, `RangeFrom`, `RangeTo`, `RangeToInclusive`, `RangeFull` | `Clone`, `PartialEq`, `Eq`, `Hash`, `Debug` |
| `Duration`(`ZERO`), `StrBuf`(빈 값), `FormatSpec`(`FormatSpec::new()`) | `Default` |

`#[non_exhaustive]`([refs-lang 6.2.10](../../refs-lang/006-meta-system/602-built-in-attributes.md))는 변형이 늘 수 있는 공개 enum과 필드가 늘 수 있는 옵션 레코드에 붙습니다. 정의한 패키지 밖에서 이런 enum을 `match`하려면 `_` 갈래가 있어야 하고, 구조체 패턴에는 `..`가 있어야 합니다.

- enum: `IoErrorKind`, `TlsErrorKind`, `CipherSuite`, `AeadAlgorithm`, `EcCurve`, `HashAlgorithm`, `rokon::Value`, `Stdio`, `FileType`, `Signal`, `JoinErrorKind`
- 레코드: `tls::ClientConfig`, `tls::ServerConfig`, `FieldInfo`, `MethodInfo`

닫힌 enum은 그대로 exhaustive입니다: `Ordering`, `Poll`, `Result`, `SeekFrom`, `Shutdown`, `Alignment`, `Sign`, `SocketAddr`, `Component`, `AtomicOrdering`, `SendError`, `TrySendError`.

`#[sealed]`([refs-lang 6.2](../../refs-lang/006-meta-system/602-built-in-attributes.md))는 라이브러리가 건전성이나 의미를 기대는 구현을 가진 트레잇에 붙습니다. 정의한 패키지만 구현할 수 있고, 바운드와 호출은 그대로입니다: `AtomicPrimitive`(2.16), `NicheInt`, `NicheFloat`(2.2), `Transfer`(2.13), `Pinnable`(1.6). `Step`(2.7)은 사용자 범위 타입을 위해 열어 둡니다. `Pod`(1.3)는 sealed가 아니며(`#[derive(Pod)]`가 사용자 struct를 검사합니다), `bool`과 `char`는 `Pod`가 아닙니다.
