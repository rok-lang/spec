# 2.18. task — M:N 스케줄러 task API

`rok::core::task` 모듈은 M:N 스케줄러 위에서 동작하는 협력적 task API 를 제공합니다. 이 모듈은 Rust Tokio 와 Go goroutine 으로부터 영감을 받았으며, work-stealing 스케줄러 위에서 task 가 **stackless coroutine** 으로 구현됩니다.

`use rok::core::**;` 가 묵시적으로 적용되므로 `task::spawn`, `task::race`, `Task<T>` 등은 별도의 import 없이도 사용할 수 있습니다. 다만 모듈 prefix `task::` 는 그대로 보존됩니다. 한편 명시적으로 비동기 코드임을 드러내고 싶은 자리에서는 `use rok::core::task::*;` 를 권장하는데, 이는 시각적 분리를 통해 비동기 흐름을 명확히 하기 위함입니다.

## 2.18.1. 핵심 API

핵심 API 의 시그니처는 다음과 같습니다.

```rok
fn block_on<F: Future>(f: F) -> F::Output;              // sync 진입점 — executor 에 붙어 f 를 끝까지 실행
fn spawn<F: Future + Send>(f: F) -> Task<F::Output> where F::Output: Send;
fn spawn_local<F: Future>(f: F) -> Task<F::Output>;     // non-Send future — 현재 worker 에 고정
fn spawn_blocking<T: Send>(f: fn() -> T + Send) -> Blocking<T>;   // blocking 작업을 executor 밖 OS 스레드에서
fn workers() -> usize;                                  // executor 안에서는 pool 크기, 밖에서는 1
fn cpu_count() -> usize;                                // 온라인 CPU 수 — pool 크기
async fn race<T>(tasks: List<Task<T>>) -> T;            // 첫 완료 task 결과
fn select<F: Future>(futs: List<F>) -> SelectFuture<F>; // await — (index, 결과), spawn 하지 않음
fn id() -> TaskId;                                      // 현재 task 의 id — task 밖이면 panic
fn try_id() -> TaskId | none;
async fn join<T>(tasks: List<Task<T>>) -> List<T>;      // 모두 완료 후
async fn timeout<T>(dur: Duration, task: Task<T>) throws (TimeoutError) -> T;   // try await 에서 던짐
async fn sleep(dur: Duration);
async fn sleep_until(when: Instant);
async fn yield_now();                                   // await — 다른 ready task 에 한 번 양보
fn registrations() -> usize;                            // 진단용 — 실행 중 pool 의 park 한 waker + 대기 timer 수 (구현이 정함)
fn readable(fd: i32) -> IoReady;                       // await — fd 가 읽기 가능해질 때까지 (refs-lang 7.7.2)
fn writable(fd: i32) -> IoReady;
```

`sleep` 과 `timeout` 은 첫 poll 부터 시간을 잽니다.

### 사용 예

실제 사용 예시는 다음과 같습니다.

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

`spawn` 은 클로저가 아니라 future 를 받으므로 `task::spawn(async { ... })` 처럼 `async` 블록이나 `async fn` 호출 결과를 넘깁니다. `Send` 가 아닌 future 는 `spawn_local` 로 현재 스레드에서 실행합니다. `join` / `race` / `timeout` 은 `await` 해서 결과를 받습니다.

### `spawn_blocking` — blocking 작업

```rok
struct Blocking<T> { /* 결과 슬롯 */ }
impl<T> Future for Blocking<T> { type Output = T; ... }
```

