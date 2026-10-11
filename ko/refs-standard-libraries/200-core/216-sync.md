# 2.16. sync — 동시성 primitive

`rok::core::sync` 모듈은 공유 상태에 안전하게 접근할 수 있도록 도와주는 primitive 트레잇과 타입을 정의합니다. rok 의 GC 와 M:N 스케줄러 환경에서 multi-task 또는 multi-thread 사이의 데이터 공유를 처리할 때 사용합니다.

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

## 2.16.2. `Atomic<T>` — lock-free 원자

가장 가벼운 primitive 인 `Atomic<T>` 부터 살펴봅니다. 내부적으로는 T 별 LLVM atomic 연산으로 구현되며, 다음과 같은 형태를 갖춥니다.

```rok
struct Atomic<T: AtomicPrimitive> {
    // 내부: AtomicU32 / AtomicU64 / AtomicPtr 등 (T 별 LLVM atomic ops)
}

trait AtomicPrimitive { /* marker — i32/i64/u32/u64/usize/isize/bool/Ptr */ }

impl<T: AtomicPrimitive> Atomic<T> {
    const fn new(value: T) -> Atomic<T>;

    fn load(self, ordering: AtomicOrdering = SeqCst) -> T;
    fn store(self, value: T, ordering: AtomicOrdering = SeqCst);
    fn swap(self, value: T, ordering: AtomicOrdering = SeqCst) -> T;
    fn compare_exchange(
        self, current: T, new: T,
        success: AtomicOrdering, failure: AtomicOrdering,
    ) -> Result<T, T>;                  // 실패 결과도 값 — 에러가 아님
    fn compare_exchange_weak(
        self, current: T, new: T,
        success: AtomicOrdering, failure: AtomicOrdering,
    ) -> Result<T, T>;                  // 실패 결과도 값 — 에러가 아님

    // 정수 전용
    fn fetch_add(self, value: T, ordering: AtomicOrdering = SeqCst) -> T;
    fn fetch_sub(self, value: T, ordering: AtomicOrdering = SeqCst) -> T;
    fn fetch_and(self, value: T, ordering: AtomicOrdering = SeqCst) -> T;
    fn fetch_or(self, value: T, ordering: AtomicOrdering = SeqCst) -> T;
    fn fetch_xor(self, value: T, ordering: AtomicOrdering = SeqCst) -> T;
}
```

### `AtomicOrdering`

원자 연산은 메모리 순서 보장 수준에 따라 다섯 가지로 나뉘며, 각 변형의 의미는 다음과 같습니다.

```rok
enum AtomicOrdering {
    Relaxed,        // 동기화 없음 — counter 등
    Acquire,        // load 후 동기화 보장
    Release,        // store 전 동기화 보장
    AcqRel,         // RMW 시 양방향
    SeqCst,         // 전역 순차 일관성 (default — 안전)
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

좀 더 복잡한 lock-free 자료 구조는 compare-and-swap 루프 위에서 구성합니다. 다음은 lock-free 스택의 push 예시입니다.

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
            Err(_) => continue,                   // 다른 task 가 먼저 — 재시도
        }
    }
}
```

## 2.16.3. `Mutex<T>` — 배타적 잠금

원자 연산만으로 표현하기 어려운 공유 상태에는 `Mutex<T>` 를 사용합니다. 내부는 futex 와 payload 로 구성되며, `using` 으로 묶은 가드가 잠금을 해제합니다.

```rok
struct Mutex<T> {
    // 내부: futex + payload
}

impl<T> Mutex<T> {
    fn new(value: T) -> Mutex<T>;

    fn lock(self) -> MutexGuard<T>;            // 스레드를 block
    fn lock_async(self) -> LockFuture<T>;      // await m.lock_async() — task 만 중단 (Future<Output = MutexGuard<T>>)
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

가드를 `using` 으로 묶으면 블록을 빠져나갈 때 `drop` 되어 잠금이 해제됩니다. `using` 없이 받은 가드는 자동으로 풀리지 않으므로 반드시 `using` 으로 묶습니다.

```rok
static STATE: Mutex<List<i32>> = Mutex::new([]);

