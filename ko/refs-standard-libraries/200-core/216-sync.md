# 2.16. sync - 동시성 primitive

`rok::core::sync` 모듈은 공유 상태에 안전하게 접근할 수 있도록 도와주는 primitive 트레잇과 타입을 정의합니다. Rok 의 GC 와 M:N 스케줄러 환경에서 multi-task 또는 multi-thread 사이의 데이터 공유를 처리할 때 사용합니다.

`use rok::core::**;` 가 묵시적으로 적용되므로 별도의 import 없이도 `Atomic`, `Mutex`, `RwLock`, `Cell` 을 곧바로 사용할 수 있습니다. 다만 동시성 코드임을 시각적으로 분명히 드러내고 싶은 자리에서는 다음과 같이 좁은 범위의 import 를 권장합니다.

```rok
use rok::core::sync::{Atomic, Mutex, RwLock, Once, Condvar, Cell};
```

## 2.16.1. 4 primitive 개요

네 가지 primitive 는 각각 서로 다른 동시성 시나리오를 담당합니다. 다음 표는 각 타입의 의미와 어울리는 사용 자리를 한눈에 정리한 것입니다.

| 타입        | 의미                                       | 사용 자리                      |
|-------------|--------------------------------------------|--------------------------------|
| `Atomic<T>` | lock-free 원자 연산 (T = primitive)        | counter, flag, sequence number |
| `Mutex<T>`  | 배타적 잠금 (single writer)                | shared mutable state           |
| `RwLock<T>` | 다중 reader 또는 단일 writer               | read-heavy state               |
| `Cell<T>`   | thread-local interior mutability (no lock) | single-threaded mutable        |

## 2.16.2. `Atomic<T>` - lock-free 원자

가장 가벼운 primitive 인 `Atomic<T>` 부터 살펴봅니다. 내부적으로는 T 별 LLVM atomic 연산으로 구현되며, 다음과 같은 형태를 갖춥니다.

```rok
struct Atomic<T> {
    // 내부: T 별 LLVM atomic 연산으로 다루는 원소 하나짜리 cell
}

trait AtomicPrimitive { /* marker: i8/i16/i32/i64/isize, u8/u16/u32/u64/usize, bool, Ptr<T> */ }

impl<T: AtomicPrimitive + PartialEq> Atomic<T> {
    const fn new(value: T) -> Atomic<T>;

    fn load(self, ordering: AtomicOrdering = SeqCst) -> T;
    fn store(self, value: T, ordering: AtomicOrdering = SeqCst);
    fn swap(self, value: T, ordering: AtomicOrdering = SeqCst) -> T;
    fn compare_exchange(
        self, current: T, new: T,
        success: AtomicOrdering, failure: AtomicOrdering,
    ) -> Result<T, T>;                  // 실패 결과도 값, 에러가 아님
    fn compare_exchange_weak(
        self, current: T, new: T,
        success: AtomicOrdering, failure: AtomicOrdering,
    ) -> Result<T, T>;                  // 실패 결과도 값, 에러가 아님

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

`wait` 는 이유 없이 돌아올 수 있으므로 호출자가 조건을 다시 확인합니다 (futex 의미).

`load` 와 `compare_exchange` / `compare_exchange_weak` 의 `failure` ordering 은 load ordering 입니다. 이 자리에 `Release` 나 `AcqRel` 을 주면 panic 합니다. 이 구현에서 `compare_exchange_weak` 는 이유 없이 실패하지 않으며 `compare_exchange` 와 똑같이 동작합니다. `compare_exchange` 가 이전 값을 `current` 와 비교하므로 `PartialEq` 가 필요하며, `Ptr<T>` 는 주소로 비교합니다.

### `AtomicOrdering`

원자 연산은 메모리 순서 보장 수준에 따라 다섯 가지로 나뉘며, 각 변형의 의미는 다음과 같습니다.

```rok
enum AtomicOrdering {
    Relaxed,        // 동기화 없음 (counter 등)
    Acquire,        // load 후 동기화 보장
    Release,        // store 전 동기화 보장
    AcqRel,         // RMW 시 양방향
    SeqCst,         // 전역 순차 일관성 (default, 안전)
}
```

이 모델은 C++ 와 Rust 의 atomic ordering 모델과 동형이며, 기본값은 안전한 `SeqCst` 입니다. 이름이 `AtomicOrdering` 인 것은 암묵 core glob 아래에서 비교용 `cmp::Ordering` 과 겹치지 않게 하기 위함이며, 변형 (`Relaxed`, `Acquire`, `Release`, `AcqRel`, `SeqCst`) 은 접두 없이 쓸 수 있습니다. ordering 인자는 컴파일 타임 상수여야 합니다. 따라서 약한 ordering 은 성능이 critical 한 자리에서만 신중하게 사용하는 것이 좋습니다.

### 사용 예

가장 흔한 용도인 카운터 증가는 다음과 같이 작성합니다.

```rok
static COUNTER: Atomic<i32> = Atomic::new(0);

