# 3.8. `path::`: UTF-8 Path(Python 방식)

`rok::std::path` 모듈은 경로 표현을 정의합니다. 핵심은 UTF-8 newtype을 우선한다는 점입니다. Rust는 `Path`와 `OsStr`를 나누는데, 이 구분은 쓰기에 부담이 됩니다. Rok은 그 부담을 피하고 Python `pathlib.Path`의 쉬운 사용법을 따릅니다.

## 3.8.1. 두 타입: `Path` / `RawPath`

Rok의 경로 표현은 두 타입으로 나뉩니다.

| 타입      | 의미                                          | 사용 빈도             |
|-----------|-----------------------------------------------|-----------------------|
| `Path`    | UTF-8 경로 (95% 이상)                         | 일반 코드 기본값      |
| `RawPath` | OS 고유 바이트 (Unix 바이트 / Windows WTF-16) | 1% (UTF-8 아닌 파일명) |

대부분의 코드는 `Path`만으로 충분합니다. `RawPath`는 오래된 Unix 시스템을 다룰 때처럼 옛 방식의 바이트가 정말 필요한 경우에만 씁니다.

## 3.8.2. `Path`: UTF-8

`Path`는 불변 값입니다. `join`, `/`, `with_*`는 모두 새 `Path`를 돌려줍니다. 경로를 제자리에서 바꾸는 `PathBuf` / `push`는 없습니다. `p /= q`는 일반 복합 대입 규칙대로 `p = p / q`입니다.

`Path`의 인터페이스는 다음과 같습니다.

```rok
struct Path { /* UTF-8 str newtype, GC heap */ }

impl Path {
    fn new(s: str) -> Path;
    fn empty() -> Path;
    fn from_components<I: IntoIterator<Item = str>>(components: I) -> Path;

    // 분해
    fn parent(self) -> Path | none;
    fn file_name(self) -> str | none;
    fn file_stem(self) -> str | none;
    fn extension(self) -> str | none;
    fn components(self) -> Components;   // Iterator<Item = Component> 구현
    fn ancestors(self) -> Ancestors;     // Iterator<Item = Path> 구현

    // 조작
    fn join<P: AsPath>(self, other: P) -> Path;   // str 도 받음
    fn with_file_name(self, name: str) -> Path;
    fn with_extension(self, ext: str) -> Path;
    fn strip_prefix<P: AsPath>(self, base: P) -> Path | none;   // base 가 접두가 아니면 none

    fn is_absolute(self) -> bool;
    fn is_relative(self) -> bool;
    fn has_root(self) -> bool;
    fn starts_with<P: AsPath>(self, base: P) -> bool;
    fn ends_with<P: AsPath>(self, suffix: P) -> bool;
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

impl Component {
    fn as_str(self) -> str;     // 경로에 적힌 그대로: "/", ".", "..", 이름
}
```

## 3.8.3. `/` 연산자: Python pathlib 방식

경로 결합을 읽기 쉽게 하려고 `/` 연산자를 둡니다.

```rok
impl Div<str> for Path {
    type Output = Path;
    #[infix("/")]
    fn div(self, rhs: str) -> Path { self.join(Path::new(rhs)) }
}

impl Div<Path> for Path { ... }

impl From<str> for Path { ... }   // Path::new
impl Display for Path { ... }     // 텍스트 그대로: /etc/nginx
impl Debug for Path { ... }       // 따옴표 친 텍스트: "/etc/nginx"
```

`p /= "x"`와 `p /= q`(`q: Path`)도 쓸 수 있으며, `p = p / ..`와 같습니다.

**효과**는 다음 예에서 분명합니다.

```rok
let p = Path::new("/etc") / "nginx" / "nginx.conf";
                                          //→ "/etc/nginx/nginx.conf"

let home = env::home_dir()? / ".config" / "myapp" / "settings.toml";
```

Python의 `Path("/etc") / "nginx" / "nginx.conf"`와 **같은 사용감**입니다. Rust `PathBuf::join`과 `push`의 장황함을 피합니다.

## 3.8.3a. `AsPath`: 경로 인자

```rok
trait AsPath {
    fn as_path(self) -> Path;
}
impl AsPath for str { ... }
impl AsPath for Path { ... }
```

`fs`의 경로 인자는 `AsPath`로 받습니다. 그래서 `str` 타입과 `Path`를 모두 넘길 수 있습니다(`fs::read("a.txt")`, `fs::read(dir / "a.txt")`).

## 3.8.4. UTF-8 불변식

`Path`는 **항상 UTF-8**입니다. `Path::new(s)`는 이미 UTF-8인 `str`을 받으므로 아무것도 변환하거나 검증하지 않습니다.

