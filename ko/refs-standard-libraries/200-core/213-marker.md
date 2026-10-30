# 2.13. marker: `Clone` / `Copy` / `Drop`

값을 복제하는 일과 더 이상 필요 없는 자원을 정리하는 일은 어느 언어에서나 다뤄야 합니다. `rok::core::marker` 모듈은 이 두 문제를 트레잇 세 개로 표현합니다. `Clone`은 값의 독립된 사본을 만드는 방법을 정의합니다. `Copy`는 비트 복사만으로 충분한 타입임을 나타내는 마커입니다. `Drop`은 값이 스코프를 떠날 때 실행해야 하는 정리 코드를 정의합니다. 이 밖에 `none` 리터럴 타입의 라이브러리 이름인 `NoneSentinel`(2.13.8)과 스레드 사이로 넘기는 값을 감싸는 `Handoff`(2.13.9)도 이 모듈에 있습니다.

Rok은 GC를 쓰므로 이 트레잇의 의미도 달라집니다. 메모리 회수는 GC가 맡으므로 `Drop`은 메모리 해제와 관계가 없습니다. `Drop`은 파일 핸들이나 소켓처럼 언어 바깥에 있는 자원을 정리하는 일(finalization)에만 씁니다. 그래서 Rok의 마커 트레잇은 Rust와도, Java/Python과도 다릅니다.

## 2.13.1. 메모리 모델: GC가 있는 언어의 Clone/Drop

Rok 런타임에서는 [Immix + Sticky Mark-Bit Generational GC](../../refs-lang/007-runtime/703-gc-algorithm.md)가 힙 객체의 수명을 자동으로 관리합니다. 객체가 언제 메모리에서 사라질지는 사용자가 정하지 않습니다. 하지만 외부 자원은 결정론적으로 정리해야 하므로, Rok의 `Drop`은 GC와 별개인 메커니즘으로 동작합니다. Rust와 비교하면 다음과 같습니다.

| 측면              | Rust (no GC)            | Rok (GC)                                        |
|-------------------|-------------------------|-------------------------------------------------|
| 메모리 회수       | `Drop`이 책임          | GC가 자동으로, `Drop`은 **관여하지 않음**      |
| `Drop::drop` 의미 | 자원 해제 + 메모리 해제 | **외부 자원 정리만** (파일 핸들, 소켓 등)      |
| 호출 시점         | 결정론적 (스코프 종료)  | **`using` 블록을 빠져나갈 때만** (GC 수집과 무관) |
| 누락 시 영향      | 자원 누수 + 메모리 누수 | 자원 누수만 (메모리는 GC가 결국 회수)          |

> [!NOTE]
> 일반 바인딩은 스코프를 벗어나도 자동으로 정리되지 않습니다. `Drop::drop`은 `using`으로 묶은 바인딩에서만 호출됩니다([refs-lang 3.17 `using`](../../refs-lang/003-expressions/317-using-expression.md)). GC는 다른 객체의 정리를 보장하지 않으며, 어떤 경우에도 `drop`을 부르지 않습니다.

메모리 회수와 자원 정리가 분리되어 있으므로 일반 객체는 `Drop`을 신경 쓰지 않아도 됩니다(GC가 회수합니다). 외부 자원을 다루는 타입만 `Drop`을 구현해 정리 시점을 결정론적으로 정합니다.

## 2.13.2. `Clone`: 깊은 복제

`Clone::clone`은 값의 독립된 사본이 필요할 때 부르는 메서드입니다. 시그니처는 단순합니다.

```rok
#[lang("clone_trait")]
trait Clone {
    fn clone(self) -> Self;
}
```

`clone`은 깊은 복제입니다. 값 안에 참조가 있으면 참조가 가리키는 데이터까지 복사합니다. 그래서 복제한 뒤에는 양쪽을 따로 바꿔도 서로 영향을 주지 않습니다. 단, GC가 불변으로 보장하는 공유 데이터는 그대로 공유해도 됩니다.

### `Clone`의 의미

가장 단순한 `Clone` 예제입니다.

```rok
let a: List<i32> = [1, 2, 3];
let b = a.clone();           // 새 List, 내부 buffer 도 새로
b.push(4);                   // a 영향 없음
a;                           //→ [1, 2, 3]
b;                           //→ [1, 2, 3, 4]
```

여기서 '깊다'는 정확히 이런 뜻입니다. `Clone`은 논리적 동등성을 유지하면서, 두 값을 서로 독립적으로 바꿀 수 있도록 보장합니다. 여기서 두 성질이 나옵니다. `let b = a.clone();` 직후에는 `PartialEq`가 정의되어 있으면 `a == b`가 성립합니다. 그 뒤로 `a`를 어떻게 바꿔도 `b`에 반영되지 않으며, 반대도 마찬가지입니다.

### 자동 호출 없음: `clone_trait` lang slot

어떤 언어에서는 `let y = x;`만 써도 컴파일러가 `clone`을 호출합니다. Rok은 명시적인 방식을 택했습니다.

```rok
let xs: List<i32> = [1, 2, 3];
let ys = xs;          // GC heap 객체 → 참조 공유 (clone 안 일어남)
```

