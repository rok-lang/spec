# 7.2. 할당

이 문서는 [객체 레이아웃](701-object-layout.md)에서 정의한 객체를 메모리에 배치하는 방식을 다룹니다. 할당 알고리듬은 fast path와 slow path 두 단계로 정의합니다.

## 7.2.1. TLAB (Thread-Local Allocation Buffer)

스케줄러 worker(= OS 스레드)마다 자기 TLAB을 가집니다. TLAB은 [Immix](../000-meta/007-glossary.md#immix) 공간에서 받은 [block](../000-meta/007-glossary.md#block-블록) 하나(또는 그 일부)로 이루어지며, bump pointer 방식으로 빠르게 할당합니다.

```
TLAB 구조.
  cursor : ptr      // 다음 할당 위치
  limit  : ptr      // TLAB 의 끝
  block  : ptr      // 시작 (재활용용)
```

cursor가 limit보다 작은 동안에는 공간이 충분합니다. cursor에 size를 더한 값이 limit을 넘으면 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer)이 부족한 것이므로 slow path로 넘어갑니다.

## 7.2.2. Fast path: 인라인 4 명령어

> [!IMPORTANT]
> 대부분의 할당은 인라인 코드로 처리합니다. 기계 명령어 4~5개로 끝나며, 동기화 비용이 없습니다.

```
1. tmp = cursor
2. new_cursor = tmp + size
3. if new_cursor > limit → goto slow
4. cursor = new_cursor
   (return tmp)
```

이 경로는 기계 명령어 4~5개로 끝납니다. branch는 거의 항상 `false`여서 빠른 경로를 따르며, TLAB 자체가 thread-local이므로 동기화 비용이 없습니다.

fast path는 컴파일러가 LLVM IR에 직접 생성하는 인라인 코드입니다. 런타임은 이를 함수 호출로 노출하지 않습니다.

## 7.2.3. Slow path: `rok_gc_alloc_slow`

TLAB이 부족하거나 객체가 클 때([LOS](../000-meta/007-glossary.md#los-large-object-space)) 씁니다.

```
extern "C" {
    fn rok_gc_alloc_slow(size: usize, align: usize, type_id: u32) -> Ptr<u8>;
}
```

런타임이 제공하는 `extern "C"` 함수이며, 컴파일러는 fast path가 실패하면 이 함수를 호출합니다.

slow path는 다음을 수행합니다.

1. 객체가 8 KiB 이상이거나 정렬이 8바이트보다 크면 LOS에 할당합니다(7.2.9).
2. 한 line보다 크고 남은 TLAB에 들어가지 않는 객체는 별도의 line 구간에 할당합니다.
3. 그 밖의 경우에는 TLAB을 새로 채웁니다. 필요하면 먼저 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 수집을 합니다(7.2.8).
4. 헤더의 `type_id`와 `gc_flags`를 초기화합니다.

## 7.2.4. 객체 헤더 초기화

객체 헤더는 slow path에서 초기화합니다. fast path에서도 컴파일러가 인라인으로 헤더를 채운 뒤 사용합니다.

```
*(ptr + 0) = type_id
*(ptr + 4) = 0u32          // gc_flags 초깃값
*(ptr + 8) = size          // gc_word: 객체 총 크기 (8.1.4)
```

payload는 이미 0으로 채워져 있으며, 사용자 코드가 필드 값을 채웁니다. 수집기는 헤더 앞이나 side table에 자체 bookkeeping 정보를 둘 수 있습니다.

## 7.2.5. 정렬

모든 할당은 8바이트 정렬을 보장합니다. TLAB cursor는 늘 8바이트의 배수이고, 객체 크기도 8바이트 배수로 올림합니다. 더 큰 정렬이 필요한 객체(`#[repr(align(N))]`)는 slow path에서 처리합니다.

## 7.2.6. TLAB

TLAB은 Immix block 안에서 비어 있는 line이 이어진 구간입니다. refill할 때 그 구간을 0으로 지운 뒤 넘기므로, fast path는 0으로 채워진 메모리를 받습니다.

## 7.2.7. TLAB 와 work-stealing 의 관계

[스케줄러](706-scheduler.md)의 work-stealing 메커니즘으로 한 worker가 다른 worker의 task를 훔칠 수 있습니다.

훔쳐 온 task는 훔친 worker의 TLAB에서 할당하며, 원래 worker의 TLAB과는 관계가 없습니다.

TLAB은 각 OS 스레드에 thread-local로 묶여 있으므로 이 모델이 그대로 작동합니다.

## 7.2.8. GC trigger

slow path는 다음 경우에 수집을 시작합니다. 조건과 minor / full 선택은 [7.3.2](703-gc-algorithm.md#732-sticky-mark-bit-세대-수집)를 따릅니다.

- 새 TLAB이나 block을 내줄 수 없을 때.
- 지난 수집 뒤의 할당량이 다음 cycle의 예산을 넘었을 때.

`rok_gc_collect`를 호출하면 명시적으로 full 수집을 합니다.

## 7.2.9. 큰 객체의 LOS 할당

8 KiB 이상이거나 8바이트보다 큰 정렬을 요구하는 객체는 TLAB을 거치지 않고 LOS에 하나씩 할당합니다. 큰 객체를 TLAB에 두면 남은 자리가 낭비되고, LOS 객체는 옮기지 않으므로 bump 할당으로 얻는 이점도 없습니다.

## 7.2.10. 고정된 객체

GC 포인터를 넘긴 `extern "C"` 호출 중에는 따로 고정할 필요가 없습니다([7.4.1](704-barriers-and-roots.md#루트가-가리키는-객체는-옮기지-않음)). C 코드가 호출 뒤에도 포인터를 보관한다면 `rok_gc_pin`으로 고정합니다. 고정은 `PINNED` 비트만 켤 뿐 객체를 복사하지 않습니다. 고정은 `rok_gc_unpin`으로 풉니다.

## 7.2.11. 할당 시간 vs heap 사용량

fast path는 상수 시간(약 4 명령어)이 걸립니다. 평균 할당 비용은 fast path가 결정합니다.

## 7.2.12. 할당 생략

최적화기는 객체가 밖으로 빠져나가지 않고 store 뒤에 읽히지도 않는 할당을 없앨 수 있습니다. 그런 할당은 safepoint가 아니며 수집기 통계에도 잡히지 않습니다. 크기를 표현할 수 없는 할당(`len * size + header` overflow)은 언제나 런타임의 "allocation too large"에 도달합니다.
