# 2.18. task — M:N 스케줄러 task API

`rok::core::task` 모듈은 M:N 스케줄러 위에서 동작하는 협력적 task API 를 제공합니다. 이 모듈은 Rust Tokio 와 Go goroutine 으로부터 영감을 받았으며, work-stealing 스케줄러 위에서 task 가 **stackless coroutine** 으로 구현되어 있습니다.

`use rok::core::**;` 가 묵시적으로 적용되므로 `task::spawn`, `task::race`, `Task<T>` 등은 별도의 import 없이도 사용할 수 있습니다. 다만 모듈 prefix `task::` 는 그대로 보존됩니다. 한편 명시적으로 비동기 코드임을 드러내고 싶은 자리에서는 `use rok::core::task::*;` 를 권장하는데, 이는 시각적 분리를 통해 비동기 흐름을 명확히 하기 위함입니다.

## 2.18.1. 핵심 API

핵심 API 의 시그니처는 다음과 같습니다.

```rok
fn block_on<F: Future>(f: F) -> F::Output;              // sync 진입점 — pool 을 띄워 f 를 끝까지 실행
fn spawn<F: Future + Send>(f: F) -> Task<F::Output> where F::Output: Send;
fn spawn_local<F: Future>(f: F) -> Task<F::Output>;     // non-Send future — 현재 worker 에 고정
fn workers() -> usize;                                  // 실행 중 pool 의 worker 수 (block_on 밖에서는 1)
fn cpu_count() -> usize;                                // 온라인 CPU 수 — pool 크기
async fn race<T>(tasks: List<Task<T>>) -> T;            // 첫 완료 task 결과
async fn join<T>(tasks: List<Task<T>>) -> List<T>;      // 모두 완료 후
async fn timeout<T>(dur: Duration, task: Task<T>) throws (TimeoutError) -> T;   // try await 에서 던짐
async fn sleep(dur: Duration);
async fn sleep_until(when: Instant);
fn yield_now();
```

### 사용 예

실제 사용 예시를 살펴보면 다음과 같습니다.

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
task::yield_now();
```

`spawn` 은 클로저가 아니라 future 를 받으므로 `task::spawn(async { ... })` 처럼 `async` 블록이나 `async fn` 호출 결과를 넘깁니다. `Send` 가 아닌 future 는 `spawn_local` 로 현재 스레드에서 실행합니다. `join` / `race` / `timeout` 은 `await` 해서 결과를 받습니다.

## 2.18.2. `Task<T>` 타입

`Task<T>` 타입은 다음과 같이 정의되어 있습니다.

```rok
struct Task<T> {
    // 내부: GC heap 의 coroutine frame + 결과 슬롯
}

impl<T> Task<T> {
    fn cancel(self);                      // 더 이상 poll 하지 않음
    fn is_cancelled(self) -> bool;
    fn is_finished(self) -> bool;
    fn join(self) -> Join<T>;             // try await t.join() — 취소됐으면 CancellationError
}

struct CancellationError {}

impl<T> Future for Task<T> {              // `await t` — 완료까지 suspend
    type Output = T;
    ...
}
```

### `IntoFuture` 통합

`Task<T>` 는 `Future<Output = T>` 를 구현하므로 `await t` 로 기다릴 수 있고, [2.20 future](220-future.md) 의 `IntoFuture` blanket impl 에도 포함되어 `Thread` 와 동일한 방식으로 await 될 수 있습니다. `IntoFuture` 트레잇과 lang slot `future_trait` 의 권위 있는 정의는 [2.20 future](220-future.md) 에 있으며, 본 절은 그 정의를 그대로 따릅니다.

## 2.18.3. 스케줄러 모델 — work-stealing

`block_on` 은 CPU 수 (`cpu_count()`) 만큼의 worker 로 이루어진 work-stealing pool 을 띄우며, 호출한 스레드가 worker 0 이 됩니다. `spawn` 한 task 는 어느 worker 에서든 실행될 수 있으므로 `Future + Send` 이고 결과도 `Send` 여야 합니다. `spawn_local` 의 task 는 그것을 만든 worker 에 고정됩니다. `block_on` 이 끝날 때 아직 실행 중인 task 는 버려집니다.

스케줄러 골격은 다음과 같이 구성됩니다.

```
Worker 0 ─┐      Worker 1 ─┐    ...    Worker N-1 ─┐
 Local Q   │      Local Q   │            Local Q    │
 LIFO slot │      LIFO slot │            LIFO slot  │
 [256 slot]│      [256 slot]│            [256 slot] │
           └─ Global Injector Queue ──────┘
           └─ I/O Driver (포터블)   ──────┘
           └─ Timer Wheel           ──────┘
           └─ Parked Worker Stack   ──────┘
```

이 구조의 주요 특성을 정리하면, 우선 로컬 큐는 Tokio 방식과 같은 256-slot ring 으로 구성됩니다. 또한 LIFO slot 은 깨어난 task 를 즉시 실행하여 cache 친화성을 높이며, 도둑 알고리듬은 steal back (FIFO) 방식으로 동작합니다. I/O Driver 는 target 에 따라 epoll, io_uring, kqueue, IOCP, wepoll, poll_oneoff 중 하나를 사용하며, Timer 는 Hashed Hierarchical Timing Wheel 을 통해 O(1) 성능을 보장합니다.

표준 라이브러리 측의 권위는 API 시그니처에 한정되며, 알고리듬 본문은 spec 외부의 runtime 및 표준 라이브러리 body 영역에 속한다는 점에 유의해야 합니다.

## 2.18.4. 협력적 취소 — `Task::cancel`

협력적 취소는 다음과 같이 사용합니다.

```rok
let t = task::spawn(async { long_running() });
// ... 다른 일 ...
t.cancel();              // 협력적 신호 — task 가 cancellation point 에서 종료
```

`cancel()` 이 호출되면 그 task 는 더 이상 poll 되지 않고 현재 중단 지점에서 끝납니다. 취소된 task 를 `try await t.join()` 으로 기다리면 `CancellationError` 가 던져지고, `await t` 로 기다리면 panic 입니다.

```rok
let r = try await t.join() catch _: CancellationError { fallback() };
```

## 2.18.5. `thread::*` — 1:1 OS thread

별도 모듈인 `thread` 는 M:N 모델이 아니라 **OS thread** 를 직접 spawn 합니다.

```rok
use rok::core::thread::*;

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

시간 표현을 위한 `Duration` 과 `Instant` 는 다음과 같이 정의되어 있습니다.

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