task::spawn(async {
    using g = STATE.lock() {
        g.update(|xs| { xs.push(42); xs });
    }   // ← g.drop() → 잠금 해제 ([2.13 marker](213-marker.md))
});
```

### deadlock 회피 — `try_lock`

deadlock 을 피하고 싶을 때는 blocking 대신 `try_lock` 으로 비차단 시도를 할 수 있습니다.

```rok
match STATE.try_lock() {
    v: MutexGuard<...> => using g = v { g.push(1); },
    none => { log("locked, skip"); },
}
```

### poisoning — **없음**

rok 의 Mutex 는 Rust 의 PoisonError 모델과 다릅니다. panic 이 발생해도 잠금은 자동으로 해제되며, 다음 task 는 정상적으로 진입할 수 있습니다. 이렇게 설계한 이유는 두 가지입니다.

1. panic 은 잡을 수 없고 프로세스를 종료하므로, panic 도중의 잠금 상태를 다른 task 가 관측하는 일이 없습니다. 에러가 `throw` 로 전파되어 `using` 블록을 벗어날 때에는 [`using` 이 Drop 을 호출](213-marker.md#2134-drop--using-해제-시-정리)하므로, `MutexGuard` 의 drop 이 잠금 해제를 책임집니다.
2. poison 의 의미는 보수적으로 보이지만 사용자 부담이 크고, 대부분의 경우 false positive 입니다.

에러로 빠져나가기 전에 일관성 복원이 정말로 필요한 자리에서는 다음과 같이 명시적으로 처리하면 됩니다.

```rok
using g = MUTEX.lock() {
    try unsafe_op(g) catch e {
        g.reset_to_known_good_state();
        throw e;
    };
}
```

## 2.16.4. `RwLock<T>` — read-write 분리

읽기가 압도적으로 많은 자리에는 `RwLock<T>` 가 더 적합합니다. 다중 reader 와 단일 writer 를 분리하여 throughput 을 끌어올립니다.

```rok
struct RwLock<T> { /* ... */ }

impl<T> RwLock<T> {
    fn new(value: T) -> RwLock<T>;

    fn read(self) -> RwLockReadGuard<T>;        // 다중 reader OK
    fn write(self) -> RwLockWriteGuard<T>;      // 단일 writer

    fn try_read(self) -> RwLockReadGuard<T> | none;
    fn try_write(self) -> RwLockWriteGuard<T> | none;
}
```

config 나 cache 처럼 read 가 압도적으로 많은 자리에서 사용하면 좋고, write 가 드문 워크로드에서는 Mutex 보다 throughput 이 우수합니다.

```rok
static CONFIG: RwLock<Config> = RwLock::new(Config::default());

using c = CONFIG.read() {        // 다른 reader 와 동시 OK
    process(c);
}   // ← read guard drop, lock 해제

using w = CONFIG.write() {       // 잠시 모든 reader 차단
    w.reload_from_file();
}
```

## 2.16.4a. `Once` / `Condvar`

Rust 와 같은 모양의 두 primitive 입니다. poisoning 은 없습니다.

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
    fn notify_one(self);
    fn notify_all(self);
}
```

`wait` 는 spurious wakeup 이 있을 수 있으므로 조건을 루프에서 다시 확인합니다.

## 2.16.5. `Cell<T>` — single-threaded interior

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

// task 간 공유 — 컴파일 에러 (Cell 은 Send / Sync 가 정의되지 않음)
task::spawn(async { c.set(100) });       // ERROR — Cell 은 cross-task 안전 X
```

### `Send` / `Sync` 마커

`Send` / `Sync` 는 [2.13.9](213-marker.md#2139-send--sync--스레드-경계-marker) 의 자동 marker 트레잇입니다. 이 장의 타입은 다음 정책을 따릅니다.

- `Atomic<T>`: Send + Sync
- `Mutex<T>` / `RwLock<T>`: Send + Sync (T: Send 시)
- `Cell<T>`: 없음 (single-task only) — 위 `task::spawn` 이 에러인 이유는 `async` 블록이 `Cell` 을 캡처해 `Send` 가 아니기 때문입니다

## 2.16.6. 모듈 매핑

각 primitive 의 구현은 다음과 같이 매핑됩니다.

| 타입         | 모듈                     | 내부 구조                        |
|--------------|--------------------------|----------------------------------|
| `Atomic<T>`  | `rok::core::sync::atomic` | LLVM atomic ops (`rok.atomic.*`) |
| `Mutex<T>`   | `rok::core::sync::mutex`  | futex / wait queue (target 별)   |
| `RwLock<T>`  | `rok::core::sync::rwlock` | futex + reader counter           |
| `Cell<T>`    | `rok::core::sync::cell`   | 단순 wrapper, no lock            |
| `MutexGuard` | `rok::core::sync::mutex`  | RAII via Drop                    |
