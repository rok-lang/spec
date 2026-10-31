# 2.18. task: M:N 스케줄러 task API

`rok::core::task` 모듈은 M:N 스케줄러 위에서 동작하는 협력적 task API입니다. Rust Tokio와 Go 고루틴에서 영감을 받았습니다. task는 work-stealing 스케줄러 위의 **스택 없는 코루틴**(stackless coroutine)으로 구현됩니다.

`use rok::core::**;`가 묵시적으로 적용되므로 `task::spawn`, `task::race`, `Task<T>` 등은 가져오지 않고 쓸 수 있습니다. 모듈 접두 `task::`는 그대로 남습니다. 비동기 코드임을 드러내 흐름을 눈에 띄게 나누고 싶은 자리에서는 `use rok::core::task::*;`를 권장합니다.

## 2.18.1. 핵심 API

핵심 API의 시그니처입니다.

```rok
fn block_on<F: Future>(f: F) -> F::Output;              // sync 진입점: executor 에 붙어 f 를 끝까지 실행
fn block_on_try<T, E>(f: TryAsyncFuture<T, E>) throws (E) -> T;   // throw 하는 async body 용 block_on. 그 에러를 여기서 throw
fn spawn<F: Future + Send>(f: F) -> Task<F::Output> where F::Output: Send;
fn spawn_local<F: Future>(f: F) -> Task<F::Output>;     // non-Send future, 현재 worker 에 고정 (OS 스레드가 아님)
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

`sleep`과 `timeout`은 첫 poll부터 시간을 잽니다. `park` / `unpark`는 [2.16](216-sync.md)의 락이 쓰는 대기 큐입니다. 직접 작성한 `Future`는 자신이 고른 키에 `ctx.waker()`([2.20](220-future.md))를 걸어 두고(park) `Pending`을 돌려줍니다. 상태를 바꾸는 쪽이 같은 키로 `unpark`를 부릅니다. 걸어 둔 항목은 그 task가 끝날 때에도 지워집니다.

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

`spawn`은 클로저가 아니라 future를 받습니다. 그래서 `task::spawn(async { ... })`처럼 `async` 블록이나 `async fn` 호출 결과를 넘깁니다. `Send`가 아닌 future는 `spawn_local`로 현재 워커에서 실행합니다. `join` / `race` / `timeout`은 `await`로 기다려 결과를 받습니다.

### `spawn_blocking`: 블로킹 작업

```rok
struct Blocking<T> { /* 결과 슬롯 */ }
impl<T> Future for Blocking<T> { type Output = T; ... }
```

`spawn_blocking(f)`는 `f`를 실행기(executor)의 워커가 아니라 블로킹 풀의 OS 스레드에서 실행합니다. 그리고 그 결과로 끝나는 future를 돌려줍니다. readiness로 기다릴 수 없는 블로킹 시스템 호출(일반 파일 I/O 등, [stdlib 3.1.4a](../300-std/301-fs.md#314a-비동기-파일-io_async))을 task 안에서 기다릴 때 씁니다.

- `f`는 첫 poll이 아니라 **호출하는 즉시** 시작합니다. 풀이 가득 차 있으면 먼저 큐에 들어갑니다.
- 블로킹 풀은 프로세스 전체에 하나입니다. 필요할 때 스레드를 늘리고(상한 있음), 한동안 일이 없는 스레드는 끝냅니다. 작업은 들어온 순서(FIFO)로 처리합니다. 상한은 스레드 512개이고, 상한에 닿으면 작업을 큐에 쌓습니다. 10초 동안 일이 없는 스레드는 끝납니다. 작업 하나는 쉬고 있는 스레드 하나에 넘깁니다.
- `await`로 기다리던 task가 취소되어도 `f`는 끝까지 실행되고, 결과만 버려집니다.

```rok
let n = await task::spawn_blocking(|| heavy_checksum(path));
```

### `spawn`의 `Send` 검사

`task::spawn`은 결과 타입도 `Send`여야 합니다(`where F::Output: Send`). 또 **프레임**이 `Send`가 아닌 future를 거부합니다([2.13.9](213-marker.md#2139-send--sync-스레드-경계-마커)). future가 어디서 왔든 같습니다. 컴파일러는 `async` 블록이나 `async fn` 호출만 검사하지 않습니다. 매개변수, struct / 튜플 / 합의 필드, 도우미 함수의 결과, 리스트의 원소, 클로저가 들고 있는 future, 다른 코루틴 프레임 안의 future도 검사합니다. future의 프레임은 다음으로 이루어집니다.

- 캡처한 값.
- `await`를 넘어 살아 있는 지역 변수와 임시값.
- 프레임이 들고 있는 다른 future. 그 프레임도 재귀로 봅니다.

```rok
let c = Cell::new(0);                      // Cell: !Send
task::spawn(async { c.set(1); });          // ERROR: !Send 캡처
task::spawn_local(async { c.set(1); });    // OK: 현재 worker 에 고정

