# 7.5. 동시 수집

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 가 Stop-The-World (STW) 만 사용하면 pause time 이 워크로드에 따라 무한히 커진다. nok 의 GC 는 전체 수집을 동시 (concurrent) 로 수행하며, STW 단계는 짧은 동기화에만 사용한다.

## 7.5.1. 동시 수집의 두 단계

```
1. Concurrent Mark         ~10 ms pause (SATB barrier 시작)
2. Concurrent Evacuate     ~1 ms pause (Brooks 포인터 시작)
```

각각 짧은 STW pause 로 시작한 뒤 백그라운드 (mutator 와 동시) 로 진행한다.

## 7.5.2. Concurrent Mark: Tricolor + SATB barrier

### Tricolor 알고리듬

객체를 세 가지 색으로 분류한다.

| 색    | 의미                         |
| ----- | ---------------------------- |
| White | 아직 도달 안 함 (회수 후보)  |
| Grey  | 도달했지만 자식은 아직 안 봄 |
| Black | 도달 + 자식 모두 처리됨      |

mark 진행:

```
1. 모든 객체를 White 로 시작.
2. 루트 → Grey.
3. Grey 객체 하나를 꺼내 — White 자식을 모두 Grey 로, 자기는 Black 으로.
4. Grey 큐가 빌 때까지 반복.
5. 끝나면 — White 객체는 죽었음 (회수).
```

### Mutator 의 간섭: SATB

mark 가 동시에 일어나는 동안에도 mutator 는 포인터 쓰기를 계속한다. 이 쓰기가 mark 를 망칠 수 있다.

문제 시나리오:

```
시점 1: GC 가 Black 객체 P 의 처리를 끝낸다 (자식 다 봄).
시점 2: Mutator 가 P 의 필드를 — 새로 만든 White 객체 Q 로 변경.
        그리고 — 이전에 P 가 가리키던 Q' (도달 가능했던) 의 마지막 참조가 끊어진다.
시점 3: GC 는 P 의 자식을 다시 보지 않는다 (이미 Black). Q 가 mark 되지 않는다.
        Q' 는 더 이상 도달 불가 — White 로 잘못 회수된다.
```

### SATB (Snapshot-At-The-Beginning) 해결

GC 시작 시점의 객체 그래프 스냅샷을 마크한다는 의미다.

barrier:

```
*p = new_value;                  // p 의 자식이 바뀜
                                  // ↓ barrier (SATB)
if (mark_in_progress) {
    old_value = *p;              // 변경 전 값
    mark_queue.push(old_value);   // 잘못 회수되지 않게 mark
}
```

매 포인터 쓰기마다 변경 전 값을 mark queue 에 푸시한다 (mark 진행 중일 때만).

이 barrier 가 활성인 동안 처음 스냅샷에서 도달 가능한 모든 객체가 mark 된다. 새로 만들어지는 객체는 Grey 로 직접 시작한다 (할당 시 set).

### Pause time

- initial mark (STW): 루트만 grey 로 표시. 매우 짧다 (~ 10 ms).
- concurrent mark: 백그라운드. mutator 는 SATB barrier 비용만 부담한다.
- final mark (짧은 STW): mutator 의 마지막 쓰기들을 처리. 매우 짧다.

## 7.5.3. Concurrent Evacuate: Brooks 포인터

