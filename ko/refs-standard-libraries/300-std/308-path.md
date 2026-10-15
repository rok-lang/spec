# 3.8. `path::` — UTF-8 Path (Python 식)

`rok::std::path` 모듈은 경로 표현을 정의합니다. 여기서 핵심은 UTF-8 newtype 을 우선시한다는 점인데, 이는 Rust 의 `Path` 와 `OsStr` 분리가 가져오는 ergonomic 부담을 회피하면서, 동시에 Python `pathlib.Path` 의 직관성을 채택하기 위함입니다.

## 3.8.1. 두 타입 — `Path` / `RawPath`

rok 의 경로 표현은 두 개의 타입으로 나뉩니다.

| 타입      | 의미                                          | 사용 빈도             |
|-----------|-----------------------------------------------|-----------------------|
| `Path`    | UTF-8 경로 (95%+)                             | 일반 코드 default     |
| `RawPath` | OS native bytes (Unix bytes / Windows WTF-16) | 1% (non-UTF-8 파일명) |

대부분의 코드는 `Path` 만 사용하면 충분합니다. `RawPath` 가 필요해지는 경우는 legacy bytes 가 정말로 요구되는 자리, 예컨대 오래된 unix 시스템과 마주칠 때에 한정됩니다.

## 3.8.2. `Path` — UTF-8

`Path` 는 불변 값입니다. `join`, `/`, `with_*` 는 모두 새 `Path` 를 돌려주며, 경로를 제자리에서 바꾸는 `PathBuf` / `push` 는 없습니다. `p /= q` 는 일반 복합 대입 규칙대로 `p = p / q` 입니다.

`Path` 의 인터페이스는 다음과 같이 구성됩니다.

```rok
struct Path { /* UTF-8 str newtype, GC heap */ }

impl Path {
    fn new() -> Path;
    fn from(s: str) -> Path;
    fn from_components<I: Iterator<Item = str>>(components: I) -> Path;

    // 분해
    fn parent(self) -> Path | none;
    fn file_name(self) -> str | none;
    fn file_stem(self) -> str | none;
    fn extension(self) -> str | none;
    fn components(self) -> Components;   // Iterator<Item = Component> 구현
    fn ancestors(self) -> Ancestors;     // Iterator<Item = Path> 구현

    // 조작
    fn join(self, other: Path) -> Path;
    fn with_file_name(self, name: str) -> Path;
    fn with_extension(self, ext: str) -> Path;
    fn strip_prefix(self, base: Path) throws (StripPrefixError) -> Path;

    fn is_absolute(self) -> bool;
    fn is_relative(self) -> bool;
    fn has_root(self) -> bool;
    fn starts_with(self, base: Path) -> bool;
    fn ends_with(self, suffix: Path) -> bool;
    fn normalize(self) -> Path;             // resolve "." / ".." (lexical)

    // OS 상호작용 (편의 — 내부적으로 fs::*)
    fn exists(self) -> bool;
    fn is_file(self) -> bool;
    fn is_dir(self) -> bool;
    fn is_symlink(self) -> bool;
    fn metadata(self) throws (IoError) -> Metadata;
    fn canonicalize(self) throws (IoError) -> Path;     // realpath (filesystem 검증)

    fn as_str(self) -> str;
    fn to_raw(self) -> RawPath;
}

enum Component {
    RootDir,                    // "/"
    Prefix(str),                // "C:" (Windows)
    CurDir,                     // "."
    ParentDir,                  // ".."
    Normal(str),                // "data"
}
```

## 3.8.3. `/` 연산자 — Python pathlib 식

경로 결합의 가독성을 한층 끌어올리기 위해 `/` 연산자를 도입합니다.

```rok
impl Div<str> for Path {
    type Output = Path;
    #[infix("/")]
    fn div(self, rhs: str) -> Path { self.join(Path::from(rhs)) }
}

impl Div<Path> for Path { ... }
```

`p /= "x"` 와 `p /= q` (`q: Path`) 도 쓸 수 있으며, `p = p / ..` 와 같습니다.

**효과** 는 다음 예시에서 분명히 드러납니다.

```rok
let p = Path::from("/etc") / "nginx" / "nginx.conf";
                                          //→ Path("/etc/nginx/nginx.conf")

let home = env::home_dir()? / ".config" / "myapp" / "settings.toml";
```

이는 Python 의 `Path("/etc") / "nginx" / "nginx.conf"` 와 **동일한 ergonomics** 를 제공합니다. 그 결과 Rust `PathBuf::join` 과 `push` 의 verbose 한 표현을 자연스럽게 회피하게 됩니다.

## 3.8.3a. `AsPath` — 경로 인자

```rok
trait AsPath {
    fn as_path(self) -> Path;
}
impl AsPath for str { ... }
impl AsPath for Path { ... }
```

