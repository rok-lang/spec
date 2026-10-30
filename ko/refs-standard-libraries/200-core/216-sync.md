# 2.16. sync: 동시성 기본 요소

`rok::core::sync` 모듈은 공유 상태에 안전하게 접근하는 기본 트레잇과 타입을 정의합니다. Rok의 GC와 M:N 스케줄러 아래에서 여러 task나 스레드가 데이터를 공유할 때 씁니다.

`use rok::core::**;`가 묵시적으로 적용되므로 따로 가져오지 않아도 `Atomic`, `Mutex`, `RwLock`, `Cell`을 쓸 수 있습니다. 동시성 코드임을 드러내고 싶은 자리에서는 다음처럼 필요한 항목만 가져오기를 권장합니다.

```rok
use rok::core::sync::{Atomic, Mutex, RwLock, Once, Condvar, Cell};
```

## 2.16.1. 기본 요소 4개 개요

네 기본 요소는 각각 다른 동시성 상황을 맡습니다. 각 타입의 의미와 쓰는 자리는 다음과 같습니다.

| 타입        | 의미                                       | 사용 자리                      |
|-------------|--------------------------------------------|--------------------------------|
| `Atomic<T>` | lock-free 원자 연산 (T = primitive)        | counter, flag, sequence number |
| `Mutex<T>`  | 배타적 잠금 (쓰는 쪽 하나)                 | 공유 가변 상태                 |
| `RwLock<T>` | 읽는 쪽 여럿 또는 쓰는 쪽 하나             | 읽기가 많은 상태               |
| `Cell<T>`   | thread-local interior mutability (no lock) | single-threaded mutable        |

## 2.16.2. `Atomic<T>`: lock-free 원자 타입

가장 가벼운 기본 요소는 `Atomic<T>`입니다. 내부는 T마다 LLVM atomic 연산으로 구현합니다. 형태는 다음과 같습니다.

```rok
struct Atomic<T: AtomicPrimitive> {
    // 내부: AtomicU32 / AtomicU64 / AtomicPtr 등 (T 별 LLVM atomic ops)
}

trait AtomicPrimitive { /* marker - i32/i64/u32/u64/usize/isize/bool/Ptr */ }

impl<T: AtomicPrimitive> Atomic<T> {
    const fn new(value: T) -> Atomic<T>;

    fn load(self, ordering: AtomicOrdering = SeqCst) -> T;
    fn store(self, value: T, ordering: AtomicOrdering = SeqCst);
    fn swap(self, value: T, ordering: AtomicOrdering = SeqCst) -> T;
    fn compare_exchange(
        self, current: T, new: T,
        success: AtomicOrdering, failure: AtomicOrdering,
    ) -> Result<T, T>;                  // 실패 결과도 값 - 에러가 아님
    fn compare_exchange_weak(
        self, current: T, new: T,
        success: AtomicOrdering, failure: AtomicOrdering,
    ) -> Result<T, T>;                  // 실패 결과도 값 - 에러가 아님

    // 정수 전용
    fn fetch_add(self, value: T, ordering: AtomicOrdering = SeqCst) -> T;
    fn fetch_sub(self, value: T, ordering: AtomicOrdering = SeqCst) -> T;
    fn fetch_and(self, value: T, ordering: AtomicOrdering = SeqCst) -> T;
    fn fetch_or(self, value: T, ordering: AtomicOrdering = SeqCst) -> T;
    fn fetch_xor(self, value: T, ordering: AtomicOrdering = SeqCst) -> T;
}

impl Atomic<i32> {
    fn wait(self, expected: i32, timeout_ns: i64 = -1);   // 값이 expected 인 동안, 최대 timeout_ns (< 0 이면 무한) 대기
    fn wake(self);                                         // 이 cell 에서 wait 중인 스레드를 모두 깨움
}
```

`wait`는 이유 없이 돌아올 수 있으므로 호출자가 조건을 다시 확인합니다(futex 의미).

### `AtomicOrdering`

원자 연산은 메모리 순서 보장 강도에 따라 다섯 가지로 나뉩니다.

```rok
enum AtomicOrdering {
    Relaxed,        // 동기화 없음 - counter 등
    Acquire,        // load 후 동기화 보장
    Release,        // store 전 동기화 보장
    AcqRel,         // RMW 시 양방향
    SeqCst,         // 전역 순차 일관성 (default - 안전)
}
```

