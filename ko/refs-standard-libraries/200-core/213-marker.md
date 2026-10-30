# 2.13. marker - `Clone` / `Copy` / `Drop`

값을 복제하고, 더 이상 필요 없어진 자원을 정리하는 일은 어떤 언어에서도 피할 수 없는 주제입니다. `rok::core::marker` 모듈은 이 두 가지 관심사를 세 트레잇으로 묶어 표현합니다. 먼저 `Clone` 은 값의 독립적인 사본을 만드는 방법을 정의하고, 그다음 `Copy` 는 단순한 비트 복사로 충분한 타입임을 알리는 표지를 제공하며, 또한 `Drop` 은 scope 를 떠나는 순간 실행해야 하는 정리 코드를 정의합니다. 이 밖에 `none` 리터럴 타입의 라이브러리 이름인 `NoneSentinel` (2.13.8) 과, 스레드 사이로 넘기는 값을 감싸는 `Handoff` (2.13.9) 도 이 모듈에 둡니다.

Rok이 GC 를 채택했다는 사실은 이 트레잇들의 의미에 직접적인 영향을 줍니다. 메모리 회수는 GC 가 책임지므로 `Drop` 은 더 이상 메모리 해제와 얽히지 않습니다. 따라서 `Drop` 은 파일 핸들이나 소켓처럼 언어 외부에 존재하는 자원을 정리하는 finalize 작업에 한정됩니다. 이러한 차이가 Rok의 marker 트레잇을 Rust 와도, Java/Python 과도 다른 자리에 놓이게 합니다.

## 2.13.1. 메모리 모델 - GC 가 있는 언어의 Clone/Drop

Rok의 런타임은 [Immix + Sticky Mark-Bit Generational GC](../../refs-lang/007-runtime/703-gc-algorithm.md) 가 heap 객체의 수명을 자동으로 관리합니다. 즉 어떤 객체가 언제 메모리에서 사라지는지를 사용자가 직접 결정할 필요가 없습니다. 그러나 외부 자원의 정리만큼은 결정론적으로 일어나야 하기 때문에, Rok의 `Drop` 은 GC 와 분리된 별도의 메커니즘으로 동작합니다. 이를 Rust 와 비교하면 다음과 같습니다.

| 측면              | Rust (no GC)            | Rok (GC)                                        |
|-------------------|-------------------------|-------------------------------------------------|
| 메모리 회수       | `Drop` 가 책임          | GC 가 자동, `Drop` **하지 않음**                |
| `Drop::drop` 의미 | 자원 해제 + 메모리 free | **외부 자원 정리만** (file handle, socket, ...) |
| 호출 시점         | 결정론적 (scope exit)   | **`using` 블록을 빠져나갈 때만** (GC 수집과 무관) |
| 누락 시 영향      | 자원 누수 + 메모리 누수 | 자원 누수만 (메모리는 GC 가 결국 회수)          |

> [!NOTE]
> 일반 바인딩에는 scope 종료 시 자동 drop 이 없습니다. `Drop::drop` 은 `using` 으로 묶은 바인딩에서만 호출됩니다 ([refs-lang 3.17 `using`](../../refs-lang/003-expressions/317-using-expression.md)). GC 는 다른 객체의 finalization 을 보장하지 않으며, 어떤 경우에도 `drop` 을 부르지 않습니다.

이처럼 메모리 회수와 자원 정리를 분리한 덕분에, 일반 객체는 `Drop` 을 신경 쓰지 않아도 GC 가 알아서 회수해 주고, 외부 자원을 다루는 타입만 `Drop` 을 구현해 결정론적인 정리 시점을 확보할 수 있습니다.

## 2.13.2. `Clone` - 깊은 복제

값의 독립적인 사본이 필요할 때 호출하는 메서드가 `Clone::clone` 입니다. 시그니처는 다음과 같이 단순합니다.

```rok
#[lang("clone_trait")]
trait Clone {
    fn clone(self) -> Self;
}
```

`clone` 은 깊은 복제를 수행합니다. 즉 내부에 참조가 있더라도 그 참조가 가리키는 데이터까지 새로 복사해, 이후에 양쪽을 따로 변경해도 서로 영향을 주지 않습니다. 다만 GC 가 immutable 로 보장하는 공유 데이터는 여전히 그대로 공유할 수 있습니다.

### `Clone` 의 의미

다음 예제는 `Clone` 의 가장 직관적인 모습을 보여줍니다.

```rok
let a: List<i32> = [1, 2, 3];
let b = a.clone();           // 새 List, 내부 buffer 도 새로
b.push(4);                   // a 영향 없음
a;                           //→ [1, 2, 3]
b;                           //→ [1, 2, 3, 4]
```