task::spawn(async { COUNTER.fetch_add(1, Relaxed); });
task::spawn(async { COUNTER.fetch_add(1, Relaxed); });
// 결과: COUNTER == 2 (race 없음)

let v = COUNTER.load(SeqCst);     //→ 2
```

### CAS (compare-and-swap)

atomic 이 직접 제공하지 않는 갱신은 compare-and-swap 루프 위에서 구성합니다. 다음은 지금까지 저장된 값 중 가장 큰 값을 유지하는 예시입니다.

```rok
fn store_max(cell: Atomic<i64>, x: i64) {
    let cur = cell.load(Acquire);
    loop {
        if cur >= x { break; }
        match cell.compare_exchange(cur, x, AcqRel, Acquire) {
            Ok(_)       => break,
            Err(actual) => cur = actual,          // 다른 task 가 먼저. 그 값으로 재시도
        }
    }
}
```

원소 타입은 `AtomicPrimitive` 여야 합니다. `List<T>` 같은 컬렉션은 `Atomic` 에 담을 수 없으며, 공유 컬렉션은 `Mutex<T>` (2.16.3) 에 둡니다.

## 2.16.3. `Mutex<T>` - 배타적 잠금

원자 연산만으로 표현하기 어려운 공유 상태에는 `Mutex<T>` 를 사용합니다. 내부는 futex 와 payload 로 구성되며, `using` 으로 묶은 가드가 잠금을 해제합니다.

```rok
struct Mutex<T> {
    // 내부: futex + payload
}

impl<T> Mutex<T> {
    fn new(value: T) -> Mutex<T>;
    unsafe fn new_unshared(value: T) -> Mutex<T>;   // 넘김 복사 없는 new