이 모델은 C++와 Rust의 atomic ordering 모델과 같은 구조입니다. 기본값은 안전한 `SeqCst`입니다. 이름을 `AtomicOrdering`으로 정해서 암묵 core glob 아래에서 비교용 `cmp::Ordering`과 겹치지 않게 했습니다. 변형(`Relaxed`, `Acquire`, `Release`, `AcqRel`, `SeqCst`)은 접두 없이 쓸 수 있습니다. ordering 인자는 컴파일 타임 상수여야 합니다. 약한 ordering은 성능이 중요한 자리에서만 신중하게 쓰기를 권장합니다.

### 사용 예

가장 흔한 용도인 카운터 증가는 다음과 같이 씁니다.

```rok
static COUNTER: Atomic<i32> = Atomic::new(0);

task::spawn(async { COUNTER.fetch_add(1, Relaxed); });
task::spawn(async { COUNTER.fetch_add(1, Relaxed); });
// 결과: COUNTER == 2 (race 없음)

let v = COUNTER.load(SeqCst);     //→ 2
```

### CAS (compare-and-swap)

더 복잡한 lock-free 자료 구조는 compare-and-swap 루프로 만듭니다. 다음은 lock-free 스택의 push 예입니다.

```rok
fn lock_free_push(stack: Atomic<List<T> | none>, x: T) {
    loop {
        let cur = stack.load(Acquire);
        let new = match cur {
            v: List<T> => v.with_appended(x),
            none       => [x],
        };
        match stack.compare_exchange(cur, new, AcqRel, Acquire) {
            Ok(_)  => break,
            Err(_) => continue,                   // 다른 task 가 먼저 - 재시도
        }
    }
}
```

## 2.16.3. `Mutex<T>`: 배타적 잠금

원자 연산만으로 표현하기 어려운 공유 상태에는 `Mutex<T>`를 씁니다. 내부는 futex와 페이로드로 이루어집니다. `using`으로 묶은 가드가 잠금을 풉니다.

```rok
struct Mutex<T> {
    // 내부: futex + payload
}

impl<T> Mutex<T> {
    fn new(value: T) -> Mutex<T>;
    unsafe fn new_unshared(value: T) -> Mutex<T>;   // 넘김 복사 없는 new

    fn lock(self) -> MutexGuard<T>;            // 스레드를 block
    fn lock_async(self) -> LockFuture<T>;      // await m.lock_async() - task 만 중단 (Future<Output = MutexGuard<T>>)
    fn try_lock(self) -> MutexGuard<T> | none; // non-blocking
}

struct MutexGuard<T> { /* RAII guard */ }

impl<T> Drop for MutexGuard<T> {
    fn drop(ref self) {
        // futex_wake + 잠금 해제
    }
}

impl<T> MutexGuard<T> {
    fn get(self) -> T;                  // 보호된 값
    fn set(self, value: T);             // 값 교체
    fn update(self, f: fn(T) -> T);     // 읽고 바꿔 씀
}
```

### 사용 예

가드를 `using`으로 묶으면 블록을 빠져나갈 때 `drop`되어 잠금이 풀립니다. `using` 없이 받은 가드는 자동으로 풀리지 않으므로 항상 `using`으로 묶어야 합니다. 가드는 호출, 메서드 호출, `await`에서 받습니다. 받은 가드로는 세 가지만 할 수 있습니다. 바인딩(`let`, 패턴, `using`)을 초기화하거나, 변수에 대입하거나, 호출자에게 넘기는(`return`, 함수의 값) 것입니다. 함수에 넘기거나, 그 위에서 메서드를 부르거나, 필드나 컬렉션에 저장하거나, 버리면 컴파일 에러입니다. `using`으로 묶은 가드는 함수에 넘길 수 있습니다. `using` 없이 묶은 가드는 넘긴다고 해서 풀리지 않습니다. `using`으로 풀리지 않는 일반 `let`(또는 `if let`)으로 가드를 묶거나, `lock`이 돌려준 가드를 버리면 컴파일 에러입니다.