`fs` 의 경로 인자는 `AsPath` 로 받으므로 `str` 과 `Path` 를 모두 넘길 수 있습니다 (`fs::read("a.txt")`, `fs::read(dir / "a.txt")`).

## 3.8.4. UTF-8 invariant

`Path` 는 **항상 UTF-8** 입니다. 따라서 `Path::from(s)` 시점에 `s` 가 이미 UTF-8 이므로 별도 검증이 필요하지 않습니다.

다만 OS 가 non-UTF-8 파일명을 반환하는 드문 상황도 존재합니다.
- Unix 의 경우 arbitrary bytes 가 가능하므로 `Path::from(...)` 은 **lossy 변환** (replacement char `U+FFFD`) 을 수행합니다.
- Windows 의 경우 WTF-16 이며, surrogate pair 처리가 별도로 이루어집니다.

엄격한 변환이 필요하다면 다음과 같이 작성합니다.

```rok
let raw: RawPath = ...;
let p: Path = try Path::try_from(raw);    // UTF-8 이 아니면 NonUtf8Error 를 throw
```

`Path::try_from` 은 지금은 `Path` 의 고유 함수입니다. 표준의 `TryFrom` 트레잇은 에러 타입이 `OverflowError` 로 고정되어 있어 ([2.11.4](../200-core/211-conversion.md#2114-tryfromu-트레잇--fallible-변환)) `NonUtf8Error` 를 던지는 이 변환은 트레잇 impl 이 아닙니다.

## 3.8.5. `RawPath` — bytes / WTF-16

OS 의 native 표현이 그대로 필요한 자리를 위해 `RawPath` 가 마련되어 있습니다.

```rok
struct RawPath { /* unix: Vec<u8>, windows: Vec<u16> */ }

impl RawPath {
    fn from_bytes(bytes: List<u8>) -> RawPath;        // unix
    fn from_wide(wide: List<u16>) -> RawPath;          // windows

    fn as_bytes(self) -> Slice<u8>;                    // unix 만
    fn as_wide(self) -> Slice<u16>;                    // windows 만

    fn try_to_path(self) throws (NonUtf8Error) -> Path;
    fn to_path_lossy(self) -> Path;
}
```

대부분의 코드는 이 타입을 마주칠 일이 없습니다. OS 와 직접 syscall 을 주고받는 자리에서만 등장한다고 보면 됩니다.

## 3.8.6. canonicalize / normalize 차이

이 두 함수는 비슷해 보이지만, 동작 방식과 비용 측면에서 분명히 구분됩니다.

| 함수             | 의미                          | 파일시스템 접근     |
|------------------|-------------------------------|---------------------|
| `normalize()`    | 순수 lexical 정리 (".." 해석) | **하지 않음**       |
| `canonicalize()` | symlink 해석 + 절대 경로 변환 | **함** (`realpath`) |

```rok
Path::from("/a/b/../c").normalize();              //→ "/a/c" (lexical)
try Path::from("/a/b/../c").canonicalize();       //→ symlink 해석된 절대 경로
```

`normalize` 는 빠르고 syscall 을 동반하지 않으므로 90% 의 자리에서 충분히 제 역할을 해냅니다.

## 3.8.7. WASI 호환

`Path` 는 WASI Preview 2 의 **capability path** 와도 자연스럽게 통합됩니다.

```rok
let dir: Dir = try Dir::root().open_dir("data");
let f = try dir.open(Path::from("config.toml"));
                                          // Path 가 dir 에 상대적 — escape 거부
```

자세한 capability 모델은 [3.1 fs](301-fs.md) 에서 다룹니다.

## 3.8.8. 다른 언어와의 비교

설계 결정의 좌표를 명확히 하기 위해, 인접 언어들과 비교해 보자.

| 측면          | Rust                      | Python `pathlib` | Go `filepath`   | rok                         |
|---------------|---------------------------|------------------|-----------------|-----------------------------|
| 타입          | `Path` / `PathBuf`        | `Path`           | `string`        | `Path`                      |
| owned vs view | borrowed `&Path` / owned  | **불변**         | **불변**        | **불변** (`/` 는 새 Path)   |
| `/` 연산자    | **없음** (`.join()` 명시) | `/`              | **없음**        | `/`                         |
| non-UTF-8     | `OsStr` (혼합)            | `bytes` (별도)   | `string` (혼란) | `RawPath` 분리              |
| canonicalize  | `.canonicalize()`         | `.resolve()`     | `filepath.Abs`  | `.canonicalize()`           |
| 메서드 chain  | OK (대부분)               | excellent        | OK              | excellent                   |

요약하자면 rok 의 **가장 큰 결정** 은 UTF-8 default 와 `/` 연산자의 결합입니다. 이로써 Python pathlib 의 ergonomics 와 Rust 수준의 performance 를 함께 가져갑니다.
