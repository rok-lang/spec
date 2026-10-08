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

slow path 가 수행하는 작업은 다음과 같습니다. 먼저 LOS 임계 검사를 거쳐 64-bit 환경에서는 8 KB, 32-bit 환경에서는 2 KB 이상의 객체라면 LOS 로 곧장 보냅니다. 그다음으로 TLAB 을 새로 받아 Immix 의 free block 가운데 하나를 thread-local 에 할당합니다. 가용 메모리가 임계 이하로 떨어진 경우에는 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 사이클을 트리거하여 백그라운드의 concurrent 작업으로 진행합니다. 마지막으로 할당 후에는 헤더를 초기화하여 `type_id` 를 설정하고 `gc_flags` 를 초깃값으로 둡니다.

## 7.2.4. 객체 헤더 초기화

객체 헤더는 slow path 에서 초기화되며, fast path 의 경우에도 컴파일러가 인라인으로 헤더를 채운 뒤 사용하게 됩니다.

```
*(ptr + 0) = type_id
*(ptr + 4) = 0u32          // gc_flags 초깃값
*(ptr + 8) = 0u64          // gc_word — Immix 일반 객체는 0
                            // LOS 객체는 size
```

이후 payload 부분은 사용자 코드가 필드 값으로 채우게 됩니다.

객체를 옮기지 않는 수집기는 `gc_word` 를 자유롭게 쓸 수 있습니다. 또한 수집기는 헤더 앞이나 side table 에 자체 bookkeeping 을 둘 수 있습니다 ([8.1.4](../008-abi/801-object-header-abi.md#814-gc_word-offset-8-u64-또는-ptr)).

## 7.2.5. 정렬

모든 할당은 8 바이트 정렬을 보장합니다. TLAB cursor 가 8 바이트의 배수로 유지되며, 객체 크기 또한 8 바이트 배수로 올림 처리됩니다. 더 큰 정렬이 요구되는 객체 (`#[repr(align(N))]`) 는 slow path 에서 처리하게 됩니다.

## 7.2.6. TLAB 크기

TLAB 의 기본 크기는 32 KB (한 Immix block 전체) 또는 16 KB (block 절반) 입니다.

크기를 너무 작게 잡으면 slow path 호출이 잦아져 비용이 늘어나고, 반대로 너무 크게 잡으면 thread 간 단편화가 커지는 동시에 GC pause 동안 walking 시간이 길어집니다. 그래서 기본값은 16 ~ 32 KB 사이로 두며, 런타임 설정으로 조정할 수 있습니다.

## 7.2.7. TLAB 와 work-stealing 의 관계

[스케줄러](706-scheduler.md) 의 work-stealing 메커니즘 덕분에 한 worker 가 다른 worker 의 task 를 훔칠 수 있습니다.

훔쳐 온 task 는 훔친 worker 의 TLAB 에서 할당되며, 원래 worker 의 TLAB 와는 무관하게 동작합니다.

이 모델은 TLAB 이 각 OS thread 에 thread-local 로 묶여 있기 때문에 자연스럽게 작동합니다.

## 7.2.8. GC trigger

slow path 에서는 다음 조건 중 하나라도 충족되면 GC 사이클을 트리거합니다. 첫째, Immix 의 free block 이 부족하여 새 block 을 내어 줄 수 없는 상태일 때입니다. 둘째, 전체 heap 사용량이 임계를 넘었을 때 (heap 크기의 X%) 입니다. 셋째, 사용자가 `gc::collect()` 와 같은 라이브러리 함수를 호출하여 명시적으로 요청했을 때이며, 이 경우는 드뭅니다.

트리거 이후의 GC 진행 절차는 [GC 알고리듬](703-gc-algorithm.md) 에서 정의됩니다.

## 7.2.9. 큰 객체의 직접 LOS 할당

```
size >= LOS_THRESHOLD → mmap 으로 직접 page 할당
```

LOS 객체는 TLAB 을 거치지 않고 OS 의 mmap 을 직접 호출하여 할당합니다. 그 이유는 두 가지입니다. 먼저 TLAB 안에 큰 객체 한 개를 두면 나머지 자리가 그대로 낭비됩니다. 또한 LOS 객체는 어차피 이동하지 않기 때문에 bump pointer 의 이점을 누릴 수 없습니다.

mmap 은 페이지 단위 (보통 4 KB / 16 KB) 로 일어나는 OS 호출이며, LOS 임계 (8 KB / 2 KB) 이상의 객체에만 적용되므로 상대적으로 드물게 발생합니다.

## 7.2.10. FFI Pin 된 객체

[`extern` 블록](../005-modules/509-extern-block.md) 의 [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 호출에 주소를 전달하는 객체는 표준 라이브러리의 `pin` 함수를 통해 Non-Moving 공간으로 옮겨집니다.

이 과정에서 일반 Immix 공간에 있던 객체를 한 번 복사하여 Non-Moving 공간에 두며, `gc_flags` 의 `pinned` 비트가 켜집니다. 이후 GC 는 해당 객체에 대해 evacuate 를 시도하지 않습니다.

`unpin` 을 호출하면 다시 일반 Immix 공간으로 옮길 수 있고, 그대로 둘 수도 있습니다.

이 메커니즘 덕분에 FFI 호출이 끝난 뒤에도 외부 코드가 들고 있는 포인터의 유효성이 보장됩니다.

## 7.2.11. 할당 시간 vs heap 사용량

TLAB 기반의 fast path 덕분에 할당은 상수 시간 (~ 4 명령어) 으로 끝나며, Java 나 Go 같은 GC 언어와 비슷한 성능을 보입니다.

대규모 객체가 많거나 GC 가 잦은 워크로드에서도 평균 할당 비용은 fast path 가 주도하게 됩니다.