`Mutex::new(value)`는 `value`를 넘겨받습니다. 그 뒤에 호출자의 핸들을 쓰면 컴파일 에러입니다. 다음도 모두 컴파일 에러입니다([2.13.9](213-marker.md#2139-send--sync-스레드-경계-마커)).

- 가드의 `using` 블록이 끝난 뒤 `get`이 돌려준 것이나 `set`이 준 것을 쓰는 것
- 가드가 쥐고 있는 동안(set으로 잠금에 새 값을 주기 전까지) 그 값을 넘기는 것
- 그것을 돌려주거나, yield하거나, 블록의 값으로 삼거나, 블록 뒤에 실행하는 코루틴에 주는 것

값은 `using` 블록 안에서 다루거나 `clone()`을 남겨 두세요. 가드는 `!Sync`이고 자기 `using` 블록 안에서만 쓸 수 있습니다. 블록이 끝난 뒤 가드의 복사본(컬렉션, 클로저, 필드 안의 것)을 쓰면 컴파일 에러입니다. 다른 스레드에서 도는 클로저에서 값을 쓰려면 먼저 읽고(`let v = g.get();`) `v`를 캡처합니다. 실행 시에는 `new`, `set`, `update`가 값이 들여오는 컬렉션의 복사본을 저장합니다(2.13.9). `new_unshared`는 복사를 건너뜁니다. 이때 `value`의 컬렉션에 다른 것이 닿지 않는다는 점(방금 만든 값)은 호출자가 보장합니다.

```rok
static STATE: Mutex<List<i32>> = Mutex::new([]);

task::spawn(async {
    using g = STATE.lock() {
        g.update(|xs| { xs.push(42); xs });
    }   // ← g.drop() → 잠금 해제 ([2.13 marker](213-marker.md))
});
```

### deadlock 회피: `try_lock`

교착 상태를 피하려면 블록하지 않고 `try_lock`으로 한 번 시도해 볼 수 있습니다.

```rok
match STATE.try_lock() {
    v: MutexGuard<...> => using g = v { g.push(1); },
    none => { log("locked, skip"); },
}
```

### poisoning: 없음

Rok의 Mutex는 Rust의 PoisonError 모델과 다릅니다. panic이 일어나도 잠금은 자동으로 풀리고, 다음 task는 정상적으로 들어갈 수 있습니다. 이렇게 설계한 이유는 두 가지입니다.

1. panic은 잡을 수 없고 프로세스를 끝내므로 panic 도중의 잠금 상태를 다른 task가 볼 일이 없습니다. 에러가 `throw`로 전파되어 `using` 블록을 벗어날 때에는 [`using`이 Drop을 호출](213-marker.md#2134-drop-using-해제-시-정리)합니다. 그래서 `MutexGuard`의 drop이 잠금을 풉니다.
2. poisoning은 보수적으로 보이지만 쓰는 쪽의 부담이 크고, 대부분 거짓 양성(false positive)입니다.

에러로 빠져나가기 전에 일관성을 꼭 되돌려야 하는 자리에서는 다음과 같이 직접 처리합니다.

```rok
using g = MUTEX.lock() {
    try unsafe_op(g) catch e {
        g.reset_to_known_good_state();
        throw e;
    };
}
```

## 2.16.4. `RwLock<T>`: 읽기와 쓰기 분리

읽기가 쓰기보다 훨씬 많은 자리에는 `RwLock<T>`가 더 맞습니다. 읽는 쪽 여럿과 쓰는 쪽 하나를 나눠 처리량을 높입니다.

```rok
struct RwLock<T> { /* ... */ }

impl<T> RwLock<T> {
    fn new(value: T) -> RwLock<T>;

    fn read(self) -> RwLockReadGuard<T>;        // 다중 reader OK
    fn write(self) -> RwLockWriteGuard<T>;      // 단일 writer

    fn try_read(self) -> RwLockReadGuard<T> | none;
    fn try_write(self) -> RwLockWriteGuard<T> | none;

    fn read_async(self) -> ReadFuture<T>;       // await l.read_async() - task 만 중단
    fn write_async(self) -> WriteFuture<T>;
}
```

설정이나 캐시처럼 읽기가 대부분인 상태에 알맞습니다. 쓰기가 드문 작업에서는 Mutex보다 처리량이 높습니다.

```rok
static CONFIG: RwLock<Config> = RwLock::new(Config::default());

using c = CONFIG.read() {        // 다른 reader 와 동시 OK
    process(c);
}   // ← read guard drop, lock 해제

using w = CONFIG.write() {       // 잠시 모든 reader 차단
    w.reload_from_file();
}
```

`RwLock<T>`는 `T`가 `Send + Sync`일 때만 `Sync`입니다([2.16.5](#send--sync-마커)). 그래서 여기서 `Config`는 `Sync`여야 합니다.

`Mutex`와 마찬가지로 `RwLock::new(value)`는 `value`를 넘겨받습니다. 그 뒤에 호출자의 핸들을 쓰면 컴파일 에러입니다. `using` 블록이 끝난 뒤 가드의 `get`이 돌려준 것이나 `set`이 준 것을 쓰는 것도 컴파일 에러입니다. 가드가 쥐고 있는 동안(set으로 잠금에 새 값을 주기 전까지) 그것을 넘기는 것, 블록 안에서 그것을 돌려주는 것도 마찬가지입니다([2.13.9](213-marker.md#2139-send--sync-스레드-경계-마커)). 값은 `using` 블록 안에서 다루거나 `clone()`을 남겨 두세요. `Mutex`와 마찬가지로 `new`와 쓰기 가드의 `set`, `update`는 복사본을 저장합니다.

`RwLock` 가드에도 같은 규칙이 적용됩니다. `read` / `write`의 가드를 `using`이 풀지 않는 일반 `let`(또는 `if let`)으로 묶거나 버리면 컴파일 에러입니다.

## 2.16.4a. `Once` / `Condvar`

Rust와 같은 모양의 기본 요소 두 가지입니다. poisoning은 없습니다.

```rok
struct Once { /* ... */ }
impl Once {
    const fn new() -> Once;
    fn call_once(self, f: fn());          // 처음 한 번만 f 실행, 동시 호출자는 끝날 때까지 대기
}

struct Condvar { /* ... */ }
impl Condvar {
    const fn new() -> Condvar;
    fn wait<T>(self, guard: MutexGuard<T>) -> MutexGuard<T>;   // 잠금을 풀고 대기, 깨면 다시 잠금
    fn wait_async<T>(self, guard: MutexGuard<T>) -> CondvarWait<T>;   // g = await cv.wait_async(g) - task 만 중단
    fn notify_one(self);
    fn notify_all(self);
}
```

`wait`는 이유 없이 깨어날(spurious wakeup) 수 있으므로 조건을 루프에서 다시 확인합니다. `notify_one`은 기다리는 쪽을 적어도 하나 깨웁니다. 기다리는 쪽이 async이면 둘 이상 깨울 수도 있습니다.

### async 잠금과 취소

`lock_async` / `read_async` / `write_async` / `wait_async`의 future는 취소된 poll에서 잠금을 잡지 않습니다([2.18.4](218-task.md#2184-협력적-취소-taskcancel)). 취소된 `wait_async`는 다음과 같이 끝납니다.

- 뮤텍스가 비어 있으면 다시 잡습니다. 그래서 호출자의 `using` 가드가 평소처럼 풉니다.
- 다른 task가 잡고 있으면 잡지 않고 떠납니다. 버려지는 가드는 그 뮤텍스를 풀지 않습니다.

## 2.16.4b. `Frozen<T>`: 읽기 전용 공유

한 번 만든 뒤 읽기만 하는 값(설정, 신뢰 앵커, 표)은 `Frozen<T>`로 공유합니다. 어느 스레드든 잠금 없이 읽을 수 있습니다.

```rok
struct Frozen<T> { /* ... */ }   // T: Send 이면 Sync

impl<T> Frozen<T> {
    const fn new(value: T) -> Frozen<T>;   // 다른 것이 닿지 않는 복사본을 둡니다
    fn get(self) -> T;                       // 읽는 쪽만의 복사본
}
```

`new`는 넘김(hand-off)과 마찬가지로 `value`가 닿는 컬렉션의 복사본을 둡니다([2.13.9](213-marker.md#2139-send--sync-스레드-경계-마커)). 그래서 바깥의 어떤 것도 그 안에 쥔 것에 닿지 않습니다. `get`은 읽는 쪽마다 따로 복사본을 줍니다. 그 복사본을 바꿔도 다른 읽는 쪽이 보는 값은 바뀌지 않습니다. `Sync`인 구성 요소(`Mutex`, 스스로 잠그는 캐시, 바뀌지 않는 `Sync` 타입의 값)는 복사하지 않고 공유합니다. 그래서 `get`의 비용은 값의 나머지 컬렉션을 복사하는 만큼입니다. 큰 읽기 전용 부분은 이 복사에서 빼 두세요. `Sync` 타입으로 두거나, 필요한 곳에서 읽는 별도의 `Frozen` 필드에 두면 됩니다.

```rok
struct Settings {
    name: str;
    limits: List<i64>;               // get 마다 복사: 작게
    table: Frozen<List<Entry>>;      // 공유: 필요한 곳에서 table.get()
}

static SETTINGS: Frozen<Settings> = Frozen::new(load());
```

표준 라이브러리의 TLS `Certificate`가 그런 타입입니다. 파싱한 뒤로 바뀌지 않고 접근자가 복사본을 줍니다. 그래서 `Frozen<ClientConfig>`의 `get`은 신뢰 앵커를 공유합니다.

## 2.16.5. `Cell<T>`: 단일 스레드 내부 가변성

`Cell<T>`는 단일 task 안에서만 동작하는 내부 가변성(interior mutability)입니다.

```rok
struct Cell<T> { /* ... */ }

impl<T> Cell<T> {
    fn new(value: T) -> Cell<T>;
    fn get(self) -> T            where T: Copy;
    fn set(ref self, value: T);
    fn replace(ref self, value: T) -> T;
    fn take(ref self) -> T       where T: Default;
}
```

`Cell<T>`는 스레드 로컬 내부 가변성이므로 여러 스레드에서는 안전하지 않습니다. 단일 task 안에서만 공유할 수 있고, 잠금이 없어서 매우 빠릅니다.

```rok
let c: Cell<i32> = Cell::new(0);
c.set(42);
let v = c.get();          //→ 42

// task 간 공유 - 컴파일 에러 (Cell 은 Send / Sync 가 정의되지 않음)
task::spawn(async { c.set(100) });       // ERROR - Cell 은 cross-task 안전 X
```

### `Send` / `Sync` 마커

`Send` / `Sync`는 [2.13.9](213-marker.md#2139-send--sync-스레드-경계-마커)의 자동 마커 트레잇입니다. 이 장의 타입은 다음 정책을 따릅니다.

- `Atomic<T>`: Send + Sync
- `Mutex<T>`: `T: Send`이면 Send + Sync
- `RwLock<T>`: `T: Send`이면 Send, `T: Send + Sync`이면 Sync. 여러 읽는 쪽이 값을 동시에 공유하므로 컬렉션(`!Sync`)의 `RwLock`은 스레드 사이에서 공유할 수 없습니다. `Mutex<T>`를 씁니다.
- `MutexGuard<T>`, `RwLockReadGuard<T>`, `RwLockWriteGuard<T>`: Send, `!Sync`. 가드는 자기 `using` 블록 안에서만 씁니다.
- `Frozen<T>`: `T: Send`이면 Sync (2.16.4b)
- `Cell<T>`: 둘 다 아님(단일 task 전용). 위 `task::spawn`이 에러인 이유는 `async` 블록이 `Cell`을 캡처해서 `Send`가 아니기 때문입니다.

## 2.16.6. 모듈 매핑

각 기본 요소의 구현 위치입니다.

| 타입         | 모듈                     | 내부 구조                        |
|--------------|--------------------------|----------------------------------|
| `Atomic<T>`  | `rok::core::sync::atomic` | LLVM atomic ops (`rok.atomic.*`) |
| `Mutex<T>`   | `rok::core::sync::mutex`  | futex / wait queue (타깃별)      |
| `RwLock<T>`  | `rok::core::sync::rwlock` | futex + reader counter           |
| `Cell<T>`    | `rok::core::sync::cell`   | 단순 래퍼, 잠금 없음             |
| `MutexGuard` | `rok::core::sync::mutex`  | RAII via Drop                    |
