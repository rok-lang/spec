# 7.6. 스케줄러

rok 의 스케줄러는 M:N [work-stealing](../000-meta/007-glossary.md#work-stealing) 방식이며, 본 문서에서는 그 구조를 정의합니다.

## 7.6.1. 통합된 worker 모델

기본은 N 개의 OS thread 로 구성되며 보통 CPU 코어 수에 맞추고, 각 thread 는 다음 네 가지 역할을 동시에 수행합니다.

```
OS Thread = {
    Scheduler Worker      // task queue (work-stealing)
    Mutator Context       // TLAB + shadow stack (see 701/702/704)
    I/O Driver Handle     // poll / epoll / iouring / kqueue / IOCP / wasi
    GC Worker Context     // participates in parallel mark / evacuate
}
```

> see 701/702/704: [객체 레이아웃과 메모리 공간](701-object-layout.md) / [할당](702-allocation.md) / [배리어와 루트 추적](704-barriers-and-roots.md). participates in parallel mark / evacuate: parallel mark / evacuate 참여.

이렇게 역할을 통합한 덕분에 추가로 thread 를 분리하지 않고도 모든 런타임 활동이 하나의 풀에서 처리될 수 있습니다.

## 7.6.2. Work-Stealing 구조

각 worker 는 로컬 deque 와 LIFO slot 을 함께 갖고 있습니다.

```
Worker:
  ├── lifo_slot   : 1-task buffer    (most recently spawned task)
  ├── local_deque : 64-task ring     (this worker's tasks)
  └── ...
```

> most recently spawned task: 가장 최근 spawn 한 task. this worker's tasks: 자신의 task 들.

task 가 새로 생성되면 먼저 LIFO slot 에 들어가고, 이미 차 있다면 local_deque 의 bottom 으로 옮겨집니다. task 를 실행할 때는 local_deque 의 bottom 에서 꺼내어 LIFO 순서로 진행하며, steal 이 일어날 때는 다른 worker 의 local_deque 의 top 즉 FIFO 쪽에서 가져옵니다.

이 구조는 자기가 spawn 한 task 를 자기가 실행해 캐시 locality 를 살리는 동시에, 다른 worker 가 일이 없으면 훔쳐 와서 load balancing 까지 얻어내는 표준적인 work-stealing 모델입니다.

## 7.6.3. Global Injector Queue

각 worker 의 local queue 외에 전역 injector queue 도 별도로 존재합니다.

```
Global injector:
  - receives tasks spawned from outside (e.g. extern callback).
  - when a worker's local queue is empty — pull from global.
```

> receives tasks spawned from outside: 외부에서 spawn 된 task 가 들어갑니다. when a worker's local queue is empty: worker 의 local queue 가 비면.

이 큐는 worker 풀 바깥에서 들어오는 task 의 진입점 역할을 맡고 있습니다.

## 7.6.4. Steal 알고리듬

worker 의 local deque 가 비고 LIFO slot 까지 비어 있다면 다음과 같은 순서로 진행됩니다.

```
1. global injector 에서 pull 시도.
2. 다른 worker 의 local deque 의 top 에서 steal 시도 (random round-robin).
3. 모두 실패하면 — sleep / park (blocking sync).
```

steal 은 worker 를 최대 N-1 개까지 시도해 보고, 그래도 모두 비어 있으면 park 상태로 전환합니다.

## 7.6.5. 키워드 vs 라이브러리 함수

[키워드](../001-lexical/105-keywords.md) 의 예약 키워드 최소화 원칙에 따라, 스케줄러 함수들은 모두 라이브러리 함수로 제공됩니다.

| 라이브러리 함수         | 의미                                                                     |
|-------------------------|--------------------------------------------------------------------------|
| `task::spawn(f)`        | 새 task 를 큐에 push. 결과 `Task<T>`.                                    |
| `task::yield_now()`     | 자발적 `yield`. 다른 task 에 양보.                                       |
| `task::sleep(d)`        | timer 큐로 이동.                                                         |
| `task::race(a, b)`      | 둘 중 먼저 끝나는 것.                                                    |
| `task::join_all([...])` | 모두 끝날 때까지.                                                        |
| `task::timeout(d, fut)` | 시간 제한.                                                               |
| `thread::spawn(f)`      | 새 OS thread 만들기 (1:1).                                               |
| `par::for_each(xs, f)`  | 병렬 for-each ([block](../000-meta/007-glossary.md#block-블록) on join). |
| `par::scope { ... }`    | 병렬 스코프 (트레일링 클로저).                                           |

이들이 키워드가 아니므로 사용자 코드에서 `spawn` 이나 `race` 같은 식별자도 자유롭게 쓸 수 있습니다.

## 7.6.6. 1:1 OS Thread: `thread::spawn`

대부분의 task 는 스케줄러가 매핑하는 M:N 방식으로 동작하지만, 블로킹 작업이나 외부 라이브러리 호출 같은 경우에는 전용 OS thread 가 자연스러울 때가 있습니다. 이때는 `thread::spawn` 을 통해 진짜 OS thread 를 만들어 1:1 로 매핑할 수 있습니다.

내부적으로는 Unix 의 `pthread_create` 또는 Windows 의 `CreateThread` 등을 직접 호출하며, 일반 task 와는 분리된 풀에서 작동합니다. 결과는 Future 와는 별개의 타입인 `Thread<T>` 로 반환됩니다.

## 7.6.7. Blocking Syscall 자동 검출

work-stealing 모델은 task 가 blocking syscall 을 호출하면 그 worker 자체가 멈춰 버린다는 문제를 안고 있습니다. 이를 해결하기 위해 blocking pool 자동 오프로드를 사용합니다.

컴파일러와 런타임이 알려진 blocking syscall 을 blocking pool worker 로 자동으로 보내고, 그 worker 는 별도의 OS thread 이기 때문에 다른 task 의 진행을 방해하지 않습니다. 자세한 메커니즘은 스케줄러 구현이 결정하도록 두었습니다.

## 7.6.8. Thread-per-core 옵션

기본 동작은 N 개의 worker 를 두고 task 가 thread 사이를 자유롭게 이동하도록 하는 형태입니다. 다만 thread-per-core 옵션을 켜면 각 task 가 고정된 worker 에 pinning 되고, worker 끼리 steal 하지 않으며, 그 결과 cache locality 가 극대화되어 NUMA 환경이나 데이터센터 워크로드에 적합한 동작을 얻을 수 있습니다.

이 옵션은 `rokc` 빌드 옵션이나 환경 변수 같은 런타임 설정으로 켜고 끌 수 있습니다.

## 7.6.9. 우선순위 / 공정성

기본은 우선순위 없는 FIFO + work-stealing 입니다. 같은 worker 안의 task 는 cache 효율을 위해 LIFO 순서로 처리되고, 다른 worker 가 steal 해 간 task 는 오래된 것을 우선하기 위해 FIFO 순서로 처리됩니다.

명시적 우선순위 지정은 본 사양의 범위 밖이며, 표준 라이브러리의 별도 표면에서 다룹니다.

## 7.6.10. 통합 GC Worker 역할

GC 사이클이 도는 동안 각 worker 는 parallel mark / evacuate 작업에도 함께 참여합니다. 별도의 GC thread 풀을 두지 않고, 같은 worker 가 mutator 와 GC worker 의 역할을 모두 맡는 구조입니다. 사이클이 시작되면 worker 가 GC 협력 모드로 전환되었다가, 사이클이 끝나면 다시 task 실행으로 복귀합니다.

이러한 통합 모델은 [동시 수집](705-concurrent-collection.md) 의 concurrent 진행과 결합되어 worker 를 적절히 활용할 수 있도록 해 줍니다.

## 7.6.11. Scheduler 와 Coroutine 의 관계

스케줄러는 [코루틴](708-coroutine.md) 의 stackless coroutine 을 사용하며, 각 task 의 상태 머신이 coroutine frame 에 저장되는 형태입니다.

task 가 `await` 또는 `yield` 를 만나면 coroutine frame 만 큐에 다시 들어가고 worker 는 다른 task 를 이어서 진행하게 되며, 해당 task 가 재개될 때는 다른 worker 가 그 frame 을 꺼내어 이어서 실행할 수도 있습니다. 이러한 모델 덕분에 수백만 개의 task 가 N 개의 worker 위에서 효율적으로 처리될 수 있는 것입니다.

## 7.6.12. Spawn 비용

`task::spawn` 은 stack 없는 coroutine frame 을 할당하는 것만으로 동작이 완료됩니다. frame 의 크기는 task 의 지역 변수에 따라 달라지지만 대략 64 ~ 256 바이트 수준이고, [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 의 fast path 에서 할당되기 때문에 그 비용이 거의 0 에 가깝습니다.

이 모델 덕분에 수백만 개의 task 를 동시에 진행하더라도 OS thread 와는 비교할 수 없을 만큼 가벼운 동작을 얻을 수 있습니다.

## 7.6.13. 전체 흐름 요약

```
user code:    task::spawn { compute() }
  ↓
compiler:     create coroutine frame + push to queue
  ↓
worker:       pop task from deque/lifo and execute
  ↓ (on await / yield)
worker:       requeue frame, run another task
  ↓ (some worker)
resume:       pop frame and continue execution
  ↓ (on normal termination)
result:       returned as Future<T>
```

> user code: 사용자 코드. compiler: 컴파일러. create coroutine frame + push to queue: coroutine frame 생성 + 큐 push. pop task from deque/lifo and execute: deque/lifo 에서 task 꺼내 실행. on await / yield: await / yield 만나면. requeue frame, run another task: frame 을 큐에 다시, 다른 task 진행. some worker: 어떤 worker 가. resume: 재개. pop frame and continue execution: frame 꺼내 이어서 실행. on normal termination: 정상 종료 시. result: 결과. returned as Future<T>: Future<T> 의 결과로 반환.