`let y = x;`는 GC 힙 객체면 참조를 공유하고, `Copy` 타입이면 비트를 복사합니다. 따라서 `Clone::clone`은 자동으로 디스패치되지 않고, 사용자가 명시적으로 호출할 때만 실행됩니다. 그래서 `Clone`은 메서드 슬롯이 아니라 트레잇 슬롯 `clone_trait`로 등록됩니다. 컴파일러는 이 슬롯을 `#[derive(Clone)]` 확장에서만 찾습니다.

### 기본 본문

원시 타입의 `clone`은 비트 복사일 뿐이므로 intrinsic에 바로 대응합니다. 컬렉션의 `clone`은 내부 버퍼를 새로 할당하고 원소를 재귀적으로 복제합니다.

```rok
impl Clone for i32 {
    #[intrinsic("rok.clone.copy")]    // bitwise copy
    fn clone(self) -> i32;
}

impl<T: Clone> Clone for List<T> {
    fn clone(self) -> List<T> {
        let r: List<T> = [];
        for x in self {
            r.push(x.clone());
        }
        r
    }
}
```

GC 객체를 복제하면 항상 새 버퍼를 할당하고 원소를 재귀적으로 복제합니다.

### `#[derive(...)]`

모든 필드가 `Clone`인 사용자 정의 struct는 대부분 본문을 직접 쓸 이유가 없습니다. 이때는 `#[derive(Clone)]`이 본문을 대신 만듭니다.

```rok
#[derive(Clone)]
struct Config {
    name: str;
    timeout: u32;
    retries: u32;
}

let c1 = Config { name: "default", timeout: 30, retries: 3 };
let c2 = c1.clone();         // 모든 필드 재귀 clone
```

derive가 만드는 본문은 모든 필드에 `Clone::clone`을 호출하고, 그 결과를 같은 구조로 묶어 돌려줍니다. 따라서 모든 필드가 재귀적으로 `Clone`을 구현해야 합니다.

## 2.13.3. `Copy`: 마커

```rok
trait Copy: Clone {
    // marker: 메서드 없음
}
```

