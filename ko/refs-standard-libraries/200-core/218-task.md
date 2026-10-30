# 2.18. task - M:N 스케줄러 task API

`rok::core::task` 모듈은 M:N 스케줄러 위에서 동작하는 협력적 task API입니다. Rust Tokio와 Go goroutine에서 영감을 받았으며, task는 work-stealing 스케줄러 위의 **stackless coroutine**으로 구현됩니다.

`use rok::core::**;`가 묵시적으로 적용되므로 `task::spawn`, `task::race`, `Task<T>` 등은 import 없이 쓸 수 있습니다. 모듈 prefix `task::`는 그대로 남습니다. 비동기 코드임을 명시적으로 드러내 흐름을 눈에 띄게 구분하고 싶은 자리에서는 `use rok::core::task::*;`를 권장합니다.

## 2.18.1. 핵심 API

핵심 API의 시그니처입니다.

```rok
fn block_on<F: Future>(f: F) -> F::Output;              // sync 진입점: executor 에 붙어 f 를 끝까지 실행
fn block_on_try<T, E>(f: TryAsyncFuture<T, E>) throws (E) -> T;   // throw 하는 async body 용 block_on. 그 에러를 여기서 throw
fn spawn<F: Future + Send>(f: F) -> Task<F::Output> where F::Output: Send;
fn spawn_local<F: Future>(f: F) -> Task<F::Output>;     // non-Send future, 현재 worker 에 고정
fn spawn_blocking<T: Send>(f: fn() -> T + Send) -> Blocking<T>;   // blocking 작업을 executor 밖 OS 스레드에서
fn workers() -> usize;                                  // executor 안에서는 pool 크기, 밖에서는 1
fn cpu_count() -> usize;                                // 온라인 CPU 수 (pool 크기)
async fn race<T>(tasks: List<Task<T>>) -> T;            // 첫 완료 task 결과
fn select<F: Future>(futs: List<F>) -> SelectFuture<F>; // await: (index, 결과), spawn 하지 않음
fn id() -> TaskId;                                      // 현재 task 의 id, task 밖이면 panic
fn try_id() -> TaskId | none;
async fn join<T>(tasks: List<Task<T>>) -> List<T>;      // 모두 완료 후
async fn timeout<T>(dur: Duration, task: Task<T>) throws (TimeoutError) -> T;   // try await 에서 던짐
async fn sleep(dur: Duration);
async fn sleep_until(when: Instant);
async fn yield_now();                                   // await: 다른 ready task 에 한 번 양보
fn registrations() -> usize;                            // 진단용: 실행 중 pool 의 park 한 waker + 대기 timer 수 (구현이 정함)
fn readable(fd: i32) -> IoReady;                       // await: fd 가 읽기 가능해질 때까지 (refs-lang 7.7.2)
fn writable(fd: i32) -> IoReady;
fn wake_io(fd: i32);                                   // fd 의 readable / writable 대기자를 깨움 (close / shutdown 뒤에 씀; 저수준)
fn park(key: usize, waker: Waker);                     // 저수준: 다음 unpark(key) 때 waker 를 깨움
fn unpark(key: usize);                                 // key 에 park 된 task 를 모두 깨움
```

`sleep`과 `timeout`은 첫 poll부터 시간을 잽니다. `park` / `unpark`는 [2.16](216-sync.md)의 lock들이 쓰는 대기 큐입니다. 직접 작성한 `Future`는 자신이 고른 key에 `ctx.waker()` ([2.20](220-future.md))를 park하고 `Pending`을 돌려주며, 상태를 바꾸는 쪽이 같은 key로 `unpark`를 부릅니다. park된 항목은 그 task가 끝날 때에도 지워집니다.

이 함수들이 돌려주는 future와 에러는 다음과 같습니다.

```rok
struct SelectFuture<F> { /* ... */ }
impl<F: Future> Future for SelectFuture<F> { type Output = (usize, F::Output); ... }

struct IoReady { /* ... */ }
impl Future for IoReady { type Output = (); ... }   // 이유 없이 완료될 수 있음. I/O 를 루프에서 재시도

struct Join<T> { /* ... */ }                        // Task::join, try await 로 기다림
impl<T> Join<T> {
    fn poll(ref self, ref ctx: Context) throws (CancellationError) -> Poll<T>;
}

struct TimeoutError {}                              // task::timeout 시간 초과; Display "timed out", Debug, Error
```

