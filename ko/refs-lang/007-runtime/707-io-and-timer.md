# 7.7. I/O 드라이버와 타이머

[스케줄러](706-scheduler.md) 의 worker 가 I/O 대기 task 와 타이머 대기 task 를 효율적으로 처리하기 위해 두 가지 메커니즘을 사용합니다.

## 7.7.1. I/O 멀티플렉싱

각 worker 는 I/O driver handle 을 가지며, OS 가 제공하는 멀티플렉싱 메커니즘을 활용합니다.

| OS / 환경             | 메커니즘                    |
|-----------------------|-----------------------------|
| Linux (표준)          | epoll                       |
| Linux (고성능)        | io_uring                    |
| BSD (FreeBSD / macOS) | kqueue                      |
| Windows               | IOCP (wepoll 으로 wsa 통일) |
| WASI                  | poll_oneoff                 |

빌드 시점에 타깃에 맞는 driver 가 컴파일되며, 이는 [조건부 컴파일](../006-meta-system/603-cfg.md) 의 `target_os` 분기를 통해 이루어집니다.

## 7.7.2. I/O 대기 패턴

I/O 대기는 다음과 같은 흐름으로 진행됩니다.

```
1. task 가 read(fd) 호출.
2. 데이터 없으면 — worker 가 그 task 를 *I/O wait queue* 로 보냄.
3. driver 가 epoll/iouring/kqueue/IOCP 에 *fd 등록*.
4. task 는 일시 중단 — worker 는 다른 task 진행.
5. fd 가 readable 되면 — driver 가 task 를 *task queue 로 복귀*.
6. 어떤 worker 가 그 task 를 꺼내 — read 재개.
```

이 흐름은 `async` `fn` 의 `await` 와 자연스럽게 결합됩니다 ([비동기](../004-functions/412-async.md) 참고).

## 7.7.3. io_uring 활용

Linux 환경에서는 io_uring 이 epoll 보다 더 빠른 성능을 보여 주는데, 특히 디스크 I/O 가 많거나 connection 수가 많은 경우에 그 차이가 두드러집니다.

- kernel ring buffer 를 사용해 syscall 횟수를 줄여 줍니다.
- batch submit 으로 여러 I/O 요청을 한 번에 보낼 수 있습니다.
- 일부 패턴에서는 zero-copy 도 가능합니다.

스케줄러는 io_uring 이 가용한 경우 이를 우선 사용하며, 사용할 수 없을 때에는 epoll 로 fallback 합니다.

## 7.7.4. I/O Driver 추상화

각 driver 는 공통된 인터페이스를 구현합니다.

```
trait IODriver {
    fn register(ref self, fd: Fd, interest: IoInterest, waker: Waker) throws (IoError);
    fn deregister(ref self, fd: Fd) throws (IoError);
    fn poll(ref self, timeout: Duration) -> List<Event>;
}
```

위 인터페이스는 개념적인 모습을 나타낸 것이며, 정확한 시그니처는 스케줄러 구현이 결정합니다. 본 사양은 공통 인터페이스가 존재한다는 사실까지만 책임을 집니다.

## 7.7.5. 타이머: Hashed Hierarchical Timing Wheel

`task::sleep(d)` 나 `task::timeout(d, ...)` 와 같은 시간 대기는 timing wheel 자료구조로 처리됩니다.

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

worker 의 main `loop` 는 다음과 같습니다.

```rok
loop {
    // 1. 큐에서 ready task 처리
    while let task: Task = queue.next_ready() {        // none 이면 종료
        execute(task);
    }

    // 2. timeout 까지 I/O 폴링 — timing wheel 의 다음 tick 까지
    let timeout = wheel.next_tick_in();
    let events = io_driver.poll(timeout);

    // 3. I/O ready 된 task 들 큐에 복귀
    for event in events {
        queue.push(event.waker.task);
    }

    // 4. wheel tick — 시간 지난 task 들 큐에 복귀
    wheel.tick();
}
```

이 통합 `loop` 가 한 worker 안에서 task, I/O, timer 를 모두 처리하는 핵심이 됩니다.

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

[비동기](../004-functions/412-async.md) 의 `task::timeout` 이나 `task::race`, 또는 사용자가 명시적으로 cancel 을 요청하면 해당 task 의 Future 가 drop 됩니다.

drop 시점에는 진행 중이던 `await` 위치에서 정리 코드가 실행되며, `finally` 블록이나 drop guard 가 정의되어 있다면 그 또한 함께 수행됩니다.

자세한 cancellation 메커니즘은 스케줄러와 코루틴 구현이 결정합니다.

## 7.7.11. Reactor 패턴과의 비교

전통적인 reactor 는 별도의 reactor thread 가 I/O 를 관리하며, worker thread 와는 분리된 구조를 가집니다.

반면 rok 의 통합 worker 모델에서는 각 worker 가 곧 자기 자신의 reactor 역할을 합니다. 그 결과 thread 사이의 unpark 나 wakeup 비용이 발생하지 않으며 cache locality 도 더 우수합니다.

대신 모든 worker 가 I/O driver 를 호출할 수 있어야 하므로, driver 자체는 thread-safe 하게 구현되어야 합니다.
