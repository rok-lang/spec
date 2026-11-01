# 2.17. channel: 메시지 패싱

`rok::core::channel` 모듈에는 task와 스레드 사이에서 메시지를 주고받는 채널 기본 요소가 있습니다. 설계는 Go의 채널과 Rust의 `mpsc::channel`에서 영감을 받았습니다.

`channel`, `Sender`, `Receiver`는 [prelude](../000-meta/002-import.md)에 들어 있습니다. 그래서 가져오지 않고도 `channel::<T>()`, `Sender<T>`처럼 짧게 쓸 수 있습니다. 출처를 드러내야 하는 자리에서는 `use rok::core::channel::*;`를 적기를 권장합니다.

## 2.17.1. 기본 형태

채널은 두 함수로 만듭니다.

```rok
const fn channel<T: Send>() -> (Sender<T>, Receiver<T>);
const fn channel_bounded<T: Send>(cap: usize) -> (Sender<T>, Receiver<T>);
```

`channel<T>()`는 **unbounded** 채널을 만듭니다. 용량이 무한하므로 실제로는 메모리 한계까지 메시지를 받아들입니다. `channel_bounded<T>(cap)`은 용량이 `cap`으로 고정된 **bounded** 채널을 만듭니다. 큐가 가득 차면 보내는 쪽이 블록됩니다. `channel_bounded(0)`은 랑데부(rendezvous) 채널입니다. 이 채널에서 `send`는 받는 쪽이 그 값을 가져갈 때까지 기다립니다. 이 채널의 `try_send`는 값을 기다리는 쪽이 없으면 `TrySendError::Full`을 던집니다. `recv`로 막힌 스레드뿐 아니라 `recv_async` / `for await`에서 중단된 task도 값을 기다리는 쪽으로 셉니다.

### 사용 예

가장 단순한 사용 예입니다.

```rok
let (tx, rx) = channel::<i32>();

task::spawn(async {
    try tx.send(1);
    try tx.send(2);
    try tx.send(3);
});

while let v: i32 = rx.recv() {
    use(v);                  //→ 1, 2, 3
}
```

## 2.17.2. `Sender<T>` / `Receiver<T>`

채널의 양 끝을 이루는 두 타입의 정의입니다.

```rok
struct Sender<T> { /* ... */ }
struct Receiver<T> { /* ... */ }

impl<T> Sender<T> {
    fn send(self, value: T) throws (SendError<T>);
    fn try_send(self, value: T) throws (TrySendError<T>);
    const fn send_async(self, value: T) -> SendFuture<T>;   // try await tx.send_async(v): task 만 중단, throws (SendError<T>)
    fn clone(self) -> Sender<T>;             // 다중 sender
    fn close(self);                          // 이 sender 를 닫음
}

impl<T> Drop for Sender<T> {             // sender 를 해제하면 닫힘
    fn drop(ref self);
}

impl<T> Receiver<T> {
    fn recv(self) -> T | none;               // none = 모든 sender 가 close
    fn try_recv(self) -> T | Empty | Closed; // 큐가 비면: 열린 sender 가 있으면 Empty, 없으면 Closed
    const fn recv_async(self) -> RecvFuture<T>;    // await rx.recv_async(): task 만 중단, Output = T | none
    fn iter(self) -> ReceiverIter<T>;         // for x in rx { ... }
}

impl<T> Stream for Receiver<T> {         // for await v in rx: task 만 중단
    type Item = T;
    ...
}

impl<T> IntoIterator for Receiver<T> {   // for x in rx: 모든 sender 가 close 되면 끝
    type Item = T;
    type IntoIter = ReceiverIter<T>;
}

enum SendError<T> { Closed(T) }
enum TrySendError<T> { Full(T), Closed(T) }
// 둘 다 Display, Debug, Error 를 구현 (메시지는 값이 아니라 경우를 말함)
#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug)]
struct Empty;      // try_recv: 큐가 비었고 아직 열린 sender 가 있음
#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug)]
struct Closed;     // try_recv: 큐가 비었고 모든 sender 가 close 됨

struct RecvFuture<T> { /* ... */ }
impl<T> Future for RecvFuture<T> {
    type Output = T | none;
    ...
}

struct SendFuture<T> { /* ... */ }
impl<T> SendFuture<T> {
    // `task::Join`(2.18.1)처럼 `try await`가 구동, 에러는 send 의 에러
    fn poll(ref self, ref ctx: Context) throws (SendError<T>) -> Poll<()>;
}

struct ReceiverIter<T> { /* ... */ }    // Iterator<Item = T>, 모든 sender 가 close 되면 끝
```

