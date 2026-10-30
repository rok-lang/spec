# 3.8. `path::` - UTF-8 Path (Python 식)

`rok::std::path` 모듈은 경로 표현을 정의합니다. 핵심은 UTF-8 newtype을 우선한다는 점입니다. Rust의 `Path`와 `OsStr` 분리가 주는 ergonomic 부담을 피하고, Python `pathlib.Path`의 쉬운 사용법을 따르기 위해서입니다.

## 3.8.1. 두 타입 - `Path` / `RawPath`

Rok의 경로 표현은 두 타입으로 나뉩니다.

| 타입      | 의미                                          | 사용 빈도             |
|-----------|-----------------------------------------------|-----------------------|
| `Path`    | UTF-8 경로 (95%+)                             | 일반 코드 default     |
| `RawPath` | OS native bytes (Unix bytes / Windows WTF-16) | 1% (non-UTF-8 파일명) |

대부분의 코드는 `Path`만으로 충분합니다. `RawPath`는 오래된 unix 시스템을 다룰 때처럼 legacy bytes가 정말 필요한 경우에만 씁니다.

## 3.8.2. `Path` - UTF-8

`Path`는 불변 값입니다. `join`, `/`, `with_*`는 모두 새 `Path`를 돌려주며, 경로를 제자리에서 바꾸는 `PathBuf` / `push`는 없습니다. `p /= q`는 일반 복합 대입 규칙대로 `p = p / q`입니다.

`Path`의 인터페이스는 다음과 같습니다.

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

    // OS 상호작용 (편의, 내부적으로 fs::*)
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

## 3.8.3. `/` 연산자 - Python pathlib 식

경로 결합을 읽기 쉽게 하려고 `/` 연산자를 둡니다.

```rok
impl Div<str> for Path {
    type Output = Path;
    #[infix("/")]
    fn div(self, rhs: str) -> Path { self.join(Path::from(rhs)) }
}

impl Div<Path> for Path { ... }
```

`p /= "x"`와 `p /= q`(`q: Path`)도 쓸 수 있으며, `p = p / ..`와 같습니다.

**효과**는 다음 예시에서 분명합니다.

```rok
let p = Path::from("/etc") / "nginx" / "nginx.conf";
                                          //→ Path("/etc/nginx/nginx.conf")

let home = env::home_dir()? / ".config" / "myapp" / "settings.toml";
```

Python의 `Path("/etc") / "nginx" / "nginx.conf"`와 **같은 ergonomics**이며, Rust `PathBuf::join`과 `push`의 장황함을 피합니다.

## 3.8.3a. `AsPath` - 경로 인자

```rok
trait AsPath {
    fn as_path(self) -> Path;
}
impl AsPath for str { ... }
impl AsPath for Path { ... }
```

`fs`의 경로 인자는 `AsPath`로 받으므로 `str` 타입과 `Path`를 모두 넘길 수 있습니다(`fs::read("a.txt")`, `fs::read(dir / "a.txt")`).

## 3.8.4. UTF-8 invariant

`Path`는 **항상 UTF-8**입니다. `Path::from(s)` 시점에 `s`는 이미 UTF-8이므로 별도 검증이 필요 없습니다.

다만 드물게 OS가 non-UTF-8 파일명을 돌려줄 때가 있습니다.
- Unix에서는 임의의 bytes가 가능하므로 `Path::from(...)`은 **lossy 변환**(replacement char `U+FFFD`)을 합니다.
- Windows에서는 이름이 WTF-16이며, surrogate pair를 따로 처리합니다.

엄격한 변환이 필요하면 다음과 같이 씁니다.

```rok
let raw: RawPath = ...;
let p: Path = try Path::try_from(raw);    // UTF-8 이 아니면 NonUtf8Error 를 throw
let q: Path = try raw.try_into();          // 같음
```

이 변환은 `impl TryFrom<RawPath> for Path { type Error = NonUtf8Error; ... }`입니다([2.11.4](../200-core/211-conversion.md#2114-tryfromu-트레잇---fallible-변환)).

## 3.8.5. `RawPath` - bytes / WTF-16

`RawPath`는 OS의 native 표현이 그대로 필요한 곳에서 씁니다.

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

Windows에서 `NonUtf8Error.valid_up_to`는 UTF-16 단위를 셉니다. `to_path_lossy`는 잘못된 maximal subsequence (Unix) / 짝 없는 surrogate (Windows) 하나하나를 U+FFFD 하나로 바꿉니다.

대부분의 코드는 이 타입을 볼 일이 없습니다. OS와 직접 syscall을 주고받는 곳에서만 나옵니다.

## 3.8.6. canonicalize / normalize 차이

두 함수는 비슷해 보이지만 동작과 비용이 분명히 다릅니다.

| 함수             | 의미                          | 파일시스템 접근     |
|------------------|-------------------------------|---------------------|
| `normalize()`    | 순수 lexical 정리 (".." 해석) | **하지 않음**       |
| `canonicalize()` | symlink 해석 + 절대 경로 변환 | **함** (`realpath`) |

```rok
Path::from("/a/b/../c").normalize();              //→ "/a/c" (lexical)
try Path::from("/a/b/../c").canonicalize();       //→ symlink 해석된 절대 경로
```

`normalize`는 빠르고 syscall을 하지 않으므로 90%의 경우에 충분합니다.

## 3.8.7. WASI 호환

`Path`는 WASI Preview 2의 **capability path**와도 자연스럽게 맞물립니다.

```rok
let dir: Dir = try Dir::root().open_dir("data");
let f = try dir.open(Path::from("config.toml"));
                                          // Path 가 dir 에 상대적, escape 거부
```

자세한 capability 모델은 [3.1 fs](301-fs.md)에서 다룹니다.

## 3.8.8. 다른 언어와의 비교

설계 결정을 분명히 하기 위해 인접 언어와 비교합니다.

| 측면          | Rust                      | Python `pathlib` | Go `filepath`   | Rok                         |
|---------------|---------------------------|------------------|-----------------|-----------------------------|
| 타입          | `Path` / `PathBuf`        | `Path`           | `string`        | `Path`                      |
| owned vs view | borrowed `&Path` / owned  | **불변**         | **불변**        | **불변** (`/`는 새 Path)   |
| `/` 연산자    | **없음** (`.join()` 명시) | `/`              | **없음**        | `/`                         |
| non-UTF-8     | `OsStr` (혼합)            | `bytes` (별도)   | `string` (혼란) | `RawPath` 분리              |
| canonicalize  | `.canonicalize()`         | `.resolve()`     | `filepath.Abs`  | `.canonicalize()`           |
| 메서드 chain  | OK (대부분)               | excellent        | OK              | excellent                   |

Rok의 **가장 큰 결정**은 UTF-8 default와 `/` 연산자를 결합한 것입니다. 이로써 Python pathlib의 ergonomics와 Rust 수준의 performance를 함께 얻습니다.