Rok에서는 모든 값이 값으로 복사됩니다. 컬렉션이나 GC 객체는 핸들이 복사되어 같은 객체를 가리킵니다. 원시 값, `str`(불변 `(ptr, len)`), struct, enum, 튜플은 그 자체가 복사됩니다([refs-lang 2.3.8](../../refs-lang/002-types/203-struct.md#238-값-의미)). 따라서 `Copy`는 **마커일 뿐**이며 대입이나 인자 전달의 의미를 바꾸지 않습니다. `Copy`는 제네릭 바운드(`T: Copy`)에서 '비트 복사가 곧 독립된 사본'인 타입을 고를 때만 씁니다.

### impl 정책

| 타입                             | `Clone`     | `Copy`           | 비고                                 |
|----------------------------------|-------------|------------------|--------------------------------------|
| `i*`, `u*`, `f*`                 | ✓           | ✓                | 원시 숫자                            |
| `bool`, `char`, `c_char`         | ✓           | ✓                |                                      |
| `str`                            | ✓           | ✓                | 불변 `(ptr, len)` 값                 |
| `int`, `float`                   | ✓           | ✓                | 불변 값, 힙의 BigInt도 공유만      |
| `List<T>`, `Dict<K,V>`, `Set<T>` | ✓ (T:Clone) | ✗                | 핸들 복사는 공유, 독립 사본 아님     |
| 튜플                             | ✓ (모두)    | ✓ (모두 Copy 시) | 컴포넌트별                           |
| `Range<T>`                       | ✓ (T:Clone) | ✓ (T:Copy)       | start/end 둘 다 Copy 시              |
| 사용자 struct / enum             | derive      | derive           | 모든 필드 같은 트레잇 구현 시        |

`Drop`을 구현한 타입은 `Copy`가 될 수 없습니다.

```rok
#[derive(Clone, Copy)]
struct Vec3 { x: f64; y: f64; z: f64; }
```

> [!IMPORTANT]
> `Copy`의 derive는 `Clone`도 함께 구현하고 모든 필드가 `Copy`일 때만 쓸 수 있습니다.

## 2.13.4. `Drop`: `using` 해제 시 정리

파일을 열었으면 닫아야 하고, 뮤텍스를 잠갔으면 풀어야 합니다. 이런 외부 자원 정리는 GC에 맡길 수 없습니다. 언제 올지 모르는 GC 스윕까지 자원을 붙들고 있으면 파일 디스크립터가 바닥날 수 있습니다. 풀리지 않은 잠금 때문에 다른 코드가 멈출 수도 있습니다. `Drop` 트레잇과 `using` 표현식이 이 문제를 풉니다.

```rok
trait Drop {
    #[lang("drop_method")]
    fn drop(ref self);
}
```

### 호출 시점: `using` 바인딩에서만

Rok은 GC 언어이므로 일반 바인딩이 스코프를 벗어날 때 `drop`을 자동으로 부르지 않습니다. `drop`은 `using`으로 묶은 바인딩에서만, 블록을 빠져나갈 때 호출됩니다([refs-lang 3.17 `using`](../../refs-lang/003-expressions/317-using-expression.md)).

- 정상 종료, `return`, `break`, 에러 전파 모두에서 호출됩니다.
- 여러 바인딩은 선언의 역순(LIFO)으로 호출됩니다.
- panic은 프로세스를 끝내므로 `drop`을 부르지 않습니다.
- drop glue: 값 자신의 `drop`이 끝나면 `Drop`이 있는 필드를 선언의 역순으로 재귀적으로 정리합니다. 튜플 원소와 enum / 합 타입이 지금 담은 페이로드도 같습니다.

```rok
fn process(path: str) throws (IoError) {
    using lock = MUTEX.lock(), file = try File::open(path) {
        do_work(file);
    }   // ← 정상 종료든 에러든 file.drop() 다음 lock.drop()
}

{
    let f = try! File::open("data.txt");
}   // ← 자동 drop 없음: f 는 GC 가 회수하지만 File::drop 은 불리지 않음
```

`Drop`을 구현하지 않은 타입을 `using`으로 묶으면 컴파일 에러입니다. 에러 메시지는 `Drop`이 필요하다고 알립니다.

### GC와 관계

GC는 `drop`을 부르지 않으며 finalizer도 없습니다. 정리가 필요한 자원은 `using`으로 묶거나 `close()` 같은 메서드로 직접 닫아야 합니다. 그러지 않으면 `unclosed_resource` 린트가 경고합니다([refs-lang 6.2.9](../../refs-lang/006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)).

### `Drop` 구현 예

가장 전형적인 구현은 OS 자원을 감싼 래퍼입니다.

```rok
struct File {
    handle: i32;
}

impl Drop for File {
    fn drop(ref self) {
        rok::bare::close_fd(self.handle);     // OS file descriptor close
    }
}
```

### `Drop`의 제약

`Drop`에는 몇 가지 제약이 있습니다. 먼저 `Drop`을 구현한 타입은 `Copy`가 될 수 없습니다. 비트 복사로 사본을 마음대로 늘리면서 정리를 결정론적으로 보장할 수는 없기 때문입니다.

```rok
#[derive(Copy)]              // ERROR - Drop 과 충돌
struct File { handle: i32; }
impl Drop for File { ... }
```

그다음 `drop`을 직접 호출하면 컴파일 에러입니다. `using` 해제 시점에 정확히 한 번 호출되어야 정확성이 보장되기 때문입니다.

```rok
let f = File::open(path);
f.drop();                    // ERROR: Drop::drop 직접 호출 금지
```

자원을 일찍 닫으려면 `File::close()`처럼 별도의 메서드를 두세요.

예외는 `rok::bare` 안의 unsafe 코드뿐입니다. 예를 들어 `unsafe fn drop_in_place`([1.3 mem](../100-bare/103-mem.md))는 수동 메모리 관리용 함수입니다. 이 함수는 drop glue를 실행합니다. 위와 같이 그 타입 자신의 `drop`을 먼저 부르고, 그다음 필드의 `drop`을 선언 역순으로 부릅니다.

또한 `drop`에는 `throws` 시그니처가 없으므로 본문에서 에러를 던질 수 없습니다. `drop` 안에서 panic이 일어나면 다른 panic과 마찬가지로 프로세스가 끝납니다.

```rok
impl Drop for X {
    fn drop(ref self) {
        if some_cond { throw "oops" }    // ERROR: drop 은 throws 가 아님
    }
}
```

### derive 정책

`Clone`과 달리 `Drop`은 derive로 구현할 수 없습니다. `using`으로 묶인 값의 `drop`이 끝나면, 컴파일러는 그 struct에서 `Drop`이 있는 필드의 `drop`을 선언 역순으로 이어서 호출합니다. 따라서 사용자가 쓰는 `Drop` 본문은 '필드 말고 더 정리해야 할 것'만 다루면 됩니다.

```rok
struct Container {
    a: File;
    b: Socket;
}
// Container 를 using 으로 묶으면 해제 시 b.drop() → a.drop()
// (필드 자체가 Drop 구현이라)

impl Drop for Container {
    fn drop(ref self) {
        log("Container released");
        // 필드 drop 은 _이 메서드 종료 후_ 컴파일러가 자동 호출
    }
}
```

## 2.13.5. lang slot 매핑

이 모듈은 lang slot 네 개로 컴파일러와 연결됩니다. 자동 호출 시점은 앞에서 설명한 정책 그대로입니다.

| slot          | 부착 항목                 | 자동 호출 시점                           |
|---------------|---------------------------|------------------------------------------|
| `clone_trait` | `Clone` 트레잇            | 없음, `#[derive(Clone)]` 확장에서 사용   |
| `drop_method` | `Drop::drop` 메서드       | `using` 바인딩 해제 시 (drop glue 포함)  |
| `send_trait`  | `Send` 트레잇 (2.13.9)    | 자동 구현, 스레드 경계 검사              |
| `sync_trait`  | `Sync` 트레잇 (2.13.9)    | 자동 구현, 스레드 경계 검사              |

자세한 내용은 [0.4 lang slot 표](../000-meta/004-lang-slot-table.md)의 0.4.2 / 0.4.3 / 0.4.5 절에 있습니다.

## 2.13.6. 트레잇 구현 매트릭스

표준 라이브러리 주요 타입과 세 트레잇의 관계는 다음과 같습니다.

| 타입                             | Clone       | Copy          | Drop                            | 동작                                 |
|----------------------------------|-------------|---------------|---------------------------------|--------------------------------------|
| `i*`, `u*`, `f*`                 | ✓           | ✓             | ✗                               | `rok.clone.copy` (비트 복사)         |
| `bool`, `char`, `c_char`         | ✓           | ✓             | ✗                               | 비트 복사                            |
| `str`                            | ✓           | ✓             | ✗                               | 불변 `(ptr, len)` 값 복사            |
| `List<T>`, `Dict<K,V>`, `Set<T>` | ✓ (T:Clone) | ✗             | ✗                               | 재귀 복제                            |
| 튜플                             | ✓ (모두)    | ✓ (모두 Copy) | ✗                               | 컴포넌트별                           |
| `Range<T>`                       | ✓ (T:Clone) | ✓ (T:Copy)    | ✗                               | derive 자동                          |
| `File`                           | ✗           | ✗             | ✓                               | `rok::bare::close_fd`                 |
| `MutexGuard<T>`                  | ✗           | ✗             | ✓ (`using` 해제 시)             | `rok::bare::unlock`                   |
| `Channel::Sender<T>`             | ✓           | ✗             | ✗ (`close()`로 닫음)           | (위와 동일)                          |

표의 원칙은 단순합니다. 모든 원시 타입과 내부 GC 타입은 `Clone`을 구현하고, 그중 비트 복사가 의미상 맞는 것만 `Copy`입니다. 컬렉션은 `Clone`이지만 `Copy`는 아닙니다. 핸들을 비트 복사하면 깊은 복제의 의미가 지켜지지 않기 때문입니다. `str`은 불변이므로 `(ptr, len)`을 복사하면 곧 독립된 사본이 됩니다. 그래서 `Copy`입니다. `Drop`은 `File`, `Socket`, `MutexGuard`처럼 외부 자원을 감싸는 래퍼만 구현합니다. 이런 타입은 자원을 중복으로 소유하지 않도록 보통 `Clone`을 구현하지 않습니다.

## 2.13.7. 다른 언어와 비교

Rok의 모델을 기존 언어와 비교하면 설계 의도가 더 분명해집니다.

| 측면              | Rust                     | Java/Python (GC)            | Rok (GC + Drop)          |
|-------------------|--------------------------|-----------------------------|--------------------------|
| 메모리 회수       | `Drop`이 책임           | GC                          | GC                       |
| 결정론적 finalize | `Drop` (컴파일러 보장)   | **없음** (finalizer 비결정) | `Drop` (스코프 한정)     |
| Copy 의미         | bitwise + `!Drop` 강제   | **없음** (모든 객체가 참조) | bitwise + `!Drop` 강제   |
| Clone 의미        | 깊은 복제                | `clone()` (Object.clone)    | 깊은 복제                |
| derive            | `#[derive(Clone, Copy)]` | `Cloneable` 인터페이스      | `#[derive(Clone, Copy)]` |

Rok의 특징은 GC와 결정론적 `Drop`이 함께 있다는 점입니다. 메모리 회수는 GC가 맡으므로 `Drop`은 메모리 해제를 뜻하지 않습니다. 일반 객체는 `Drop`을 구현하지 않아도 됩니다. 외부 자원 정리가 필요한 타입만 `Drop`을 구현하고, 쓰는 쪽은 `using`으로 묶습니다.

이 모델은 Java/Python finalizer의 비결정성과, GC가 없는 Rust가 치르는 비용을 함께 피합니다.

## 2.13.8. `NoneSentinel`: `none` 리터럴의 타입

[refs-lang 1.9.2](../../refs-lang/001-lexical/109-other-literals.md)의 `none` 리터럴의 타입은 컴파일러에 내장된 `none` 타입입니다. 이 타입은 값이 하나뿐인 크기 0 타입입니다. `rok::core::marker`는 타입 별칭으로 이 타입에 `NoneSentinel`이라는 라이브러리 이름만 붙입니다. lang slot은 관여하지 않습니다.

```rok
pub type NoneSentinel = none;
```

이 이름을 직접 쓸 일은 거의 없습니다. `T | none` 같은 합 타입 표기([refs-lang 2.6](../../refs-lang/002-types/206-sum-types.md))에서 `none` 자리가 곧 이 타입입니다. `none` 리터럴, `none` 타입 표기, `NoneSentinel`은 모두 같은 내장 타입을 가리킵니다.

## 2.13.9. `Send` / `Sync`: 스레드 경계 마커

```rok
#[lang("send_trait")]
trait Send { }      // 다른 스레드로 옮겨도 안전

#[lang("sync_trait")]
trait Sync { }      // 공유 참조로 여러 스레드에서 읽어도 안전
```

두 트레잇은 메서드가 없는 **자동 마커 트레잇**이며, 컴파일러가 구조적으로 구현합니다.

- 원시 숫자, `bool`, `char`, `c_char`, `str`, `none`은 `Send + Sync`입니다.
- `struct`, `enum`, `union`, 튜플, 배열, 합 타입은 모든 필드(구성 요소)가 `Send`이면 `Send`, 모두 `Sync`이면 `Sync`입니다. 제네릭 컨테이너의 `Send`는 같은 규칙으로 `T`를 따릅니다.
- 컬렉션(`List`, `Dict`, `Set`, `StrBuf`, `Deque`, `BinaryHeap`, `OrderedDict`, `OrderedSet`, `Slice`)은 `Send`이고 `!Sync`입니다. 가변 핸들을 공유하기 때문입니다. 스레드 사이에서는 `Mutex` / `RwLock` / 채널로 공유합니다.
- Rok에는 move 의미론이 없습니다. 값을 넘겨도(`Mutex::new(xs)`, `RwLock::new(xs)`, `tx.send(xs)` / `try_send` / `send_async`, `Handoff::new(xs)`) 보낸 쪽의 핸들은 그대로 남습니다. 그 핸들은 새 소유자가 이제 다른 스레드와 공유하는 객체를 가리킵니다. 가드에서 꺼낸(`get`) 핸들, 가드에 넣은(`set`) 핸들, `update`의 클로저가 닿은 핸들도 가드의 `using` 블록이 끝난 뒤 여전히 보호된 객체를 가리킵니다. 그 뒤에 그런 값을 어느 경로에서든 쓰면 컴파일 에러(`E202`, [refs-lang 9.9.4](../../refs-lang/009-tools/909-debug-tools.md#994-진단--에러-메시지))입니다. 컴파일러는 다음 경로로 값을 추적합니다.
  - 복사(`let ys = xs`, struct 필드나 튜플 원소, 필드 대입)
  - 그 값을 넘기는 필드(`Mutex::new(s.items)`)
  - 그 값을 담는 객체(`all.push(xs)`, 그 값을 캡처하는 클로저). 추적은 양방향입니다. `xs`를 넘기면 `all`도 쓸 수 없습니다.
  - 컬렉션의 원소. 컬렉션에 저장한 것과 거기서 읽은 것(`xs[i]`, `get`, 반복, 그 슬라이스)은 한 값입니다. 그래서 어느 쪽을 넘겨도 다른 쪽은 쓸 수 없습니다(`let x = xs[0]; tx.send(xs); x.len()`, `tx.send(xs[0]); xs[1].len()`). 원소를 넘긴 컬렉션에는 `len`, `is_empty`, `capacity`, `clear`, `truncate`, `reserve`, `shrink_to_fit`, `push`, `insert`(와 `Deque`의 `push_back` / `push_front`)만 쓸 수 있습니다. `clear`는 컬렉션을 비우므로, 그 뒤에는 다시 온전히 쓸 수 있습니다. 다른 컬렉션에서 복사한 컬렉션(`extend`, `+=`, `collect`, `par::filter`)은 원본의 원소를 공유합니다. `pop`이나 `remove`로 꺼낸 원소(`List`, `Deque`, `Dict`, `OrderedDict`, `BinaryHeap`)는 컬렉션을 떠난 것입니다. 그 원소를 넘겨도 컬렉션은 계속 쓸 수 있습니다. 다만 컬렉션이 그 객체를 두 번 담고 있을 수 있으면 그렇지 않습니다. 한 값을 두 번 저장했거나, 저장한 두 값이 한 객체를 공유하거나, 한 컬렉션의 원소를 반복문에서 다른 컬렉션에 저장했을 때가 그렇습니다(원소를 위치로 구별하지 않습니다).
  - 무언가를 넘기는 직접 호출의 인자. 한 객체를 두 인자로 넘기고, 호출된 함수가 한쪽으로 넘기고 다른 쪽으로 쓰면 컴파일 에러입니다. `f(tx, a, a)`, `go`가 `self.items`를 쓸 때의 `h.go(tx, h.items)`, 원소를 꺼내 보내는 리스트와 그 원소 하나가 그런 예입니다. 한 컬렉션의 두 원소는 서로 다른 객체로 봅니다(`f(tx, rows[0], rows[1])`은 허용).
  - `ref` 매개변수. 호출된 함수가 그 일부를 넘긴 뒤, 넘기는 모든 경로에서 그 부분에 새 값을 줄 수 있습니다(`tx.send(self.buf)` 뒤의 `self.buf = []`). 그러면 호출자의 변수는 새 값을 돌려받아 계속 쓸 수 있습니다. 옛 값을 가리키는 다른 핸들은 쓸 수 없습니다. 값 매개변수는 복사본입니다. 그 필드에 새 값을 줘도 호출자에게 닿지 않으므로, 호출자는 넘긴 것을 그대로 쥐고 있습니다.
  - 호출. 함수가 매개변수에 하는 일은 그 호출자에게도 똑같이 적용됩니다. 매개변수를 넘기기, 그것이나 그 일부를 돌려주기, 다른 매개변수의 객체나 `ref` 매개변수에 저장하기, 돌려주는 클로저에 캡처하기, 가드 매개변수가 담은 것을 돌려주기가 여기에 듭니다. 미리 빌드한 라이브러리를 넘어서도 마찬가지입니다([refs-lang 9.4.7](../../refs-lang/009-tools/904-rokhir-bytecode.md#947-body)).

  함수 값으로 하는 호출은 같은 매개변수 타입으로 값처럼 쓰인 모든 함수에 닿는다고 봅니다. 조건은 컴파일러가 경로마다 아는 값을 검사할 때만 따라갑니다. 그런 값은 두 가지입니다. 하나는 상수로 정하거나 복사한 `bool`, 합 타입 또는 enum 변형입니다(`let sent = false; if c { tx.send(xs); sent = true; } if !sent { xs.len() }`). 다른 하나는 호출이 어떻게 끝났는지입니다. 어떤 함수가 에러 없이 반환할 때나 `none` / `false`가 아닌 값을 돌려줄 때만 매개변수를 넘긴다면, 다른 결과로 가는 호출자의 `catch`, `try?`, `??`, `else`, `if` 경로에서는 그 매개변수를 넘기지 않은 것으로 봅니다. `async fn`에 대한 `try await`의 `catch`도 마찬가지입니다. 그 밖의 조건은 따라가지 않습니다. 한 경로에서 넘긴 값은 그 경로가 다른 경로와 합쳐지는 곳에서 사라집니다.

  값을 계속 쓰려면 `clone()`(독립된 값)을 넘기거나, 먼저 변수나 필드에 새 값을 줍니다(struct 사본이면 그 필드만). `set`은 앞선 `get`이 돌려준 것을 풀어 줍니다. `Sync` 값의 사용, 값의 태그 읽기(`x is none`, `match x`), `unsafe`([refs-lang 4.15.2](../../refs-lang/004-functions/415-unsafe.md#4152-unsafe----블록-표현식)) 안의 사용은 예외입니다. `Handoff` 프로토콜은 그것을 만드는 함수 안에서 끝납니다. 호출자는 그 넘김을 보지 않습니다. spawn으로 띄운 클로저의 캡처는 검사하지 않아도 됩니다. 그 캡처는 `Send + Sync`여야 하기 때문입니다. `task::spawn`은 future를 넘깁니다. task의 프레임은 future가 캡처한 것의 복사본을 받습니다. 띄운 쪽에서는 send 뒤와 마찬가지로 그 future와 future가 캡처한 것이 사라집니다([2.18.1](218-task.md#2181-핵심-api)).

  다음도 컴파일 에러입니다. 첫째, `Sync` 객체에서 읽었지만 그 자체는 `Sync`가 아닌 값(사용자의 `unsafe impl Sync` 타입의 필드, `thread::local`의 값)을 넘기는 것입니다. 그런 값은 넘기지 않아도 다른 코드가 닿습니다. 대신 `clone()`을 넘깁니다. 채널의 `recv`, task의 결과, 잠금 가드가 주는 값은 받는 쪽의 것이므로 이 규칙에 들지 않습니다. 둘째, 가드가 아직 쥐고 있는 동안(`set`이 잠금에 새 값을 줄 때까지) 가드의 `get`이 돌려준 것을 넘기는 것입니다. 셋째, 그것을 `using` 블록 안에서 돌려주거나, 거기서 yield로 내보내거나(제너레이터의 `yield g.get()`), 블록의 값으로 삼거나(`let x = using g = m.lock() { g.get() }`, `f(using g = m.lock() { g.get() })`), 블록 안에서 만들어 블록 뒤에 실행하는 코루틴(`async fn`의 future, 제너레이터: `fut = grow(g.get()[0])`)에 주는 것입니다. 값을 쓰기 전에 블록이 끝나면서 잠금을 풀기 때문입니다. 가드는 자기 `using` 블록 안에서만 쓸 수 있습니다. 그 블록이 끝난 뒤 가드의 복사본을 쓰면 컴파일 에러입니다. 복사본에는 컬렉션에 넣은 것, 클로저가 잡은 것, 호출된 함수가 저장한 것, 두 번째 `using`으로 묶은 것, 코루틴이 들고 있는 것(`async fn`이나 제너레이터의 인자, `async` 블록의 캡처)이 모두 듭니다. 가드는 `!Sync`입니다([2.16.3](216-sync.md#2163-mutext-배타적-잠금)). `par::for_each`, `par::map`, `par::filter`는 항목마다 워커에 주므로, 두 항목이 `Sync`가 아닌 컬렉션 하나에 닿으면 안 됩니다. 이것은 호출마다 실행 중에 확인합니다([2.19.1](219-par.md#2191-핵심-api)). 두 항목이 한 컬렉션에 닿으면 호출한 스레드에서 항목을 하나씩 실행하고, 호출 위치마다 한 번 경고합니다. 슬라이스는 읽기만 하므로, 어느 항목도 리스트 자체를 들고 있지 않으면 항목이 한 리스트를 그 슬라이스로 공유해도 됩니다.

  이 규칙 밖에 남는 것은 세 가지입니다. `unsafe` 안의 코드, 위의 인자 검사를 건너뛰는 함수 값이나 클로저로 하는 호출, 그리고 한 객체를 두 번 이상 내보내는 제너레이터를 모은 컬렉션의 원소를 하나씩 넘기는 경우입니다. 그런 컬렉션은 그 객체를 두 번 담는다고 보지 않습니다.

  넘김은 실행 중에도 복사합니다. 그래서 위 규칙이 놓치는 경로(와 `unsafe` 안의 코드)에서도, 넘긴 값이 값으로 닿는 컬렉션은 보낸 쪽과 새 소유자 사이에 공유되지 않습니다(예외는 아래에 적습니다). `Mutex::new`, `RwLock::new`, 가드의 `set`과 `update`, `send` / `try_send` / `send_async`, 그리고 task, 스레드, `par` 스코프 작업의 결과(와 `try_scope`의 에러)는 값이 닿는 모든 컬렉션의 복사본을 새 소유자에게 줍니다. 복사는 struct, 튜플, 배열, enum, 합 타입의 구성 요소와 컬렉션의 원소를 깊이에 상관없이 따라갑니다. 단계마다 스택을 쓰지 않고 작업 목록으로 따라갑니다. 여러 번 닿는 컬렉션은 한 번만 복사하므로, 복사본에도 값의 공유 관계와 순환이 그대로 남습니다. 복사 대상은 `List`, `Dict`, `Deque`, `StrBuf`, `Slice`와 그 위에 만든 컬렉션, 그리고 `GcPtr`([1.6.3](../100-bare/106-gc-pin.md))입니다. 코루틴 future(`AsyncFuture`, `TryAsyncFuture`)는 자기 프레임을 따로 받습니다. 그 프레임에는 같은 복사로 만든, 캡처한 것의 복사본이 들어 있습니다. 이미 시작했거나 끝난 future는 넘길 수 없습니다. 프레임(이나 캡처한 future의 프레임)에 `[ref x]` 캡처가 있는 future도 넘길 수 없습니다. 넘기면 panic이 일어납니다([2.18.1](218-task.md#2181-핵심-api)). `Task`는 복사하지 않습니다. 대신 한 번 넘긴 뒤에는 그것을 `await`로 기다릴 때마다 결과의 복사본을 따로 받습니다([2.18.2](218-task.md#2182-taskt-타입)). 리스트와 그 버퍼를 보는 모든 뷰는 복사한 버퍼 하나를 계속 공유합니다. `str`의 바이트를 보는 뷰는 그 바이트만 복사합니다(바이트는 바뀔 수 없습니다). `update`는 가드가 이미 쥔 것을 그대로 두고, 클로저가 들여오는 것만 복사합니다. 보낸 쪽의 핸들은 그대로 둡니다. 그래서 위 규칙이 놓친 핸들은 다른 스레드가 쓰지 않는 객체를 가리킵니다. 에러를 던지는 send는 같은 값(호출자의 값이나 그 복사본)을 돌려줍니다. 복사에는 복사하는 양에 비례하는 시간과 메모리가 듭니다. GC 참조가 없는 일반 값이 연속해 있으면 메모리 복사 한 번으로 끝납니다. `rok::core::marker::hand_over(x)`는 같은 복사본을 돌려줍니다. 공유한 채로 두는 것은 `Sync`인 구성 요소, `unsafe impl Send`가 있는 타입의 값(task, 파일, 소켓처럼 스스로 공유를 다룹니다), 제너레이터와 스트림(`Send`가 아니어서 넘기지 않습니다)입니다. `Handoff::new`와 `Mutex::new_unshared`(2.16.3)는 복사하지 않습니다. 가드에서 꺼낸 핸들(`get`, `update` 클로저의 인자)은 실행 중에 복사하지 않습니다. 대신 위 규칙이 그 핸들과 가드 자체를 `using` 블록 밖까지 따라갑니다. 모든 복사본, `return`, `yield`, 블록의 값, 블록 안에서 만든 코루틴의 프레임이 그 경로입니다.
- 다음은 제외됩니다. `Cell<T>`, 원시 포인터(`Ptr<T>`, `ConstPtr<T>`, `CPtr<T>`), 맨 `any`와 바운드에 `Send` / `Sync`가 없는 `any Tr`, 바운드 없는 함수 타입, `[ref]` 캡처가 있는 클로저([refs-lang 4.6.9](../../refs-lang/004-functions/406-capture-semantics.md#469-병렬과-동시성-문맥의-추가-제약))입니다. `any Tr`은 바운드에 `Send`와 `Sync`가 모두 있을 때만 `Send`입니다. 클로저의 캡처처럼 `any`의 사본은 그 박스를 공유하므로, 스레드를 건너는 `any` 안에는 컬렉션이 있을 수 없습니다. 이 밖의 예외는 각 타입의 장에서 정합니다.
- 함수 타입은 `fn(..) -> R + Send`처럼 바운드를 적었을 때만 그 트레잇을 구현합니다([refs-lang 4.4](../../refs-lang/004-functions/404-function-type.md#send--sync-바운드)).

자동 구현이 맞지 않는 타입은 직접 선언합니다. 내부에 원시 포인터를 두지만 스레드 안전하게 감싼 타입은 `unsafe impl`로 직접 구현합니다(opt-in, 안전성은 작성자 책임). 구조상 자동 구현되지만 스레드 안전하지 않은 타입은 `impl !Send`로 구현을 끕니다(opt-out). `Cell<T>`의 제외도 이 방식으로 선언합니다.

```rok
struct Handle { raw: Ptr<u8>; }
unsafe impl Send for Handle { }
unsafe impl Sync for Handle { }

impl<T> !Send for Cell<T> { }
impl<T> !Sync for Cell<T> { }
impl<T> !Send for Ptr<T> { }
impl<T> !Sync for Ptr<T> { }
impl<T> !Send for CPtr<T> { }
impl<T> !Sync for CPtr<T> { }
```

스레드나 task로 값을 넘기는 API(`thread::spawn`, `task::spawn`, `par::*`, `channel`)는 이 두 트레잇으로 인자를 제한합니다. 동기화 도구의 정책은 [2.16 sync](216-sync.md)를 따릅니다.

### `Handoff<T>`와 hand-off 장치

```rok
pub struct Handoff<T> { value: T; }

impl<T> Handoff<T> {
    pub const unsafe fn new(value: T) -> Handoff<T>;
    pub const fn get(self) -> T;
}

unsafe impl<T: Send> Send for Handoff<T> {}
unsafe impl<T: Send> Sync for Handoff<T> {}

pub fn hand_over<T>(x: T) -> T;
```

`Handoff<T>`는 외부 프로토콜(join, 잠금, 채널 넘김) 아래에서 값을 스레드 사이로 옮깁니다. 이 프로토콜은 두 스레드가 값을 동시에 쓰지 못하게 막습니다. `T`가 `Send`이면 `T`가 `Sync`가 아니어도(컬렉션) `Send + Sync`입니다. `new`는 `unsafe`입니다. 호출자는 두 스레드가 그 값이나 그 핸들의 다른 사본을 동시에 쓰지 않음을 보장합니다. `new`는 사본을 만들지 않으며(위 참고), `get`은 감싼 값을 돌려줍니다.

`hand_over(x)`는 넘김(hand-off)이 새 소유자에게 주는 값을 돌려줍니다. `x`가 닿는 모든 컬렉션을 복사한 `x`이며, 닿는 컬렉션이 없으면 `x` 그대로입니다.

`Transfer`와 `Memo`는 넘김 사본이 다른 모듈의 표준 컬렉션에 닿을 수 있도록 공개했을 뿐입니다.

```rok
pub trait Transfer {
    fn transfer_copy(self, m: Memo) -> Self;
    fn transfer_keep(self, m: Memo) -> Self;
    fn transfer_visit(self, m: Memo) -> Self;
}

pub struct Memo { /* 비공개 필드 */ }
```

컴파일러는 정해진 표준 핸들 타입 목록에만 `Transfer`를 호출합니다. 그 밖의 타입에 둔 구현은 무시합니다. `Memo`는 한 번의 넘김이 만든 사본을 원본 객체별로 기록하고, 아직 고쳐야 할 복사된 버퍼도 기록합니다. `Memo`의 메서드와 그 옆의 `transfer*` intrinsic은 `pub(pkg)`입니다. 라이브러리 내부용이며 안정 API가 아닙니다.

## 2.13.10. `Default`: 기본값

```rok
trait Default {
    fn default() -> Self;
}
```

| 타입                              | `default()`        |
|-----------------------------------|--------------------|
| 정수, `int`                       | `0`                |
| 실수, `float`                     | `0.0`              |
| `bool`                            | `false`            |
| `char`                            | `'\0'`             |
| `str`                             | `""`               |
| `List`, `Dict`, `Set` 등 컬렉션   | 빈 컬렉션          |
| `T \| none`                       | `none`             |

`Default`는 derive로 구현할 수 있으며, 모든 필드의 `default()`로 값을 만듭니다.

## 2.13.11. derive 가능한 트레잇

`#[derive(...)]`([refs-lang 6.2.1](../../refs-lang/006-meta-system/602-built-in-attributes.md#621-derive-자동-트레잇-구현))로 만들 수 있는 내장 트레잇은 다음뿐입니다.

| 트레잇        | 생성되는 구현                               | 요구                                |
|---------------|---------------------------------------------|-------------------------------------|
| `Clone`       | 필드별 `clone`                              | 모든 필드 `Clone`                   |
| `Copy`        | marker                                      | `Clone`도 derive/구현, `Drop` 아님 |
| `Debug`       | 필드 이름과 값을 `f"{x:?}"` 형식으로        | 모든 필드 `Debug`                   |
| `Default`     | 필드별 `default()`                          | 모든 필드 `Default`                 |
| `PartialEq`   | 필드별 `==`의 AND                          | 모든 필드 `PartialEq`               |
| `Eq`          | marker                                      | `PartialEq`도 derive/구현          |
| `PartialOrd`  | 필드 선언 순서의 사전식 비교                | 모든 필드 `PartialOrd`, `PartialEq` |
| `Ord`         | 필드 선언 순서의 사전식 비교                | `Eq`, `PartialOrd`도 derive/구현   |
| `Hash`        | 필드별 해시의 결합 (결합 함수는 정하지 않음) | 모든 필드 `Hash`                   |

- `Display`는 derive로 구현할 수 없습니다([2.10](210-fmt.md)).
- `Eq` / `Ord`를 상위 트레잇 없이 derive로 구현하면 컴파일 에러입니다.
- `union`에 `#[derive]`를 붙이면 컴파일 에러입니다.
