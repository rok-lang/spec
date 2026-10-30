# 7.6. 스케줄러

Rok의 스케줄러는 M:N [work-stealing](../000-meta/007-glossary.md#work-stealing) 방식입니다. 이 문서는 그 구조를 정의합니다.

## 7.6.1. 통합된 worker 모델

프로세스에는 executor가 하나 있으며, 처음 쓸 때 시작합니다. worker는 CPU 수만큼의 OS 스레드이고, 여기에 blocking lane과 I/O driver가 더해집니다. `task::block_on`은 호출한 스레드를 guest worker로 붙이며, 이 함수가 반환된 뒤에도 남은 task는 계속 실행됩니다. `par::*`와 `spawn_blocking`도 같은 executor를 씁니다([stdlib 2.18.3](../../refs-standard-libraries/200-core/218-task.md#2183-스케줄러-모델---work-stealing)). 각 worker는 다음 역할을 함께 맡습니다.

```
worker = {
    scheduler   // task 큐 (work-stealing)
    mutator     // TLAB + shadow stack (701 / 702 / 704)
}
```

I/O driver는 worker가 겸하지 않고, 프로세스마다 하나 있는 reactor 스레드가 맡습니다([7.7.1](707-io-and-timer.md#771-io-멀티플렉싱)). STW 수집은 수집을 시작한 스레드가, 동시 mark는 marker 스레드가 진행합니다(7.6.10).

## 7.6.2. Work-Stealing 구조

각 worker는 로컬 deque를 가집니다(lock-free Chase-Lev deque, [8.5.4 `rok_deque_*`](../008-abi/805-extern-symbols.md#rok_deque_---work-stealing-deque)).

```
Worker:
  ├── local_deque : growable ring    (this worker's tasks, GC-traced)
  └── ...
```

새로 깨어나거나 spawn된 task는 그 worker deque의 끝(bottom)에 들어갑니다. **소유 worker도 thief도 가장 오래된 task(top)부터** 꺼냅니다. 소유자가 가장 새 task부터 꺼내면, 스스로를 계속 다시 깨우는 task(`await task::yield_now()`를 반복하는 task 등)가 자기 뒤에 쌓인 task를 굶길 수 있기 때문입니다.

이 work-stealing 모델에서 worker는 대개 자기가 spawn한 task를 직접 실행하므로 캐시 locality가 유지됩니다. 일이 없는 worker는 다른 worker의 task를 훔쳐 와 load balancing이 이루어집니다. 구현은 위의 굶김이 생기지 않는 범위에서 LIFO slot 같은 "가장 최근 task 우선" 최적화를 둘 수 있습니다.

## 7.6.3. Global Injector Queue

worker마다 있는 local queue와 별도로, 전역 injector queue가 하나 있습니다.

```
Global injector:
  - receives tasks spawned from outside (e.g. extern callback).
  - when a worker's local queue is empty, pull from global.
```

이 큐는 worker 풀 바깥에서 들어오는 task의 진입점입니다.

## 7.6.4. Steal 알고리듬

worker의 local deque가 비어 있으면 다음 순서로 진행합니다.

```
1. global injector 에서 pull 시도.
2. 다른 worker 의 local deque 의 top 에서 steal 을 시도합니다. 대상은 worker 마다의 의사 난수 index 에서 시작해 round-robin 으로 돕니다.
3. 모두 실패하면 sleep / park (blocking sync).
```

steal은 최대 N-1개의 worker를 시도합니다. 모두 비어 있으면 worker는 park 상태로 바뀝니다.

## 7.6.5. 키워드 vs 라이브러리 함수

[키워드](../001-lexical/105-keywords.md)에서 정한 예약 키워드 최소화 원칙에 따라, 스케줄러 함수는 모두 라이브러리 함수입니다.

| 라이브러리 함수         | 의미                                                                     |
|-------------------------|--------------------------------------------------------------------------|
| `task::spawn(fut)`      | future를 새 task로 큐에 push. 결과 `Task<T>`.                            |
| `task::yield_now()`     | 자발적 양보 (`await`). 다른 ready task를 한 번 실행시킴.                 |
| `task::sleep(d)`        | timer 큐로 이동.                                                         |
| `task::race([...])`     | 먼저 끝나는 task (`List<Task<T>>`, await).                               |
| `task::join([...])`     | 모두 끝날 때까지 (`List<Task<T>>`, await).                               |
| `task::timeout(d, t)`   | 시간 제한. 초과하면 `try await`에서 `TimeoutError`.                      |
| `thread::spawn(f)`      | 새 OS 스레드 만들기 (1:1).                                               |
| `par::for_each(xs, f)`  | 병렬 for-each ([block](../000-meta/007-glossary.md#block-블록) on join). |
| `par::scope { ... }`    | 병렬 스코프 (트레일링 클로저).                                           |

이 함수들은 키워드가 아니므로 사용자 코드에서 `spawn`이나 `race` 같은 식별자를 자유롭게 쓸 수 있습니다.

## 7.6.6. 1:1 OS Thread: `thread::spawn`

대부분의 task는 스케줄러가 매핑하는 M:N 방식으로 동작합니다. 하지만 블로킹 작업이나 외부 라이브러리 호출에는 전용 OS 스레드가 더 맞을 때가 있습니다. 이때는 `thread::spawn`으로 실제 OS 스레드를 만들어 1:1로 매핑합니다.

내부에서는 Unix의 `pthread_create`나 Windows의 `CreateThread` 등을 직접 호출하며, 일반 task와 분리된 풀에서 동작합니다. 결과는 Future와 다른 타입인 `JoinHandle<T>`로 반환됩니다.

## 7.6.7. Blocking Syscall 자동 검출

work-stealing 모델에서는 task가 blocking syscall을 호출하면 그 worker 자체가 멈춥니다. 이 문제는 blocking pool로 자동 오프로드해서 해결합니다.

컴파일러와 런타임은 알려진 blocking syscall을 blocking pool worker로 자동으로 보냅니다. 이 worker는 별도의 OS 스레드이므로 다른 task의 진행을 막지 않습니다. 자세한 메커니즘은 스케줄러 구현에 맡깁니다.

> [!NOTE]
> 참조 런타임에서는, task가 blocking 호출에 머물러 있는 pool 스레드가 자기 worker를 새 스레드에 넘깁니다. `spawn_local`로 띄운 task는 자기 worker를 고정하며, 그 worker는 다른 스레드로 넘겨지지 않습니다.

## 7.6.8. Thread-per-core 옵션

기본적으로는 worker를 N개 두고, task가 스레드 사이를 자유롭게 옮겨 다닙니다. thread-per-core 옵션을 켜면 각 task가 정해진 worker에 고정(pinning)되고 worker끼리 steal하지 않습니다. 그 결과 cache locality가 극대화되어 NUMA 환경이나 데이터센터 워크로드에 알맞습니다.

환경 변수 `ROK_THREAD_PER_CORE=1`로 켜고, `ROK_WORKERS=<n>`으로 N을 정합니다(기본값: 온라인 CPU 수).

> [!NOTE]
> task는 worker에 고정됩니다. 하지만 task가 block된 worker는 다른 OS 스레드에서 이어질 수 있으므로([7.6.7](#767-blocking-syscall-자동-검출)), 고정된 task라도 스레드 정체성은 안정적이지 않습니다.

## 7.6.9. 우선순위 / 공정성

기본은 우선순위 없는 FIFO + work-stealing입니다. 소유 worker와 thief 모두 deque에서 가장 오래된 task부터 처리합니다(7.6.2).

바쁜 worker도 timer를 미루지 않습니다. task를 32개 실행할 때마다 기한이 지난 timer를 처리해 해당 task를 큐에 되돌립니다. 그래서 늘 일이 있는 worker에서도 `task::sleep` / `task::timeout`이 제때 깨어납니다.

명시적 우선순위 지정은 이 사양의 범위 밖이며, 표준 라이브러리의 별도 표면에서 다룹니다.

## 7.6.10. 통합 GC Worker 역할

STW 수집은 할당이나 `rok_gc_collect` 호출로 수집을 시작한 스레드가 진행하고, 다른 worker는 safepoint에서 멈춰 기다립니다([7.4.3](704-barriers-and-roots.md#743-safepoint-polling)). 수집이 끝나면 worker는 다시 task를 실행합니다. 동시 mark는 별도의 marker 스레드 하나가 맡습니다([7.5](705-concurrent-collection.md)).

## 7.6.11. Scheduler 와 Coroutine 의 관계

스케줄러는 [코루틴](708-coroutine.md)에서 정의한 stackless coroutine을 씁니다. 각 task의 상태 기계는 coroutine frame에 저장됩니다.

task가 `await`나 `yield`를 만나면 coroutine frame만 큐에 다시 들어가고, worker는 다른 task를 실행합니다. task가 재개될 때는 다른 worker가 그 frame을 꺼내 이어서 실행할 수도 있습니다. 이 모델 덕분에 task 수백만 개를 worker N개로 처리할 수 있습니다.

## 7.6.12. Spawn 비용

`task::spawn`은 스택 없는 coroutine frame 하나를 할당하면 끝납니다. frame 크기는 task의 지역 변수에 따라 다르지만 대략 64~256바이트이고, [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer)의 fast path에서 할당하므로 비용은 거의 0입니다.

그래서 task 수백만 개를 동시에 진행해도 OS 스레드와 비교할 수 없을 만큼 가볍습니다.

## 7.6.13. 전체 흐름 요약

```
user code:    task::spawn(async { compute() })
  ↓
compiler:     create coroutine frame + push to queue
  ↓
worker:       pop task from deque/lifo and execute
  ↓ (on await / yield)
worker:       requeue frame, run another task
  ↓ (some worker)
resume:       pop frame and continue execution
  ↓ (on normal termination)
result:       delivered as Output of Task<T> (Task<T>: Future<Output = T>)
```
