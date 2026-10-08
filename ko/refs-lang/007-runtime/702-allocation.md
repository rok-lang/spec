# 7.2. 할당

[객체 레이아웃](701-object-layout.md) 의 객체를 메모리에 배치하는 방식. fast path 와 slow path 두 단계의 알고리듬을 정의한다.

## 7.2.1. TLAB (Thread-Local Allocation Buffer)

각 스케줄러 worker (= OS thread) 는 자기 TLAB 을 가진다. TLAB 은 [Immix](../000-meta/007-glossary.md#immix) 공간에서 받은 [block](../000-meta/007-glossary.md#block-블록) 하나 (또는 일부) 로, bump pointer 로 빠르게 할당한다.

```
TLAB 구조:
  cursor : ptr      // 다음 할당 위치
  limit  : ptr      // TLAB 의 끝
  block  : ptr      // 시작 (재활용용)
```

- cursor < limit: 공간이 충분하다.
- cursor + size > limit: [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 부족, slow path 로 간다.

## 7.2.2. Fast path: 인라인 4 명령어

> [!IMPORTANT]
> 대부분의 할당은 인라인 코드로 처리한다. 기계 명령어 4 ~ 5 개, 동기화 비용 0.

```
1. tmp = cursor
2. new_cursor = tmp + size
3. if new_cursor > limit → goto slow
4. cursor = new_cursor
   (return tmp)
```

- 기계 명령어 4 ~ 5 개.
- branch 는 거의 `false` (빠른 경로).
- TLAB 자체가 thread-local 이라 동기화 비용이 0 이다.

이 fast path 는 컴파일러가 LLVM IR 에 직접 생성한다 (인라인). 런타임이 함수 호출 형태로 제공하지 않는다.

## 7.2.3. Slow path: `nok_gc_alloc_slow`

TLAB 이 부족하거나 큰 객체 ([LOS](../000-meta/007-glossary.md#los-large-object-space)) 인 경우다.

```
extern "C" {
    fn nok_gc_alloc_slow(size: usize, align: usize, type_id: u32) -> Ptr<u8>;
}
```

런타임이 제공하는 `extern "C"` 함수. 컴파일러는 fast path 가 실패하면 이 함수를 호출한다.

slow path 의 작업:

1. LOS 임계 검사. 8 KB (64-bit) / 2 KB (32-bit) 이상이면 LOS 로 직접 보낸다.
2. TLAB 새로 받기. Immix 의 free block 중 하나를 thread-local 에 할당한다.
3. [GC](../000-meta/007-glossary.md#gc-garbage-collector) 트리거. 가용 메모리가 임계 이하면 GC 사이클을 시작한다 (concurrent → 백그라운드).
4. 할당 후 헤더 초기화. `type_id` 를 설정하고 `gc_flags` 를 초깃값으로 둔다.

## 7.2.4. 객체 헤더 초기화

slow path 에서 헤더를 초기화한다. fast path 에서도 컴파일러가 인라인으로 헤더를 채운 뒤 사용한다.

```
*(ptr + 0) = type_id
*(ptr + 4) = 0u32          // gc_flags 초깃값
*(ptr + 8) = 0u64          // gc_word — Immix 일반 객체는 0
                            // LOS 객체는 size
```

이후 payload 부분은 사용자 코드가 필드 값으로 채운다.

## 7.2.5. 정렬

모든 할당은 8 바이트 정렬을 보장한다.

- TLAB cursor 가 8 바이트 배수다.
- 객체 크기는 8 바이트 배수로 올린다.
- 큰 정렬이 필요한 객체 (`#[repr(align(N))]`) 는 slow path 에서 처리한다.

## 7.2.6. TLAB 크기

TLAB 의 기본 크기는 32 KB (한 Immix block 전체) 또는 16 KB (block 절반).

- 너무 작으면 slow path 호출이 잦아져 비용이 늘어난다.
- 너무 크면 thread 간 단편화가 커지고 GC pause 동안 walking 시간이 길어진다.

기본은 16 ~ 32 KB. 런타임 설정으로 조정할 수 있다.

## 7.2.7. TLAB 와 work-stealing 의 관계

[스케줄러](706-scheduler.md) 의 work-stealing 으로 한 worker 가 다른 worker 의 task 를 훔칠 수 있다.

훔친 task 는 훔친 worker 의 TLAB 에서 할당된다. 원래 worker 의 TLAB 와는 무관하다.

이 모델은 TLAB 이 각 OS thread 에 thread-local 이라 자연스럽게 작동한다.

## 7.2.8. GC trigger

slow path 에서 다음 조건 중 하나가 되면 GC 사이클을 트리거한다.

- Immix 의 free block 부족. 새 block 을 줄 수 없는 상태.
- 전체 heap 사용량이 임계 초과 (heap 크기의 X%).
- 명시 호출. 사용자가 `gc::collect()` 같은 라이브러리 함수를 호출 (드묾).

트리거 이후의 GC 진행은 [GC 알고리듬](703-gc-algorithm.md) 에서 정의한다.

## 7.2.9. 큰 객체의 직접 LOS 할당

```
size >= LOS_THRESHOLD → mmap 으로 직접 page 할당
```

LOS 객체는 TLAB 을 거치지 않고 OS mmap 을 직접 호출한다. 이유:

- TLAB 안에 큰 객체 한 개를 두면 나머지 자리가 낭비된다.
- LOS 객체는 어차피 이동하지 않는다. bump pointer 의 이점이 없다.

mmap 은 페이지 단위 (보통 4 KB / 16 KB) 로 일어나는 OS 호출이다. LOS 임계 (8 KB / 2 KB) 이상의 객체에만 적용되므로 상대적으로 드물다.

## 7.2.10. FFI Pin 된 객체

[`extern` 블록](../005-modules/509-extern-block.md) 의 [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 호출에 주소를 넘기는 객체는 표준 라이브러리의 `pin` 함수로 Non-Moving 공간에 옮긴다.

- 일반 Immix 공간에 있던 객체를 한 번 복사하여 Non-Moving 공간에 둔다.
- `gc_flags` 의 `pinned` 비트가 켜진다.
- 이후 GC 는 evacuate 를 시도하지 않는다.

`unpin` 호출 시 다시 일반 Immix 공간으로 옮길 수 있다 (또는 그대로 둔다).

이 메커니즘은 FFI 호출이 끝난 뒤에도 외부 코드가 들고 있는 포인터가 유효함을 보장한다.

## 7.2.11. 할당 시간 vs heap 사용량

TLAB 기반 fast path 덕분에 할당이 상수 시간이다 (~ 4 명령어). Java / Go 같은 GC 언어와 비슷한 성능이다.

대규모 객체나 잦은 GC 가 발생하는 워크로드에서도 평균 할당 비용은 fast path 가 주도한다.
