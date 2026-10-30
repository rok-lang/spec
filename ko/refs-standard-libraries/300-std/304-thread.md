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
struct Thread { /* OS handle - 제네릭 아님 */ }
struct JoinHandle<T> { /* join 결과 T */ }
struct ThreadId { /* opaque */ }

struct JoinError { code: i32; }          // OS join 에러 번호 (errno), JOIN_LOST, 또는 JOIN_TAKEN
const JOIN_LOST: i32 = -1;               // 실패한 REPL 셀과 함께 끝난 스레드
const JOIN_TAKEN: i32 = -2;              // 이미 join 또는 detach 됨
// Display, Debug, Error 를 구현

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
    fn detach(self);                     // join 하지 않음 - 스레드는 끝까지 실행
    fn thread(self) -> Thread;
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

// thread-local storage
fn local<T>(init: fn() -> T + Sync) -> ThreadLocal<T>;     // 각 스레드에서 처음 쓸 때 init 호출

impl<T> ThreadLocal<T> {
    fn get(self) -> T;                       // 현재 스레드의 값
    fn set(self, value: T);
    fn with<R>(self, f: fn(T) -> R) -> R;
}
```

`Thread::unpark`는 그 스레드의 park 토큰을 세우고 깨웁니다. 토큰은 한 번의 `park`가 소비합니다.

### panic과 `JoinError`

어느 스레드에서든 panic이 나면 프로세스 전체가 종료 코드 101로 끝납니다. 그래서 `join`이 다른 스레드의 panic을 받을 일은 없습니다. `JoinError`가 나타내는 것은 두 가지입니다. OS 수준의 join 실패, 또는 실패한 REPL 셀과 함께 끝난 스레드의 `JOIN_LOST`([refs-lang 9.7.0](../../refs-lang/009-tools/907-jit-cells.md#970-셀-세션의-규칙))를 나타냅니다. 후자는 "thread join failed: the thread ended with a failed REPL cell"로 표시됩니다.

`JOIN_TAKEN` (-2): 그 핸들이나 그 복사본에 이미 join이나 detach를 했습니다. 두 번째 `join`은 이 에러를 던지고, 두 번째 `detach`는 아무것도 하지 않습니다.

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

## 3.4.5. 구현 / 타깃

플랫폼별 지원 현황은 다음과 같습니다.

| 항목            | linux/macos/windows   | wasm32-wasi / 브라우저    | wasm32-wasi-threads |
|-----------------|-----------------------|---------------------------|---------------------|
| `spawn`         | ✓                     | `Unsupported`를 던짐     | ✓                   |
| `Builder`       | ✓                     | `Unsupported`를 던짐     | ✓                   |
| `local`         | ✓                     | ✓ (스레드 하나)           | ✓                   |
| `park`/`unpark` | futex / WaitOnAddress | ✓                         | ✓                   |

`wasm32-wasi`와 브라우저에는 스레드가 하나뿐이고, `thread::current().name()`은 `none`입니다. `wasm32-wasi-threads`에서 `rok_cpu_count()`와 사용 가능한 병렬도는 4입니다. WASI는 코어 수를 알려 줄 수 없기 때문입니다.
