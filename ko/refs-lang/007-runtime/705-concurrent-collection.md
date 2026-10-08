# 7.5. 동시 수집

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 가 Stop-The-World (STW) 만 사용한다면 pause time 이 워크로드에 따라 무한히 커질 수 있습니다. 그래서 rok 의 GC 는 전체 수집을 동시 (concurrent) 로 수행하며, STW 단계는 짧은 동기화에만 한정해서 사용합니다.

## 7.5.1. 동시 수집의 두 단계

```
1. Concurrent Mark         ~10 ms pause (SATB barrier 시작)
2. Concurrent Evacuate     ~1 ms pause (Brooks 포인터 시작)
```

각 단계는 짧은 STW pause 로 시작한 뒤 백그라운드에서 mutator 와 동시에 진행됩니다.

## 7.5.2. Concurrent Mark: Tricolor + SATB barrier

### Tricolor 알고리듬

객체를 세 가지 색으로 분류하여 다룹니다.

| 색    | 의미                                |
|-------|-------------------------------------|
| White | 아직 도달 하지 않습니다 (회수 후보) |
| Grey  | 도달했지만 자식은 아직 안 봄        |
| Black | 도달 + 자식 모두 처리됨             |

mark 의 진행 과정은 다음과 같습니다.

```
1. 모든 객체를 White 로 시작.
2. 루트 → Grey.
3. Grey 객체 하나를 꺼내 — White 자식을 모두 Grey 로, 자기는 Black 으로.
4. Grey 큐가 빌 때까지 반복.
5. 끝나면 — White 객체는 죽었음 (회수).
```

### Mutator 의 간섭: SATB

mark 가 동시에 진행되는 동안에도 mutator 는 포인터 쓰기를 계속하기 때문에, 이 쓰기가 mark 결과를 망쳐 버릴 수 있습니다.

문제가 되는 시나리오를 살펴보면 다음과 같습니다.

```
시점 1: GC 가 Black 객체 P 의 처리를 끝낸다 (자식 다 봄).
시점 2: Mutator 가 P 의 필드를 — 새로 만든 White 객체 Q 로 변경.
        그리고 — 이전에 P 가 가리키던 Q' (도달 가능했던) 의 마지막 참조가 끊어진다.
시점 3: GC 는 P 의 자식을 다시 보지 않는다 (이미 Black). Q 가 mark 되지 않는다.
        Q' 는 더 이상 도달 불가 — White 로 잘못 회수된다.
```

### SATB (Snapshot-At-The-Beginning) 해결

SATB 는 GC 시작 시점의 객체 그래프 스냅샷을 그대로 마크한다는 뜻입니다.

barrier 는 다음과 같은 모양을 갖습니다.

```
*p = new_value;                  // p 의 자식이 바뀜
                                  // ↓ barrier (SATB)
if (mark_in_progress) {
    old_value = *p;              // 변경 전 값
    mark_queue.push(old_value);   // 잘못 회수되지 않게 mark
}
```

매 포인터 쓰기마다 변경 전 값을 mark queue 에 푸시합니다. 단, 이 동작은 mark 가 진행 중일 때만 일어납니다.

이 barrier 가 활성화되어 있는 동안에는, 처음 스냅샷에서 도달 가능했던 모든 객체가 빠짐없이 mark 됩니다. 새로 만들어지는 객체는 할당 시점에 set 되어 Grey 로 직접 시작됩니다.

### Pause time

- initial mark (STW): 루트만 grey 로 표시하므로 매우 짧습니다 (~ 10 ms).
- concurrent mark: 백그라운드로 진행되며, mutator 는 SATB barrier 비용만 부담합니다.
- final mark (짧은 STW): mutator 의 마지막 쓰기들을 처리하는 단계로, 매우 짧습니다.

## 7.5.3. Concurrent Evacuate: Brooks 포인터