fn make() -> AsyncFuture<()> { let c = Cell::new(0); async { c.set(1); } }
task::spawn(make());                       // ERROR: helper 가 만든 future 의 frame 도 검사
```

같은 검사가 클로저를 다른 스레드로 보내는 자리에도 적용됩니다. 그런 자리는 `thread::spawn`, `thread::Builder::spawn`, `par`의 `Scope::spawn`, `task::spawn_blocking`입니다. 클로저가 `Send`가 아닌 프레임의 future를 들고 있으면 ``` `X` would move a future to another thread: ... ``` 에러입니다.

이 검사는 보수적으로 근사해도 됩니다. 구현은 실제로는 `Send`인 future를 거부해도 됩니다. 하지만 `Send`가 아닌 값을 다른 스레드로 넘기는 future를 받아들이면 안 됩니다. 검사는 다음 값 안으로는 들어가지 않습니다.

- 타입이 함수 값을 담을 수 없는 값. 이런 값은 future의 프레임을 숨길 수 없습니다.
- `unsafe impl Send`를 명시한 타입의 값(예: `Task<T>`). 그 타입이 안전성을 책임집니다.

`async fn`의 결과 타입 `AsyncFuture<T>`는 `T: Send`일 때 `Send`입니다. 에러를 던지는 판인 `TryAsyncFuture<T, E>`는 `T`와 `E`가 모두 `Send`일 때 그렇습니다. 캡처와 프레임은 위 검사가 따로 봅니다. 캡처가 `Send`이지만 `Sync`가 아닌 프레임은 받아들입니다. `async fn`이 받는 리스트, `async` 블록이 캡처하는 리스트나 future가 그런 예입니다. `task::spawn`은 채널 send처럼 future를 넘기기 때문입니다([2.13.9](213-marker.md#2139-send--sync-스레드-경계-마커)). task는 future가 캡처한 것의 복사본을 담은 자기 프레임을 받습니다. 이 복사본은 넘김 한 번으로 만듭니다. 그래서 캡처끼리 공유하던 것(같은 리스트 두 번, 리스트와 그 위의 future)은 계속 공유됩니다. spawn 뒤에 그 future나 그것이 캡처한 것을 쓰면 send 뒤와 마찬가지로 컴파일 에러(`E202`)입니다. 한 번도 폴링되지 않은 future만 넘길 수 있습니다. 다음 future를 넘기면 panic이 일어나고, 아무것도 공유하지 않습니다. 이미 시작한 것(`select`, 사용자의 `Future`, 또는 직접 한 번 폴링한 것), 끝난 것, 그리고 자기 프레임이나 캡처한 future의 프레임에 `[ref x]` 캡처가 있는 것입니다. 채널로 보내거나 `Mutex`나 `Frozen`에 넣는 코루틴 future도 같은 방식으로 넘겨지고, 그때마다 자기 프레임을 받습니다. `Task`가 넘겨진 task(`Frozen::get`, 채널, 락, `par` 항목)를 `await`하면 매번 결과의 복사본을 따로 받습니다. 한 스레드 안의 `Task` 복사본은 그 결과를 공유합니다.

### `select`: 먼저 끝나는 future

```rok
fn select<F: Future>(futs: List<F>) -> SelectFuture<F>;   // Output = (usize, F::Output)

let (i, v) = await task::select([fetch_a(), fetch_b()]);
```

`race`와 달리 future를 띄우지 않고, 기다리는 task **안에서** 실행합니다.

- 목록 순서대로 폴링합니다. 같은 poll에서 둘 이상이 끝나면 인덱스가 낮은 쪽이 이깁니다.
- 하나가 끝나면 이미 시작한 나머지는 모두 취소됩니다. 취소된 상태로 한 번 더 폴링되어 `finally` / `using`이 실행됩니다(2.18.4). 한 번도 폴링되지 않은 future는 실행되지 않습니다.
- 진 future가 만든 깨우기 등록(타이머, parking)은 모두 풀립니다.
- 기다리는 task가 취소되면 자식 future도 취소됩니다.
- 목록이 비어 있으면 panic이 일어납니다.
- future는 모두 한 타입이어야 합니다. 결과 타입이 다르면 `async { Msg::A(await a) }`처럼 감쌉니다.

`select`와 `sleep`으로 타임아웃을 만들 수도 있습니다.

```rok
enum Ev { Line(str), Timeout }

