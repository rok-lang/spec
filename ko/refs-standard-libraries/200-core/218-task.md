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
async fn yield_now();                                   // await — 다른 ready task 에 한 번 양보
fn readable(fd: i32) -> IoReady;                       // await — fd 가 읽기 가능해질 때까지 (refs-lang 7.7.2)
fn writable(fd: i32) -> IoReady;
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
await task::yield_now();
```

`spawn` 은 클로저가 아니라 future 를 받으므로 `task::spawn(async { ... })` 처럼 `async` 블록이나 `async fn` 호출 결과를 넘깁니다. `Send` 가 아닌 future 는 `spawn_local` 로 현재 스레드에서 실행합니다. `join` / `race` / `timeout` 은 `await` 해서 결과를 받습니다.

### `spawn` 의 `Send` 검사

`task::spawn` 은 다음 값 가운데 `Send` 가 아닌 것이 있는 future 를 거부합니다 ([2.13.9](213-marker.md#2139-send--sync--스레드-경계-marker)).

- `async` 블록 / 클로저가 캡처한 값.
- 코루틴 안에서 `await` 를 넘어 살아 있는 지역 (frame 에 남는 값).

```rok
let c = Cell::new(0);                      // Cell: !Send
task::spawn(async { c.set(1); });          // ERROR — !Send 캡처
task::spawn_local(async { c.set(1); });    // OK — 현재 worker 에 고정
```

이 검사는 근사해도 됩니다. 구현은 `Send` 인 future 를 거부할 수 있지만 (보수적), `Send` 가 아닌 값을 넘기는 future 를 받아서는 안 됩니다. 예를 들어 `spawn` 인자가 `async` 블록, `async fn` 호출, 또는 그것으로 초기화된 지역 변수일 때만 frame 을 들여다보고, 그 밖의 future 는 타입의 `Send` 구현만 보는 구현이 허용됩니다.

`async fn` 의 결과 타입 `AsyncFuture<T>` 는 `T: Send` 일 때, throwing 판 `TryAsyncFuture<T, E>` 는 `T` 와 `E` 가 모두 `Send` 일 때 `Send` 입니다. 캡처와 frame 은 위 검사가 따로 봅니다.

## 2.18.2. `Task<T>` 타입

`Task<T>` 타입은 다음과 같이 정의되어 있습니다.

```rok
struct Task<T> {
    // 내부: GC heap 의 coroutine frame + 결과 슬롯
}

impl<T> Task<T> {
    fn cancel(self);                      // 취소 요청 — 다음 await 에서 끝남 (2.18.4)
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
 Local     │      Local     │            Local      │
 deque     │      deque     │            deque      │
 (Chase-Lev)      (Chase-Lev)            (Chase-Lev)│
           └─ Global Injector Queue ──────┘
           └─ I/O Driver (포터블)   ──────┘
           └─ Timer (heap / wheel)  ──────┘
           └─ Parked Worker Stack   ──────┘
```

이 구조의 주요 특성을 정리하면, 우선 로컬 큐는 lock-free Chase-Lev deque 이며, 소유 worker 와 도둑 모두 가장 오래된 task 부터 꺼냅니다 (스스로를 다시 깨우는 task 가 다른 task 를 굶기지 않도록, [refs-lang 7.6.2](../../refs-lang/007-runtime/706-scheduler.md#762-work-stealing-구조)). 바쁜 worker 는 task 32 개마다 기한이 지난 timer 를 처리합니다. I/O Driver 는 프로세스당 reactor thread 하나가 epoll / kqueue 를 edge-triggered 로 기다리며 (io_uring / IOCP 는 향후, [refs-lang 7.7](../../refs-lang/007-runtime/707-io-and-timer.md)), Timer 구조는 구현이 정합니다 (binary heap 또는 timing wheel, [refs-lang 7.7.5](../../refs-lang/007-runtime/707-io-and-timer.md#775-타이머-구조)).

표준 라이브러리 측의 권위는 API 시그니처에 한정되며, 알고리듬 본문은 spec 외부의 runtime 및 표준 라이브러리 body 영역에 속한다는 점에 유의해야 합니다.

## 2.18.4. 협력적 취소 — `Task::cancel`

협력적 취소는 다음과 같이 사용합니다.

```rok
let t = task::spawn(async { long_running() });
// ... 다른 일 ...
t.cancel();              // 협력적 신호 — task 가 cancellation point 에서 종료
```

`cancel()` 은 취소를 **요청** 합니다. 규칙은 다음과 같습니다 ([refs-lang 7.7.10](../../refs-lang/007-runtime/707-io-and-timer.md#7710-cancellation)).

- 모든 `await` 가 cancellation point 입니다. 취소된 task 는 다음 `await` 에서 `return` 처럼 본문을 떠나며, 지나는 `finally` 와 `using` 정리가 안쪽부터 실행됩니다.
- 취소는 task 안에서 `catch` 할 수 있는 에러가 아닙니다. task 안으로 `CancellationError` 가 던져지지 않습니다.
- blocking syscall 은 cancellation point 가 아닙니다. 그 호출이 끝난 뒤 다음 `await` 에서 떠납니다.
- `CancellationError` 는 기다리는 쪽이 받습니다. 취소된 task 를 `try await t.join()` 으로 기다리면 `CancellationError` 가 던져지고, `await t` 로 기다리면 panic 입니다.

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
