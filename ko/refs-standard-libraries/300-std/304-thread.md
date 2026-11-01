# 3.4. `thread::`: OS 스레드(1:1 모델)

`rok::std::thread` 모듈은 OS 수준 스레드를 다루는 진입점입니다. [2.18 task](../200-core/218-task.md)에서 정리한 대로, 스레드는 M:N 협력적 스케줄러 위에서 도는 task와 별개의 추상입니다. 이 모듈로 네이티브 스레드를 직접 띄웁니다.

## 3.4.1. thread와 task 중 고르기

상황별 선택 기준은 다음과 같습니다.

| 자리                           | 권장                                        |
|--------------------------------|---------------------------------------------|
| 일반 동시성 / async I/O        | `task::spawn` (가벼움)                      |
| CPU 중심 계산                  | `task::spawn` 또는 `par::*` (work-stealing) |
| 블로킹 C 라이브러리 호출       | `thread::spawn` (task 스케줄러를 막지 않음) |
| OS 스레드 API가 필요한 곳     | `thread::spawn` (TLS, signal mask, etc.)    |
| 임베디드 / no-task             | `thread::spawn`                             |

task로 가볍고 충분히 표현할 수 있는 곳에서는 task를 먼저 씁니다. OS 자원을 직접 다뤄야 하는 곳에서만 스레드로 내려갑니다.

## 3.4.2. API

기본 타입과 함수 시그니처는 다음과 같습니다.

```rok
struct Thread { /* OS handle, 제네릭 아님 */ }
struct JoinHandle<T> { /* join 결과 T */ }
struct ThreadId { /* opaque */ }       // PartialEq 구현

struct JoinError { .. }                 // 비공개 필드 (0.5.10)
impl JoinError {
    fn kind(self) -> JoinErrorKind;
    fn message(self) -> str;             // Display 가 보이는 그대로
}
// Display, Debug, Error, PartialEq, Eq, Clone 을 구현

#[non_exhaustive]
enum JoinErrorKind {
    Os(i32),                             // OS 의 join 에러 번호 (errno)
    Lost,                                // 실패한 REPL 셀과 함께 끝난 스레드
    Taken,                               // 이미 join 또는 detach 됨
}                                        // + PartialEq, Debug

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
fn panicking() -> bool;                        // panic 한 스레드에서, 프로세스가 끝나는 동안 true

// thread-local storage
fn local<T>(init: fn() -> T + Sync) -> ThreadLocal<T>;     // 각 스레드에서 처음 쓸 때 init 호출

impl<T> ThreadLocal<T> {
    fn with<R>(self, f: fn(T) -> R) -> R;    // 이 스레드의 값을 f에 빌려 준 결과 (범위 한정)
    fn set(self, value: T);                  // 이 스레드의 값 교체
    fn replace(self, value: T) -> T;         // 교체하고 이전 값을 돌려줌
    unsafe fn as_ptr(self) -> Ptr<T>;        // 이 스레드의 값, 스레드가 끝날 때까지 고정됨
}
impl<T: Copy> ThreadLocal<T> {
    fn get(self) -> T;                       // 이 스레드의 값의 복사본
}
impl<T: Default> ThreadLocal<T> {
    fn take(self) -> T;                      // 값을 꺼내고 T::default()를 남김
}
```

`Thread::unpark`는 그 스레드의 park 토큰을 세우고 깨웁니다. 토큰은 한 번의 `park`가 소비합니다.

`thread::spawn`은 기본 설정의 `Builder::spawn`을 `try!`로 감싼 함수입니다. `Builder::spawn`이 에러를 던지는 경우(타깃에 스레드가 없거나 OS가 새 스레드를 거부)에는 `thread::spawn`에서 panic이 일어납니다. 그 에러를 다루려면 `Builder::new().spawn(f)`를 씁니다.