let (_, ev) = await task::select([
    async { Ev::Line(await read_line()) },
    async { await task::sleep(Duration::sec(5)); Ev::Timeout },
]);
```

### `TaskId`

`task::id()`는 실행 중인 task의 id입니다. task 밖에서 부르면 panic이 일어납니다(`try_id()`는 `none`을 돌려줍니다). `Task::id()`는 그 task의 id입니다. id는 프로세스 전역 카운터에서 나오고 다시 쓰이지 않습니다.

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

`Task<T>`는 `Future<Output = T>`를 구현하므로 `await t`로 기다릴 수 있습니다. [2.20 future](220-future.md)의 `IntoFuture` 블랭킷 impl에도 포함되므로 `Thread`와 같은 방식으로 기다릴 수 있습니다. `IntoFuture` 트레잇과 lang slot `future_trait`의 권위 있는 정의는 [2.20 future](220-future.md)에 있고, 이 절은 그 정의를 따릅니다.

## 2.18.3. 스케줄러 모델: work-stealing

프로세스에는 실행기(executor)가 하나 있습니다. 실행기는 처음 쓸 때 시작합니다. CPU 수(`cpu_count()`)만큼의 워커 스레드, 블로킹 레인, I/O 드라이버로 이루어집니다. `block_on`은 부른 스레드를 손님 워커로 이 실행기에 붙여 `f`를 끝까지 실행합니다. `block_on`이 돌아올 때 실행 중인 task는 계속 실행됩니다. 다만 `block_on`은 그 스레드에 고정된 `spawn_local` task도 끝난 뒤에야 돌아옵니다. 그 task는 다른 스레드로 옮겨 갈 수 없기 때문입니다. `par::*`와 `spawn_blocking`도 같은 executor를 씁니다.

`spawn`으로 띄운 task는 어느 워커에서든 실행될 수 있습니다. 그래서 `Future + Send`여야 하고 결과도 `Send`여야 합니다. `spawn_local`의 task는 그것을 만든 워커에 고정됩니다. 고정되는 것은 워커이지 OS 스레드가 아닙니다. 워커는 다른 OS 스레드에서 이어질 수 있으므로([refs-lang 7.6.7](../../refs-lang/007-runtime/706-scheduler.md#767-블로킹-syscall-자동-검출)), `spawn_local` task는 어느 OS 스레드에서 도는지나 그 스레드의 스레드 로컬 상태에 기대면 안 됩니다. (참조 런타임은 살아 있는 `spawn_local` task가 있는 워커를 넘기지 않습니다.) 다만 `block_on` 스레드가 폴링하지만 소유하지 않은 task도 있습니다. 기다리는 동안 맡은 `spawn` task입니다. 이런 task에서 부른 `spawn_local`은 대신 풀 워커에 고정됩니다. 그래서 `block_on`은 자기 task의 로컬 task만 기다립니다. 이런 poll 한 번에서 만든 로컬 task는 모두 같은 풀 워커로 가고, 그 poll이 끝난 뒤에야 시작합니다. 원래 그 task를 실행했을 워커에서와 같습니다.

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

로컬 큐는 lock-free Chase-Lev 덱입니다. 소유 워커와 훔치는 워커 모두 가장 오래된 task부터 꺼냅니다. 스스로를 계속 다시 깨우는 task가 다른 task를 굶기지 않게 하려는 것입니다([refs-lang 7.6.2](../../refs-lang/007-runtime/706-scheduler.md#762-work-stealing-구조)). 바쁜 워커는 task 32개마다 기한이 지난 타이머를 처리합니다. I/O 드라이버에서는 프로세스당 리액터 스레드 하나가 epoll / kqueue를 edge-triggered로 기다립니다. io_uring / IOCP는 향후 과제입니다([refs-lang 7.7](../../refs-lang/007-runtime/707-io-and-timer.md)). 타이머 구조는 구현이 정합니다(이진 힙이나 타이밍 휠, [refs-lang 7.7.4](../../refs-lang/007-runtime/707-io-and-timer.md#774-타이머-구조)).

표준 라이브러리의 권위는 API 시그니처까지입니다. 알고리듬 본문은 사양 밖의 런타임과 표준 라이브러리 본문에 속합니다.

`rok::core::task`는 런타임 훅도 내보냅니다. `TaskRef`, `Pool`, `Worker`, `wake_task`, `cancel_requested`, `unwinding`, 그리고 `host_enter` / `host_leave` / `host_run`(WebAssembly 컴포넌트의 이벤트 루프 진입점)입니다. 이 훅은 컴파일러와 런타임을 위해 공개되어 있을 뿐 런타임 내부용입니다. 안정된 API가 아닙니다.

## 2.18.4. 협력적 취소: `Task::cancel`

협력적 취소는 다음과 같이 씁니다.

```rok
let t = task::spawn(async { long_running() });
// ... 다른 일 ...
t.cancel();              // 협력적 신호. task 가 cancellation point 에서 종료
```

`cancel()`은 취소를 **요청**합니다. 규칙은 다음과 같습니다([refs-lang 7.7.9](../../refs-lang/007-runtime/707-io-and-timer.md#779-취소)).

- 모든 `await`가 취소 지점(cancellation point)입니다. 취소된 task는 다음 `await`에서 `return`처럼 본문을 떠납니다. 그때 지나는 `finally`와 `using` 정리가 안쪽부터 실행됩니다.
- 취소는 task 안에서 `catch`할 수 있는 에러가 아닙니다. task 안으로 `CancellationError`를 던지지 않습니다.
- 블로킹 시스템 호출은 취소 지점이 아닙니다. task는 그 호출이 끝난 뒤 다음 `await`에서 떠납니다.
- 소켓 I/O(`*_async`), `readable` / `writable`, `sleep` / `timeout`, `lock_async`, `recv_async` / `send_async`에서 기다리던 task는 취소되면 깨어나고 다음 `await`에서 떠납니다.
- task가 끝나면 그 task의 깨우기 등록(I/O readiness, 타이머, park한 대기)은 모두 풀립니다.
- `spawn_blocking`이나 그 위의 `fs::*_async`를 기다리다 취소되면 이미 시작된 작업은 백그라운드에서 끝까지 실행되고 결과만 버려집니다. 쓰기 같은 부수 효과는 일어납니다. 그 사이에 끝난 결과도 버립니다. 취소된 poll은 `Pending`을 돌려줍니다.
- 기다리던 future가 값(메시지, 락 guard)을 넘겨준 poll은 그 값을 지킵니다. 취소 검사는 "기다린 코루틴이 취소 때문에 떠났는가"를 봅니다. 값은 남고, task는 다음 `await`에서 떠납니다([refs-lang 7.7.9](../../refs-lang/007-runtime/707-io-and-timer.md#779-취소)).
- 마지막 취소 지점을 지난 뒤 취소된 task는 정상적으로 끝나고 결과가 있습니다. `await t` / `join`은 그 결과를 돌려줍니다.
- `CancellationError`는 결과 없이 끝난 task를 기다리는 쪽이 받습니다. 그런 task를 `try await t.join()`으로 기다리면 `CancellationError`가 던져집니다. `await t`로 기다리면 panic이 일어납니다.

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

## 2.18.5. `thread::*`: 1:1 OS 스레드

별도 모듈인 `thread`는 M:N 모델을 쓰지 않고 **OS 스레드**를 직접 띄웁니다.

```rok
use rok::std::thread;

