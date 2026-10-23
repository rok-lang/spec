# 7.2. 할당

[객체 레이아웃](701-object-layout.md) 에서 정의한 객체를 메모리에 배치하는 방식을 다룹니다. 할당 알고리듬은 fast path 와 slow path 의 두 단계로 정의됩니다.

## 7.2.1. TLAB (Thread-Local Allocation Buffer)

각 스케줄러 worker (= OS thread) 는 자기 자신의 TLAB 을 가집니다. TLAB 은 [Immix](../000-meta/007-glossary.md#immix) 공간에서 받은 [block](../000-meta/007-glossary.md#block-블록) 하나 (또는 그 일부) 로 구성되며, bump pointer 방식으로 빠르게 할당합니다.

```
TLAB 구조.
  cursor : ptr      // 다음 할당 위치
  limit  : ptr      // TLAB 의 끝
  block  : ptr      // 시작 (재활용용)
```

cursor 가 limit 보다 작은 동안에는 공간이 충분합니다. 반면 cursor 에 size 를 더한 값이 limit 을 넘는다면 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 이 부족한 상태이므로 slow path 로 빠지게 됩니다.

## 7.2.2. Fast path: 인라인 4 명령어

> [!IMPORTANT]
> 대부분의 할당은 인라인 코드로 처리됩니다. 기계 명령어 4 ~ 5 개로 끝나며, 동기화 비용은 0 입니다.

```
1. tmp = cursor
2. new_cursor = tmp + size
3. if new_cursor > limit → goto slow
4. cursor = new_cursor
   (return tmp)
```

이 경로는 기계 명령어 4 ~ 5 개로 완결됩니다. branch 는 거의 `false` 로 빠른 경로를 따라가며, TLAB 자체가 thread-local 이기 때문에 동기화 비용이 0 입니다.

이 fast path 는 컴파일러가 LLVM IR 에 직접 생성하는 인라인 코드로 제공되며, 런타임이 함수 호출 형태로 노출하지는 않습니다.

## 7.2.3. Slow path: `rok_gc_alloc_slow`

TLAB 이 부족하거나 객체가 큰 경우 ([LOS](../000-meta/007-glossary.md#los-large-object-space)) 에 사용됩니다.

```
extern "C" {
    fn rok_gc_alloc_slow(size: usize, align: usize, type_id: u32) -> Ptr<u8>;
}
```

이 함수는 런타임이 제공하는 `extern "C"` 함수이며, 컴파일러는 fast path 가 실패했을 때 이 함수를 호출합니다.

slow path 는 다음을 합니다.

1. 8 KiB 이상이거나 정렬이 8 바이트보다 크면 LOS 에 할당합니다 (7.2.9).
2. 한 line 보다 크고 남은 TLAB 에 들어가지 않는 객체는 자기 line 구간에 할당합니다.
3. 그 밖에는 TLAB 을 새로 채웁니다. 필요하면 먼저 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 수집을 합니다 (7.2.8).
4. 헤더의 `type_id` 와 `gc_flags` 를 초기화합니다.

## 7.2.4. 객체 헤더 초기화

객체 헤더는 slow path 에서 초기화되며, fast path 의 경우에도 컴파일러가 인라인으로 헤더를 채운 뒤 사용합니다.

```
*(ptr + 0) = type_id
*(ptr + 4) = 0u32          // gc_flags 초깃값
*(ptr + 8) = size          // gc_word - 객체 총 크기 (8.1.4)
```

payload 는 이미 0 이며, 사용자 코드가 필드 값으로 채웁니다. 수집기는 헤더 앞이나 side table 에 자체 bookkeeping 을 둘 수 있습니다.

## 7.2.5. 정렬

모든 할당은 8 바이트 정렬을 보장합니다. TLAB cursor 가 8 바이트의 배수로 유지되며, 객체 크기 또한 8 바이트 배수로 올림 처리됩니다. 더 큰 정렬이 요구되는 객체 (`#[repr(align(N))]`) 는 slow path 에서 처리합니다.

## 7.2.6. TLAB

TLAB 은 Immix block 안의 비어 있는 line 의 연속 구간입니다. refill 때 그 구간을 0 으로 지워 넘깁니다. 그래서 fast path 는 0 으로 채워진 메모리를 받습니다.

## 7.2.7. TLAB 와 work-stealing 의 관계

[스케줄러](706-scheduler.md) 의 work-stealing 메커니즘 덕분에 한 worker 가 다른 worker 의 task 를 훔칠 수 있습니다.

훔쳐 온 task 는 훔친 worker 의 TLAB 에서 할당되며, 원래 worker 의 TLAB 와는 무관하게 동작합니다.

이 모델은 TLAB 이 각 OS thread 에 thread-local 로 묶여 있기 때문에 자연스럽게 작동합니다.

## 7.2.8. GC trigger

slow path 는 다음 경우에 수집을 시작합니다. 조건과 minor / full 의 선택은 [7.3.2](703-gc-algorithm.md#732-sticky-mark-bit-세대-수집) 를 따릅니다.

- 새 TLAB 이나 block 을 내줄 수 없을 때.
- 지난 수집 뒤의 할당량이 다음 cycle 의 예산을 넘었을 때.

`rok_gc_collect` 는 명시적으로 full 수집을 합니다.

## 7.2.9. 큰 객체의 LOS 할당

8 KiB 이상이거나 8 바이트보다 큰 정렬을 요구하는 객체는 TLAB 을 거치지 않고 LOS 에 하나씩 할당합니다. 큰 객체를 TLAB 에 두면 남은 자리가 낭비되고, LOS 객체는 옮기지 않으므로 bump 할당의 이점이 없습니다.

## 7.2.10. 고정된 객체

GC 포인터를 넘긴 `extern "C"` 호출 동안에는 따로 고정할 필요가 없습니다 ([7.4.1](704-barriers-and-roots.md#루트가-가리키는-객체는-옮기지-않음)). C 가 호출 뒤에도 포인터를 보관하면 `rok_gc_pin` 으로 고정합니다. 고정은 `PINNED` 비트를 켤 뿐 객체를 복사하지 않습니다. `rok_gc_unpin` 이 풉니다.

## 7.2.11. 할당 시간 vs heap 사용량

fast path 는 상수 시간 (~ 4 명령어) 입니다. 평균 할당 비용은 fast path 가 정합니다.

## 7.2.12. 할당 생략

최적화기는 객체가 밖으로 나가지 않고 store 뒤에 읽히지도 않는 할당을 없앨 수 있습니다. 그런 할당은 safepoint 가 아니며 수집기의 통계에 세지 않습니다. 크기를 표현할 수 없는 할당 (`len * size + header` overflow) 은 언제나 런타임의 "allocation too large" 에 닿습니다.