OS에는 UTF-8이 아닌 이름이 있을 수 있습니다. Unix에서는 임의의 바이트, Windows의 UTF-16에서는 짝 없는 서로게이트입니다. `fs`와 `env`는 이런 이름을 손실 변환하지 않습니다. 그런 이름을 `Path`로 돌려줄 호출은 `InvalidData`인 `IoError`를 throw합니다. 코드가 다른 곳(FFI 등)에서 얻은 OS 바이트로 만든 `RawPath`(3.8.5)는 명시적으로 변환합니다.

```rok
let raw: RawPath = ...;
let p: Path = try Path::try_from(raw);    // UTF-8 이 아니면 Utf8Error 를 throw
let q: Path = try raw.try_into();          // 같음
```

이 변환은 `impl TryFrom<RawPath> for Path { type Error = Utf8Error; ... }`입니다([2.11.4](../200-core/211-conversion.md#2114-tryfromu-트레잇-실패할-수-있는-변환)).

## 3.8.5. `RawPath`: bytes / WTF-16

`RawPath`는 OS 고유 표현이 그대로 필요한 곳에서 씁니다.

```rok
struct RawPath { /* windows: List<u16>, 그 밖: List<u8> */ }   // + Clone, PartialEq, Eq, Hash, Debug

impl RawPath {
    fn from_bytes<B: AsBytes>(bytes: B) -> RawPath;    // windows 를 뺀 모든 타깃
    fn from_wide(wide: Slice<u16>) -> RawPath;         // windows

    fn as_bytes(self) -> Slice<u8>;                    // windows 를 뺀 모든 타깃
    fn as_wide(self) -> Slice<u16>;                    // windows 만

    fn to_path(self) throws (Utf8Error) -> Path;       // RawStr::to_str 와 같음
    fn to_path_lossy(self) -> Path;
}
```

`to_path`는 `str::from_utf8`, `RawStr::to_str`와 같은 에러인 `Utf8Error`([2.15.6](../200-core/215-string.md))를 던지고, Windows에서는 그 `valid_up_to()`가 UTF-16 단위를 셉니다. `to_path_lossy`는 잘못된 최대 부분열(Unix) / 짝 없는 서로게이트(Windows) 하나하나를 U+FFFD 하나로 바꿉니다.

대부분의 코드는 이 타입을 볼 일이 없습니다. OS와 직접 시스템 콜을 주고받는 곳에서만 나옵니다.

## 3.8.6. canonicalize와 normalize의 차이

두 함수는 비슷해 보이지만 동작과 비용이 분명히 다릅니다.

| 함수             | 의미                          | 파일시스템 접근     |
|------------------|-------------------------------|---------------------|
| `normalize()`    | 순수 문자열 정리 (".." 해석)  | **하지 않음**       |
| `canonicalize()` | 심볼릭 링크 해석 + 절대 경로 변환 | **함** (`realpath`) |

```rok
Path::new("/a/b/../c").normalize();              //→ "/a/c" (lexical)
try Path::new("/a/b/../c").canonicalize();       //→ symlink 해석된 절대 경로
```

`normalize`는 빠르고 시스템 콜을 하지 않으므로 90%의 경우에 충분합니다.

## 3.8.7. WASI 호환

`Path`는 WASI Preview 2의 **capability path**와도 자연스럽게 맞물립니다.

```rok
let dir: Dir = try Dir::root().open_dir("data");
let f = try dir.open(Path::new("config.toml"));
                                          // Path 가 dir 에 상대적, escape 거부
```

자세한 capability 모델은 [3.1 fs](301-fs.md)에 있습니다.

## 3.8.8. 다른 언어와 비교

설계 결정을 분명히 하려고 이웃 언어와 비교합니다.

| 측면          | Rust                      | Python `pathlib` | Go `filepath`   | Rok                         |
|---------------|---------------------------|------------------|-----------------|-----------------------------|
| 타입          | `Path` / `PathBuf`        | `Path`           | `string`        | `Path`                      |
| 소유 vs 뷰    | 빌린 `&Path` / 소유       | **불변**         | **불변**        | **불변** (`/`는 새 Path)   |
| `/` 연산자    | **없음** (`.join()` 명시) | `/`              | **없음**        | `/`                         |
| UTF-8 아님    | `OsStr` (혼합)            | `bytes` (별도)   | `string` (혼란) | `RawPath` 분리              |
| canonicalize  | `.canonicalize()`         | `.resolve()`     | `filepath.Abs`  | `.canonicalize()`           |
| 메서드 체인   | OK (대부분)               | 매우 좋음        | OK              | 매우 좋음                   |

Rok의 **가장 큰 결정**은 UTF-8 기본값과 `/` 연산자를 결합한 것입니다. 그 결과 Python pathlib의 사용감과 Rust 수준의 성능을 함께 얻습니다.