    fn lock(self) -> MutexGuard<T>;            // 스레드를 block
    fn lock_async(self) -> LockFuture<T>;      // await m.lock_async(): task 만 중단 (Future<Output = MutexGuard<T>>)
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

guard 는 자신이 보호하는 값의 메서드를 전달하지 않습니다. `MutexGuard<List<i32>>` 에 `g.push(1)` 을 쓰면 에러입니다. 값에는 `get`, `set`, `update` 로 접근합니다.

### 사용 예

가드를 `using` 으로 묶으면 블록을 빠져나갈 때 `drop` 되어 잠금이 해제됩니다. `using` 없이 받은 가드는 자동으로 풀리지 않으므로 반드시 `using` 으로 묶습니다. 가드는 호출, 메서드 호출, `await` 에서 받습니다. 받은 가드는 바인딩 (`let`, 패턴, `using`) 을 초기화하거나, 변수에 대입하거나, 호출자에게 넘기는 (`return`, 함수의 값) 데에만 쓸 수 있습니다. 함수에 넘기거나, 그 위에서 메서드를 부르거나, 필드나 컬렉션에 저장하거나, 버리면 컴파일 에러입니다. `using` 으로 묶은 가드는 함수에 넘길 수 있으며, `using` 없이 묶은 가드는 넘긴다고 해서 해제되지 않습니다. `using` 으로 해제되지 않는 일반 `let` (또는 `if let`) 으로 가드를 묶거나, `lock` 이 돌려준 가드를 버리면 컴파일 에러입니다.

`Mutex::new(value)` 는 `value` 를 넘겨받습니다: 그 뒤에 호출자의 핸들을 쓰면 컴파일 에러이며, guard 의 `using` 블록이 끝난 뒤 `get` 이 돌려준 것이나 `set` 이 준 것을 쓰거나, guard 가 쥔 동안 그것을 넘기거나, 블록 안에서 그것을 돌려주거나, yield 하거나, 블록의 값으로 삼거나, 블록 뒤에 실행하는 코루틴에 주는 것도 그렇습니다 ([2.13.9](213-marker.md#2139-send--sync---스레드-경계-marker)). `using` 블록 안에서 다루거나 `clone()` 을 남깁니다. guard 는 `!Sync` 이고 자기 `using` 블록 안에서만 쓸 수 있습니다: 블록이 끝난 뒤 guard 의 복사본 (컬렉션, 클로저, 필드 안의 것) 을 쓰면 컴파일 에러입니다. 다른 스레드에서 도는 클로저에서 값을 쓰려면 먼저 읽고 (`let v = g.get();`) `v` 를 잡습니다. 실행 중에는 `new`, `set`, `update` 가 값이 들여오는 컬렉션의 복사본을 저장합니다 (2.13.9). `new_unshared` 는 복사하지 않으며, 호출자가 `value` 의 컬렉션에 다른 것이 닿지 않음 (방금 만든 값) 을 보장합니다.

```rok
static STATE: Mutex<List<i32>> = Mutex::new([]);

task::spawn(async {
    using g = STATE.lock() {
        g.update(|xs| { xs.push(42); xs });
    }   // ← g.drop() → 잠금 해제 ([2.13 marker](213-marker.md))
});
```

### deadlock 회피 - `try_lock`

deadlock 을 피하고 싶을 때는 blocking 대신 `try_lock` 으로 비차단 시도를 할 수 있습니다.

```rok
match STATE.try_lock() {
    v: MutexGuard<List<i32>> => using g = v { g.update(|xs| { xs.push(1); xs }); },
    none => { log("locked, skip"); },
}
```

### poisoning - **없음**

Rok 의 Mutex 는 Rust 의 PoisonError 모델과 다릅니다. panic 이 발생해도 잠금은 자동으로 해제되며, 다음 task 는 정상적으로 진입할 수 있습니다. 이렇게 설계한 이유는 두 가지입니다.

1. panic 은 잡을 수 없고 프로세스를 종료하므로, panic 도중의 잠금 상태를 다른 task 가 관측하는 일이 없습니다. 에러가 `throw` 로 전파되어 `using` 블록을 벗어날 때에는 [`using` 이 Drop 을 호출](213-marker.md#2134-drop---using-해제-시-정리)하므로, `MutexGuard` 의 drop 이 잠금 해제를 책임집니다.
2. poison 의 의미는 보수적으로 보이지만 사용자 부담이 크고, 대부분의 경우 false positive 입니다.

에러로 빠져나가기 전에 일관성 복원이 정말로 필요한 자리에서는 다음과 같이 명시적으로 처리하면 됩니다.

```rok
using g = MUTEX.lock() {
    try unsafe_op(g) catch e {
        g.set(known_good_state());
        throw e;
    };
}
```

## 2.16.4. `RwLock<T>` - read-write 분리

읽기가 압도적으로 많은 자리에는 `RwLock<T>` 가 더 적합합니다. 다중 reader 와 단일 writer 를 분리하여 throughput 을 끌어올립니다.

```rok
struct RwLock<T> { /* ... */ }

impl<T> RwLock<T> {
    fn new(value: T) -> RwLock<T>;

    fn read(self) -> RwLockReadGuard<T>;        // 다중 reader OK
    fn write(self) -> RwLockWriteGuard<T>;      // 단일 writer

    fn try_read(self) -> RwLockReadGuard<T> | none;
    fn try_write(self) -> RwLockWriteGuard<T> | none;

    fn read_async(self) -> ReadFuture<T>;       // await l.read_async(): task 만 중단
    fn write_async(self) -> WriteFuture<T>;
}
```

config 나 cache 처럼 read 가 압도적으로 많은 자리에서 사용하면 좋고, write 가 드문 워크로드에서는 Mutex 보다 throughput 이 우수합니다.

```rok
static CONFIG: RwLock<Config> = RwLock::new(Config::default());

using c = CONFIG.read() {        // 다른 reader 와 동시 OK
    process(c);
}   // ← read guard drop, lock 해제

using w = CONFIG.write() {       // 잠시 모든 reader 차단
    w.set(Config::load_from_file());
}
```

`RwLock<T>` 는 `T` 가 `Send + Sync` 일 때만 `Sync` 이므로 ([2.16.5](#send--sync-마커)), 여기서 `Config` 는 `Sync` 여야 합니다.

`Mutex` 와 마찬가지로 `RwLock::new(value)` 는 `value` 를 넘겨받습니다: 그 뒤에 호출자의 핸들을 쓰면 컴파일 에러이며, `using` 블록이 끝난 뒤 guard 의 `get` 이 돌려준 것이나 `set` 이 준 것을 쓰거나, guard 가 쥔 동안 그것을 넘기거나, 블록 안에서 그것을 돌려주는 것도 그렇습니다 ([2.13.9](213-marker.md#2139-send--sync---스레드-경계-marker)). `using` 블록 안에서 다루거나 `clone()` 을 남깁니다. `Mutex` 와 마찬가지로 `new` 와 쓰기 guard 의 `set`, `update` 는 복사본을 저장합니다.

`RwLock` 가드에도 같은 규칙이 적용됩니다: `read` / `write` 의 가드를 `using` 이 해제하지 않는 일반 `let` (또는 `if let`) 으로 묶거나 버리면 컴파일 에러입니다.

## 2.16.4a. `Once` / `Condvar`

Rust 와 같은 모양의 두 primitive 입니다. poisoning 은 없습니다.

```rok
struct Once { /* ... */ }
impl Once {
    const fn new() -> Once;
    fn call_once(self, f: fn());          // 처음 한 번만 f 실행, 동시 호출자는 끝날 때까지 대기
    fn is_completed(self) -> bool;        // call_once 가 한 번 끝났으면 true
}

struct Condvar { /* ... */ }
impl Condvar {
    const fn new() -> Condvar;
    fn wait<T>(self, guard: MutexGuard<T>) -> MutexGuard<T>;   // 잠금을 풀고 대기, 깨면 다시 잠금
    fn wait_async<T>(self, guard: MutexGuard<T>) -> CondvarWait<T>;   // g = await cv.wait_async(g): task 만 중단
    fn notify_one(self);
    fn notify_all(self);
}
```

`wait` 는 spurious wakeup 이 있을 수 있으므로 조건을 루프에서 다시 확인합니다. `notify_one` 은 적어도 하나의 waiter 를 깨우며, async waiter 는 둘 이상 깨울 수도 있습니다.

### async 잠금과 취소

`lock_async` / `read_async` / `write_async` / `wait_async` 의 future 는 취소된 poll 에서 잠금을 잡지 않습니다 ([2.18.4](218-task.md#2184-협력적-취소---taskcancel)). 취소된 `wait_async` 는 다음과 같이 끝납니다.

- mutex 가 비어 있으면 다시 잡습니다. 그래서 호출자의 `using` guard 가 평소처럼 풉니다.
- 다른 task 가 잡고 있으면 잡지 않고 떠나며, 버려지는 guard 는 그 mutex 를 풀지 않습니다.

## 2.16.4b. `Frozen<T>` - 읽기 전용 공유

한 번 만들고 그 뒤로는 읽기만 하는 값 (설정, 신뢰 앵커, 표) 은 `Frozen<T>` 로 공유합니다. 어느 스레드든 잠금 없이 읽을 수 있습니다.

```rok
struct Frozen<T> { /* ... */ }   // T: Send 이면 Sync

impl<T> Frozen<T> {
    const fn new(value: T) -> Frozen<T>;   // 다른 것이 닿지 않는 복사본을 둡니다
    fn get(self) -> T;                       // 읽는 쪽만의 복사본
}
```

`new` 는 넘김과 같이 `value` 가 닿는 컬렉션의 복사본을 두므로 ([2.13.9](213-marker.md#2139-send--sync---스레드-경계-marker)), 바깥의 어떤 것도 그것이 쥔 것에 닿지 않습니다. `get` 은 읽는 쪽마다 자기 복사본을 줍니다: 그것을 바꿔도 다른 읽는 쪽이 보는 것은 바뀌지 않습니다. `Sync` 인 구성 요소 (`Mutex`, 스스로 잠그는 캐시, 바뀌지 않는 `Sync` 타입의 값) 는 복사하지 않고 공유하므로, `get` 은 값의 나머지 컬렉션을 복사하는 만큼만 듭니다. 큰 읽기 전용 부분은 그 복사에서 빼 둡니다: `Sync` 타입으로 두거나, 필요한 곳에서 읽는 자기만의 `Frozen` 필드에 둡니다.

```rok
struct Settings {
    name: str;
    limits: List<i64>;               // get 마다 복사: 작게
    table: Frozen<List<Entry>>;      // 공유: 필요한 곳에서 table.get()
}

static SETTINGS: Frozen<Settings> = Frozen::new(load());
```

표준 라이브러리의 TLS `Certificate` 가 그런 타입입니다: 파싱한 뒤로 바뀌지 않고 접근자가 복사본을 주므로, `Frozen<ClientConfig>` 의 `get` 은 신뢰 앵커를 공유합니다.

## 2.16.5. `Cell<T>` - single-threaded interior

마지막으로 `Cell<T>` 는 단일 task 안에서만 동작하는 interior mutability 를 제공합니다.

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

`Cell<T>` 는 thread-local interior mutability 를 의미하므로 multi-thread 에서는 안전하지 않습니다. 단일 task 안에서만 공유할 수 있고, 잠금이 없으므로 매우 빠르다는 특징이 있습니다.

```rok
let c: Cell<i32> = Cell::new(0);
c.set(42);
let v = c.get();          //→ 42

// task 간 공유: 컴파일 에러 (Cell 은 Send / Sync 가 정의되지 않음)
task::spawn(async { c.set(100) });       // ERROR: Cell 은 cross-task 안전 X
```

### `Send` / `Sync` 마커

`Send` / `Sync` 는 [2.13.9](213-marker.md#2139-send--sync---스레드-경계-marker) 의 자동 marker 트레잇입니다. 이 장의 타입은 다음 정책을 따릅니다.

- `Atomic<T>`: Send + Sync
- `Mutex<T>`: `T: Send` 이면 Send + Sync
- `RwLock<T>`: `T: Send` 이면 Send, `T: Send + Sync` 이면 Sync. 여러 reader 가 값을 동시에 공유하므로, 컬렉션 (`!Sync`) 의 `RwLock` 은 스레드 사이에서 공유할 수 없습니다. `Mutex<T>` 를 씁니다.
- `MutexGuard<T>`, `RwLockReadGuard<T>`, `RwLockWriteGuard<T>`: Send, `!Sync`. guard 는 자기 `using` 블록 안에서만 씁니다
- `Frozen<T>`: `T: Send` 이면 Sync (2.16.4b)
- `Cell<T>`: 없음 (single-task only). 위 `task::spawn` 이 에러인 이유는 `async` 블록이 `Cell` 을 캡처해 `Send` 가 아니기 때문입니다

## 2.16.6. 모듈 매핑

각 primitive 의 구현은 다음과 같이 매핑됩니다.

| 타입         | 모듈                     | 내부 구조                        |
|--------------|--------------------------|----------------------------------|
| `Atomic<T>`  | `rok::core::sync::atomic` | LLVM atomic ops (`rok.atomic.*`) |
| `Mutex<T>`   | `rok::core::sync::mutex`  | futex / wait queue (target 별)   |
| `RwLock<T>`  | `rok::core::sync::rwlock` | futex + reader counter           |
| `Cell<T>`    | `rok::core::sync::cell`   | 단순 wrapper, no lock            |
| `MutexGuard` | `rok::core::sync::mutex`  | RAII via Drop                    |