`spawn_blocking(f)` 는 `f` 를 executor 의 worker 가 아닌 blocking pool 의 OS 스레드에서 실행하고, 그 결과로 끝나는 future 를 돌려줍니다. readiness 로 기다릴 수 없는 blocking syscall (일반 파일 I/O 등, [stdlib 3.1.4a](../300-std/301-fs.md#314a-비동기-파일-io-_async)) 을 task 안에서 기다릴 때 씁니다.

- `f` 는 첫 poll 이 아니라 **호출하는 즉시** 시작합니다 (pool 이 가득 차면 먼저 큐에 들어감).
- blocking pool 은 프로세스 전역 하나이며, 필요할 때 스레드를 늘리고 (상한 있음) 한동안 일이 없는 스레드는 끝냅니다. 작업은 들어온 순서 (FIFO) 로 처리합니다. 상한은 스레드 512 개이며, 상한에 닿으면 작업을 큐에 쌓습니다. 10 초 동안 일이 없는 스레드는 끝납니다. 작업 하나는 쉬고 있는 스레드 하나에 넘깁니다.
- `await` 하던 task 가 취소되어도 `f` 는 끝까지 실행되며, 결과만 버려집니다.

```rok
let n = await task::spawn_blocking(|| heavy_checksum(path));
```

### `spawn` 의 `Send` 검사

`task::spawn` 은 결과 타입도 `Send` 여야 하며 (`where F::Output: Send`), **frame** 이 `Send` 가 아닌 future 를 거부합니다 ([2.13.9](213-marker.md#2139-send--sync--스레드-경계-marker)). future 가 어디서 왔든 같습니다: `async` 블록이나 `async fn` 호출뿐 아니라 매개변수, struct / 튜플 / 합의 필드, helper 함수의 결과, 리스트의 원소, 클로저가 들고 있는 future, 다른 코루틴 frame 안의 future 도 검사합니다. future 의 frame 은 다음입니다.

- 캡처한 값.
- `await` 를 넘어 살아 있는 지역 변수와 임시값.
- frame 이 들고 있는 다른 future (그 frame 도 재귀로).

```rok
let c = Cell::new(0);                      // Cell: !Send
task::spawn(async { c.set(1); });          // ERROR — !Send 캡처
task::spawn_local(async { c.set(1); });    // OK — 현재 worker 에 고정

fn make() -> AsyncFuture<()> { let c = Cell::new(0); async { c.set(1); } }
task::spawn(make());                       // ERROR — helper 가 만든 future 의 frame 도 검사
```

같은 검사가 다른 스레드로 클로저를 보내는 자리에도 적용됩니다: `thread::spawn`, `thread::Builder::spawn`, `par` 의 `Scope::spawn`, `task::spawn_blocking`. 클로저가 `Send` 가 아닌 frame 의 future 를 들고 있으면 ``` `X` would move a future to another thread: … ``` 에러입니다.

이 검사는 보수적으로 근사해도 됩니다. 구현은 실제로는 `Send` 인 future 를 거부할 수 있지만, `Send` 가 아닌 값을 다른 스레드로 넘기는 future 를 받아서는 안 됩니다. 다음 값은 따라 들어가지 않습니다.

- 타입이 함수 값을 담을 수 없는 값 (future 의 frame 을 숨길 수 없음).
- `unsafe impl Send` 를 명시한 타입의 값 (예: `Task<T>`). 그 타입이 안전성을 책임집니다.

`async fn` 의 결과 타입 `AsyncFuture<T>` 는 `T: Send` 일 때, throwing 판 `TryAsyncFuture<T, E>` 는 `T` 와 `E` 가 모두 `Send` 일 때 `Send` 입니다. 캡처와 frame 은 위 검사가 따로 봅니다.

### `select` — 먼저 끝나는 future

```rok
fn select<F: Future>(futs: List<F>) -> SelectFuture<F>;   // Output = (usize, F::Output)

let (i, v) = await task::select([fetch_a(), fetch_b()]);
```

`race` 와 달리 future 들을 spawn 하지 않고, 기다리는 task **안에서** 실행합니다.

- 목록 순서대로 poll 하며, 같은 poll 에서 둘 이상이 끝나면 낮은 index 가 이깁니다.
- 하나가 끝나면, 이미 시작한 나머지는 모두 취소됩니다. 취소된 상태로 한 번 더 poll 되어 `finally` / `using` 이 실행됩니다 (2.18.4). 한 번도 poll 되지 않은 future 는 실행되지 않습니다.
- 진 future 들이 만든 깨우기 등록 (timer, parking) 은 모두 풀립니다.
- 기다리는 task 가 취소되면 자식 future 들도 취소됩니다.
- 빈 목록이면 panic 입니다.
- future 들은 한 타입이어야 합니다. 결과 타입이 다르면 감쌉니다: `async { Msg::A(await a) }`.

`select` 와 `sleep` 으로 timeout 을 만들 수도 있습니다.

```rok
enum Ev { Line(str), Timeout }

let (_, ev) = await task::select([
    async { Ev::Line(await read_line()) },
    async { await task::sleep(Duration::sec(5)); Ev::Timeout },
]);
```

### `TaskId`

`task::id()` 는 실행 중인 task 의 id 이며, task 밖에서 부르면 panic 합니다 (`try_id()` 는 `none`). `Task::id()` 는 그 task 의 id 입니다. id 는 프로세스 전역 카운터에서 나오며 다시 쓰이지 않습니다.

## 2.18.2. `Task<T>` 타입

`Task<T>` 타입은 다음과 같이 정의됩니다.

```rok
struct Task<T> {
    // 내부: GC heap 의 coroutine frame + 결과 슬롯
}

impl<T> Task<T> {
    fn cancel(self);                      // 취소 요청 — 다음 await 에서 끝남 (2.18.4)
    fn is_cancelled(self) -> bool;
    fn is_finished(self) -> bool;
    fn join(self) -> Join<T>;             // try await t.join() — 결과 없이 취소됐으면 CancellationError
    fn id(self) -> TaskId;
}

struct TaskId { /* u64 */ }
impl TaskId {
    fn as_u64(self) -> u64;
}
// TaskId: PartialEq / Eq / Hash / Display / Debug

struct CancellationError {}

impl<T> Future for Task<T> {              // `await t` — 완료까지 suspend
    type Output = T;
    ...
}
```

### `IntoFuture` 통합

`Task<T>` 는 `Future<Output = T>` 를 구현하므로 `await t` 로 기다릴 수 있고, [2.20 future](220-future.md) 의 `IntoFuture` blanket impl 에도 포함되어 `Thread` 와 동일한 방식으로 await 될 수 있습니다. `IntoFuture` 트레잇과 lang slot `future_trait` 의 권위 있는 정의는 [2.20 future](220-future.md) 에 있으며, 본 절은 그 정의를 그대로 따릅니다.

## 2.18.3. 스케줄러 모델 — work-stealing

프로세스에는 executor 가 하나 있습니다. 처음 쓸 때 시작하며, CPU 수 (`cpu_count()`) 만큼의 worker thread, blocking lane, I/O driver 로 이루어집니다. `block_on` 은 부른 스레드를 guest worker 로 이 executor 에 붙여 `f` 를 끝까지 실행합니다. `block_on` 이 돌아올 때 실행 중인 task 는 계속 실행됩니다. `par::*` 와 `spawn_blocking` 도 같은 executor 를 씁니다.

`spawn` 한 task 는 어느 worker 에서든 실행될 수 있으므로 `Future + Send` 이고 결과도 `Send` 여야 합니다. `spawn_local` 의 task 는 그것을 만든 worker 에 고정됩니다.

스케줄러 골격은 다음과 같이 구성됩니다.

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

이 구조의 주요 특성을 정리하면, 우선 로컬 큐는 lock-free Chase-Lev deque 이며, 소유 worker 와 도둑 모두 가장 오래된 task 부터 꺼냅니다 (스스로를 다시 깨우는 task 가 다른 task 를 굶기지 않도록, [refs-lang 7.6.2](../../refs-lang/007-runtime/706-scheduler.md#762-work-stealing-구조)). 바쁜 worker 는 task 32 개마다 기한이 지난 timer 를 처리합니다. I/O Driver 는 프로세스당 reactor thread 하나가 epoll / kqueue 를 edge-triggered 로 기다리며 (io_uring / IOCP 는 향후, [refs-lang 7.7](../../refs-lang/007-runtime/707-io-and-timer.md)), Timer 구조는 구현이 정합니다 (binary heap 또는 timing wheel, [refs-lang 7.7.4](../../refs-lang/007-runtime/707-io-and-timer.md#774-타이머-구조)).

표준 라이브러리 측의 권위는 API 시그니처에 한정되며, 알고리듬 본문은 spec 외부의 runtime 및 표준 라이브러리 body 영역에 속한다는 점에 유의해야 합니다.

## 2.18.4. 협력적 취소 — `Task::cancel`

협력적 취소는 다음과 같이 사용합니다.

```rok
let t = task::spawn(async { long_running() });
// ... 다른 일 ...
t.cancel();              // 협력적 신호 — task 가 cancellation point 에서 종료
```

`cancel()` 은 취소를 **요청** 합니다. 규칙은 다음과 같습니다 ([refs-lang 7.7.9](../../refs-lang/007-runtime/707-io-and-timer.md#779-cancellation)).

- 모든 `await` 가 cancellation point 입니다. 취소된 task 는 다음 `await` 에서 `return` 처럼 본문을 떠나며, 지나는 `finally` 와 `using` 정리가 안쪽부터 실행됩니다.
- 취소는 task 안에서 `catch` 할 수 있는 에러가 아닙니다. task 안으로 `CancellationError` 가 던져지지 않습니다.
- blocking syscall 은 cancellation point 가 아닙니다. 그 호출이 끝난 뒤 다음 `await` 에서 떠납니다.
- 소켓 I/O (`*_async`), `readable` / `writable`, `sleep` / `timeout`, `lock_async`, `recv_async` / `send_async` 에서 기다리던 task 는 취소되면 깨어나고 다음 `await` 에서 떠납니다.
- task 가 끝나면 그 task 의 깨우기 등록 (I/O readiness, timer, park 한 대기) 은 모두 풀립니다.
- `spawn_blocking` 과 그 위의 `fs::*_async` 를 기다리던 중 취소되면, 이미 시작된 작업은 백그라운드에서 끝까지 실행되고 결과만 버려집니다. 쓰기 같은 부수 효과는 일어납니다. 그 사이에 끝난 결과도 버립니다. 취소된 poll 은 `Pending` 을 돌려줍니다.
- 기다리던 future 가 값을 넘겨준 (메시지, 잠금 guard) poll 은 그 값을 지킵니다. 취소 검사는 "기다린 코루틴이 취소 때문에 떠났는가" 를 보며, 값은 남고 다음 `await` 에서 떠납니다 ([refs-lang 7.7.9](../../refs-lang/007-runtime/707-io-and-timer.md#779-cancellation)).
- 마지막 cancellation point 를 지난 뒤 취소된 task 는 정상적으로 끝나고 결과를 가집니다. `await t` / `join` 은 그 결과를 돌려줍니다.
- `CancellationError` 는 결과 없이 끝난 task 를 기다리는 쪽이 받습니다. 그런 task 를 `try await t.join()` 으로 기다리면 `CancellationError` 가 던져지고, `await t` 로 기다리면 panic 입니다.

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

## 2.18.5. `thread::*` — 1:1 OS thread

별도 모듈인 `thread` 는 M:N 모델이 아니라 **OS thread** 를 직접 spawn 합니다.

```rok
use rok::std::thread;

let t = thread::spawn { os_level_work() };
let r = t.join();              // OS-level 동기 대기
```

두 모델 사이의 차이를 비교하면 다음 표와 같습니다.

| 측면        | `task::spawn`              | `thread::spawn`         |
|-------------|----------------------------|-------------------------|
| 모델        | M:N (사용자 공간 스케줄러) | 1:1 (OS thread)         |
| 비용        | 가벼움 (~1KB frame)        | 무거움 (~MB stack)      |
| 협력적 취소 | ✓                          | ✗ (OS-level kill 필요)  |
| async 통합  | ✓                          | △ (joiner 만 await)     |
| 사용처      | 일반 동시성 (default)      | CPU-bound, blocking I/O |

기본적으로는 `task::spawn` 을 사용하면 되고, `thread::spawn` 은 명시적으로 OS thread 가 요구되는 상황에서만 선택하면 됩니다.

## 2.18.6. `Duration` / `Instant`

시간 표현을 위한 `Duration` 과 `Instant` 는 다음과 같이 정의됩니다.

```rok
struct Duration { /* ns 단위 i64 */ }
struct Instant { /* monotonic clock */ }

impl Duration {
    fn ns(n: i64) -> Duration;
    fn us(n: i64) -> Duration;
    fn ms(n: i64) -> Duration;
    fn sec(n: i64) -> Duration;
    fn min(n: i64) -> Duration;
    fn hours(n: i64) -> Duration;
}

impl Instant {
    fn now() -> Instant;
    fn elapsed(self) -> Duration;
}
```

이를 활용하여 `await task::sleep(Duration::ms(100))` 또는 `await task::sleep_until(t)` 와 같이 호출하면 정확한 monotonic time 을 보장받을 수 있습니다. 두 타입은 `rok::core::task::time` 에 있으며 (2.18.7), `rok::std::time` 이 이를 다시 내보냅니다.

## 2.18.7. 모듈 매핑

각 항목이 자리 잡는 모듈은 다음 표와 같습니다.

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
