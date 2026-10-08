# 7.6. 스케줄러

nok 의 스케줄러는 M:N [work-stealing](../000-meta/007-glossary.md#work-stealing) 이다. 본 문서는 그 구조를 정의한다.

## 7.6.1. 통합된 worker 모델

기본은 N 개의 OS thread (보통 CPU 코어 수). 각 thread 가 네 가지 역할을 동시에 수행한다.

```
OS Thread = {
    Scheduler Worker      // task queue (work-stealing)
    Mutator Context       // TLAB + shadow stack ([객체 레이아웃과 메모리 공간](701-object-layout.md)/[할당](702-allocation.md)/[배리어와 루트 추적](704-barriers-and-roots.md))
    I/O Driver Handle     // poll / epoll / iouring / kqueue / IOCP / wasi
    GC Worker Context     // parallel mark / evacuate 참여
}
```

이 통합 덕분에 추가 thread 분리 없이 모든 런타임 활동이 한 풀에서 처리된다.

## 7.6.2. Work-Stealing 구조

각 worker 는 로컬 deque 와 LIFO slot 을 갖는다.

```
Worker:
  ├── lifo_slot   : 1-task buffer    (가장 최근 spawn 한 task)
  ├── local_deque : 64-task ring     (자신의 task 들)
  └── ...
```

- task 생성: LIFO slot 에 먼저 넣고, 차면 local_deque 의 bottom 으로.
- task 실행: local_deque 의 bottom 에서 (LIFO).
- steal: 다른 worker 의 local_deque 의 top 에서 (FIFO 쪽).

이 구조는 캐시 locality (자기가 spawn 한 task 를 자기가 실행) 와 load balancing (다른 worker 가 일 없으면 훔침) 을 동시에 얻는 표준적 work-stealing 모델이다.

## 7.6.3. Global Injector Queue

각 worker 의 local queue 외에 전역 injector queue 가 있다.

```
Global injector:
  - thread 가 외부 (예: extern callback) 에서 task 를 spawn 할 때 들어간다.
  - worker 의 local queue 가 비면 — global 에서 pull 한다.
```

이 큐는 worker 풀 바깥에서 들어오는 task 의 진입점이다.

## 7.6.4. Steal 알고리듬

worker 의 local deque 가 비고 LIFO slot 도 비면:

```
1. global injector 에서 pull 시도.
2. 다른 worker 의 local deque 의 top 에서 steal 시도 (random round-robin).
3. 모두 실패하면 — sleep / park (blocking sync).
```

steal 은 worker 최대 N-1 개를 시도해 모두 비면 park 한다.

## 7.6.5. 키워드 vs 라이브러리 함수

[키워드](../001-lexical/105-keywords.md) 의 예약 키워드 최소화 원칙에 따라, 스케줄러 함수들은 모두 라이브러리 함수다.

| 라이브러리 함수         | 의미                                                                     |
| ----------------------- | ------------------------------------------------------------------------ |
| `task::spawn(f)`        | 새 task 를 큐에 push. 결과 `Task<T>`.                                    |
| `task::yield_now()`     | 자발적 `yield`. 다른 task 에 양보.                                       |
| `task::sleep(d)`        | timer 큐로 이동.                                                         |
| `task::race(a, b)`      | 둘 중 먼저 끝나는 것.                                                    |
| `task::join_all([...])` | 모두 끝날 때까지.                                                        |
| `task::timeout(d, fut)` | 시간 제한.                                                               |
| `thread::spawn(f)`      | 새 OS thread 만들기 (1:1).                                               |
| `par::for_each(xs, f)`  | 병렬 for-each ([block](../000-meta/007-glossary.md#block-블록) on join). |
| `par::scope { ... }`    | 병렬 스코프 (트레일링 클로저).                                           |

키워드가 아니므로 사용자 코드에서 `spawn` / `race` 같은 식별자를 자유롭게 쓸 수 있다.

## 7.6.6. 1:1 OS Thread: `thread::spawn`

대부분의 task 는 M:N (스케줄러가 매핑) 이다. 블로킹 작업이나 외부 라이브러리 호출 등은 전용 OS thread 가 자연스러울 때가 있다.

`thread::spawn` 은 진짜 OS thread 를 만들어 1:1 로 매핑한다.

- `pthread_create` (Unix) / `CreateThread` (Windows) 등을 직접 호출.
- 일반 task 와 분리된 풀에서 작동한다.
- 결과는 `Thread<T>` (Future 와는 별개 타입).

## 7.6.7. Blocking Syscall 자동 검출

work-stealing 모델은 task 가 blocking syscall 을 하면 그 worker 가 멈추는 문제가 있다.

해결책은 blocking pool 자동 오프로드.

- 컴파일러 / 런타임이 알려진 blocking syscall 을 blocking pool worker 로 자동 보낸다.
- 그 worker 는 별도 OS thread 라 다른 task 의 진행을 막지 않는다.

자세한 메커니즘은 스케줄러 구현이 결정한다.

## 7.6.8. Thread-per-core 옵션

기본은 N = CPU 코어 수의 worker 이며, task 가 thread 사이를 자유롭게 이동한다.

thread-per-core 옵션을 켜면:

- 각 task 가 고정된 worker 에 pinning 된다.
- worker 끼리 steal 하지 않는다.
- cache locality 가 극대화된다. NUMA / 데이터센터 워크로드에 적합.

런타임 설정 (`nokc` 빌드 옵션 또는 환경 변수) 으로 켜고 끈다.

## 7.6.9. 우선순위 / 공정성

기본은 우선순위 없는 FIFO + work-stealing 이다.

- 같은 worker 의 task 는 LIFO 순서 (cache 효율).
- 다른 worker 가 steal 한 task 는 FIFO 순서 (오래된 것 우선).

명시적 우선순위 지정 (`task::spawn_with_priority` 등) 은 추후 표준 라이브러리에서 제공 가능 (본 사양 범위 밖).

## 7.6.10. 통합 GC Worker 역할

GC 사이클 동안 각 worker 가 parallel mark / evacuate 에 참여한다.

- 별도의 GC thread 풀이 없다. 같은 worker 가 mutator 와 GC worker 역할을 모두 맡는다.
- 사이클이 시작되면 worker 가 GC 협력 모드로 전환한다.
- 사이클이 끝나면 다시 task 실행으로 복귀한다.

이 통합 모델은 [동시 수집](705-concurrent-collection.md) 의 concurrent 진행과 결합되어 worker 를 적절히 활용한다.

## 7.6.11. Scheduler 와 Coroutine 의 관계

[코루틴](708-coroutine.md) 의 stackless coroutine 을 쓴다. 각 task 의 상태 머신이 coroutine frame 에 저장된다.

- task 가 `await` / `yield` 하면 coroutine frame 만 큐에 다시 들어가고 worker 는 다른 task 를 진행한다.
- task 가 재개될 때는 다른 worker 가 그 frame 을 꺼내 이어서 실행할 수 있다.

이 모델 덕분에 수백만 개의 task 가 N 개 worker 에서 효율적으로 처리된다.

## 7.6.12. Spawn 비용

`task::spawn` 은 stack 없는 coroutine frame 을 할당하기만 하면 끝이다.

- frame 크기 ≈ 64 ~ 256 바이트 (task 의 지역 변수에 따라 다름).
- [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 에서 fast path 로 할당. 비용이 거의 0 이다.

이 모델 덕분에 수백만 task 의 동시 진행이 OS thread 와 비교할 수 없이 가볍다.

## 7.6.13. 전체 흐름 요약

```
사용자 코드: task::spawn { compute() }
  ↓
컴파일러: coroutine frame 생성 + 큐 push
  ↓
worker: deque/lifo 에서 task 꺼내 실행
  ↓ (await / yield 만나면)
worker: frame 을 큐에 다시, 다른 task 진행
  ↓ (어떤 worker 가)
재개: frame 꺼내 이어서 실행
  ↓ (정상 종료 시)
결과: Future<T> 의 결과로 반환
```
