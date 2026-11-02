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
| `Int` | (intrinsic 없음, 임의 정밀도) | 표준 라이브러리 정의 |
```

## 0.5.4. 의미 정의(semantics)

각 메서드의 의미는 산문으로 풀어 씁니다. 형식과 예시뿐 아니라 한계와 에러 상황도 다룹니다.

```markdown
### `Add::add(self, rhs)`

산술 덧셈. 두 값을 더한 결과를 반환합니다.

- `i32 + i32`: 오버플로에서 panic (모든 프로필에서 검사, `-C overflow-checks=no`이면 wrap)
- `f64 + f64`: IEEE 754 덧셈
- `Int + Int`: 임의 정밀도. 오버플로 없음

오버플로 처리는 [2.1 숫자 타입](../200-core/201-numbers.md)의 고정 폭 산술
규칙을 따릅니다. saturating / checked / wrapping 변형은 별도 메서드(`saturating_add` 등)
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
5. 연산자는 던질 수 없습니다([2.3](../200-core/203-ops.md)). 실패할 수 있는 연산자는 `Result`를 돌려줍니다(`Finite<F> + Finite<F>`는 `Result<Finite<F>, OverflowError>`).


## 0.5.10. 에러 타입

표준 라이브러리의 에러 타입은 모두 다음 규칙을 따릅니다.

1. **필드는 비공개입니다.** 에러는 구조체 리터럴이 아니라 생성자(`new(..)`, 또는 `IoError::from_os_code` 같은 이름 붙은 생성자)로 만들고, 사실을 이름으로 삼은 getter(`kind()`, `position()`, `path()`, `os_code()`)로 읽습니다. 라이브러리만 만드는 에러(`LayoutError`, `NulError`)에는 공개 생성자가 없을 수 있습니다. 원인을 나열한 enum인 에러(`FromBytesWithNulError`, `VarError`)는 데이터를 변형 필드에 담고, 패턴으로 읽습니다. 모든 에러 타입에는 `message() -> str`이 있습니다. 무엇이 잘못됐는지를 `Display`가 덧붙일 수 있는 문맥 없이 돌려줍니다.
2. **트레잇 묶음.** 모든 에러 타입은 `Display`, `Debug`, `Error`, `PartialEq`, `Eq`, `Clone`을 구현합니다. 타입 매개변수의 값을 담는 에러(`SendError<T>`, `TrySendError<T>`)는 앞의 셋을 구현합니다.
3. **종류.** 종류 getter는 `str`이나 정수가 아니라 enum(`IoErrorKind`, `TlsErrorKind`, `JoinErrorKind`)을 돌려주며, 그 enum은 `#[non_exhaustive]`입니다(`DecodeErrorKind`도 같습니다). OS 에러 코드는 `os_code() -> i32 | none`이고, OS에서 온 에러가 아니면 `none`입니다.
4. **unit 구조체.** 필드가 없는 에러는 unit 구조체(`struct TimeoutError;`)이며, `TimeoutError {}`가 아니라 `TimeoutError`로 씁니다.
5. **실패 영역마다 에러 하나.** core는 함께 쓰는 에러(`ValueError`, `ParseError`, `OverflowError`, `ConversionError`, `TypeError`, `Utf8Error`)와 자기 실패 영역의 에러(`ZeroError`, `DecodeError`, `FmtError`, `CancellationError`, `TimeoutError`)를 둡니다. std 모듈은 실패 영역마다 에러 타입을 하나 둡니다(`IoError`, `TlsError`, `RokonError`. `env`는 변수 읽기의 `VarError`와 `join_paths`의 `JoinPathsError`를, `process`는 `IoError` 옆에 핸들러 설치의 `SignalError`를 둡니다). 두 에러가 같은 실패를 다루는 일은 없습니다. UTF-8 검사가 실패하면 모두 `Utf8Error`를 던집니다.
6. **메시지.** 메시지는 소문자로 시작하고(고유명사와 타입 이름은 예외), 끝에 마침표나 `error:` 접두사를 붙이지 않습니다. 연산자의 panic은 Rust의 문구(`index 5 out of bounds for length 3`)를 씁니다. 메서드의 panic은 `Type::method: `(자유 함수는 `module::fn: `)로 시작하고, 문제를 말한 뒤, 대신 쓸 것이 있으면 `; <대신 쓸 것>`을 붙입니다. 예: `i32::isqrt: the number is negative; use `checked_isqrt` to get `none` instead`. 잘못된 인자에 panic하는 공개 함수는 `#[track_caller]`입니다.