`await handle`([2.20.5](../200-core/220-future.md#2205-intofuture-await-가능한-값))은 task 안에서 스레드를 기다립니다. join은 executor의 블로킹 레인([`task::spawn_blocking`](../200-core/218-task.md#2181-핵심-api))에서 돌므로 task만 기다립니다. 결과는 `T`입니다. OS join 실패(`JoinError`)는 panic이 됩니다. 이 실패를 다루려면 `join`을 호출합니다.

### panic과 `JoinError`

어느 스레드에서든 panic이 나면 프로세스 전체가 종료 코드 101로 끝납니다. 그래서 `join`이 다른 스레드의 panic을 받을 일은 없습니다. `thread::panicking()`은 프로세스가 끝나는 동안 panic한 스레드에서 도는 코드(임베더의 종료 훅)에서만 true입니다. `JoinError`가 나타내는 것은 두 가지입니다. OS 수준의 join 실패(`JoinErrorKind::Os`), 또는 실패한 REPL 셀과 함께 끝난 스레드의 `JoinErrorKind::Lost`([refs-lang 9.7.0](../../refs-lang/009-tools/907-jit-cells.md#970-셀-세션의-규칙))를 나타냅니다. 후자는 "thread join failed: the thread ended with a failed REPL cell"로 표시됩니다.

`JoinErrorKind::Taken`: 그 핸들이나 그 복사본에 이미 join이나 detach를 했습니다. 두 번째 `join`은 이 에러를 던지고, 두 번째 `detach`는 아무것도 하지 않습니다.

## 3.4.3. 사용 예

가장 간단한 spawn-join 패턴, builder, 스레드를 여러 개 띄우는 예를 차례로 보여 줍니다.

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

## 3.4.4. 스레드 로컬 저장소

스레드마다 독립된 저장소가 필요하면 `thread::local`을 씁니다.

```rok
let counter = thread::local(|| 0);

try thread::spawn(|| {
    counter.set(counter.get() + 1);         // 이 스레드의 값만 바뀜
}).join();
```

모양은 Rust의 `LocalKey`를 따릅니다. `with(f)`는 현재 스레드의 값을 `f`에 빌려 주고 `f`가 돌려준 것을 돌려줍니다. 이 값은 범위가 한정된 뷰이며 잠금 가드의 값처럼 추적됩니다([stdlib 2.16.3](../200-core/216-sync.md#2163-mutext-배타적-잠금)). 값을 `f` 밖으로 돌려주거나, 호출 뒤까지 저장하거나(캡처, 필드, `Cell`), 보내거나, 값을 간직하는 함수에 넘기면 컴파일 에러 E202입니다. `with`에 `async` 클로저를 주는 것도 에러입니다. 그 future가 `await`를 넘어 값을 들고 있다가 다른 worker 스레드에서 이어질 수 있기 때문입니다. 대신 `clone()`이나 `f`가 계산한 것을 남겨 둡니다.

```rok
static SEEN: ThreadLocal<List<str>> = thread::local(|| -> List<str> { [] });

SEEN.with(|l| l.push("a"));             // 이 스레드의 리스트를 그 자리에서 바꿈
let n = SEEN.with(|l| l.len());          // f가 계산한 것은 밖으로 나옴
let copy = SEEN.with(|l| l.clone());     // 복사본도 나옴
let all = SEEN.take();                   // 리스트 자체를 꺼내고 []를 남김
```

소유하는 형태는 값을 넣고 꺼냅니다. `set(v)`는 `v`를 저장하고, `replace(v)`는 `v`를 저장하고 이전 값을 돌려주며, `take()`는 값을 돌려주고 `T::default()`를 남깁니다. `get()`은 복사본을 돌려주므로 `T: Copy`가 필요합니다. 다른 `T`에 쓰면 컴파일 에러입니다(`with`, `replace`, `take`를 씁니다). `unsafe fn as_ptr()`는 현재 스레드의 값을 가리킵니다. 그 자리는 스레드가 끝날 때까지 고정되며, 포인터로 읽고 쓰는 일은 호출자가 책임집니다.

0.1.6에는 모든 `T`에 대한 `get()`과, `await`를 넘어 든 `get()` 값을 보고하는 `thread_local_across_await` 린트가 있었습니다. 0.2.0은 둘 다 없앱니다. `with`는 `await`할 수 없으므로 그런 경쟁을 쓸 수 없습니다. 린트 이름은 린트 수준에 여전히 쓸 수 있고 아무것도 하지 않습니다([refs-lang 6.2.9](../../refs-lang/006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)).

## 3.4.5. 구현 / 타깃

플랫폼별 지원 현황은 다음과 같습니다.

| 항목            | linux/macos/windows   | wasm32-wasip2 / 브라우저    | wasm32-wasip1-threads |
|-----------------|-----------------------|---------------------------|---------------------|
| `spawn`         | ✓                     | panic (`Builder::spawn`의 `Unsupported`) | ✓     |
| `Builder::spawn` | ✓                    | `Unsupported`를 던짐     | ✓                   |
| `local`         | ✓                     | ✓ (스레드 하나)           | ✓                   |
| `park`/`unpark` | futex / WaitOnAddress | ✓                         | ✓                   |

`wasm32-wasip2`와 브라우저에는 스레드가 하나뿐이고, `thread::current().name()`은 `none`입니다. `wasm32-wasip1-threads`에서 `rok_cpu_count()`와 사용 가능한 병렬도는 4입니다. WASI는 코어 수를 알려 줄 수 없기 때문입니다.
