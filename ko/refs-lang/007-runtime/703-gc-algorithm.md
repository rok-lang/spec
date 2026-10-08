# 7.3. GC 알고리듬

rok 의 GC 는 [Immix](../000-meta/007-glossary.md#immix) mark-region[^immix] 과 Sticky Mark-Bit generational (Whippet mmc 방식)[^whippet] 을 결합한 형태로 설계되어 있습니다. 이 두 가지 기법을 함께 사용함으로써 메모리 효율과 처리 성능을 동시에 확보하고자 합니다.

본 문서에서는 알고리듬의 동작 원리만을 다루며, 동시 수집과 배리어에 관한 내용은 [배리어와 루트](704-barriers-and-roots.md) 에서 따로 정의하고 있습니다.

## 7.3.1. Immix mark-region

기존의 GC 는 크게 두 가지 축으로 발전해 왔습니다.

- mark-sweep 방식은 객체를 마크하고 죽은 자리를 free list 로 관리하는 방법인데, 단편화가 발생한다는 한계를 지닙니다.
- mark-compact 또는 copy 방식은 살아있는 객체를 한 곳으로 모아 옮기는 방법으로, 단편화가 전혀 없는 대신 복사 비용이 크다는 단점이 있습니다.

Immix 는 이 두 방식의 장점을 결합한 알고리듬에 해당합니다.

### 32 KB block + 128 B line

```
block (32 KB)
  ├── line (128 B)
  ├── line (128 B)
  ├── ...
  └── line (128 B)         (256 line / block)
```

메모리는 [block](../000-meta/007-glossary.md#block-블록) 단위로 OS 로부터 받아오며 (mmap), 각 block 의 내부는 다시 line 단위로 분할됩니다. 객체는 line 경계를 넘어 존재할 수도 있는데, 큰 객체의 경우 여러 line 을 동시에 점유하는 형태가 됩니다.

### mark: line 단위

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 는 어떤 line 에 살아있는 객체가 있는지를 비트맵을 통해 추적합니다. 객체를 mark 할 때에는 그 객체가 차지하는 모든 line 을 함께 mark 하며, mark 가 끝나면 unmarked line 만을 회수하게 됩니다.

이러한 line 단위 mark 가 바로 핵심에 해당합니다. 객체 mark 와 line mark 를 분리해 두면 살아있는 객체 사이의 작은 틈새까지도 line 단위로 기록되기 때문에, 그 공간을 다시 재활용할 수 있게 됩니다.

### sweep: block 분류

mark 가 끝난 뒤에는 각 block 을 다음 세 가지 중 하나로 분류합니다.

| 분류       | 의미                       | 처리                         |
|------------|----------------------------|------------------------------|
| clean      | 모든 line 이 free          | block 자체를 free pool 로    |
| recyclable | 일부 line 만 살아있음      | 사용 가능한 line 들로 재활용 |
| full       | 거의 모든 line 이 살아있음 | 그대로 둠 (단편화 가능)      |

recyclable block 의 경우 사이사이 비어 있는 line 들에 새로운 객체 할당이 들어가는 방식으로 활용됩니다.

### 단편화 처리: opportunistic evacuation

block 안의 단편화가 심한 경우에는 그 block 의 살아있는 객체를 다른 block 으로 evacuate 하게 됩니다.

이때 대상이 되는 block 은 사용량이 적은 recyclable block 이며, evacuate 는 결국 복사를 의미합니다. 객체를 옮긴 뒤 옛 자리에는 forwarding pointer 를 남겨 두고, 이후 mark 단계에서 forwarding pointer 를 따라가 새 위치를 mark 하는 식으로 동작합니다.

evacuation 을 항상 수행하는 것은 아닙니다. 단편화 임계를 넘었을 때에만 진행하며, 바로 이 점이 "opportunistic" 이 의미하는 바입니다.

이 방식을 통해 mark-compact 대비 메모리 효율은 14%, throughput 은 7 ~ 25% 정도의 개선을 얻을 수 있다고 보고되어 있습니다 (논문 결과).

## 7.3.2. Sticky Mark-Bit Generational (Whippet mmc)

Immix 위에 세대 추적 기능을 추가하는 단계입니다. 일반적인 generational GC 처럼 nursery 와 old space 를 분리하는 대신, 같은 공간 안에서 비트만으로 세대를 추적합니다.

### sticky 비트

객체의 `gc_flags` ([객체 레이아웃과 메모리 공간](701-object-layout.md)) 에는 `sticky` 비트가 마련되어 있습니다. 객체가 처음 mark 될 때 이 `sticky` 비트가 켜지며, 이후로 `sticky` 가 켜진 객체는 오래된 객체로 간주됩니다.

### Young GC: sticky 건너뜀

```
Young GC:
  - sticky 비트가 *꺼진* 객체만 mark / sweep.
  - sticky 가 켜진 객체는 *살아있다고 가정* — 따로 검사 안 함.
  - mark 끝난 unmarked young 객체 회수.
  - 새로 mark 된 객체에 sticky 비트 set (다음 GC 부터 old).
```

대부분의 객체는 짧게 살다가 소멸하기 때문에 (generational hypothesis), young GC 만으로도 가비지의 대부분을 회수할 수 있게 됩니다.

### Old GC: sticky 리셋

```
Old GC (덜 자주):
  - 모든 sticky 비트 *리셋*.
  - 전체 mark / sweep.
  - 도달한 객체는 다시 sticky set.
```

old GC 는 young GC 에 비해 훨씬 덜 자주 일어나며, 일반적으로는 young GC 의 1/N 정도의 빈도로 수행됩니다.

### 별도 nursery 공간 없음

전통적인 generational GC 에서는 별도의 nursery 공간을 두고, 나이 든 객체를 promote (다른 공간으로 복사) 하는 방식을 취합니다. 그러나 rok 의 [sticky mark-bit](../000-meta/007-glossary.md#sticky-mark-bit) 은 공간을 분리하지 않고도 같은 효과를 얻을 수 있도록 해 줍니다.

이 방식의 장점은 다음과 같습니다.

- 공간 낭비가 전혀 없습니다. nursery 가 비어 있어도 그로 인한 낭비가 발생하지 않습니다.
- promotion 복사도 필요 없습니다. 객체는 제자리에 그대로 두고 sticky 비트만 켜면 충분합니다.
- 로직 자체가 단순해집니다. 공간 사이의 객체 이동이 전혀 없기 때문입니다.

물론 단점도 있는데, 매 객체마다 sticky 검사가 필요하기 때문에 일부 수집 비용이 추가된다는 점입니다.

전체적으로 볼 때 Whippet mmc 의 결과는 전통적인 nursery 방식과 동등하거나 오히려 더 나은 throughput 을 보여 줍니다.

## 7.3.3. 사이클: Young 과 Old 의 교차

```
Young → Young → Young → Young → ... → Old → Young → Young → ...
```

대부분의 사이클은 young 으로 처리되며, 특정 조건에서만 old 가 일어나게 됩니다.

trigger 조건은 다음과 같이 정리할 수 있습니다. 첫째, young GC 횟수 임계가 있는데, young 이 N 번 일어날 때마다 한 번씩 old 가 트리거됩니다. 둘째, heap 점유율로, old 공간이 차오르면 (즉 sticky 객체가 많아지면) old 가 트리거됩니다. 셋째, 수동 호출인데, 사용자나 시스템이 명시적으로 호출하는 경우입니다 (드물게 사용됩니다).

## 7.3.4. 객체 mark 의 절차

GC 는 다음과 같은 순서로 mark 를 진행합니다.

1. 루트부터 시작합니다. 루트의 정의는 [배리어와 루트](704-barriers-and-roots.md) 에서 확인할 수 있습니다.
2. 도달한 각 객체에 대해 다음을 수행합니다.
   - 헤더의 mark 비트를 set 합니다.
   - 그 객체가 차지하는 line 들을 mark 합니다.
   - `type_id → scan_table` 을 통해 자식 포인터들을 따라갑니다.
3. 큐가 모두 비면 mark 가 종료됩니다.

여기서 사용되는 [scan_table](../000-meta/007-glossary.md#scan_table) 은 `rok_gc_register_type` 으로 [Extern "C" 심볼](../008-abi/805-extern-symbols.md) 의 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 에 따라 등록되는 정보입니다. 자세한 형식은 [ScanTable 직렬화 포맷](../008-abi/802-scan-table.md) 에서 정의하고 있습니다.

## 7.3.5. sweep 의 절차

mark 이후에는 block 별로 line 비트맵을 살펴보고 분류 작업을 진행합니다.

block 의 모든 line 이 unmarked 인 경우에는 block 자체를 free 시킵니다. 일부만 marked 인 경우에는 recyclable 로 처리하여 빈 line 들을 다시 할당할 수 있게 만듭니다. 거의 모두 marked 인 경우에는 full 로 분류해 그대로 두거나, 필요에 따라 evacuation 후보로 삼습니다.

evacuation 후보가 충분히 모이면 다음 mark 사이클에서 evacuate 를 진행하게 됩니다.

## 7.3.6. LXR-style RC + trace 결합 (선택)

LXR 논문[^lxr] 에서 제시하는 Reference Counting + Trace hybrid 모델 또한 적용합니다. 짧게 사는 객체는 RC 를 통해 빠르게 회수하고, 긴 사이클이나 누수성 객체는 trace 를 통해 검출하는 방식입니다. 이 두 메커니즘은 동시에 동작하면서 서로의 약점을 보완해 줍니다.

자세한 내용은 GC 구현이 결정할 사항입니다. 본 사양은 LXR 모델을 적용한다는 사실까지를 책임지는 범위로 합니다.

## 참고 문헌

[^immix]: Blackburn & McKinley, _Immix: A Mark-Region Garbage Collector with Space Efficiency, Fast Collection, and Mutator Performance_, PLDI 2008.

[^whippet]: Wingo, Whippet: a sub-toolkit for embeddable garbage collection. <https://wingolog.org/tags/whippet>

[^lxr]: Zhao, Blackburn, McKinley, Low-Latency, High-Throughput Garbage Collection, PLDI 2022.