## 0.5.11. 이름 짓기

표준 라이브러리의 모든 이름은 이 표를 따릅니다. 표에 어긋나는 이름은 라이브러리의 결함이지 선례가 아닙니다.

| 종류 | 규칙 |
|---|---|
| 생성자 | `new`(필수 인자), `with_<x>`(인자 하나 더), `from_<src>`(변환, `Formatter::from_sink`), 특별한 값은 `empty()` / `zeroed()`. 단위 생성자는 단위 이름(`Duration::secs(n)`, `ms`, `ns`). `List::filled(n, x)`(n개 복사)와 `BinaryHeap::by_ordering(cmp)`는 이름 붙은 예외. `to_<x>`는 쓰지 않음 |
| 변환 | `as_`는 비용 없는 뷰, `to_`는 새로 소유하는 값(비용 있음), `into_`는 소비. 문자열 타입이 `str`이므로 `to_str`, `read_to_str`, `from_str`이고 `to_string`은 쓰지 않음 |
| 수 변환 | 손실 없는 변환과 검사하는 변환은 `From` / `TryFrom` / `SaturatingFrom` / `WrappingFrom`으로만. 고유 `to_<num>`은 손실 있는 반올림(`to_f64`)에만 쓰고, `Int` / `Float`에는 `TryFrom` 옆에 검사하는 `to_<num>() -> T \| none`도 둠([2.1.2](../200-core/201-numbers.md)). Rust처럼 `Ipv4Addr::to_bits` / `from_bits`는 `From<Ipv4Addr> for u32`, `From<u32> for Ipv4Addr` 옆에 둠 |
| 게터 | `get_` 접두사 없음. `get`은 키나 인덱스로 찾을 때, 또는 래퍼의 값(`NonZero::get`)에만. 이름 붙은 예외는 guard의 `get_clone()` 하나로, 범위가 정해진 `get()` 옆의 깊은 복사본([2.16](../200-core/216-sync.md)) |
| 술어 | `is_` / `has_`, 또는 Rust에서 굳은 단어(`success`, `readonly`) |
| 세터 | `set_<x>(ref self, ..)`는 unit을 돌려줌. 핸들(파일, 소켓, 커서, 리더)은 OS 상태나 공유 상태를 바꾸므로 세터가 `self`를 받음(`File::set_len`, `TcpStream::set_nodelay`, `Cursor::set_position`). 빌더는 필드 이름을 그대로 쓰고 `Self`를 돌려줌 |
| 크기 | `len`은 컨테이너의 원소 수나 바이트 수, `<x>_len`은 길이 속성, `count`는 센 것(`count_ones`), `size`는 메모리 크기 |
| 추가 / 제거 | 양 끝은 `push` / `pop`, 키나 인덱스는 `insert` / `remove`, 여럿은 `extend`, `clear`, `retain`, 꺼내 옮기기는 `take` |
| 순회 | `iter()`는 `<Type>Iter`를 돌려줌(`BinaryHeap::iter`는 `HeapIter`). 다른 순회 메서드는 정의한 모듈에서 메서드 이름의 CamelCase를 돌려줌(아래). 단 `Iterator::inspect`는 `InspectIter`를 돌려줌(`Inspect`는 2.12의 리플렉션 트레잇). 소유하는 스트림 어댑터(`into_lines`, `into_bytes`, `into_split`, `into_take`, 그 `_async` 형태, `into_incoming`)는 뷰의 타입을 감싼 `io::Owned<V, R>`를 돌려줌([2.23.1](../200-core/223-io.md#2231-read)) |
| 리플렉션 | 접미사 없는 형태는 타입의 답(`T::type_name()`, `T::size()`), `_of_val` 형태는 실행 중 값의 답(`x.type_name_of_val()`, `x.size_of_val()`, `x.field_names_of_val()`) |
| async | `<name>_async`는 막히는 `<name>`의 짝. `async_` 접두사는 쓰지 않음 |
| 약어 | 풀어 씀. 정해진 집합만 예외: `len`, `fd`, `io`, `utf8`, `ns` / `us` / `ms` / `secs` / `mins`, `tls`, `der` / `pem`, `mmap`, `pid`, `alpn`, `ocsp`, `dns`, 그리고 Rust에서 굳은 줄임말 `addr`, `dir`, `temp`, `exe`, `env`, `var` / `vars`, `args`, `arg`, `ty`, `os`, `std`(`Stdio`), `ptr`, `buf`, `recv`, `ttl`, `subsec`, 수학 이름 `abs`, `rem`, `ln`, `exp`, `cbrt`, `min` / `max` |
| 상수 | `SCREAMING_SNAKE`. 상수와 짝을 이루는 함수는 두지 않음(`f64::INFINITY`) |
| 모듈 | 단수 명사나 영역 이름(`fs`, `net`, `iter`). `collections`와 `ops`는 복수 그대로 |

순회 타입. `str`: `chars()` -> `Chars`, `bytes()` -> `Bytes`, `char_indices()` -> `CharIndices`, `lines()` -> `Lines`, `split()` -> `Split`, `splitn()` -> `SplitN`, `rsplit()` -> `RSplit`, `rsplitn()` -> `RSplitN`, `split_whitespace()` -> `SplitWhitespace`, `match_indices()` -> `MatchIndices`. `Dict`: `keys()` -> `Keys`, `values()` -> `Values`. `List` / `Slice`: `windows()` -> `slice::Windows`, `chunks()` -> `slice::Chunks`(어댑터 `iter::Windows` / `iter::Chunks`와 나란히). `Receiver::iter()` -> `ReceiverIter`. 두 모듈이 같은 끝 이름을 쓰면 모듈과 함께 적습니다(`slice::Windows`, `str::Bytes`).

## 0.5.12. 트레잇 범위, `#[non_exhaustive]`, `#[sealed]`

표준 라이브러리의 모든 공개 타입은 자기 부류의 트레잇을 구현합니다.

| 타입 부류 | 구현하는 트레잇 |
|---|---|
| 모든 공개 타입 | `Debug` |
| 값 타입(주소, 경로, id, 시간, 종류, 알고리즘, 범위, 스펙) | `Clone`, `PartialEq`; 같음이 전체적이면 `Eq`; `Eq`이면 `Hash`; 값에 순서가 있으면 `PartialOrd` / `Ord`; 작으면 `Copy` |
| 빈 값이나 0이 분명한 타입 | `Default` |
| 텍스트 같은 타입 | `Display` |

부류별 `Debug` 출력은 다음과 같습니다.

- 값은 내용을 보여 줍니다: `0..3`, `0..=3`, `Start(3)`, `Permissions { mode: 0o644, readonly: false }`. 주소는 `Display`처럼 텍스트를 보여 줍니다(`127.0.0.1:80`, [3.2.4](../300-std/302-net.md#324-socketaddr--ipaddr)). `StrBuf`는 담은 텍스트를 따옴표로 감싸 보여 줍니다.
- 핸들은 타입과 그 핸들이 가리키는 것을 보여 줍니다: `File { path: "a.txt" }`, `Child { pid: 42 }`, `Thread { id: ThreadId(3), name: "worker" }`. 싸게 보여 줄 것이 없는 핸들은 타입 이름만 보여 줍니다(`TcpStream`, `Mutex`).
- 반복자, 스트림, future는 타입 이름을 보여 줍니다(`Map`, `Chars`, `AsyncFuture`).
- 비밀을 담은 값은 그 비밀을 보여 주지 않습니다. 타입과 공개 매개변수만 한 가지 꼴 `Type { field: value, .. }`로 보여 줍니다: `x25519::PrivateKey { .. }`, `Hmac { .. }`, `DecapsulationKey768 { .. }`, `rsa::PrivateKey { bits: 2048, .. }`, `Secret { len: 32, .. }`.

`Dict` 키와 `Set` 원소로 쓰고, 비교하고, 정렬하는 라이브러리 타입의 값 트레잇입니다.

| 타입 | 트레잇 |
|---|---|
| `Path` | `Clone`, `Eq`, `Hash`, `PartialOrd`, `Ord`(텍스트 순), `Default`(빈 경로), `Display`(텍스트) |
| `Ipv4Addr`, `Ipv6Addr` | `Clone`, `Copy`, `Eq`, `Hash`, `PartialOrd`, `Ord` |
| `IpAddr` | `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash`, `Debug` |
| `Domain`, `SocketType`, `Protocol`, `Access` | `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash`, `Debug` |
| `SocketAddr`, `SocketAddrV4`, `SocketAddrV6` | `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash`, `Debug` |
| `Ordering` | `Clone`, `Copy`, `Hash`, `PartialOrd`, `Ord`(`Less < Equal < Greater`) |
| `IoErrorKind` | `Clone`, `Copy`, `Eq`, `Hash`, `Display`(`not found`) |
| `TlsErrorKind`, `CipherSuite`, `AeadAlgorithm`, `EcCurve`, `HashAlgorithm` | `Hash` |
| `Signal`, `FileType`, `JoinErrorKind` | `Clone`, `Copy`, `Eq`, `Hash` |
| `Component`, `RawStr`, `RawPath` | `Clone`, `Eq`, `Hash` |
| `SeekFrom`, `Shutdown`, `Alignment`, `Sign`, `LoadOrdering`, `StoreOrdering`, `UpdateOrdering` | `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash`, `Debug` |
| `ThreadId` | `Clone`, `Copy`, `Eq`, `Hash`, `Debug` |
| `Permissions`, `Layout`, `ExitStatus` | `Clone`, `Copy`, `PartialEq`, `Eq`, `Hash` |
| `CStr`, `CString`, `Certificate`(DER 바이트 기준), `FormatSpec` | `Clone`, `PartialEq`, `Eq`, `Hash` |
| `Range` | `Debug`, `Hash` |
| `RangeInclusive`, `RangeFrom`, `RangeTo`, `RangeToInclusive`, `RangeFull` | `Clone`, `PartialEq`, `Eq`, `Hash`, `Debug` |
| `Duration`(`ZERO`), `StrBuf`(빈 값), `FormatSpec`(`FormatSpec::new()`) | `Default` |

`#[non_exhaustive]`([refs-lang 6.2.10](../../refs-lang/006-meta-system/602-built-in-attributes.md))는 변형이 늘 수 있는 공개 enum과 필드가 늘 수 있는 옵션 레코드에 붙습니다. 정의한 패키지 밖에서 이런 enum을 `match`하려면 `_` 갈래가 있어야 하고, 구조체 패턴에는 `..`가 있어야 합니다.

- enum: `IoErrorKind`, `TlsErrorKind`, `CipherSuite`, `AeadAlgorithm`, `EcCurve`, `HashAlgorithm`, `DecodeErrorKind`, `base64::Alphabet`, `rokon::Value`, `Stdio`, `FileType`, `Signal`, `JoinErrorKind`, `fs::Access`, `net::Domain`, `net::SocketType`
- 레코드: `tls::ClientConfig`, `tls::ServerConfig`, `FieldInfo`, `MethodInfo`, `fs::OpenFlags`, `fs::CreateFlags`, `os::unix::fs::OsOpenFlags` / `OsCreateFlags`, `os::windows::fs::OsOpenFlags` / `OsCreateFlags` / `CreateFileParams`

닫힌 enum은 그대로 exhaustive입니다: `Ordering`, `Poll`, `Result`, `SeekFrom`, `Shutdown`, `Alignment`, `Sign`, `IpAddr`, `SocketAddr`, `Component`, `LoadOrdering`, `StoreOrdering`, `UpdateOrdering`, `hex::Case`, `base64::Padding`, `VarError`, `FromBytesWithNulError`, `SendError`, `TrySendError`. 두 목록을 합치면 모든 공개 enum이 들어갑니다.

`#[sealed]`([refs-lang 6.2](../../refs-lang/006-meta-system/602-built-in-attributes.md))는 라이브러리가 건전성이나 의미를 기대는 구현을 가진 트레잇에 붙습니다. 정의한 패키지만 구현할 수 있고, 바운드와 호출은 그대로입니다: `AtomicPrimitive`(2.16), `NicheInt`, `NicheFloat`(2.2), `Pinnable`(1.6). `Step`(2.7)은 사용자 범위 타입을 위해 열어 둡니다. `Pod`(1.3)는 sealed가 아니며(`#[derive(Pod)]`가 사용자 struct를 검사합니다), `bool`과 `char`는 `Pod`가 아닙니다.

## 0.5.13. async 짝

`x_async`는 블로킹하는 `x`의 async 짝입니다. 인자와 에러는 같고, `try await s.x_async(..)`처럼 await하며 task만 중단합니다. `x`가 리시버를 제자리에서 다루면 짝은 `ref self`를 받고, `x`가 리시버를 소비하면 `self`를 받습니다(`close_async`).

스트림 연산의 짝은 타입마다 따로 둔 메서드가 아니라 트레잇에서 옵니다. `Read`, `BufRead`, `Write`, `Seek`, `Close`를 구현하는 타입은 `AsyncRead`, `AsyncBufRead`, `AsyncWrite`, `AsyncSeek`, `AsyncClose`를 구현해서 `read_async`, `fill_buf_async`, `write_async`, `flush_async`, `seek_async`, `close_async`를 얻습니다([2.23.5](../200-core/223-io.md#2235-async-트레잇)). 스트림 입출력이 아닌 연산만 고유 `_async` 메서드입니다. 열기와 연결(`open_async`, `connect_async`), `accept_async`, `shutdown_async`, 데이터그램과 fd 넘기기 호출, TLS 제어, 프로세스 대기(`wait_async`, `output_async`, `status_async`)가 그렇습니다.

| 타입 | async 트레잇 | 고유 짝 |
|---|---|---|
| `File` | `AsyncRead`, `AsyncWrite`, `AsyncSeek`, `AsyncClose` | `open_async`와 다른 열기 함수, `sync_all_async`, `set_len_async`, `metadata_async`, `lock_exclusive_async`, `lock_shared_async`(`unlock`은 기다리지 않음) |
| `TcpStream` | `AsyncRead`, `AsyncWrite`, `AsyncClose` | `connect_async`, `shutdown_async` |
| `TcpListener` | `AsyncClose` | `accept_async` |
| `UdpSocket` | `AsyncClose` | `connect_async`, `send_async`, `recv_async`, `send_to_async`, `recv_from_async` |
| `Socket` | `AsyncRead`, `AsyncWrite`, `AsyncClose` | `connect_async`, `accept_async`, `shutdown_async`, `send_to_async`, `recv_from_async` |
| `UnixStream` | `AsyncRead`, `AsyncWrite`, `AsyncClose` | `connect_async`, `shutdown_async`, `send_with_fd_async`, `recv_with_fd_async` |
| `UnixListener` | `AsyncClose` | `accept_async` |
| `UnixDatagram` | `AsyncClose` | `connect_async`, `send_async`, `recv_async`, `send_with_fd_async`, `recv_with_fd_async` |
| `tls::Client`, `tls::Server` | `AsyncRead`, `AsyncWrite`, `AsyncClose` | `new_async`, `with_config_async`, `close_notify_async`, `key_update_async` |
| `net::ReadHalf<T>` | `AsyncRead`, `AsyncClose` | `shutdown_async` |
| `net::WriteHalf<T>` | `AsyncWrite`, `AsyncClose` | `shutdown_async`, `key_update_async` (TLS) |
| `ChildStdin` | `AsyncWrite`, `AsyncClose` | |
| `ChildStdout`, `ChildStderr` | `AsyncRead`, `AsyncClose` | |
| `Child` | | `wait_async` |
| `Command` | | `output_async`, `status_async` |
| `BufReader`, `BufWriter`, `LineWriter`, `Cursor` | 안쪽 스트림을 따름([2.23.6](../200-core/223-io.md#2236-버퍼와-cursor)) | |
| `io::Owned<V, R>`, `io::Chain<A, B>` | `AsyncClose`는 reader를 따름(`R`, `A`와 `B`)([2.23.5](../200-core/223-io.md#2235-async-트레잇)) | |

`Sender`의 `close`는 I/O가 아니므로 짝이 없습니다. `File`과 `Dir`의 짝은 `ref self`를 받고, `close_async`만 핸들을 소비합니다.

## 0.5.14. 문서와 속성

표준 라이브러리의 모든 공개 항목에는 `///` 문서 주석이 있습니다. 함수, 타입, 트레잇, 트레잇 메서드, 고유 메서드가 모두 해당하고, `const fn`과 `async fn`, `cfg` 뒤에 있는 항목도 마찬가지입니다. 문서는 항목이 하는 일과 돌려주는 값을 말합니다. 던지거나 `none`을 돌려주거나 panic하는 함수는 언제 그러는지 말하고, `unsafe` 함수는 호출하는 쪽이 지켜야 할 조건을 말합니다. 라이브러리 자체 검사(`rokc check --check-library`)는 `missing_docs`를 `deny`로 봅니다([refs-lang 6.2.9](../../refs-lang/006-meta-system/602-built-in-attributes.md)).

결과에 순서가 있는 함수는 그 순서를 밝힙니다.

| 항목 | 순서 |
|---|---|
| `Dict`, `Set`과 그 반복자 | 삽입 순서 |
| `Set::union`, `Set::symmetric_difference` | `self`에서 온 멤버를 그 순서대로, 이어서 `other`에만 있는 멤버를 그 순서대로 |
| `Set::intersection`, `Set::difference` | `self`의 순서 |
| `fs::walk` | 깊이 우선 전위 순서(디렉터리가 그 항목보다 먼저), 각 디렉터리의 항목은 이름순 |
| `net::resolve` | 시스템 리졸버의 순서, 선호하는 주소가 먼저 |
| `env::vars`, `env::vars_os` | OS가 환경을 담아 둔 순서, 정렬하지 않음 |
| `task::race` | 먼저 끝난 task가 이기고, 같은 깨어남에서 끝난 것으로 보이면 인덱스가 가장 작은 것이 이김. 나머지는 취소됨 |
| `task::select` | 목록 순서로 poll, 동시면 인덱스가 작은 쪽 |

`#[must_use]`([refs-lang 6.2.11](../../refs-lang/006-meta-system/602-built-in-attributes.md#6211-쓰지-않은-결과-unused_result-discardable-must_use))는 다음에 붙습니다. 타입에는 값을 어떻게 쓰는지 알려 주는 메시지가 붙습니다.

| 종류 | 항목 |
|---|---|
| 반복자 | `str`(`Chars`, `CharIndices`, `Split`, `Lines` 등), `List`(`ListIter`), `Slice`(`SliceIter`, `Windows`, `Chunks`), `SliceMut`(`SliceMutIter`), `Dict`(`Keys`, `Values`, `DictIter`), `Set`(`SetIter`), `Deque`(`DequeIter`), `BinaryHeap`(`HeapIter`), `Receiver`(`ReceiverIter`)의 반복자, `iter` 어댑터와 `TryIterator` 어댑터(`TryMap`, `MapErr`, `Results` 등), `GenIter`, `TryGenIter`, `env::Vars`, `env::VarsOs`, `fs::ReadDir`, `fs::Walk`, `io::Lines`, `io::Split`, `io::Bytes`, `net::Incoming`, `path::Components`, `path::Ancestors` |
| 스트림 | `AsyncStream`, `TryAsyncStream`, `stream` 어댑터와 조합자, `TryStream` 어댑터, `io::BytesStream`, `io::LinesStream`, `fs::ReadDirStream` |
| future | `async fn` 호출이나 `async` 블록의 값, `LockFuture`, `ReadFuture`, `WriteFuture`, `CondvarWait`, `RecvFuture`, `SendFuture`, `task::Join`, `SelectFuture` |
| 빌더와 결과 | `process::Command`, `thread::Builder`, `thread::JoinHandle`, `Result` |
| 함수 | `str`와 `Path`의 값 메서드, 모든 `Duration` 함수와 `Instant`, `SystemTime`, `BootTime`의 산술(`duration_since`, `checked_add`, `checked_sub`), `Iterator`, `DoubleEndedIterator`, `TryIterator`의 순수한 종결 메서드(`find`, `rfind`, `min`, `max`, `min_by_key`, `max_by_key`, `reduce`, `last`, `position`), `Set::union`, `intersection`, `difference`, `symmetric_difference`, 정수와 부동소수점 기본 타입, `Int`, `Float`의 모든 메서드 |

`spawn`과 `spawn_blocking`이 돌려주는 `Task`는 `#[must_use]`가 아닙니다. 기다리든 말든 작업은 실행됩니다. 가드에는 표시가 필요 없습니다. 문장에서 `MutexGuard`를 버리면 오류입니다. `#[discardable]` 함수는 refs-lang 6.2.11에 나온 것들입니다.