let t = thread::spawn { os_level_work() };
let r = t.join();              // OS-level 동기 대기
```

두 모델의 차이는 다음과 같습니다.

| 측면        | `task::spawn`              | `thread::spawn`         |
|-------------|----------------------------|-------------------------|
| 모델        | M:N (사용자 공간 스케줄러) | 1:1 (OS thread)         |
| 비용        | 가벼움 (~1KB 프레임)       | 무거움 (~MB 스택)       |
| 협력적 취소 | ✓                          | ✗ (OS 수준 kill 필요)   |
| async 통합  | ✓                          | △ (joiner만 await)     |
| 사용처      | 일반 동시성 (기본)         | CPU-bound, 블로킹 I/O   |

기본으로는 `task::spawn`을 쓰고, OS 스레드가 꼭 필요할 때만 `thread::spawn`을 고르세요.

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

오버플로가 나면 `ns`..`days`와 `+` `-` `*` 연산자는 debug 빌드에서 panic이 일어나고, 그 밖에는 포화(saturate)합니다. `checked_*` / `saturating_*` 형태로 동작을 직접 고를 수 있습니다. `Duration`은 부호가 있습니다. 그래서 `earlier`가 더 나중이면 `duration_since`는 음수 기간을 돌려줍니다. 이때 `checked_duration_since`는 `none`을, `saturating_duration_since`는 `Duration::ZERO`를 돌려줍니다.

`await task::sleep(Duration::ms(100))`, `await task::sleep_until(t)` 같은 호출은 정확한 단조 시간(monotonic time)을 보장합니다. 두 타입은 `rok::core::task::time`에 있고(2.18.7), `rok::std::time`이 다시 내보냅니다.

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
