# 7.7. I/O 드라이버와 타이머

[스케줄러](706-scheduler.md) 의 worker 가 I/O 대기 task 와 타이머 대기 task 를 효율적으로 처리하기 위해 두 가지 메커니즘을 사용합니다.

## 7.7.1. I/O 멀티플렉싱

프로세스마다 **reactor thread 하나** 가 OS 의 readiness 멀티플렉서에서 기다리다가, 준비된 fd 를 기다리던 task 를 깨웁니다.

| OS / 환경             | 메커니즘                            | 상태 |
|-----------------------|-------------------------------------|------|
| Linux                 | epoll (`EPOLLET`, edge-triggered)   | 제공 |
| BSD (FreeBSD / macOS) | kqueue (`EV_CLEAR`, edge-triggered) | 제공 |
| Linux (고성능)        | io_uring                            | 향후 |
| Windows               | IOCP                                | 향후 |
| WASI                  | poll_oneoff                         | 향후 |

빌드 시점에 타깃에 맞는 driver 가 컴파일되며, 이는 [조건부 컴파일](../006-meta-system/603-cfg.md) 의 `target_os` 분기를 통해 이루어집니다.

reactor thread 는 첫 async I/O 대기 때 시작합니다. 구현은 최적화로 worker 마다 reactor 를 둘 수 있습니다 (7.7.11). 어느 쪽이든 관찰되는 의미는 같습니다.

## 7.7.2. I/O 대기 패턴

I/O 대기는 다음과 같은 흐름으로 진행됩니다.

```
1. task 가 non-blocking 소켓에 read 를 시도 → WouldBlock.
2. task 가 await task::readable(fd) — fd 를 reactor 에 (다시) 등록하고 park.
3. worker 는 다른 task 를 실행.
4. reactor 가 fd 의 readiness edge 를 받으면 그 fd 에 park 한 task 를 unpark.
5. 어떤 worker 가 그 task 를 꺼내 read 를 다시 시도.
```

이 흐름은 `async` `fn` 의 `await` 와 결합됩니다 ([비동기](../004-functions/412-async.md) 참고). 등록은 edge-triggered 이므로 대기마다 다시 겁니다. `task::readable(fd)` / `task::writable(fd)` future 는 park 한 뒤 fd 를 한 번 더 확인해 그사이 지나간 edge 를 놓치지 않습니다. 깨어났는데 아직 준비되지 않았을 수 있으므로 호출자는 연산을 다시 시도합니다.

```rok
fn readable(fd: i32) -> IoReady;     // rok::core::task — Future<Output = ()>
fn writable(fd: i32) -> IoReady;
```