여기서 "깊다" 라는 표현이 정확히 무엇을 의미하는지 짚어 둘 필요가 있습니다. `Clone` 은 logical equality 를 유지하면서 값 사이의 mutation 독립성을 보장합니다. 구체적으로 두 가지 성질이 따라옵니다. 먼저 `let b = a.clone();` 직후에는 `PartialEq` 가 정의되어 있다면 `a == b` 가 성립합니다. 그다음 그 이후 `a` 에 어떤 변경을 가해도 `b` 에 반영되지 않으며, 반대 방향도 마찬가지로 독립적입니다.

### 자동 호출 없음 - `clone_trait` lang slot

언어에 따라서는 `let y = x;` 만 써도 컴파일러가 알아서 `clone` 을 호출하기도 합니다. Rok은 이 부분에서 명시적인 쪽을 택했습니다.

```rok
let xs: List<i32> = [1, 2, 3];
let ys = xs;          // GC heap 객체 → 참조 공유 (clone 안 일어남)
```

`let y = x;` 는 GC heap 객체에 대해서는 참조 공유를 수행하고, `Copy` 타입에 대해서는 비트 복사를 수행합니다. 따라서 `Clone::clone` 은 자동으로 디스패치되지 않으며 사용자가 명시적으로 호출했을 때만 실행됩니다. 이 때문에 `Clone` 은 메서드 슬롯이 아니라 트레잇 슬롯 `clone_trait` 로 등록되며, 컴파일러는 이 슬롯을 `#[derive(Clone)]` 확장에서만 찾습니다.

### default 본문

primitive 타입의 `clone` 은 결국 비트 복사로 귀결되므로 intrinsic 으로 직접 매핑하고, 컬렉션의 `clone` 은 내부 buffer 를 새로 할당하면서 원소를 재귀적으로 복제합니다.

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

GC 객체의 clone 은 항상 새 buffer 를 할당하고 원소를 재귀적으로 clone 합니다.

### `#[derive(...)]`

대부분의 사용자 정의 struct 는 모든 필드가 `Clone` 이라면 직접 본문을 작성할 이유가 없습니다. 이러한 경우 `#[derive(Clone)]` 이 작업을 대신합니다.

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

derive 가 합성하는 본문은 모든 필드의 `Clone::clone` 을 호출한 결과를 같은 구조로 묶어 돌려주는 형태입니다. 따라서 모든 필드가 재귀적으로 `Clone` 을 구현하고 있어야 한다는 요구 조건이 자연스럽게 따라옵니다.

## 2.13.3. `Copy` - marker

```rok
trait Copy: Clone {
    // marker: 메서드 없음
}
```

