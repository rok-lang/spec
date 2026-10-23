# 7.6. 스케줄러

rok 의 스케줄러는 M:N [work-stealing](../000-meta/007-glossary.md#work-stealing) 방식이며, 본 문서에서는 그 구조를 정의합니다.

## 7.6.1. 통합된 worker 모델

프로세스에는 executor 가 하나 있으며 처음 쓸 때 시작합니다. worker 는 CPU 수만큼의 OS thread 이고, 여기에 blocking lane 과 I/O driver 가 붙습니다. `task::block_on` 은 부른 스레드를 guest worker 로 붙이며, 돌아올 때 남은 task 는 계속 실행됩니다. `par::*` 와 `spawn_blocking` 도 같은 executor 를 씁니다 ([stdlib 2.18.3](../../refs-standard-libraries/200-core/218-task.md#2183-스케줄러-모델---work-stealing)). 각 worker 는 다음 역할을 겸합니다.

```
worker = {
    scheduler   // task 큐 (work-stealing)
    mutator     // TLAB + shadow stack (701 / 702 / 704)
}
```

I/O driver 는 worker 가 겸하지 않습니다. 프로세스당 reactor thread 하나가 맡습니다 ([7.7.1](707-io-and-timer.md#771-io-멀티플렉싱)). STW 수집은 수집을 시작한 thread 가, 동시 mark 는 marker thread 가 진행합니다 (7.6.10).

## 7.6.2. Work-Stealing 구조

각 worker 는 로컬 deque 를 갖습니다 (lock-free Chase-Lev deque, [8.5.4 `rok_deque_*`](../008-abi/805-extern-symbols.md#rok_deque_---work-stealing-deque)).

```
Worker:
  ├── local_deque : growable ring    (this worker's tasks, GC-traced)
  └── ...
```

새로 깨어나거나 spawn 된 task 는 그 worker 의 deque 끝 (bottom) 에 들어갑니다. **소유 worker 도 thief 도 가장 오래된 task (top) 부터** 꺼냅니다. 소유자가 가장 새 task 부터 꺼내면, 스스로를 다시 깨우는 task (`await task::yield_now()` 를 도는 task 등) 가 자기 뒤에 쌓인 task 들을 굶길 수 있기 때문입니다.

이 구조는 자기가 spawn 한 task 를 대개 자기가 실행해 캐시 locality 를 살리는 동시에, 다른 worker 가 일이 없으면 훔쳐 와서 load balancing 까지 얻어내는 work-stealing 모델입니다. LIFO slot 같은 "가장 최근 task 우선" 최적화는, 위의 굶김이 생기지 않는 범위에서 구현이 둘 수 있습니다.

## 7.6.3. Global Injector Queue

각 worker 의 local queue 외에 전역 injector queue 도 별도로 존재합니다.

```
Global injector:
  - receives tasks spawned from outside (e.g. extern callback).
  - when a worker's local queue is empty - pull from global.
```

이 큐는 worker 풀 바깥에서 들어오는 task 의 진입점 역할을 맡고 있습니다.

## 7.6.4. Steal 알고리듬

worker 의 local deque 가 비어 있다면 다음과 같은 순서로 진행됩니다.

```
1. global injector 에서 pull 시도.
2. 다른 worker 의 local deque 의 top 에서 steal 을 시도합니다. 대상은 worker 마다의 의사 난수 index 에서 시작해 round-robin 으로 돕니다.
3. 모두 실패하면 - sleep / park (blocking sync).
```

steal 은 worker 를 최대 N-1 개까지 시도해 보고, 그래도 모두 비어 있으면 park 상태로 전환합니다.

## 7.6.5. 키워드 vs 라이브러리 함수

[키워드](../001-lexical/105-keywords.md) 의 예약 키워드 최소화 원칙에 따라, 스케줄러 함수들은 모두 라이브러리 함수로 제공됩니다.

| 라이브러리 함수         | 의미                                                                     |
|-------------------------|--------------------------------------------------------------------------|
| `task::spawn(fut)`      | future 를 새 task 로 큐에 push. 결과 `Task<T>`.                          |
| `task::yield_now()`     | 자발적 양보 (`await`). 다른 ready task 를 한 번 실행시킴.                |
| `task::sleep(d)`        | timer 큐로 이동.                                                         |
| `task::race([...])`     | 먼저 끝나는 task (`List<Task<T>>`, await).                               |
| `task::join([...])`     | 모두 끝날 때까지 (`List<Task<T>>`, await).                               |
| `task::timeout(d, t)`   | 시간 제한. 초과하면 `try await` 에서 `TimeoutError`.                     |
| `thread::spawn(f)`      | 새 OS thread 만들기 (1:1).                                               |
| `par::for_each(xs, f)`  | 병렬 for-each ([block](../000-meta/007-glossary.md#block-블록) on join). |
| `par::scope { ... }`    | 병렬 스코프 (트레일링 클로저).                                           |

이들이 키워드가 아니므로 사용자 코드에서 `spawn` 이나 `race` 같은 식별자도 자유롭게 쓸 수 있습니다.

## 7.6.6. 1:1 OS Thread: `thread::spawn`

대부분의 task 는 스케줄러가 매핑하는 M:N 방식으로 동작하지만, 블로킹 작업이나 외부 라이브러리 호출 같은 경우에는 전용 OS thread 가 자연스러울 때가 있습니다. 이때는 `thread::spawn` 을 통해 진짜 OS thread 를 만들어 1:1 로 매핑할 수 있습니다.

내부적으로는 Unix 의 `pthread_create` 또는 Windows 의 `CreateThread` 등을 직접 호출하며, 일반 task 와는 분리된 풀에서 작동합니다. 결과는 Future 와는 별개의 타입인 `JoinHandle<T>` 로 반환됩니다.

## 7.6.7. Blocking Syscall 자동 검출

work-stealing 모델은 task 가 blocking syscall 을 호출하면 그 worker 자체가 멈춰 버린다는 문제를 안고 있습니다. 이를 해결하기 위해 blocking pool 자동 오프로드를 사용합니다.

컴파일러와 런타임이 알려진 blocking syscall 을 blocking pool worker 로 자동으로 보내고, 그 worker 는 별도의 OS thread 이기 때문에 다른 task 의 진행을 방해하지 않습니다. 자세한 메커니즘은 스케줄러 구현이 결정하도록 두었습니다.

> [!NOTE]
> 참조 런타임에서는 task 가 blocking 호출에 머무는 pool thread 가 자기 worker 를 새 thread 에 넘깁니다. `spawn_local` 로 띄운 task 는 worker 를 고정하며, 그 worker 는 넘겨지지 않습니다.

## 7.6.8. Thread-per-core 옵션

기본 동작은 N 개의 worker 를 두고 task 가 thread 사이를 자유롭게 이동하도록 하는 형태입니다. 다만 thread-per-core 옵션을 켜면 각 task 가 고정된 worker 에 pinning 되고, worker 끼리 steal 하지 않으며, 그 결과 cache locality 가 극대화되어 NUMA 환경이나 데이터센터 워크로드에 적합한 동작을 얻을 수 있습니다.

환경 변수 `ROK_THREAD_PER_CORE=1` 로 켜며, `ROK_WORKERS=<n>` 이 N 을 정합니다 (기본값: 온라인 CPU 수).

> [!NOTE]
> task 는 worker 에 고정됩니다. task 가 block 된 worker 는 다른 OS 스레드에서 이어질 수 있으므로 ([7.6.7](#767-blocking-syscall-자동-검출)), 고정된 task 에서도 스레드 정체성은 안정적이지 않습니다.

## 7.6.9. 우선순위 / 공정성

기본은 우선순위 없는 FIFO + work-stealing 입니다. 소유 worker 와 thief 모두 deque 에서 가장 오래된 task 부터 처리합니다 (7.6.2).

바쁜 worker 도 timer 를 미루지 않습니다. task 를 32 개 실행할 때마다 기한이 지난 timer 를 처리해 그 task 를 큐에 되돌립니다. 그래서 늘 일이 있는 worker 위에서도 `task::sleep` / `task::timeout` 이 제때 깨어납니다.

명시적 우선순위 지정은 본 사양의 범위 밖이며, 표준 라이브러리의 별도 표면에서 다룹니다.

## 7.6.10. 통합 GC Worker 역할

STW 수집은 할당이나 `rok_gc_collect` 에서 수집을 시작한 thread 가 진행하고, 다른 worker 는 safepoint 에서 멈춰 기다립니다 ([7.4.3](704-barriers-and-roots.md#743-safepoint-polling)). 수집이 끝나면 worker 는 task 실행으로 돌아갑니다. 동시 mark 는 별도의 marker thread 하나가 합니다 ([7.5](705-concurrent-collection.md)).

## 7.6.11. Scheduler 와 Coroutine 의 관계

스케줄러는 [코루틴](708-coroutine.md) 의 stackless coroutine 을 사용하며, 각 task 의 상태 머신이 coroutine frame 에 저장되는 형태입니다.

task 가 `await` 또는 `yield` 를 만나면 coroutine frame 만 큐에 다시 들어가고 worker 는 다른 task 를 이어서 진행하게 되며, 해당 task 가 재개될 때는 다른 worker 가 그 frame 을 꺼내어 이어서 실행할 수도 있습니다. 이러한 모델 덕분에 수백만 개의 task 가 N 개의 worker 위에서 효율적으로 처리될 수 있는 것입니다.

## 7.6.12. Spawn 비용

`task::spawn` 은 stack 없는 coroutine frame 을 할당하는 것만으로 동작이 완료됩니다. frame 의 크기는 task 의 지역 변수에 따라 달라지지만 대략 64 ~ 256 바이트 수준이고, [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 의 fast path 에서 할당되기 때문에 그 비용이 거의 0 에 가깝습니다.

이 모델 덕분에 수백만 개의 task 를 동시에 진행하더라도 OS thread 와는 비교할 수 없을 만큼 가벼운 동작을 얻을 수 있습니다.

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