[GC 알고리듬](703-gc-algorithm.md) 의 opportunistic evacuation 이 단편화가 심한 [block](../000-meta/007-glossary.md#block-블록) 을 evacuate 합니다.

evacuation 은 결국 객체 복사이기 때문에, 이를 동시에 수행하려면 mutator 의 접근과 GC 의 복사를 서로 동기화해야 합니다.

### Brooks 포인터

각 객체 헤더에는 forwarding 포인터 슬롯이 마련되어 있습니다. evacuate 가 일어나면 옛 자리에 새 자리의 포인터를 두는 방식으로 동작합니다.

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

mutator 가 객체 X 의 필드에 접근할 때는 다음 절차를 따릅니다.

```
1. 객체의 forwarded 비트를 검사.
2. 켜져 있으면 — gc_word 를 따라가 새 자리에 접근.
3. 꺼져 있으면 — 옛 자리에 접근.
```

이 검사가 모든 객체 접근에 추가되기는 하지만, 분기 1 개에 불과해 비용이 작은 편이며, LLVM 이 hot path 를 최적화해 줍니다.

### Pause time

- initial evacuate (STW): barrier 를 활성화하는 단계로, 약 1 ms 정도입니다.
- concurrent evacuate: 백그라운드에서 객체를 복사하며, mutator 는 forwarding barrier 비용을 부담합니다.
- final fixup (짧은 STW): 모든 forwarding 포인터를 정리하는 단계입니다.

## 7.5.4. LXR-style RC + Trace

[GC 알고리듬](703-gc-algorithm.md) 에서 언급한 LXR (PLDI 2022) 모델은 Reference Counting 과 Trace 를 결합한 형태입니다.

### RC 부분

객체 헤더에 참조 카운트 비트, 또는 별도의 카운터 필드를 둡니다.

- 포인터 복사나 삭제 시점에 카운트를 증감시킵니다 (write barrier 를 통해서 처리합니다).
- 카운트가 0 에 도달하면 즉시 회수가 가능합니다.

### Trace 부분

RC 만으로는 순환 참조를 회수할 수 없습니다. 서로를 가리키고 있어 카운트가 항상 1 이상 유지되기 때문입니다.

순환 회수를 위해서는 주기적인 trace (mark / sweep) 를 추가해야 하며, 위에서 설명한 concurrent mark 가 그 역할을 맡아 줍니다.

### 두 메커니즘의 협력

- 짧게 살거나 비순환인 객체는 RC 가 즉시 회수합니다 (낮은 latency 를 보장합니다).
- 긴 사이클이나 순환에 해당하는 경우에는 Trace 가 검출합니다 (드물게 일어납니다).

이러한 결합 덕분에 최상급 throughput (논문 결과 기준) 과 낮은 평균 pause 를 동시에 얻을 수 있습니다.

자세한 RC 메커니즘은 GC 구현이 결정하며, 본 사양은 LXR 모델을 적용하는 지점까지만 책임집니다.

## 7.5.5. STW Parallel 단계 없음

전통적인 GC 는 짧은 STW parallel 단계 (한 번 멈추고 모든 worker 가 동시에 mark 하는 방식) 도 함께 사용합니다. 반면 rok 은 처음부터 concurrent 만 사용하며, STW parallel 전용 단계는 따로 두지 않습니다.

이는 latency 를 우선시한 결정입니다. throughput 은 LXR 과 [Immix](../000-meta/007-glossary.md#immix) 로 보전합니다.

## 7.5.6. Pause time 목표

| 단계                   | 목표                                    |
|------------------------|-----------------------------------------|
| Initial mark (STW)     | ~ 10 ms                                 |
| Concurrent mark        | 백그라운드 (사용자가 인지하지 못합니다) |
| Final mark (STW)       | ~ 1 ms                                  |
| Initial evacuate (STW) | ~ 1 ms                                  |
| Concurrent evacuate    | 백그라운드                              |
| Final fixup (STW)      | < 1 ms                                  |

사용자가 인지하는 전체 pause 시간은 15 ms 미만을 목표로 잡고 있습니다.

이 정도 수준이면 인터랙티브하거나 실시간성이 요구되는 워크로드 (REPL, Jupyter 등) 에 충분히 대응할 수 있습니다.

## 7.5.7. 동시 수집의 비용

mutator 가 추가로 부담하게 되는 비용은 다음과 같습니다.

- Write barrier (Incremental-Update + SATB): 대부분 분기 1 ~ 2 개 정도입니다.
- Forwarding barrier: 객체 접근마다 분기 1 개가 추가됩니다.
- Safepoint poll: 함수와 루프에 분기 1 개가 들어갑니다.

이를 모두 합치면 throughput overhead 는 5 ~ 10 % 수준이며, 구체적인 수치는 워크로드에 따라 달라집니다.

이 비용은 낮은 pause 와 concurrent 진행을 얻기 위한 대가라고 보면 됩니다.

## 7.5.8. 트리거 정책

GC 사이클이 언제 시작되는지를 정리하면 다음과 같습니다.

- young GC: [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 가 부족해질 때, 또는 heap 사용량이 임계에 닿을 때 시작됩니다.
- old GC: young 사이클이 N 번 돌 때마다, 또는 sticky 객체 비율이 임계에 닿을 때 시작됩니다.
- concurrent evacuate: 단편화 임계를 넘은 block 이 N 개 이상 쌓였을 때 시작됩니다.

정확한 임계와 정책은 GC 구현의 런타임 설정에서 결정하며, 조정이 가능합니다.