Rok 에서는 모든 값이 값으로 복사됩니다. 컬렉션이나 GC 객체는 핸들이 복사되어 같은 객체를 가리키고, 원시 값, `str` (불변 `(ptr, len)`), 구조체, enum, 튜플은 그 자체가 복사됩니다 ([refs-lang 2.3.8](../../refs-lang/002-types/203-struct.md#238-값-의미)). 따라서 `Copy` 는 **marker 일 뿐** 이며, 대입이나 인자 전달의 의미를 바꾸지 않습니다. `Copy` 는 제네릭 bound (`T: Copy`) 에서 "비트 복사가 곧 독립된 사본" 인 타입을 고를 때만 쓰입니다.

### impl 정책

| 타입                             | `Clone`     | `Copy`           | 비고                                 |
|----------------------------------|-------------|------------------|--------------------------------------|
| `i*`, `u*`, `f*`                 | ✓           | ✓                | 원시 숫자                            |
| `bool`, `char`, `c_char`         | ✓           | ✓                |                                      |
| `str`                            | ✓           | ✓                | 불변 `(ptr, len)` 값                 |
| `int`, `float`                   | ✓           | ✓                | 불변 값, heap 의 BigInt 도 공유만    |
| `List<T>`, `Dict<K,V>`, `Set<T>` | ✓ (T:Clone) | ✗                | 핸들 복사는 공유, 독립 사본 아님     |
| 튜플                             | ✓ (모두)    | ✓ (모두 Copy 시) | 컴포넌트별                           |
| `Range<T>`                       | ✓ (T:Clone) | ✓ (T:Copy)       | start/end 둘 다 Copy 시              |
| 사용자 struct / enum             | derive      | derive           | 모든 필드 같은 트레잇 구현 시        |

`Drop` 을 구현한 타입은 `Copy` 가 될 수 없습니다.

```rok
#[derive(Clone, Copy)]
struct Vec3 { x: f64; y: f64; z: f64; }
```

> [!IMPORTANT]
> `Copy` derive 는 `Clone` 도 함께 구현될 때만, 그리고 모든 필드가 `Copy` 일 때만 가능합니다.

## 2.13.4. `Drop` - `using` 해제 시 정리

파일을 열었으면 닫아야 하고, mutex 를 잠갔으면 풀어야 합니다. 이러한 외부 자원 정리는 GC 의 손에 맡길 수 없습니다. 언제 일어날지 알 수 없는 sweep 시점까지 자원을 붙들고 있다면, 파일 디스크립터가 고갈되거나 잠금이 풀리지 않은 채로 다른 코드가 멈춰 서는 일이 벌어지기 때문입니다. `Drop` 트레잇과 `using` 표현식이 이 문제를 해결합니다.

```rok
trait Drop {
    #[lang("drop_method")]
    fn drop(ref self);
}
```

### 호출 시점 - `using` 바인딩에서만

Rok은 GC 언어이므로 일반 바인딩이 scope 를 벗어날 때 자동으로 `drop` 을 부르지 않습니다. `drop` 은 `using` 으로 묶은 바인딩에서만, 블록을 빠져나갈 때 호출됩니다 ([refs-lang 3.17 `using`](../../refs-lang/003-expressions/317-using-expression.md)).

- 정상 종료, `return`, `break`, 에러 전파 모두에서 호출됩니다.
- 여러 바인딩은 선언의 역순 (LIFO) 으로 호출됩니다.
- panic 은 프로세스를 끝내므로 `drop` 을 부르지 않습니다.
- drop glue: 값 자신의 `drop` 뒤에, `Drop` 을 가진 필드가 선언의 역순으로 재귀적으로 정리됩니다. 튜플 원소와, enum / 합 타입이 지금 담은 payload 도 같습니다.

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

`Drop` 을 구현하지 않은 타입을 `using` 으로 묶으면 `Drop` 이 필요하다는 컴파일 에러가 납니다.

### GC 와의 관계

GC 는 `drop` 을 부르지 않으며 finalizer 는 없습니다. 정리가 필요한 자원은 `using` 으로 묶거나 `close()` 같은 메서드로 직접 닫으며, 그러지 않으면 `unclosed_resource` lint 가 경고합니다 ([refs-lang 6.2.9](../../refs-lang/006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)).

### `Drop` 구현 예

가장 전형적인 구현은 OS 자원을 감싼 wrapper 입니다.

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

### `Drop` 의 제약

`Drop` 은 강력한 만큼 몇 가지 제약을 동반합니다. 먼저 `Drop` 을 구현한 타입은 `Copy` 가 될 수 없습니다. 비트 복사로 사본을 무한히 늘리면서 결정론적인 정리를 보장한다는 것은 모순이기 때문입니다.

```rok
#[derive(Copy)]              // ERROR - Drop 과 충돌
struct File { handle: i32; }
impl Drop for File { ... }
```

그다음 사용자가 `drop` 을 직접 호출하면 컴파일 에러입니다. `using` 해제 시점에 한 번만 호출되어야 정확성이 보장되기 때문입니다.

```rok
let f = File::open(path);
f.drop();                    // ERROR: Drop::drop 직접 호출 금지
```

따라서 조기에 자원을 닫고 싶다면 `File::close()` 처럼 별도의 메서드를 노출하는 쪽이 옳습니다.

예외는 `rok::bare` 안의 unsafe 코드뿐입니다. 예를 들어 `unsafe fn drop_in_place` ([1.3 mem](../100-bare/103-mem.md)) 는 수동 메모리 관리를 위한 함수입니다. drop glue 를 실행합니다: 위와 같이 그 타입 자신의 `drop`, 그 다음 필드들의 `drop` 을 선언 역순으로.

또한 `drop` 은 `throws` 시그니처를 갖지 않으므로 본문에서 에러를 던질 수 없습니다. `drop` 안에서 panic 이 일어나면 다른 panic 과 마찬가지로 프로세스가 종료됩니다.

```rok
impl Drop for X {
    fn drop(ref self) {
        if some_cond { throw "oops" }    // ERROR: drop 은 throws 가 아님
    }
}
```

### derive 정책

`Clone` 과 달리 `Drop` 은 derive 할 수 없습니다. `using` 으로 묶인 값의 `drop` 이 끝나면, 컴파일러는 그 struct 의 `Drop` 필드들의 `drop` 을 선언 역순으로 이어서 호출합니다. 따라서 사용자가 작성하는 `Drop` 본문은 "필드 이외에 추가로 정리해야 할 것" 만 다루면 됩니다.

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

이 모듈이 컴파일러와 연결되는 지점은 네 개의 lang slot 입니다. 각각의 자동 호출 시점은 앞서 살펴본 정책 그대로입니다.

| slot          | 부착 항목                 | 자동 호출 시점                           |
|---------------|---------------------------|------------------------------------------|
| `clone_trait` | `Clone` 트레잇            | 없음, `#[derive(Clone)]` 확장에서 사용   |
| `drop_method` | `Drop::drop` 메서드       | `using` 바인딩 해제 시 (drop glue 포함)  |
| `send_trait`  | `Send` 트레잇 (2.13.9)    | 자동 구현, 스레드 경계 검사              |
| `sync_trait`  | `Sync` 트레잇 (2.13.9)    | 자동 구현, 스레드 경계 검사              |

상세한 내용은 [0.4 lang slot 표](../000-meta/004-lang-slot-table.md) 0.4.2 / 0.4.3 / 0.4.5 절을 참고합니다.

## 2.13.6. 트레잇 구현 매트릭스

표준 라이브러리의 주요 타입이 세 트레잇과 어떻게 맺어지는지 한눈에 정리하면 다음과 같습니다.

| 타입                             | Clone       | Copy          | Drop                            | 동작                                 |
|----------------------------------|-------------|---------------|---------------------------------|--------------------------------------|
| `i*`, `u*`, `f*`                 | ✓           | ✓             | ✗                               | `rok.clone.copy` (bitwise)           |
| `bool`, `char`, `c_char`         | ✓           | ✓             | ✗                               | bitwise                              |
| `str`                            | ✓           | ✓             | ✗                               | 불변 `(ptr, len)` 값 복사            |
| `List<T>`, `Dict<K,V>`, `Set<T>` | ✓ (T:Clone) | ✗             | ✗                               | 재귀 clone                           |
| 튜플                             | ✓ (모두)    | ✓ (모두 Copy) | ✗                               | 컴포넌트별                           |
| `Range<T>`                       | ✓ (T:Clone) | ✓ (T:Copy)    | ✗                               | derive 자동                          |
| `File`                           | ✗           | ✗             | ✓                               | `rok::bare::close_fd`                 |
| `MutexGuard<T>`                  | ✗           | ✗             | ✓ (`using` 해제 시)             | `rok::bare::unlock`                   |
| `Channel::Sender<T>`             | ✓           | ✗             | ✗ (`close()` 로 닫음)           | (위와 동일)                          |

표 뒤에 자리 잡은 원칙은 단순합니다. 먼저 모든 primitive 와 내부 GC 타입은 `Clone` 을 구현하고, 그 가운데 비트 복사가 의미적으로 옳은 것들만 `Copy` 가 됩니다. 컬렉션은 `Clone` 이지만 `Copy` 는 아닙니다. 핸들의 비트 복사가 깊은 복제의 의미를 보존하지 않기 때문입니다. `str` 은 불변이므로 그 `(ptr, len)` 의 복사가 곧 독립된 사본이어서 `Copy` 입니다. 또한 `Drop` 은 `File`, `Socket`, `MutexGuard` 처럼 외부 자원을 감싸는 wrapper 만 구현합니다. 이 타입들은 자원 중복 소유를 막기 위해 보통 `Clone` 을 구현하지 않습니다.

## 2.13.7. 다른 언어와의 비교

지금까지 설명한 Rok의 모델이 기존 언어들과 어떻게 다른지 비교해 보면, 설계 의도가 좀 더 분명해집니다.

| 측면              | Rust                     | Java/Python (GC)            | Rok (GC + Drop)          |
|-------------------|--------------------------|-----------------------------|--------------------------|
| 메모리 회수       | `Drop` 가 책임           | GC                          | GC                       |
| 결정론적 finalize | `Drop` (컴파일러 보장)   | **없음** (finalizer 비결정) | `Drop` (scope-bound)     |
| Copy 의미         | bitwise + `!Drop` 강제   | **없음** (모든 객체 ref)    | bitwise + `!Drop` 강제   |
| Clone 의미        | 깊은 복제                | `clone()` (Object.clone)    | 깊은 복제                |
| derive            | `#[derive(Clone, Copy)]` | `Cloneable` 인터페이스      | `#[derive(Clone, Copy)]` |

Rok이 두드러지는 지점은 결국 GC 와 결정론적 `Drop` 의 공존입니다. 메모리 회수는 GC 가 맡기 때문에 `Drop` 이 메모리 해제를 의미하지 않으며, 그래서 일반 객체는 `Drop` 을 구현할 필요가 전혀 없습니다. 따라서 외부 자원 정리가 필요한 타입만 `Drop` 을 구현하고, 사용하는 쪽은 `using` 으로 묶으면 충분합니다.

이 모델은 한쪽으로는 Java/Python 의 finalizer 비결정성 문제를, 다른 한쪽으로는 GC 가 없는 데에서 오는 Rust 의 비용을 동시에 회피합니다.

## 2.13.8. `NoneSentinel` - `none` 리터럴의 타입

[refs-lang 1.9.2](../../refs-lang/001-lexical/109-other-literals.md) 의 `none` 리터럴은 컴파일러에 내장된 `none` 타입, 곧 값이 하나뿐인 zero-sized 타입을 가집니다. `rok::core::marker` 는 타입 별칭으로 이 타입에 `NoneSentinel` 이라는 라이브러리 이름을 붙일 뿐이며, lang slot 은 관여하지 않습니다.

```rok
pub type NoneSentinel = none;
```

사용자는 이 이름을 직접 쓸 일이 거의 없습니다. `T | none` 같은 합 타입 표기 ([refs-lang 2.6](../../refs-lang/002-types/206-sum-types.md)) 에서 `none` 자리가 곧 이 타입이며, `none` 리터럴, `none` 타입 표기, `NoneSentinel` 은 모두 같은 내장 타입을 가리킵니다.

## 2.13.9. `Send` / `Sync` - 스레드 경계 marker

```rok
#[lang("send_trait")]
trait Send { }      // 다른 스레드로 옮겨도 안전

#[lang("sync_trait")]
trait Sync { }      // 공유 참조로 여러 스레드에서 읽어도 안전
```

두 트레잇은 메서드가 없는 **자동 marker 트레잇** 이며, 컴파일러가 구조적으로 구현합니다.

- 원시 숫자, `bool`, `char`, `c_char`, `str`, `none` 은 `Send + Sync` 입니다.
- `struct`, `enum`, `union`, 튜플, 배열, 합 타입은 모든 필드 (구성 요소) 가 `Send` 이면 `Send`, 모두 `Sync` 이면 `Sync` 입니다. 제네릭 컨테이너의 `Send` 는 같은 규칙으로 `T` 에 따릅니다.
- 컬렉션 (`List`, `Dict`, `Set`, `StrBuf`, `Deque`, `BinaryHeap`, `OrderedDict`, `OrderedSet`, `Slice`) 은 `Send` 이고 `!Sync` 입니다. 가변 핸들을 공유하기 때문입니다. 스레드 사이에서는 `Mutex` / `RwLock` / 채널로 공유합니다.
- Rok 에는 move 의미가 없습니다. 값을 넘겨도 (`Mutex::new(xs)`, `RwLock::new(xs)`, `tx.send(xs)` / `try_send` / `send_async`, `Handoff::new(xs)`) 보낸 쪽의 핸들은 새 소유자가 이제 다른 스레드와 공유하는 객체를 가리킵니다. guard 에서 꺼낸 (`get`), guard 에 넣은 (`set`), `update` 의 클로저가 닿은 핸들도 guard 의 `using` 블록이 끝난 뒤 여전히 보호된 객체를 가리킵니다. 그 뒤에 그런 값을 어느 경로에서든 쓰는 것은 컴파일 에러 (`E202`, [refs-lang 9.9.4](../../refs-lang/009-tools/909-debug-tools.md#994-진단--에러-메시지)) 입니다. 값은 다음을 거쳐 추적합니다:
  - 복사 (`let ys = xs`, struct 필드나 tuple 원소, 필드 대입);
  - 그 값을 넘기는 필드 (`Mutex::new(s.items)`);
  - 그 값을 담는 객체 (`all.push(xs)`, 그 값을 캡처하는 클로저). 양쪽 모두입니다: `xs` 를 넘기면 `all` 도 쓸 수 없습니다;
  - 컬렉션의 원소. 컬렉션에 저장한 것과 거기서 읽은 것 (`xs[i]`, `get`, 반복, 그 slice) 은 한 값이므로, 어느 쪽을 넘겨도 다른 쪽은 쓸 수 없습니다 (`let x = xs[0]; tx.send(xs); x.len()`, `tx.send(xs[0]); xs[1].len()`). 원소를 넘긴 컬렉션에는 `len`, `is_empty`, `capacity`, `clear`, `truncate`, `reserve`, `shrink_to_fit`, `push`, `insert` (와 `Deque` 의 `push_back` / `push_front`) 만 쓸 수 있습니다. `clear` 는 컬렉션을 비우므로 다시 온전히 쓸 수 있습니다. 다른 컬렉션에서 복사한 컬렉션 (`extend`, `+=`, `collect`, `par::filter`) 은 원본의 원소를 공유합니다. `pop` 이나 `remove` 로 꺼낸 원소 (`List`, `Deque`, `Dict`, `OrderedDict`, `BinaryHeap`) 는 컬렉션을 떠났습니다: 그것을 넘겨도 컬렉션은 계속 쓸 수 있습니다. 다만 컬렉션이 그 객체를 두 번 담고 있을 수 있으면 그렇지 않습니다. 한 값을 두 번 저장했거나, 저장한 두 값이 한 객체를 공유하거나, 한 컬렉션의 원소를 반복문에서 다른 컬렉션에 저장했을 때가 그렇습니다 (원소를 위치로 구별하지 않습니다);
  - 무언가를 넘기는 직접 호출의 인자. 한 객체를 두 인자로 넘기고, 호출된 함수가 그중 하나로 넘기고 다른 하나로 쓰면 (`f(tx, a, a)`, `go` 가 `self.items` 를 쓰는 `h.go(tx, h.items)`, pop 해서 보내는 리스트와 그 원소 하나) 컴파일 에러입니다. 한 컬렉션의 두 원소는 서로 다른 객체로 봅니다 (`f(tx, rows[0], rows[1])` 은 허용);
  - `ref` 매개변수. 호출된 함수가 그 일부를 넘긴 뒤, 넘기는 모든 경로에서 그 부분에 새 값을 주면 (`tx.send(self.buf)` 뒤의 `self.buf = []`), 호출자의 변수는 새 값을 돌려받아 계속 쓸 수 있습니다. 옛 값의 다른 핸들은 그렇지 않습니다. 값 매개변수는 복사본이므로 그 필드에 새 값을 줘도 호출자에게 닿지 않고, 호출자는 넘긴 것을 여전히 쥡니다;
  - 호출. 매개변수를 넘기거나, 그것이나 그 일부를 돌려주거나, 다른 매개변수의 객체나 `ref` 매개변수에 저장하거나, 돌려주는 클로저에 캡처하거나, guard 매개변수가 담은 것을 돌려주는 함수는 그 호출자에게도 그렇게 하며, 미리 빌드한 라이브러리를 넘어서도 마찬가지입니다 ([refs-lang 9.4.7](../../refs-lang/009-tools/904-rokhir-bytecode.md#947-body)).

  함수 값을 통한 호출은 그 매개변수 타입으로 값으로 쓰인 모든 함수에 닿는다고 봅니다. 조건은 컴파일러가 경로마다 아는 값을 검사할 때 따라갑니다: 상수로 정하거나 복사한 `bool` 이나 합 또는 enum 변형 (`let sent = false; if c { tx.send(xs); sent = true; } if !sent { xs.len() }`), 그리고 호출이 어떻게 끝났는지. 에러 없이 반환할 때, 또는 `none` / `false` 가 아닌 값을 돌려줄 때만 매개변수를 넘기는 함수는, 다른 결과에 대한 호출자의 `catch`, `try?`, `??`, `else`, `if` 경로에서는 그것을 넘기지 않습니다. `async fn` 에 대한 `try await` 의 `catch` 도 마찬가지입니다. 그 밖의 조건은 따라가지 않습니다: 한 경로에서 넘긴 값은 그 경로가 다른 경로와 합쳐지는 곳에서 사라집니다.

  값을 계속 쓰려면 `clone()` (독립된 값) 을 넘기거나, 먼저 변수나 필드에 새 값을 줍니다 (struct 사본이면 그 필드만). `set` 은 앞선 `get` 이 돌려준 것을 풀어 줍니다. `Sync` 값의 사용, 값의 tag 읽기 (`x is none`, `match x`), `unsafe` ([refs-lang 4.15.2](../../refs-lang/004-functions/415-unsafe.md#4152-unsafe------블록-표현식)) 안의 사용은 예외입니다. `Handoff` 프로토콜은 그것을 만드는 함수 안에서 끝납니다: 호출자는 그 넘김을 보지 않습니다. spawn 한 클로저의 캡처는 검사가 필요 없습니다: 그것들은 `Send + Sync` 여야 합니다. `task::spawn` 은 future 를 넘깁니다: task 의 frame 은 future 가 캡처한 것의 복사본을 받고, spawn 한 쪽에서는 send 뒤와 같이 그 future 와 그것이 캡처한 것이 사라집니다 ([2.18.1](218-task.md#2181-핵심-api)).

  다음도 컴파일 에러입니다: `Sync` 객체에서 읽은, 그 자체는 `Sync` 가 아닌 값 (사용자의 `unsafe impl Sync` 타입의 필드, `thread::local` 의 값) 을 넘기는 것. 그런 값은 넘김 없이도 다른 코드가 닿습니다. `clone()` 을 넘깁니다. 채널의 `recv`, task 의 결과, 잠금 guard 가 주는 값은 받는 쪽의 것이므로 이 규칙에 들지 않습니다. guard 가 아직 쥔 동안 (`set` 이 잠금에 새 값을 줄 때까지) guard 의 `get` 이 돌려준 것을 넘기는 것, 그리고 그것을 `using` 블록 안에서 돌려주거나, 거기서 yield 하거나 (generator 의 `yield g.get()`), 블록의 값으로 삼거나 (`let x = using g = m.lock() { g.get() }`, `f(using g = m.lock() { g.get() })`), 블록 안에서 만들어 블록 뒤에 실행하는 코루틴 (`async fn` 의 future, generator: `fut = grow(g.get()[0])`) 에 주는 것: 값을 쓰기 전에 블록의 끝이 잠금을 풉니다. guard 는 자기 `using` 블록 안에서만 쓸 수 있습니다: 그 블록이 끝난 뒤 guard 의 복사본 (컬렉션에 넣은 것, 클로저가 잡은 것, 호출된 함수가 저장한 것, 두 번째 `using` 으로 묶은 것, 코루틴이 들고 있는 것: `async fn` 이나 generator 의 인자, `async` 블록의 캡처) 을 쓰면 컴파일 에러입니다. guard 는 `!Sync` 입니다 ([2.16.3](216-sync.md#2163-mutext---배타적-잠금)). `par::for_each`, `par::map`, `par::filter` 는 항목마다 worker 에 주므로, 두 항목이 `Sync` 가 아닌 컬렉션 하나에 닿으면 안 됩니다. 호출마다 실행 중에 이것을 확인합니다 ([2.19.1](219-par.md#2191-핵심-api)): 두 항목이 한 컬렉션에 닿으면, 항목들을 부른 스레드에서 하나씩 실행하고 호출 위치마다 한 번 경고합니다. slice 는 읽기만 하므로, 어느 항목도 리스트 자체를 들고 있지 않으면 항목들이 한 리스트의 slice 를 통해 그것을 공유해도 됩니다.

  이 규칙 밖에 남는 것: `unsafe` 안의 코드, 위의 인자 검사를 건너뛰는 함수 값이나 클로저를 통한 호출, 그리고 한 객체를 두 번 이상 yield 하는 generator 를 모은 컬렉션의 원소를 하나씩 넘기는 경우. 그런 컬렉션은 그 객체를 두 번 담는다고 보지 않습니다.

  넘김은 실행 중에도 복사하므로, 위의 규칙 밖에 남는 경로 (와 `unsafe` 안의 코드) 에서도 넘긴 값이 값으로 닿는 컬렉션은 보낸 쪽과 새 소유자 사이에 공유되지 않습니다 (예외는 아래에 적습니다). `Mutex::new`, `RwLock::new`, guard 의 `set` 과 `update`, `send` / `try_send` / `send_async`, 그리고 task, 스레드, `par` scope 작업의 결과 (와 `try_scope` 의 에러) 는 값이 닿는 모든 컬렉션의 복사본을 새 소유자에게 줍니다: struct, 튜플, 배열, enum, 합 타입의 구성 요소와 컬렉션의 원소를 거쳐, 깊이에 상관없이, 단계마다 스택을 쓰지 않고 작업 목록으로 따라갑니다. 여러 번 닿는 컬렉션은 한 번만 복사하므로, 복사본은 값의 공유 관계와 순환을 그대로 지닙니다. 복사하는 것은 `List`, `Dict`, `Deque`, `StrBuf`, `Slice` 와 그 위에 만든 컬렉션, 그리고 `GcPtr` ([1.6.3](../100-bare/106-gc-pin.md)) 입니다. 코루틴 future (`AsyncFuture`, `TryAsyncFuture`) 는 같은 복사로 만든, 캡처한 것의 복사본을 담은 자기 frame 을 받습니다. 시작했거나 끝난 것, 또는 frame (이나 캡처한 future 의 frame) 에 `[ref x]` 캡처가 있는 것은 넘길 수 없으며 넘기면 panic 합니다 ([2.18.1](218-task.md#2181-핵심-api)). `Task` 는 복사하지 않지만, 한 번 넘겨진 뒤에는 그것을 `await` 할 때마다 결과의 자기 복사본을 받습니다 ([2.18.2](218-task.md#2182-taskt-타입)). 리스트와 그 buffer 를 보는 모든 view 는 복사한 buffer 하나를 계속 공유하고, `str` 의 바이트를 보는 view 는 그 바이트만 복사합니다 (바뀔 수 없습니다). `update` 는 guard 가 이미 쥔 것을 그대로 두고, 클로저가 들여오는 것만 복사합니다. 보낸 쪽의 핸들은 그대로 두므로, 위의 규칙이 놓친 핸들은 다른 스레드가 쓰지 않는 객체를 가리킵니다. throw 하는 send 는 같은 값 (호출자의 값이나 그 복사본) 을 돌려줍니다. 복사에는 복사하는 양에 비례하는 시간과 메모리가 들며, plain 값 (GC 참조 없음) 의 연속은 한 번의 메모리 복사입니다. `rok::core::marker::hand_over(x)` 는 같은 복사본을 돌려줍니다. 공유한 채로 두는 것: `Sync` 인 구성 요소, `unsafe impl Send` 가 있는 타입의 값 (task, 파일, 소켓처럼 스스로 공유를 다룹니다), generator 와 stream (`Send` 가 아니어서 넘겨지지 않습니다). `Handoff::new` 와 `Mutex::new_unshared` (2.16.3) 는 복사하지 않습니다. guard 에서 꺼낸 핸들 (`get`, `update` 클로저의 인자) 은 실행 중에 복사하지 않습니다: 위의 규칙이 그것과 guard 자체를 모든 복사본, `return`, `yield`, 블록의 값, 블록 안에서 만든 코루틴의 frame 을 거쳐 `using` 블록 밖까지 따라갑니다.
- 다음은 제외됩니다: `Cell<T>`, 원시 포인터 (`Ptr<T>`, `ConstPtr<T>`, `CPtr<T>`), 맨 `any` 와 bound 에 `Send` / `Sync` 가 없는 `any Tr` (`any Tr` 은 bound 에 `Send` 와 `Sync` 가 모두 있을 때만 `Send` 입니다: 클로저의 캡처처럼 `any` 의 사본은 그 box 를 공유하므로, 스레드를 건너는 `any` 안에는 컬렉션이 있을 수 없습니다), bound 없는 함수 타입, `[ref]` 캡처가 있는 클로저 ([refs-lang 4.6.9](../../refs-lang/004-functions/406-capture-semantics.md#469-병렬과-동시성-문맥의-추가-제약)). 이 밖의 예외는 각 타입의 장에서 정합니다.
- 함수 타입은 `fn(..) -> R + Send` 처럼 bound 를 적었을 때만 그 트레잇을 가집니다 ([refs-lang 4.4](../../refs-lang/004-functions/404-function-type.md#send--sync-bound)).

자동 구현이 맞지 않는 타입은 직접 선언합니다. 내부에 원시 포인터를 두지만 스레드 안전하게 감싼 타입은 `unsafe impl` 로 opt-in 하고 (안전성은 작성자 책임), 구조상 자동 구현되지만 스레드 안전하지 않은 타입은 `impl !Send` 로 opt-out 합니다. `Cell<T>` 의 제외도 이 방식으로 선언됩니다.

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

스레드나 task 로 값을 넘기는 API (`thread::spawn`, `task::spawn`, `par::*`, `channel`) 는 이 두 트레잇으로 인자를 제한합니다. 동기화 primitive 의 정책은 [2.16 sync](216-sync.md) 를 따릅니다.

### `Handoff<T>` 와 hand-off 장치

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

`Handoff<T>` 는 두 스레드가 동시에 쓰지 못하게 막는 외부 프로토콜(join, lock, 채널 hand-off) 아래에서 값을 스레드 사이로 옮깁니다. `T` 가 `Send` 이면 `T` 가 `Sync` 가 아니어도(컬렉션) `Send + Sync` 입니다. `new` 는 `unsafe` 이며, 호출자가 두 스레드가 그 값이나 그 핸들의 다른 사본을 동시에 쓰지 않음을 보장합니다. `new` 는 사본을 만들지 않으며(위 참고), `get` 은 감싼 값을 돌려줍니다.

`hand_over(x)` 는 hand-off 가 새 소유자에게 주는 값을 돌려줍니다. `x` 가 닿는 모든 컬렉션을 복사한 `x` 이며, 닿는 컬렉션이 없으면 `x` 그대로입니다.

`Transfer` 와 `Memo` 는 hand-off 사본이 다른 모듈의 표준 컬렉션에 닿을 수 있도록 공개되어 있을 뿐입니다.

```rok
pub trait Transfer {
    fn transfer_copy(self, m: Memo) -> Self;
    fn transfer_keep(self, m: Memo) -> Self;
    fn transfer_visit(self, m: Memo) -> Self;
}

pub struct Memo { /* 비공개 필드 */ }
```

컴파일러는 정해진 표준 핸들 타입 목록에 대해서만 `Transfer` 를 호출하며, 그 밖의 타입에 둔 impl 은 무시됩니다. `Memo` 는 한 번의 hand-off 가 만든 사본을 원본 객체별로, 그리고 아직 고쳐야 할 복사된 버퍼를 기록합니다. `Memo` 의 메서드와 그 옆의 `transfer*` intrinsic 은 `pub(pkg)` 로, 라이브러리 내부용이며 안정 API 가 아닙니다.

## 2.13.10. `Default` - 기본값

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

`Default` 는 derive 할 수 있으며, 모든 필드의 `default()` 로 값을 만듭니다.

## 2.13.11. derive 가능한 트레잇

`#[derive(...)]` ([refs-lang 6.2.1](../../refs-lang/006-meta-system/602-built-in-attributes.md#621-derive-자동-트레잇-구현)) 로 만들 수 있는 내장 트레잇은 다음뿐입니다.

| 트레잇        | 생성되는 구현                               | 요구                                |
|---------------|---------------------------------------------|-------------------------------------|
| `Clone`       | 필드별 `clone`                              | 모든 필드 `Clone`                   |
| `Copy`        | marker                                      | `Clone` 도 derive/구현, `Drop` 아님 |
| `Debug`       | 필드 이름과 값을 `f"{x:?}"` 형식으로        | 모든 필드 `Debug`                   |
| `Default`     | 필드별 `default()`                          | 모든 필드 `Default`                 |
| `PartialEq`   | 필드별 `==` 의 AND                          | 모든 필드 `PartialEq`               |
| `Eq`          | marker                                      | `PartialEq` 도 derive/구현          |
| `PartialOrd`  | 필드 선언 순서의 사전식 비교                | 모든 필드 `PartialOrd`, `PartialEq` |
| `Ord`         | 필드 선언 순서의 사전식 비교                | `Eq`, `PartialOrd` 도 derive/구현   |
| `Hash`        | 필드별 해시의 결합 (결합 함수는 정하지 않음) | 모든 필드 `Hash`                   |

- `Display` 는 derive 할 수 없습니다 ([2.10](210-fmt.md)).
- `Eq` / `Ord` 를 그 상위 트레잇 없이 derive 하면 컴파일 에러입니다.
- `union` 에 `#[derive]` 를 붙이면 컴파일 에러입니다.
