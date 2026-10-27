# 2.17. channel - Message passing

The `rok::core::channel` module provides channel primitives for exchanging messages between tasks and threads. The design is inspired by Go's channels and Rust's `mpsc::channel`.

Because `use rok::core::**;` applies implicitly, names such as `channel`, `Sender` and `Receiver` are usable without any import, in short forms like `channel::<T>()` or `Sender<T>`. Where the origin should be visible, writing `use rok::core::channel::*;` explicitly is recommended.

## 2.17.1. Basic form

Channels are created by two functions.

```rok
fn channel<T: Send>() -> (Sender<T>, Receiver<T>);
fn channel_bounded<T: Send>(cap: usize) -> (Sender<T>, Receiver<T>);
```

`channel<T>()` creates an **unbounded** channel: its capacity is infinite, so in practice it accepts messages up to the memory limit. `channel_bounded<T>(cap)` creates a **bounded** channel whose capacity is fixed at `cap`; when the queue is full, the sender blocks. `channel_bounded(0)` is a rendezvous channel: `send` waits until a receiver takes the value. On this channel `try_send` throws `TrySendError::Full` if no receiver is waiting. Not only a thread blocked in `recv` but also a task suspended in `recv_async` / `for await` counts as a waiting receiver.

### Example

The simplest usage:

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

The two types forming the ends of a channel are defined as follows.

```rok
struct Sender<T> { /* ... */ }
struct Receiver<T> { /* ... */ }

impl<T> Sender<T> {
    fn send(self, value: T) throws (SendError<T>);
    fn try_send(self, value: T) throws (TrySendError<T>);
    fn send_async(self, value: T) -> SendFuture<T>;   // try await tx.send_async(v): suspends only the task, throws (SendError<T>)
    fn clone(self) -> Sender<T>;             // multiple senders
    fn close(self);                          // closes this sender
}

impl<T> Receiver<T> {
    fn recv(self) -> T | none;               // none = every sender closed
    fn try_recv(self) -> T | none;           // none = empty or closed
    fn recv_async(self) -> RecvFuture<T>;    // await rx.recv_async(): suspends only the task, Output = T | none
    fn iter(self) -> ChannelIter<T>;         // for x in rx { ... }
}

impl<T> Stream for Receiver<T> {         // for await v in rx: suspends only the task
    type Item = T;
    ...
}

impl<T> IntoIterator for Receiver<T> {   // for x in rx: ends when every sender is closed
    type Item = T;
    type IntoIter = ChannelIter<T>;
}

enum SendError<T> { Closed(T) }
enum TrySendError<T> { Full(T), Closed(T) }
// both implement Display, Debug and Error (the message names the case, not the value)
```

If `T: Send`, then `Sender<T>` and `Receiver<T>` are also `Send` and can be moved to other tasks or threads ([2.13.9](213-marker.md#2139-send--sync---thread-boundary-markers)). `send`, `try_send` and `send_async` queue a copy of the value's collections (2.13.9): the receiver shares no collection with the sender.

`recv` and `send` (when a bounded channel is full, or until a rendezvous value is taken) block the thread. To suspend only the task in async code, use `await rx.recv_async()`, `for await v in rx`, or `try await tx.send_async(v)`. `send_async` suspends the task while a bounded channel is full, or on a rendezvous channel until a receiver takes the value. Like `send`, if this sender has already been `close()`d it throws `SendError::Closed` carrying the value. A cancelled `recv_async` does not take a value, and a cancelled `send_async` does not send its value ([2.18.4](218-task.md#2184-cooperative-cancellation---taskcancel)). `Mutex::lock` / `lock_async` make the same distinction ([2.16](216-sync.md)).

## 2.17.3. Close semantics

When a channel closes and how the signal is delivered: once every sender is `close()`d, the channel is closed. The receiver then gets the close signal as `recv()` returning `none`. Rok has no automatic drop at scope exit ([2.13.4](213-marker.md#2134-drop---cleanup-on-using-release)), so closing a sender is an explicit `close()`.

Closing a channel with `close()`:

```rok
let (tx, rx) = channel::<i32>();
try tx.send(1);
tx.close();            // explicit close

rx.recv();             //→ 1
rx.recv();             //→ none  (closed)
```

## 2.17.4. Multi-producer

`Sender<T>` implements `Clone`, so multiple tasks can send messages to the same channel.

```rok
let (tx, rx) = channel::<str>();
let tx2 = tx.clone();

task::spawn(async { try tx.send("hello"); });
task::spawn(async { try tx2.send("world"); });

let v1 = rx.recv();
let v2 = rx.recv();
```

`Receiver<T>` does not implement Clone, so it acts as a single consumer. This is the mpsc (multi producer single consumer) model. If multiple consumers are needed, use a separate fan-out task that distributes the messages.

## 2.17.5. `for` / `Stream` integration

`Receiver<T>` implements `IntoIterator<Item = T>`, so it can be used directly in a `for` statement.

```rok
for v in rx {
    use(v);                  // repeats until the channel is closed
}
```

It also implements the asynchronous iterator `Stream<Item = T>`, so `for await` works the same way.

```rok
for await v in rx {
    await handle(v);
}
```

See [2.7 iter](207-iter.md) and [2.8 stream](208-stream.md) for details.

## 2.17.6. Module mapping

Where each type and function is implemented:

| Type           | Module                         | Internal structure       |
|----------------|--------------------------------|--------------------------|
| `Sender<T>`    | `rok::core::channel::sender`   | atomic ref count + queue |
| `Receiver<T>`  | `rok::core::channel::receiver` | queue + parking          |
| `channel<T>()` | `rok::core::channel`           | Arc-like internal state  |

For the internal queue, bounded channels use a ring buffer similar to Tokio mpsc, and unbounded channels use a segmented linked list. A bounded channel's queue is lock-free; an unbounded one's segment list may use a lock.
