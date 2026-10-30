# 3.4. `thread::` - OS 스레드 (1:1 모델)

`rok::std::thread` 모듈은 OS 수준 스레드를 다루는 진입점입니다. [2.18 task](../200-core/218-task.md)에서 정리한 대로, M:N 협력적 스케줄러 위에서 도는 task와는 별개의 추상이며, 이 모듈로 native thread를 직접 spawn합니다.

## 3.4.1. 언제 thread vs task

상황별 선택 기준은 다음과 같습니다.

| 자리                           | 권장                                        |
|--------------------------------|---------------------------------------------|
| 일반 동시성 / async I/O        | `task::spawn` (가벼움)                      |
| CPU-bound 계산                 | `task::spawn` 또는 `par::*` (work-stealing) |
| Blocking C 라이브러리 호출     | `thread::spawn` (task scheduler block 회피) |
| OS thread API가 필요한 곳     | `thread::spawn` (TLS, signal mask, etc.)    |
| 임베디드 / no-task             | `thread::spawn`                             |

task로 가볍고 충분히 표현할 수 있는 곳에서는 task가 우선이며, OS 자원을 직접 다뤄야 하는 곳에서만 thread로 내려갑니다.

## 3.4.2. API

기본 타입과 함수 시그니처는 다음과 같습니다.

```rok
struct Thread { /* OS handle, 제네릭 아님 */ }
struct JoinHandle<T> { /* join 결과 T */ }
struct ThreadId { /* opaque */ }       // PartialEq 구현

struct JoinError { code: i32; }          // OS join 에러 번호 (errno), JOIN_LOST, 또는 JOIN_TAKEN
const JOIN_LOST: i32 = -1;               // 실패한 REPL 셀과 함께 끝난 스레드
const JOIN_TAKEN: i32 = -2;              // 이미 join 또는 detach 됨
// Display, Debug, Error 를 구현

fn spawn<T: Send>(f: fn() -> T + Send) -> JoinHandle<T>;   // = try! Builder::new().spawn(f)

struct Builder { /* opaque: 이름, stack 크기 */ }

impl Builder {
    fn new() -> Builder;                 // 이름 없음, 기본 stack
    fn name(self, name: str) -> Builder;
    fn stack_size(self, size: usize) -> Builder;   // 바이트 (기본 8 MiB)
    fn spawn<T: Send>(self, f: fn() -> T + Send) throws (IoError) -> JoinHandle<T>;
}

impl<T> JoinHandle<T> {
    fn join(self) throws (JoinError) -> T;
    fn detach(self);                     // join 하지 않음. 스레드는 끝까지 실행
    fn thread(self) -> Thread;
}

impl<T: Send> IntoFuture for JoinHandle<T> { ... }   // `await handle` 은 join 하고 T 를 냄

impl ThreadId {
    fn as_u64(self) -> u64;              // 런타임이 매긴 스레드 id
}

impl Thread {
    fn id(self) -> ThreadId;
    fn name(self) -> str | none;        // Builder::name 으로 준 이름 (OS 스레드 이름으로도 설정), 없으면 none
    fn unpark(self);
}

// 현재 thread
fn current() -> Thread;
fn yield_now();
fn sleep(dur: Duration);
fn park();                                     // unpark 될 때까지 block (깨운 이유 없이 돌아올 수 있음)
fn park_timeout(dur: Duration);
fn panicking() -> bool;                        // 항상 false: panic 은 그 전에 프로세스를 끝냄

// thread-local storage
fn local<T>(init: fn() -> T + Sync) -> ThreadLocal<T>;     // 각 스레드에서 처음 쓸 때 init 호출

impl<T> ThreadLocal<T> {
    fn get(self) -> T;                       // 현재 스레드의 값
    fn set(self, value: T);
    fn with<R>(self, f: fn(T) -> R) -> R;
}
```

`Thread::unpark`는 그 스레드의 park 토큰을 세우고 깨웁니다. 토큰은 한 번의 `park`가 소비합니다.

`thread::spawn`은 기본값 `Builder::spawn`을 `try!`로 감싼 것입니다. `Builder::spawn`이 throw하는 경우(target에 스레드가 없거나 OS가 새 스레드를 거부) `thread::spawn`은 panic합니다. 그 오류를 다루려면 `Builder::new().spawn(f)`를 씁니다.

`await handle`([2.20.5](../200-core/220-future.md#2205-intofuture---await-가능한-값))은 task 안에서 스레드를 기다립니다. join은 executor의 blocking lane([`task::spawn_blocking`](../200-core/218-task.md#2181-핵심-api))에서 돌므로 task만 기다립니다. 결과는 `T`이며, OS join 실패(`JoinError`)는 panic하므로 이를 다루려면 `join`을 호출합니다.

### panic 과 `JoinError`

어느 스레드에서든 panic이 나면 프로세스 전체가 종료 코드 101로 끝납니다. 그래서 `join`이 다른 스레드의 panic을 받을 일은 없습니다. `JoinError`는 OS 수준의 join 실패, 또는 실패한 REPL 셀과 함께 끝난 스레드의 `JOIN_LOST`([refs-lang 9.7.0](../../refs-lang/009-tools/907-jit-cells.md#970-셀-세션의-규칙))를 나타냅니다. 후자는 "thread join failed: the thread ended with a failed REPL cell"로 표시됩니다.

`JOIN_TAKEN` (-2): 그 handle이나 그 복사본이 이미 join되었거나 detach되었습니다. 두 번째 `join`은 이 에러를 던지고, 두 번째 `detach`는 아무것도 하지 않습니다.

## 3.4.3. 사용 예

가장 간단한 spawn-join 패턴부터 builder, 여러 번 spawn하는 예까지 차례로 보여 줍니다.

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
    .map(|i| thread::spawn(|| { compute(i) }))
    .collect();

let results: List<i64> = handles
    .into_iter()
    .map(|h| try! h.join())
    .collect();
```

## 3.4.4. thread-local storage

스레드마다 독립된 저장소가 필요하면 `thread::local`을 씁니다.

```rok
let counter = thread::local(|| 0);

try thread::spawn(|| {
    counter.set(counter.get() + 1);         // 이 스레드의 값만 바뀜
}).join();
```

## 3.4.5. impl / target

플랫폼별 지원 현황은 다음과 같습니다.

| 항목            | linux/macos/windows   | wasm32-wasi / 브라우저    | wasm32-wasi-threads |
|-----------------|-----------------------|---------------------------|---------------------|
| `spawn`         | ✓                     | panic (`Builder::spawn`의 `Unsupported`) | ✓     |
| `Builder::spawn` | ✓                    | `Unsupported`를 throw    | ✓                   |
| `local`         | ✓                     | ✓ (스레드 하나)           | ✓                   |
| `park`/`unpark` | futex / WaitOnAddress | ✓                         | ✓                   |

`wasm32-wasi`와 브라우저에는 스레드가 하나뿐이며 `thread::current().name()`은 `none`입니다. `wasm32-wasi-threads`에서 `rok_cpu_count()`와 사용 가능한 병렬도는 4입니다. WASI는 코어 수를 알려 줄 수 없기 때문입니다.
