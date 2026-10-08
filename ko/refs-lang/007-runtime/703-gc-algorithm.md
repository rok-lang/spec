# 7.3. GC 알고리듬

nok 의 GC 는 [Immix](../000-meta/007-glossary.md#immix) mark-region[^immix] 과 Sticky Mark-Bit generational (Whippet mmc 방식)[^whippet] 의 결합이다.

본 문서는 알고리듬의 동작 원리만 다룬다. 동시 수집과 배리어는 [배리어와 루트](704-barriers-and-roots.md) 에서 정의한다.

## 7.3.1. Immix mark-region

기존 GC 의 두 축:

- mark-sweep: 마크하고 죽은 자리를 free list 로 관리한다. 단편화 발생.
- mark-compact / copy: 살아있는 객체를 모아 옮긴다. 단편화 0, 복사 비용 큼.

Immix 는 둘의 결합이다.

### 32 KB block + 128 B line

```
block (32 KB)
  ├── line (128 B)
  ├── line (128 B)
  ├── ...
  └── line (128 B)         (256 line / block)
```

- 메모리는 [block](../000-meta/007-glossary.md#block-블록) 단위로 OS 에서 받는다 (mmap).
- 각 block 안은 line 단위로 분할된다.
- 객체는 line 경계를 넘어 살 수 있다 (큰 객체는 여러 line 을 점유).

### mark: line 단위

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 는 어떤 line 에 살아있는 객체가 있는지를 비트맵으로 추적한다.

- 객체 mark 시 그 객체가 차지하는 모든 line 을 mark 한다.
- mark 가 끝나면 unmarked line 만 회수한다.

이 line 단위 mark 가 핵심이다. 객체 mark 와 line mark 를 분리해 두면 살아있는 객체 사이의 작은 틈새도 line 단위로 기록되어 재활용할 수 있다.

### sweep: block 분류

mark 후 각 block 을 다음 셋 중 하나로 분류한다.

| 분류       | 의미                       | 처리                         |
| ---------- | -------------------------- | ---------------------------- |
| clean      | 모든 line 이 free          | block 자체를 free pool 로    |
| recyclable | 일부 line 만 살아있음      | 사용 가능한 line 들로 재활용 |
| full       | 거의 모든 line 이 살아있음 | 그대로 둠 (단편화 가능)      |

recyclable block 에는 새 객체 할당이 사이사이 비어 있는 line 들에 들어간다.

### 단편화 처리: opportunistic evacuation

block 안의 단편화가 심한 경우 그 block 의 살아있는 객체를 다른 block 으로 evacuate 한다.

- 대상 block 은 사용량이 적은 recyclable block.
- evacuate 는 복사다. 객체를 옮기고 옛 자리에 forwarding pointer 를 남긴다.
- 이후 mark 단계가 forwarding pointer 를 따라가 새 위치를 mark 한다.

evacuation 을 항상 수행하지는 않는다. 단편화 임계를 넘었을 때에만 한다. 이게 "opportunistic" 의 의미다.

이 방식이 mark-compact 대비 메모리 효율 14%, throughput 7 ~ 25% 개선을 낸다 (논문 결과).

## 7.3.2. Sticky Mark-Bit Generational (Whippet mmc)

Immix 위에 세대 추적을 얹는다. 일반 generational GC 처럼 nursery / old space 를 분리하는 대신 같은 공간 안에서 비트로 추적한다.

### sticky 비트

객체의 `gc_flags` ([객체 레이아웃과 메모리 공간](701-object-layout.md)) 에 `sticky` 비트가 있다.

- 처음 mark 될 때 `sticky` 비트가 켜진다.
- 이후 `sticky` 가 켜진 객체는 오래된 객체로 본다.

### Young GC: sticky 건너뜀

```
Young GC:
  - sticky 비트가 *꺼진* 객체만 mark / sweep.
  - sticky 가 켜진 객체는 *살아있다고 가정* — 따로 검사 안 함.
  - mark 끝난 unmarked young 객체 회수.
  - 새로 mark 된 객체에 sticky 비트 set (다음 GC 부터 old).
```

대부분의 객체는 짧게 살다가 죽는다 (generational hypothesis). young GC 만으로도 가비지의 대부분을 회수할 수 있다.

### Old GC: sticky 리셋

```
Old GC (덜 자주):
  - 모든 sticky 비트 *리셋*.
  - 전체 mark / sweep.
  - 도달한 객체는 다시 sticky set.
```

old GC 는 훨씬 덜 자주 일어난다. 일반적으로 young GC 의 1/N 빈도다.

### 별도 nursery 공간 없음

전통적 generational GC 는 별도 nursery 공간을 두고, 나이 든 객체를 promote (다른 공간으로 복사) 한다. nok 의 [sticky mark-bit](../000-meta/007-glossary.md#sticky-mark-bit) 은 공간을 분리하지 않고도 같은 효과를 낸다.

장점:

- 공간 낭비 0. nursery 가 비어도 낭비가 없다.
- promotion 복사 0. 객체는 제자리에 두고 sticky 비트만 켠다.
- 로직 단순. 공간 사이의 객체 이동이 없다.

단점:

- 일부 수집 비용. 매 객체마다 sticky 검사가 필요하다.

전체적으로 Whippet mmc 의 결과는 전통 nursery 와 동등하거나 더 나은 throughput 을 보인다.

## 7.3.3. 사이클: Young 과 Old 의 교차

```
Young → Young → Young → Young → ... → Old → Young → Young → ...
```

대부분의 사이클은 young 이다. 특정 조건에서 old 가 일어난다.

trigger 조건:

- young GC 횟수 임계. young 이 N 번 일어날 때마다 한 번 old 를 트리거한다.
- heap 점유율. old 공간이 차오르면 (sticky 객체가 많아지면) old 를 트리거한다.
- 수동. 사용자 또는 시스템이 명시 호출 (드묾).

## 7.3.4. 객체 mark 의 절차

GC 는 다음 순서로 mark 한다.

1. 루트부터 시작한다. 루트는 [배리어와 루트](704-barriers-and-roots.md) 에서 정의한다.
2. 도달한 각 객체:
   - 헤더 mark 비트를 set 한다.
   - 그 객체가 차지하는 line 들을 mark 한다.
   - `type_id → scan_table` 로 자식 포인터들을 따라간다.
3. 큐가 비면 mark 종료.

[scan_table](../000-meta/007-glossary.md#scan_table) 은 `nok_gc_register_type` 으로 [Extern "C" 심볼](../008-abi/805-extern-symbols.md) 의 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 에 따라 등록된 정보다. 자세한 형식은 [ScanTable 직렬화 포맷](../008-abi/802-scan-table.md) 에서 정의한다.

## 7.3.5. sweep 의 절차

mark 후 block 별로 line 비트맵을 보고 분류한다.

- block 의 모든 line 이 unmarked → block 자체를 free.
- 일부만 marked → recyclable, 빈 line 들을 재할당 가능.
- 거의 모두 marked → full, 그대로 두거나 evacuation 후보.

evacuation 후보가 충분하면 다음 mark 사이클에서 evacuate 를 진행한다.

## 7.3.6. LXR-style RC + trace 결합 (선택)

LXR 논문[^lxr] 의 Reference Counting + Trace hybrid 모델도 적용한다.

- 짧게 사는 객체는 RC 로 빠르게 회수한다.
- 긴 사이클이나 누수성 객체는 trace 로 검출한다.
- 두 메커니즘이 동시에 동작하며 서로 보완한다.

자세한 내용은 GC 구현이 결정한다. 본 사양은 LXR 모델을 적용한다는 사실까지 책임진다.

## 참고 문헌

[^immix]: Blackburn & McKinley, _Immix: A Mark-Region Garbage Collector with Space Efficiency, Fast Collection, and Mutator Performance_, PLDI 2008.

[^whippet]: Wingo, _Whippet: a sub-toolkit for embeddable garbage collection_. <https://wingolog.org/tags/whippet>

[^lxr]: Zhao, Blackburn, McKinley, _Low-Latency, High-Throughput Garbage Collection_, PLDI 2022.