[GC 알고리듬](703-gc-algorithm.md) 의 opportunistic evacuation 이 단편화가 심한 [block](../000-meta/007-glossary.md#block-블록) 을 evacuate 한다.

evacuation 은 객체 복사다. 동시에 일어나려면 mutator 의 접근과 GC 의 복사를 동기화해야 한다.

### Brooks 포인터

각 객체 헤더에는 forwarding 포인터 슬롯이 있다. evacuate 시 옛 자리에 새 자리의 포인터를 둔다.

```
객체 X (옛 자리, evacuate 후):
  type_id    : ...
  gc_flags   : forwarded = 1
  gc_word    : ptr to new location
  payload    : (안 봄)

객체 X (새 자리):
  type_id    : ...
  gc_flags   : forwarded = 0
  gc_word    : 0
  payload    : ... (복사됨)
```

mutator 가 객체 X 의 필드에 접근하면:

```
1. 객체의 forwarded 비트를 검사.
2. 켜져 있으면 — gc_word 를 따라가 새 자리에 접근.
3. 꺼져 있으면 — 옛 자리에 접근.
```

이 검사가 모든 객체 접근에 추가되지만, 분기 1 개라 비용이 작다 (LLVM 이 hot path 를 최적화).

### Pause time

- initial evacuate (STW): barrier 활성화. ~ 1 ms.
- concurrent evacuate: 백그라운드 객체 복사. mutator 는 forwarding barrier 비용을 부담한다.
- final fixup (짧은 STW): 모든 forwarding 포인터 정리.

## 7.5.4. LXR-style RC + Trace

[GC 알고리듬](703-gc-algorithm.md) 에서 언급한 LXR (PLDI 2022) 모델은 Reference Counting 과 Trace 를 결합한 것이다.

### RC 부분

객체 헤더에 참조 카운트 비트, 또는 별도 카운터 필드를 둔다.

- 포인터 복사 / 삭제 시 카운트를 증감 (write barrier 로).
- 카운트가 0 에 도달하면 즉시 회수 가능.

### Trace 부분

RC 만으로는 순환 참조를 회수하지 못한다 (서로 가리켜 카운트 ≥ 1).

순환 회수를 위해 주기적 trace (mark / sweep) 를 추가한다. 위의 concurrent mark 가 그 역할을 맡는다.

### 두 메커니즘의 협력

- 짧게 살거나 비순환인 객체: RC 가 즉시 회수 (낮은 latency).
- 긴 사이클이나 순환: Trace 가 검출 (드묾).

이 결합이 최상급 throughput (논문 결과) 과 낮은 평균 pause 를 낸다.

자세한 RC 메커니즘은 GC 구현이 결정한다. 본 사양은 LXR 모델 적용까지 책임진다.

## 7.5.5. STW Parallel 단계 없음

전통적 GC 는 짧은 STW parallel 단계 (한 번 멈추고 모든 worker 가 mark) 도 사용한다. nok 은 처음부터 concurrent 만 쓴다. STW parallel 전용 단계는 없다.

이는 latency 우선의 결정이다. throughput 은 LXR / [Immix](../000-meta/007-glossary.md#immix) 로 보전한다.

## 7.5.6. Pause time 목표

| 단계                   | 목표                                |
| ---------------------- | ----------------------------------- |
| Initial mark (STW)     | ~ 10 ms                             |
| Concurrent mark        | 백그라운드 (사용자가 인지하지 못함) |
| Final mark (STW)       | ~ 1 ms                              |
| Initial evacuate (STW) | ~ 1 ms                              |
| Concurrent evacuate    | 백그라운드                          |
| Final fixup (STW)      | < 1 ms                              |

사용자가 인지하는 전체 pause 시간은 < 15 ms 가 목표.

이는 인터랙티브 / 실시간 워크로드 (REPL / Jupyter) 에 충분하다.

## 7.5.7. 동시 수집의 비용

mutator 가 받는 추가 비용:

- Write barrier (Incremental-Update + SATB): 대부분 분기 1 ~ 2 개.
- Forwarding barrier: 객체 접근에 분기 1 개.
- Safepoint poll: 함수 / 루프에 분기 1 개.

총합은 throughput overhead 5 ~ 10 % 수준 (구체적 수치는 워크로드에 따라 다름).

이 비용은 낮은 pause 와 concurrent 진행의 대가다.

## 7.5.8. 트리거 정책

GC 사이클이 언제 시작되는가:

- young GC: [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 부족 시, heap 사용량이 임계에 닿을 때.
- old GC: young 사이클 N 번마다, 또는 sticky 객체 비율이 임계에 닿을 때.
- concurrent evacuate: 단편화 임계를 넘은 block 이 N 개 이상.

정확한 임계와 정책은 GC 구현의 런타임 설정으로 결정한다 (조정 가능).