### 사용 예

사용 예입니다.

```rok
use rok::core::task::*;

let t = task::spawn(async { compute_heavy() });
let result = await t;

let results = await task::join([
    task::spawn(async { fetch("a") }),
    task::spawn(async { fetch("b") }),
    task::spawn(async { fetch("c") }),
]);

let first = await task::race([
    task::spawn(async { source_a() }),
    task::spawn(async { source_b() }),
]);

await task::sleep(Duration::ms(100));
await task::yield_now();
```

`spawn`은 클로저가 아니라 future를 받으므로 `task::spawn(async { ... })`처럼 `async` 블록이나 `async fn` 호출 결과를 넘깁니다. `Send`가 아닌 future는 `spawn_local`로 현재 스레드에서 실행합니다. `join` / `race` / `timeout`은 `await`해서 결과를 받습니다.

### `spawn_blocking` - blocking 작업

```rok
struct Blocking<T> { /* 결과 슬롯 */ }
impl<T> Future for Blocking<T> { type Output = T; ... }
```

`spawn_blocking(f)`는 `f`를 executor의 worker가 아닌 blocking pool의 OS 스레드에서 실행하고, 그 결과로 끝나는 future를 돌려줍니다. readiness로 기다릴 수 없는 blocking syscall (일반 파일 I/O 등, [stdlib 3.1.4a](../300-std/301-fs.md#314a-비동기-파일-io-_async))을 task 안에서 기다릴 때 씁니다.

- `f`는 첫 poll이 아니라 **호출하는 즉시** 시작합니다 (pool이 가득 차면 먼저 큐에 들어감).
- blocking pool은 프로세스 전체에 하나이며, 필요할 때 스레드를 늘리고(상한 있음) 한동안 일이 없는 스레드는 끝냅니다. 작업은 들어온 순서(FIFO)로 처리합니다. 상한은 스레드 512개이며, 상한에 닿으면 작업을 큐에 쌓습니다. 10초 동안 일이 없는 스레드는 끝납니다. 작업 하나는 쉬고 있는 스레드 하나에 넘깁니다.
- `await`하던 task가 취소되어도 `f`는 끝까지 실행되며, 결과만 버려집니다.

```rok
let n = await task::spawn_blocking(|| heavy_checksum(path));
```

### `spawn` 의 `Send` 검사

`task::spawn`은 결과 타입도 `Send`여야 하며 (`where F::Output: Send`), **frame**이 `Send`가 아닌 future를 거부합니다 ([2.13.9](213-marker.md#2139-send--sync---스레드-경계-marker)). future가 어디서 왔든 같습니다. `async` 블록이나 `async fn` 호출뿐 아니라 매개변수, struct / 튜플 / 합의 필드, helper 함수의 결과, 리스트의 원소, 클로저가 들고 있는 future, 다른 코루틴 frame 안의 future도 검사합니다. future의 frame은 다음으로 이루어집니다.

- 캡처한 값.
- `await`를 넘어 살아 있는 지역 변수와 임시값.
- frame이 들고 있는 다른 future (그 frame도 재귀로).

```rok
let c = Cell::new(0);                      // Cell: !Send
task::spawn(async { c.set(1); });          // ERROR: !Send 캡처
task::spawn_local(async { c.set(1); });    // OK: 현재 worker 에 고정

fn make() -> AsyncFuture<()> { let c = Cell::new(0); async { c.set(1); } }
task::spawn(make());                       // ERROR: helper 가 만든 future 의 frame 도 검사
```

같은 검사가 클로저를 다른 스레드로 보내는 자리에도 적용됩니다. `thread::spawn`, `thread::Builder::spawn`, `par`의 `Scope::spawn`, `task::spawn_blocking`. 클로저가 `Send`가 아닌 frame의 future를 들고 있으면 ``` `X` would move a future to another thread: ... ``` 에러입니다.

이 검사는 보수적으로 근사해도 됩니다. 구현은 실제로는 `Send`인 future를 거부할 수 있지만, `Send`가 아닌 값을 다른 스레드로 넘기는 future를 받아들여서는 안 됩니다. 검사는 다음 값 안으로는 들어가지 않습니다.

- 타입이 함수 값을 담을 수 없는 값 (future의 frame을 숨길 수 없음).
- `unsafe impl Send`를 명시한 타입의 값 (예: `Task<T>`). 그 타입이 안전성을 책임집니다.

`async fn`의 결과 타입 `AsyncFuture<T>`는 `T: Send`일 때, throwing 판 `TryAsyncFuture<T, E>`는 `T`와 `E`가 모두 `Send`일 때 `Send`입니다. 캡처와 frame은 위 검사가 따로 봅니다. 캡처가 `Send`이지만 `Sync`가 아닌 frame(`async fn`이 받는 리스트, `async` 블록이 캡처하는 리스트나 future)은 받아들입니다. `task::spawn`은 채널 send처럼 future를 넘깁니다 ([2.13.9](213-marker.md#2139-send--sync---스레드-경계-marker)). task는 future가 캡처한 것의 복사본을 담은 자기 frame을 받는데, 이 복사본은 넘김 한 번으로 만들므로 캡처들이 공유하던 것(같은 리스트 두 번, 리스트와 그 위의 future)은 계속 공유됩니다. spawn 뒤에 그 future나 그것이 캡처한 것을 쓰면 send 뒤와 같이 컴파일 에러 (`E202`)입니다. 한 번도 poll되지 않은 future만 넘길 수 있습니다. 시작한 것(`select`, 사용자의 `Future`, 또는 직접 한 번 poll한 것)이나 끝난 것, 또는 frame이나 캡처한 future의 frame에 `[ref x]` 캡처가 있는 것을 넘기면 panic하며, 아무것도 공유하지 않습니다. 채널로 보내거나 `Mutex`나 `Frozen`에 넣는 코루틴 future도 같은 방식으로 넘겨지며, 그때마다 자기 frame을 받습니다. `Task`가 넘겨진 task(`Frozen::get`, 채널, 잠금, `par` 항목)를 `await`하면 매번 결과의 복사본을 따로 받으며, 한 스레드 안의 `Task` 복사본들은 그것을 공유합니다.

### `select` - 먼저 끝나는 future

```rok
fn select<F: Future>(futs: List<F>) -> SelectFuture<F>;   // Output = (usize, F::Output)

let (i, v) = await task::select([fetch_a(), fetch_b()]);
```

`race`와 달리 future들을 spawn하지 않고, 기다리는 task **안에서** 실행합니다.

- 목록 순서대로 poll하며, 같은 poll에서 둘 이상이 끝나면 낮은 index가 이깁니다.
- 하나가 끝나면, 이미 시작한 나머지는 모두 취소됩니다. 취소된 상태로 한 번 더 poll되어 `finally` / `using`이 실행됩니다 (2.18.4). 한 번도 poll되지 않은 future는 실행되지 않습니다.
- 진 future들이 만든 깨우기 등록 (timer, parking)은 모두 풀립니다.
- 기다리는 task가 취소되면 자식 future들도 취소됩니다.
- 빈 목록이면 panic입니다.
- future들은 한 타입이어야 합니다. 결과 타입이 다르면 `async { Msg::A(await a) }`처럼 감쌉니다.

`select`와 `sleep`으로 timeout을 만들 수도 있습니다.

```rok
enum Ev { Line(str), Timeout }

let (_, ev) = await task::select([
    async { Ev::Line(await read_line()) },
    async { await task::sleep(Duration::sec(5)); Ev::Timeout },
]);
```

### `TaskId`

`task::id()`는 실행 중인 task의 id이며, task 밖에서 부르면 panic합니다 (`try_id()`는 `none`). `Task::id()`는 그 task의 id입니다. id는 프로세스 전역 카운터에서 나오며 다시 쓰이지 않습니다.

## 2.18.2. `Task<T>` 타입

`Task<T>` 타입의 정의입니다.

```rok
struct Task<T> {
    // 내부: GC heap 의 coroutine frame + 결과 슬롯
}

impl<T> Task<T> {
    fn cancel(self);                      // 취소 요청, 다음 await 에서 끝남 (2.18.4)
    fn is_cancelled(self) -> bool;
    fn is_finished(self) -> bool;
    fn join(self) -> Join<T>;             // try await t.join(): 결과 없이 취소됐으면 CancellationError
    fn id(self) -> TaskId;
}

struct TaskId { /* u64 */ }
impl TaskId {
    fn as_u64(self) -> u64;
}
// TaskId: PartialEq / Eq / Hash / Display / Debug

struct CancellationError {}

impl<T> Future for Task<T> {              // `await t`: 완료까지 suspend
    type Output = T;
    ...
}
```

### `IntoFuture` 통합

`Task<T>`는 `Future<Output = T>`를 구현하므로 `await t`로 기다릴 수 있고, [2.20 future](220-future.md)의 `IntoFuture` blanket impl에도 포함되므로 `Thread`와 같은 방식으로 await할 수 있습니다. `IntoFuture` 트레잇과 lang slot `future_trait`의 권위 있는 정의는 [2.20 future](220-future.md)에 있으며, 이 절은 그 정의를 따릅니다.

## 2.18.3. 스케줄러 모델 - work-stealing

프로세스에는 executor가 하나 있습니다. 처음 쓸 때 시작하며, CPU 수(`cpu_count()`)만큼의 worker thread, blocking lane, I/O driver로 이루어집니다. `block_on`은 부른 스레드를 guest worker로 이 executor에 붙여 `f`를 끝까지 실행합니다. `block_on`이 돌아올 때 실행 중인 task는 계속 실행됩니다. 다만 `block_on`은 그 스레드에 고정된 `spawn_local` task들도 끝난 뒤에야 돌아옵니다. 그 task들은 다른 스레드로 옮겨 갈 수 없기 때문입니다. `par::*`와 `spawn_blocking`도 같은 executor를 씁니다.

`spawn`한 task는 어느 worker에서든 실행될 수 있으므로 `Future + Send`이고 결과도 `Send`여야 합니다. `spawn_local`의 task는 그것을 만든 worker에 고정됩니다. 다만 `block_on` 스레드가 poll하지만 소유하지 않은 task(기다리는 동안 맡은 `spawn` task)에서 부른 `spawn_local`은 대신 pool worker에 고정됩니다. 그래서 `block_on`은 자기 task들의 local task만 기다립니다. 이런 poll 한 번에서 만든 local task들은 모두 같은 pool worker로 가고, 그 poll이 끝난 뒤에야 시작합니다. 원래 그 task들을 실행했을 worker에서와 같습니다.

스케줄러 골격은 다음과 같습니다.

```
Worker 0 ─┐      Worker 1 ─┐    ...    Worker N-1 ─┐
 Local     │      Local     │            Local      │
 deque     │      deque     │            deque      │
 (Chase-Lev)      (Chase-Lev)            (Chase-Lev)│
           └─ Global Injector Queue ──────┘
           └─ I/O Driver (포터블)   ──────┘
           └─ Timer (heap)  ───────────────┘
           └─ Parked Worker Stack   ──────┘
```

로컬 큐는 lock-free Chase-Lev deque이며, 소유 worker와 훔치는 worker 모두 가장 오래된 task부터 꺼냅니다(스스로를 계속 다시 깨우는 task가 다른 task를 굶기지 않도록, [refs-lang 7.6.2](../../refs-lang/007-runtime/706-scheduler.md#762-work-stealing-구조)). 바쁜 worker는 task 32개마다 기한이 지난 timer를 처리합니다. I/O driver는 프로세스당 reactor thread 하나가 epoll / kqueue를 edge-triggered로 기다리며(io_uring / IOCP는 향후 과제, [refs-lang 7.7](../../refs-lang/007-runtime/707-io-and-timer.md)), timer 구조는 구현이 정합니다(binary heap 또는 timing wheel, [refs-lang 7.7.4](../../refs-lang/007-runtime/707-io-and-timer.md#774-타이머-구조)).

표준 라이브러리의 권위는 API 시그니처까지입니다. 알고리듬 본문은 spec 밖의 runtime과 표준 라이브러리 body에 속합니다.

`rok::core::task`는 runtime hook도 내보냅니다. `TaskRef`, `Pool`, `Worker`, `wake_task`, `cancel_requested`, `unwinding`, 그리고 `host_enter` / `host_leave` / `host_run` (WebAssembly component의 event loop 진입점)입니다. 이들은 컴파일러와 runtime을 위해 공개되어 있을 뿐 runtime 내부용이며 안정된 API가 아닙니다.

## 2.18.4. 협력적 취소 - `Task::cancel`

협력적 취소는 다음과 같이 씁니다.

```rok
let t = task::spawn(async { long_running() });
// ... 다른 일 ...
t.cancel();              // 협력적 신호. task 가 cancellation point 에서 종료
```

`cancel()`은 취소를 **요청**합니다. 규칙은 다음과 같습니다 ([refs-lang 7.7.9](../../refs-lang/007-runtime/707-io-and-timer.md#779-cancellation)).

- 모든 `await`가 cancellation point입니다. 취소된 task는 다음 `await`에서 `return`처럼 본문을 떠나며, 지나는 `finally`와 `using` 정리가 안쪽부터 실행됩니다.
- 취소는 task 안에서 `catch`할 수 있는 에러가 아닙니다. task 안으로 `CancellationError`를 던지지 않습니다.
- blocking syscall은 cancellation point가 아닙니다. 그 호출이 끝난 뒤 다음 `await`에서 떠납니다.
- 소켓 I/O (`*_async`), `readable` / `writable`, `sleep` / `timeout`, `lock_async`, `recv_async` / `send_async`에서 기다리던 task는 취소되면 깨어나고 다음 `await`에서 떠납니다.
- task가 끝나면 그 task의 깨우기 등록 (I/O readiness, timer, park한 대기)은 모두 풀립니다.
- `spawn_blocking`과 그 위의 `fs::*_async`를 기다리던 중 취소되면, 이미 시작된 작업은 백그라운드에서 끝까지 실행되고 결과만 버려집니다. 쓰기 같은 부수 효과는 일어납니다. 그 사이에 끝난 결과도 버립니다. 취소된 poll은 `Pending`을 돌려줍니다.
- 기다리던 future가 값을 넘겨준 (메시지, 잠금 guard) poll은 그 값을 지킵니다. 취소 검사는 "기다린 코루틴이 취소 때문에 떠났는가"를 보며, 값은 남고 다음 `await`에서 떠납니다 ([refs-lang 7.7.9](../../refs-lang/007-runtime/707-io-and-timer.md#779-cancellation)).
- 마지막 cancellation point를 지난 뒤 취소된 task는 정상적으로 끝나고 결과를 가집니다. `await t` / `join`은 그 결과를 돌려줍니다.
- `CancellationError`는 결과 없이 끝난 task를 기다리는 쪽이 받습니다. 그런 task를 `try await t.join()`으로 기다리면 `CancellationError`가 던져지고, `await t`로 기다리면 panic입니다.

```rok
let t = task::spawn(async {
    using log = open_log();                      // 취소되면 여기서 drop
    loop {
        await task::sleep(Duration::sec(1));     // cancellation point
        tick(log);
    }
});
t.cancel();
```

```rok
let r = try await t.join() catch _: CancellationError { fallback() };
```

## 2.18.5. `thread::*` - 1:1 OS thread

별도 모듈인 `thread`는 M:N 모델이 아니라 **OS thread**를 직접 spawn합니다.

```rok
use rok::std::thread;

let t = thread::spawn { os_level_work() };
let r = t.join();              // OS-level 동기 대기
```

두 모델의 차이는 다음과 같습니다.

| 측면        | `task::spawn`              | `thread::spawn`         |
|-------------|----------------------------|-------------------------|
| 모델        | M:N (사용자 공간 스케줄러) | 1:1 (OS thread)         |
| 비용        | 가벼움 (~1KB frame)        | 무거움 (~MB stack)      |
| 협력적 취소 | ✓                          | ✗ (OS-level kill 필요)  |
| async 통합  | ✓                          | △ (joiner만 await)     |
| 사용처      | 일반 동시성 (default)      | CPU-bound, blocking I/O |

기본으로는 `task::spawn`을 쓰고, OS thread가 꼭 필요한 경우에만 `thread::spawn`을 고르세요.

## 2.18.6. `Duration` / `Instant`

시간을 나타내는 `Duration`과 `Instant`의 정의입니다.

```rok
struct Duration { /* i64 나노초, 비공개 */ }
struct Instant { /* 단조 시계, 비공개 */ }

impl Duration {
    const fn ns(n: i64) -> Duration;
    const fn us(n: i64) -> Duration;
    const fn ms(n: i64) -> Duration;
    const fn sec(n: i64) -> Duration;
    const fn min(n: i64) -> Duration;
    const fn hours(n: i64) -> Duration;
    const fn days(n: i64) -> Duration;
    const fn from_secs_f64(s: f64) -> Duration;
    const ZERO: Duration;
    const MAX: Duration;

    const fn as_ns(self) -> i64;
    const fn as_us(self) -> i64;
    const fn as_ms(self) -> i64;
    const fn as_secs(self) -> i64;
    const fn as_secs_f64(self) -> f64;
    const fn subsec_ns(self) -> u32;
    const fn subsec_us(self) -> u32;
    const fn subsec_ms(self) -> u32;

    const fn checked_add(self, rhs: Duration) -> Duration | none;
    const fn checked_sub(self, rhs: Duration) -> Duration | none;
    const fn checked_mul(self, factor: i64) -> Duration | none;
    const fn checked_div(self, divisor: i64) -> Duration | none;
    const fn saturating_add(self, rhs: Duration) -> Duration;
    const fn saturating_sub(self, rhs: Duration) -> Duration;
    const fn saturating_mul(self, factor: i64) -> Duration;
}
// Add, Sub (Duration); Mul<i64>, Div<i64>; Mul<Duration> for f64; Display, Debug

impl Instant {
    fn now() -> Instant;
    fn elapsed(self) -> Duration;
    const fn duration_since(self, earlier: Instant) -> Duration;
    const fn checked_duration_since(self, earlier: Instant) -> Duration | none;
    const fn saturating_duration_since(self, earlier: Instant) -> Duration;
    const fn checked_add(self, d: Duration) -> Instant | none;
    const fn checked_sub(self, d: Duration) -> Instant | none;
}
// Add<Duration>, Sub<Duration>, Sub<Instant> -> Duration
```

overflow가 나면 `ns`..`days`와 `+` `-` `*` 연산자는 debug 빌드에서 panic, 그 외에는 포화(saturate)합니다. `checked_*` / `saturating_*` 형태로 동작을 명시적으로 고를 수 있습니다. `Duration`은 부호가 있으므로 `earlier`가 더 나중이면 `duration_since`는 음수 duration을 돌려주고, `checked_duration_since`는 `none`, `saturating_duration_since`는 `Duration::ZERO`를 돌려줍니다.

`await task::sleep(Duration::ms(100))`, `await task::sleep_until(t)` 같은 호출은 정확한 monotonic time을 보장합니다. 두 타입은 `rok::core::task::time`에 있으며 (2.18.7), `rok::std::time`이 다시 내보냅니다.

## 2.18.7. 모듈 매핑

각 항목이 있는 모듈입니다.

| 항목                 | 모듈                                 | 동작                     |
|----------------------|--------------------------------------|--------------------------|
| `task::spawn`        | `rok::core::task` + scheduler runtime | scheduler push           |
| `task::race`         | `rok::core::task`                     | first-completed selector |
| `task::join`         | `rok::core::task`                     | join all                 |
| `task::timeout`      | `rok::core::task`                     | race + timer             |
| `Task<T>`            | `rok::core::task`                     | coroutine frame ref      |
| `Duration`/`Instant` | `rok::core::task::time`               | i64 ns + monotonic clock |
| `Future` (lang slot) | `rok::core::future`                   | poll-based               |
| `IntoFuture`         | `rok::core::future`                   | await 진입점             |