`T: Send`이면 `Sender<T>`와 `Receiver<T>`도 `Send`이므로 다른 task나 스레드로 옮길 수 있습니다([2.13.9](213-marker.md#2139-send--sync-스레드-경계-마커)). `send`, `try_send`, `send_async`는 값이 닿는 컬렉션의 복사본을 큐에 넣습니다(2.13.9). 그래서 받는 쪽은 보낸 쪽과 컬렉션을 공유하지 않습니다.

`recv`와 `send`는 스레드를 블록합니다. 보내기는 bounded 채널이 가득 찼을 때, 그리고 랑데부 채널에서 받는 쪽이 값을 가져갈 때까지 블록합니다. async 코드에서 task만 중단하려면 `await rx.recv_async()`, `for await v in rx`, `try await tx.send_async(v)`를 씁니다. `send_async`는 bounded 채널이 가득 찬 동안 task를 중단합니다. 랑데부 채널에서는 받는 쪽이 값을 가져갈 때까지 중단합니다. `send`와 마찬가지로 이 sender가 이미 `close()`되었으면 값을 담은 `SendError::Closed`를 던집니다. 취소된 `recv_async`는 값을 꺼내지 않고, 취소된 `send_async`는 값을 보내지 않습니다([2.18.4](218-task.md#2184-협력적-취소-taskcancel)). `Mutex::lock` / `lock_async`도 같은 방식으로 구분합니다([2.16](216-sync.md)).

## 2.17.3. 닫기 의미

모든 sender가 `close()`나 해제로 닫히면 채널은 닫힌 상태가 됩니다. 이때 받는 쪽에서는 `recv()`가 `none`을 반환합니다. 이것이 닫힘 신호입니다.

`close()`로 채널을 닫는 예입니다.

```rok
let (tx, rx) = channel::<i32>();
try tx.send(1);
tx.close();            // 명시 close

rx.recv();             //→ 1
rx.recv();             //→ none  (closed)
```

`Sender`는 `Drop` 자원입니다([2.13.4](213-marker.md#2134-drop-해제-시-정리)). `let tx2 = tx`는 sender를 옮기고, `clone()`은 따로 주인을 가진 sender를 하나 더 만들며, `close()`는 그 sender만 닫습니다. 컴파일러는 마지막 주인이 스코프를 벗어나는 곳에서 sender를 해제하고, task는 끝날 때 `async` 블록이 캡처한 sender를 해제합니다. 모든 sender가 닫히거나 해제되면 채널이 닫힙니다. sender 해제는 실패하지 않으므로 직접 닫지 않아도 됩니다. 스레드의 클로저는 캡처한 값을 해제하지 않으므로, 스레드에서는 할 일을 마친 뒤 sender를 닫습니다.

## 2.17.4. 다중 생산자와 다중 소비자

`Sender<T>`는 `Clone`을 구현하므로 여러 task가 같은 채널에 메시지를 보낼 수 있습니다.

```rok
let (tx, rx) = channel::<str>();
let tx2 = tx.clone();

task::spawn(async { try tx.send("hello"); });
task::spawn(async { try tx2.send("world"); });

let v1 = rx.recv();
let v2 = rx.recv();
```

`Receiver<T>`는 `Clone`을 구현하지 않지만 핸들입니다. 복사본(`let rx2 = rx;`, 또는 다른 task나 스레드로 옮긴 것)은 같은 큐를 공유합니다. 따라서 채널은 다중 생산자, 다중 소비자(MPMC)입니다. `Receiver` 복사본 여럿이 동시에 `recv`(또는 `try_recv`, `recv_async`, `for`)를 불러도 되고, 보낸 값 하나는 그중 먼저 가져간 쪽 하나에만 갑니다. 모든 값을 보는 받는 쪽은 없습니다. 모든 sender가 `close()`되고 큐가 비면 모든 복사본이 `none`을 받습니다. 값 하나를 여러 소비자에게 주려면 소비자마다 채널을 두고 값을 전달하는 fan-out task를 둡니다.

## 2.17.5. `for` / `Stream` 통합

`Receiver<T>`는 `IntoIterator<Item = T>`를 구현하므로 `for` 문에 바로 쓸 수 있습니다.

```rok
for v in rx {
    use(v);                  // 채널 close 까지 반복
}
```

비동기 반복자인 `Stream<Item = T>`도 구현하므로 `for await`도 같은 방식으로 쓸 수 있습니다.

```rok
for await v in rx {
    await handle(v);
}
```

자세한 내용은 [2.7 iter](207-iter.md)와 [2.8 stream](208-stream.md)에 있습니다.

## 2.17.6. 모듈 매핑

모든 타입과 함수는 `rok::core::channel`에 정의되고, 그 경로 하나만 있습니다([0.1.6](../000-meta/001-tier-model.md#016-항목마다-경로-하나)).

| 타입           | 모듈                 | 내부 구조                |
|----------------|----------------------|--------------------------|
| `Sender<T>`    | `rok::core::channel` | atomic ref count + queue |
| `Receiver<T>`  | `rok::core::channel` | queue + parking          |
| `channel<T>()`, `channel_bounded<T>(cap)` | `rok::core::channel` | Arc-like internal state |
| `SendError<T>`, `TrySendError<T>` | `rok::core::channel` | 보내지 못한 값을 담은 에러 |
| `Empty`, `Closed` | `rok::core::channel` | `try_recv`의 결과 |
| `ReceiverIter<T>` | `rok::core::channel` | `for x in rx`의 반복자 |
| `RecvFuture<T>`, `SendFuture<T>` | `rok::core::channel` | `recv_async` / `send_async`의 future |

내부 큐로 bounded 채널은 Tokio mpsc와 비슷한 링 버퍼를 쓰고, unbounded 채널은 세그먼트 연결 리스트를 씁니다. bounded 채널의 큐는 lock-free입니다. unbounded 채널의 세그먼트 리스트는 락을 써도 됩니다.