blocking 호출 (`TcpStream::read` 등) 은 이 경로를 쓰지 않고 호출한 OS 스레드를 막습니다. task 를 중단하는 I/O 는 표준 라이브러리의 `*_async` 메서드입니다 ([stdlib 3.2.0](../../refs-standard-libraries/300-std/302-net.md#320-공통-규칙)).

일반 파일은 readiness 모델로 기다릴 수 없습니다 (epoll / kqueue 에서 늘 준비됨). 그래서 일반 파일의 async I/O (`fs::*_async`, [stdlib 3.1.4a](../../refs-standard-libraries/300-std/301-fs.md#314a-비동기-파일-io-_async)) 는 blocking syscall 을 다른 OS 스레드에서 실행하고 (`task::spawn_blocking`) 그동안 task 를 중단합니다. io_uring 이 들어오면 그 위로 옮길 수 있습니다.

## 7.7.3. io_uring / IOCP

io_uring 과 IOCP 는 completion 모델 backend 로, 향후 과제입니다. io_uring 은 kernel ring buffer 와 batch submit 으로 syscall 수를 줄이며, 일반 파일 I/O 도 비동기로 처리할 수 있습니다. 들어오면 가용한 환경에서 우선 사용하고 epoll 로 fallback 합니다.

## 7.7.4. I/O Driver 추상화

각 driver 는 공통된 인터페이스를 구현합니다.

```
trait IODriver {
    fn register(ref self, fd: Fd, interest: IoInterest) throws (IoError);   // edge-triggered
    fn poll(ref self, timeout: Duration) -> List<Event>;
}
```

위 인터페이스는 개념적인 모습을 나타낸 것이며, 정확한 시그니처는 런타임 구현이 결정합니다. driver 는 waker 를 들지 않습니다. 깨울 task 는 fd 와 방향 (읽기 / 쓰기) 으로 정한 parking key 로 `rok::core::task` 의 parking 표에서 찾습니다. 런타임 심볼은 [8.5.5](../008-abi/805-extern-symbols.md#855-io--timer-영역) 를 참조합니다.

## 7.7.5. 타이머 구조

`task::sleep(d)` 나 `task::timeout(d, ...)` 와 같은 시간 대기를 담는 자료구조는 구현이 정합니다. 마감 시각 순의 binary heap (`O(log n)` insert) 도, 아래의 timing wheel 도 됩니다. 관찰되는 의미는 "마감 시각이 지난 뒤 깨운다" 하나입니다.

권장 설계는 Hashed Hierarchical Timing Wheel 입니다.

```
Timing Wheel:
  - 여러 단계의 wheel (밀리초 → 초 → 분 → ...).
  - 각 단계는 *고정 슬롯* 의 ring buffer.
  - tick 마다 현재 슬롯의 task 들을 처리.
  - 큰 단계는 — 더 큰 시간 단위, 슬롯 수 적음.
```

이 자료구조는 `O(1)` insert 와 `O(1)` cancel 을 제공하므로 대량의 task 를 처리하기에 효율적입니다.

성능 목표는 약 16M task 까지 동시에 처리할 수 있는 수준입니다.

## 7.7.6. 타이머 정밀도

기본 tick 단위는 1 ms (밀리초) 입니다. 마이크로초 수준의 더 정밀한 시간이 필요한 경우에는 busy-wait 이나 hardware timer 와 같은 별도 메커니즘을 라이브러리가 제공해 줍니다.

밀리초 단위는 대부분의 `async` 코드, 타임아웃, sleep 처리에 충분한 정밀도를 제공합니다.

## 7.7.7. 타이머와 I/O 의 통합

worker 의 main `loop` 는 다음과 같습니다. I/O readiness 는 reactor thread 가 parking 표를 거쳐 task 를 큐에 되돌려 놓으므로, worker 는 driver 를 직접 poll 하지 않습니다.

```rok
loop {
    // 1. 큐에서 ready task 처리 (reactor 가 깨운 task 포함)
    while let task: Task = queue.next_ready() {        // none 이면 종료
        execute(task);
    }

    // 2. 시간 지난 timer 의 task 들 큐에 복귀
    timers.fire_due();

    // 3. 할 일이 없으면 다음 timer 까지 park (unpark 로 깨어남)
    park_until(timers.next_deadline());
}
```

reactor thread 의 `loop` 는 다음과 같습니다.

```rok
loop {
    let events = io_driver.poll();                     // 막힘 — GC 에는 native 구간
    for event in events {
        task::unpark(parking_key(event.fd, event.direction));
    }
}
```

## 7.7.8. Park / Unpark

worker 의 큐가 완전히 비어 있고 I/O 도 idle 상태일 때에는 worker 를 park 시켜 OS sleep 상태로 둡니다.

- park 는 `pthread_cond_wait` 또는 OS 별 sleep 메커니즘을 사용합니다.
- unpark 는 다른 worker 나 I/O 이벤트가 깨워 줍니다.

park 가 진행되는 동안 CPU 사용량은 0 이므로, idle worker 가 많아지더라도 추가 비용이 발생하지 않습니다.

## 7.7.9. Worker 수의 동적 조절

기본값은 N = CPU 코어 수로 고정되어 있습니다.

조정 가능한 옵션은 다음과 같습니다.

- blocking syscall 이 검출되면 임시로 worker 를 추가합니다 (해당 worker 가 blocking 상태인 동안에만 유지됩니다).
- 과도하게 idle 한 상태에서는 worker 하나를 일시적으로 park 하고 다른 worker 가 더 많은 일을 처리하도록 합니다.

이러한 동적 조절은 런타임 설정 사항이며, 기본은 off 상태입니다. 기능을 켜면 자동으로 작동합니다.

## 7.7.10. Cancellation

`Task::cancel` 은 취소를 **요청** 합니다 ([stdlib 2.18.4](../../refs-standard-libraries/200-core/218-task.md#2184-협력적-취소--taskcancel)). 런타임은 플래그를 세우고 task 를 깨웁니다.

- 모든 `await` 가 cancellation point 입니다. `await` 는 들어갈 때, 결과를 받은 뒤, 중단하기 직전에 플래그를 봅니다.
- 취소된 task 는 다음 `await` 에서 `return` 처럼 본문을 떠납니다. 지나는 `finally` 블록과 `using` 의 `drop` 이 안쪽부터 실행됩니다. 중첩된 `async fn` 호출은 가장 안쪽 코루틴부터 정리됩니다.
- 이것은 task 안에서 `catch` 할 수 있는 에러가 아닙니다. throwing 이 아닌 async 본문도 같은 경로로 끝납니다. 결과는 버려집니다.
- blocking syscall (`TcpStream::read` 등) 은 cancellation point 가 아닙니다. 끝난 뒤 다음 `await` 에서 떠납니다.

`task::timeout` 이 시간을 넘기거나 `task::race` 가 승자를 정한 뒤에도, 진 쪽 task 는 계속 실행됩니다. 멈추려면 `cancel` 을 부릅니다.

## 7.7.11. Reactor 패턴과의 비교

rok 은 전통적인 reactor 구조를 씁니다. 별도의 reactor thread 하나가 I/O 를 기다리고, worker thread 들과는 parking 표를 통해서만 만납니다. 그 대가로 I/O 깨움이 thread 를 건넙니다 (unpark 한 번).

구현은 최적화로 각 worker 가 자기 reactor 를 겸하게 할 수 있습니다. 그러면 thread 사이의 깨움 비용이 줄고 cache locality 가 좋아지지만, 모든 worker 가 driver 를 부를 수 있어야 하므로 driver 가 thread-safe 해야 합니다. 어느 구조든 7.7.2 의 의미 (edge 를 놓치지 않음, 가짜 깨움 가능) 는 같습니다.
