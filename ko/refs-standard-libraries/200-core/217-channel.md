# 2.17. channel — 메시지 패싱

`rok::core::channel` 모듈은 task 와 thread 사이에서 메시지를 주고받기 위한 채널 primitive 를 제공합니다. 설계 측면에서는 Go 의 channel 과 Rust 의 `mpsc::channel` 에서 영감을 받았습니다.

이 모듈은 `use rok::core::**;` 가 묵시적으로 적용되므로, `channel` 과 `Sender`, `Receiver` 같은 이름을 별도의 import 없이 곧바로 사용할 수 있습니다. 즉 `channel::<T>()` 나 `Sender<T>` 처럼 짧게 표기하면 됩니다. 다만 시각적으로 출처를 분명히 드러내야 하는 자리에서는 `use rok::core::channel::*;` 를 명시적으로 적어 두는 편을 권장합니다.

## 2.17.1. 기본 형태

채널을 만드는 함수는 두 가지 형태로 제공됩니다.

```rok
fn channel<T: Send>() -> (Sender<T>, Receiver<T>);
fn channel_bounded<T: Send>(cap: usize) -> (Sender<T>, Receiver<T>);
```

먼저 `channel<T>()` 는 **unbounded** 채널을 만듭니다. capacity 가 무한하다는 의미이며, 실질적으로는 메모리 한계까지 메시지를 받아들입니다. 반면 `channel_bounded<T>(cap)` 은 **bounded** 채널을 만들며, capacity 가 `cap` 으로 고정되어 큐가 가득 차면 sender 쪽이 block 됩니다. `channel_bounded(0)` 은 rendezvous 채널로, `send` 는 receiver 가 그 값을 받을 때까지 기다립니다. 이 채널의 `try_send` 는 기다리는 receiver 가 없으면 `TrySendError::Full` 을 던집니다. `recv` 로 막힌 스레드뿐 아니라 `recv_async` / `for await` 에서 중단된 task 도 기다리는 receiver 로 셉니다.

### 사용 예

다음은 가장 단순한 형태의 사용 예시입니다.

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

채널의 양 끝을 이루는 두 타입은 다음과 같이 정의됩니다.

```rok
struct Sender<T> { /* ... */ }
struct Receiver<T> { /* ... */ }

impl<T> Sender<T> {
    fn send(self, value: T) throws (SendError<T>);
    fn try_send(self, value: T) throws (TrySendError<T>);
    fn send_async(self, value: T) -> SendFuture<T>;   // try await tx.send_async(v) — task 만 중단, throws (SendError<T>)
    fn clone(self) -> Sender<T>;             // 다중 sender
    fn close(self);                          // 이 sender 를 닫음
}

impl<T> Receiver<T> {
    fn recv(self) -> T | none;               // none = 모든 sender 가 close
    fn try_recv(self) -> T | none;           // none = 비어있음 또는 closed
    fn recv_async(self) -> RecvFuture<T>;    // await rx.recv_async() — task 만 중단, Output = T | none
    fn iter(self) -> ChannelIter<T>;         // for x in rx { ... }
}

impl<T> Stream for Receiver<T> {         // for await v in rx — task 만 중단
    type Item = T;
    ...
}

impl<T> IntoIterator for Receiver<T> {   // for x in rx — 모든 sender 가 close 되면 끝
    type Item = T;
    type IntoIter = ChannelIter<T>;
}

enum SendError<T> { Closed(T) }
enum TrySendError<T> { Full(T), Closed(T) }
// 둘 다 Display, Debug, Error 를 구현 (메시지는 값이 아니라 경우를 말함)
```

