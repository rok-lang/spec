# 3.4. `thread::` — OS 스레드 (1:1 모델)

`rok::std::thread` 모듈은 OS-level 스레드를 다루기 위한 진입점입니다. M:N 협력적 스케줄러 위에서 동작하는 task 와는 [2.18 task](../200-core/218-task.md) 에서 정리한 대로 별도의 추상이며, 이 모듈은 native thread 를 직접 spawn 하는 경로를 제공합니다.

## 3.4.1. 언제 thread vs task

어떤 상황에서 어느 쪽을 골라야 하는지는 다음 표로 가늠할 수 있습니다.

| 자리                           | 권장                                        |
|--------------------------------|---------------------------------------------|
| 일반 동시성 / async I/O        | `task::spawn` (가벼움)                      |
| CPU-bound 계산                 | `task::spawn` 또는 `par::*` (work-stealing) |
| Blocking C 라이브러리 호출     | `thread::spawn` (task scheduler block 회피) |
| OS thread API 가 요구되는 자리 | `thread::spawn` (TLS, signal mask, etc.)    |
| 임베디드 / no-task             | `thread::spawn`                             |

요컨대 task 가 충분히 가볍고 표현력 있게 처리할 수 있는 영역에서는 task 가 우선이고, OS 자원을 직접 다뤄야 하는 자리에서만 thread 로 내려가는 편이 자연스럽습니다.

## 3.4.2. API

기본 타입과 함수 시그니처는 아래와 같습니다.

```rok
struct Thread<T> { /* OS handle */ }
struct JoinHandle<T> { /* JoinHandle */ }
struct ThreadId { /* opaque */ }

fn spawn<T: Send>(f: fn() -> T + Send) -> JoinHandle<T>;

struct Builder {
    name: str | none;
    stack_size: usize | none;
}

impl Builder {
    fn new() -> Builder;
    fn name(self, name: str) -> Builder;
    fn stack_size(self, size: usize) -> Builder;
    fn spawn<T: Send>(self, f: fn() -> T + Send) throws (IoError) -> JoinHandle<T>;
}

impl<T> JoinHandle<T> {
    fn join(self) throws (JoinError) -> T;
    fn thread(self) -> ref Thread<T>;
}

impl<T> Thread<T> {
    fn id(self) -> ThreadId;
    fn name(self) -> str | none;
    fn unpark(self);
}

// 현재 thread
fn current() -> Thread<()>;
fn yield_now();
fn sleep(dur: Duration);
fn park();                                     // unpark 까지 block
fn park_timeout(dur: Duration);
fn panicking() -> bool;

// thread-local storage
fn local<T>(init: fn() -> T + Sync) -> ThreadLocal<T>;     // 각 스레드에서 init 호출
```

## 3.4.3. 사용 예

가장 간단한 spawn-join 패턴부터 builder, 다중 spawn 까지 차례로 살펴봅니다.

```rok
use rok::std::thread;

let h = thread::spawn(|| {
    heavy_blocking_work()
});
let result = try h.join();

// builder
let h = try thread::Builder::new()
    .name("worker-1")
    .stack_size(4 * 1024 * 1024)
    .spawn(|| { ... });

// 다중
let handles: List<JoinHandle<i64>> = (0..10)
    .map(|i| thread::spawn(move || { compute(i) }))
    .collect();

let results: List<i64> = handles
    .into_iter()
    .map(|h| try! h.join())
    .collect();
```

## 3.4.4. thread-local storage

각 스레드마다 독립된 저장소가 필요하다면 `thread::local` 을 활용합니다.

```rok
let counter = thread::local(|| 0);

thread::spawn(|| {
    counter.with(|c| { c.set(c.get() + 1); });
}).join();
```

## 3.4.5. impl / target

플랫폼별 지원 현황은 다음과 같이 정리됩니다.

| 항목            | linux/macos/windows   | wasi                      |
|-----------------|-----------------------|---------------------------|
| `spawn`         | ✓                     | ✓ (wasi-threads, partial) |
| `Builder`       | ✓                     | partial                   |
| `local`         | ✓                     | partial                   |
| `park`/`unpark` | futex / WaitOnAddress | partial                   |

데스크톱 계열 OS 에서는 일관된 동작을 보장하지만, wasi 는 thread proposal 의 진전에 맞춰 점진적으로 채워질 영역입니다.
