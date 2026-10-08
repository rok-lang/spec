# 7.7. I/O 드라이버와 타이머

[스케줄러](706-scheduler.md) 의 worker 가 I/O 대기 task 와 타이머 대기 task 를 효율적으로 처리하기 위한 두 가지 메커니즘.

## 7.7.1. I/O 멀티플렉싱

각 worker 는 I/O driver handle 을 가지고 OS 의 멀티플렉싱 메커니즘을 사용한다.

| OS / 환경             | 메커니즘                    |
| --------------------- | --------------------------- |
| Linux (표준)          | epoll                       |
| Linux (고성능)        | io_uring                    |
| BSD (FreeBSD / macOS) | kqueue                      |
| Windows               | IOCP (wepoll 으로 wsa 통일) |
| WASI                  | poll_oneoff                 |

빌드 시점에 타깃에 맞는 driver 가 컴파일된다 ([조건부 컴파일](../006-meta-system/603-cfg.md) 의 `target_os` 분기).

## 7.7.2. I/O 대기 패턴

```
1. task 가 read(fd) 호출.
2. 데이터 없으면 — worker 가 그 task 를 *I/O wait queue* 로 보냄.
3. driver 가 epoll/iouring/kqueue/IOCP 에 *fd 등록*.
4. task 는 일시 중단 — worker 는 다른 task 진행.
5. fd 가 readable 되면 — driver 가 task 를 *task queue 로 복귀*.
6. 어떤 worker 가 그 task 를 꺼내 — read 재개.
```

이 흐름은 `async` `fn` 의 `await` 와 자연스럽게 결합된다 ([비동기](../004-functions/412-async.md)).

## 7.7.3. io_uring 활용

Linux 에서는 io_uring 이 epoll 보다 빠르다 (특히 디스크 I/O 나 connection 이 많은 경우).

- kernel ring buffer 로 syscall 횟수가 줄어든다.
- batch submit. 여러 I/O 를 한 번에 보낸다.
- zero-copy. 일부 패턴에서 가능.

스케줄러는 io_uring 이 가용하면 우선 사용한다. 없으면 epoll 로 fallback 한다.

## 7.7.4. I/O Driver 추상화

각 driver 는 공통 인터페이스를 구현한다.

```
trait IODriver {
    fn register(ref self, fd: Fd, interest: IoInterest, waker: Waker) throws (IoError);
    fn deregister(ref self, fd: Fd) throws (IoError);
    fn poll(ref self, timeout: Duration) -> List<Event>;
}
```

위는 개념적 모습이다. 정확한 시그니처는 스케줄러 구현이 결정한다. 본 사양은 공통 인터페이스가 있다는 사실까지만 책임진다.

## 7.7.5. 타이머: Hashed Hierarchical Timing Wheel

`task::sleep(d)` / `task::timeout(d, ...)` 같은 시간 대기는 timing wheel 자료구조로 처리된다.

```
Timing Wheel:
  - 여러 단계의 wheel (밀리초 → 초 → 분 → ...).
  - 각 단계는 *고정 슬롯* 의 ring buffer.
  - tick 마다 현재 슬롯의 task 들을 처리.
  - 큰 단계는 — 더 큰 시간 단위, 슬롯 수 적음.
```

이 자료구조는 `O(1)` insert, `O(1)` cancel 이며 대량 task 처리에 효율적이다.

성능 목표: 동시 처리 ~ 16M task 가능.

## 7.7.6. 타이머 정밀도

기본 tick 단위는 1 ms (밀리초) 다. 더 정밀한 시간 (마이크로초) 이 필요하면 별도 메커니즘 (busy-wait, hardware timer 등) 을 라이브러리가 제공한다.

밀리초 단위는 대부분의 `async` / 타임아웃 / sleep 에 충분하다.

## 7.7.7. 타이머와 I/O 의 통합

worker 의 main `loop`:

```nok
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

이 통합 `loop` 가 한 worker 에서 task / I/O / timer 를 모두 처리하는 핵심이다.

## 7.7.8. Park / Unpark

worker 의 큐가 완전히 비고 I/O 도 idle 이면 worker 를 park (OS sleep) 시킨다.

- park: `pthread_cond_wait` 또는 OS 별 sleep 메커니즘.
- unpark: 다른 worker 나 I/O 이벤트가 깨운다.

park 동안 CPU 사용은 0 이다. idle worker 가 많아도 비용이 없다.

## 7.7.9. Worker 수의 동적 조절

기본은 N = CPU 코어 수로 고정이다.

조정 가능한 옵션:

- blocking syscall 검출 시 임시 worker 추가 (그 worker 가 blocking 인 동안).
- 과도한 idle 시 worker 하나를 일시 park 하고 다른 worker 가 더 처리.

이 동적 조절은 런타임 설정 사항이다. 기본은 off, 켜면 자동으로 작동한다.

## 7.7.10. Cancellation

[비동기](../004-functions/412-async.md) 의 `task::timeout` / `task::race` 또는 사용자가 명시적으로 cancel 하면 그 task 의 Future 가 drop 된다.

drop 은 진행 중인 `await` 위치에서 정리 코드를 실행한다 (`finally` 블록이나 drop guard 가 있다면).

자세한 cancellation 메커니즘은 스케줄러 / 코루틴 구현이 결정한다.

## 7.7.11. Reactor 패턴과의 비교

전통적 reactor 는 별도의 reactor thread 가 I/O 를 관리하며, worker thread 와 분리된다.

nok 의 통합 worker 모델에서는 각 worker 가 곧 자기 자신의 reactor 다. thread 사이의 unpark / wakeup 비용이 없고 cache locality 가 좋다.

대신 모든 worker 가 I/O driver 를 호출할 수 있어야 한다 (driver 자체는 thread-safe).