`T: Send` 이면 `Sender<T>` 와 `Receiver<T>` 도 `Send` 이므로 다른 task 나 스레드로 옮길 수 있습니다 ([2.13.9](213-marker.md#2139-send--sync--스레드-경계-marker)).

`recv` 와 `send` (bounded 가 가득 찬 경우, rendezvous 에서 값이 받아질 때까지) 는 스레드를 block 합니다. async 코드에서 task 만 중단하려면 `await rx.recv_async()`, `for await v in rx`, `try await tx.send_async(v)` 를 씁니다. `send_async` 는 bounded 채널이 가득 찬 동안, rendezvous 채널에서는 receiver 가 값을 가져갈 때까지 task 를 중단합니다. `send` 와 같이, 이 sender 가 이미 `close()` 되었으면 값을 담은 `SendError::Closed` 를 던집니다. 취소된 `recv_async` 는 값을 꺼내지 않고, 취소된 `send_async` 는 값을 보내지 않습니다 ([2.18.4](218-task.md#2184-협력적-취소--taskcancel)). `Mutex::lock` / `lock_async` 도 같은 구분입니다 ([2.16](216-sync.md)).

## 2.17.3. close 의미

채널이 닫히는 시점과 그 신호가 전달되는 방식은 다음과 같습니다. 우선 모든 sender 가 `close()` 되면 채널은 closed 상태가 됩니다. 이때 receiver 쪽에서는 `recv()` 가 `none` 을 반환하여 close 신호를 받게 됩니다. rok 은 scope 종료 시 자동 drop 이 없으므로 ([2.13.4](213-marker.md#2134-drop--using-해제-시-정리)), sender 를 닫는 것은 명시적인 `close()` 입니다.

다음은 `close()` 로 채널을 닫는 예시입니다.

```rok
let (tx, rx) = channel::<i32>();
try tx.send(1);
tx.close();            // 명시 close

rx.recv();             //→ 1
rx.recv();             //→ none  (closed)
```

## 2.17.4. multi-producer

`Sender<T>` 는 `Clone` 을 구현하므로, 여러 task 가 같은 채널에 메시지를 보낼 수 있습니다.

```rok
let (tx, rx) = channel::<str>();
let tx2 = tx.clone();

task::spawn(async { try tx.send("hello"); });
task::spawn(async { try tx2.send("world"); });

let v1 = rx.recv();
let v2 = rx.recv();
```

반면 `Receiver<T>` 는 Clone 을 구현하지 않으므로 single consumer 로 동작합니다. 이는 mpsc — multi producer single consumer — 모델을 의미합니다. 만약 multi-consumer 가 필요한 상황이라면 별도의 fan-out task 를 두어 분배하는 방식을 택하면 됩니다.

## 2.17.5. `for` / `Stream` 통합

`Receiver<T>` 는 `IntoIterator<Item = T>` 를 구현하므로 `for` 문에서 자연스럽게 사용할 수 있습니다.

```rok
for v in rx {
    use(v);                  // 채널 close 까지 반복
}
```

이에 더해 비동기 iterator 인 `Stream<Item = T>` 도 함께 구현되므로, `for await` 구문도 동일하게 사용 가능합니다.

```rok
for await v in rx {
    await handle(v);
}
```

자세한 내용은 [2.7 iter](207-iter.md) 와 [2.8 stream](208-stream.md) 를 참고합니다.

## 2.17.6. 모듈 매핑

각 타입과 함수가 어디에 구현되는지는 다음 표로 정리됩니다.

| 타입           | 모듈                          | 내부 구조                |
|----------------|-------------------------------|--------------------------|
| `Sender<T>`    | `rok::core::channel::sender`   | atomic ref count + queue |
| `Receiver<T>`  | `rok::core::channel::receiver` | queue + parking          |
| `channel<T>()` | `rok::core::channel`           | Arc-like internal state  |

내부 큐 구조는 bounded 채널의 경우 Tokio mpsc 와 유사한 ring buffer 를 사용하고, unbounded 채널의 경우 segmented linked-list 를 사용합니다. bounded 채널의 큐는 lock-free 이며, unbounded 채널의 segment 리스트는 lock 을 쓸 수 있습니다.
