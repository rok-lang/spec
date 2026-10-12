# 7.4. 배리어와 루트 추적

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 가 정확히 동작하려면 어디부터 mark 를 시작해야 하는지 (루트) 와 프로그램 실행 중에 발생하는 변경 (write barrier) 을 추적해야 합니다. 본 문서는 이 두 메커니즘을 정의합니다.

## 7.4.1. 루트 추적: 컴파일러가 직접 emit 하는 Shadow Stack

### Shadow stack 선택

가장 흔하게 쓰이는 두 가지 루트 추적 방식은 다음과 같습니다.

| 방식                     | 장점                     | 단점                           |
|--------------------------|--------------------------|--------------------------------|
| LLVM statepoint          | 정밀 stackmap            | x86_64 / aarch64 / ppc 만 지원 |
| Shadow Stack (직접 emit) | 모든 타깃에서 같은 코드  | 추가 스택 메모리와 push/pop    |

[플랫폼 타깃](../000-meta/006-platform-targets.md) 의 Tier 1 에는 32-bit ARM (`armv7`), 32-bit x86 (`i686`), WASM 도 포함되어 있습니다. LLVM statepoint 는 이들을 지원하지 않으므로, 모든 Tier 1 에 적용할 수 있는 방식은 [shadow stack](../000-meta/007-glossary.md#shadow-stack) 뿐이라고 할 수 있습니다.

LLVM 의 `gc "shadow-stack"` 전략 (`llvm.gcroot`) 은 쓰지 않습니다. 이 전략의 체인 머리 `llvm_gc_root_chain` 은 프로세스 전체에 하나뿐인 global 이라 thread 마다 shadow stack 을 둘 수 없기 때문입니다.

> [!IMPORTANT]
> rok 의 루트 추적은 컴파일러가 frame push / pop 을 직접 emit 하는 shadow stack 입니다. 체인 머리는 thread 마다 하나씩 있는 export 된 thread-local 심볼 `rok_tls_shadow_stack_top` 입니다 ([8.3.8](../008-abi/803-tls-slots.md#838-컴파일러-emit-의-형식)). 모든 Tier 1 타깃 (32-bit ARM / x86 / WASM 포함) 에서 같은 방식으로 동작합니다.

### Shadow Stack 의 구조

각 OS thread 는 별도의 shadow stack 을 가집니다. shadow stack 은 각 함수의 기계 스택 위에 놓인 frame 들을 `prev` 로 이은 연결 리스트입니다.

```
rok_tls_shadow_stack_top ──► frame N      (current function)
                             ├── prev ──► frame N-1 (caller) ──► ... ──► null
                             ├── count = 2
                             ├── slots[0]  ──► &word (local x)      ──► GC object
                             └── slots[1]  ──► &word (spill slot)   ──► GC object
```

> current function: 현재 함수. caller: 호출자. local x: 지역 변수 x 의 스택 슬롯. spill slot: 레지스터 spill 슬롯. GC object: GC 객체.

```
ShadowFrame {
    prev:  ptr          // 호출자의 frame (없으면 null)
    count: usize        // slots 개수 (pointer 폭)
    slots: [ptr; count] // GC 참조 word 의 주소
}
```

`slots` 에는 GC 참조 값이 아니라 **GC 참조를 담은 word 의 주소** 가 들어갑니다. 그 word 는 지역 변수의 스택 슬롯이나 spill 슬롯이며, 배열 원소나 `struct` 필드 안의 word 도 같은 방식으로 가리킵니다. 따라서 변수에 대입할 때 frame 을 따로 갱신하지 않습니다.

### 컴파일러의 emit 규칙

GC 참조를 들고 있으면서 호출이나 할당을 하는 함수만 frame 을 만듭니다. 그런 함수에 컴파일러는 다음 코드를 직접 emit 합니다.

1. 진입 시 GC 참조 word 들을 0 으로 채웁니다.
2. 기계 스택에 `ShadowFrame` 을 잡고 `slots` 에 그 word 들의 주소를 기록합니다. `prev` 에 현재 `rok_tls_shadow_stack_top` 을 저장한 뒤 머리를 이 frame 으로 바꿉니다 (link).
3. 모든 `return` 직전에 머리를 `prev` 로 되돌립니다 (unlink). 에러 반환도 일반 `return` 이므로 같은 경로를 지납니다. panic 은 프로세스를 끝내므로 unlink 가 필요 없습니다.

변수가 더 이상 쓰이지 않게 되면 (liveness 기준), 그 root 슬롯이 가리키는 word 를 null 로 지워도 됩니다. 그 word 를 다시 읽지 않는 한 의미는 바뀌지 않으며, frame 이 죽은 객체를 붙잡아 두지 않게 됩니다.

GC 는 mark 시에 모든 thread 의 체인 머리에서 출발해 frame 을 walk 하며, 각 `slots[i]` 가 가리키는 word 의 값을 root 로 mark 합니다.

코루틴 frame 은 GC heap 객체이므로 ([7.8](708-coroutine.md)) 그 안의 변수는 shadow stack 이 아니라 frame 객체의 scan table 로 추적됩니다.

### 초기 구현

이 장의 GC 설계 (Immix, sticky mark-bit, 동시 수집) 는 목표 설계입니다. 현재 구현의 heap 은 객체를 옮기지 않는 Immix 식 mark-region 입니다 (32 KiB block, 128 B line, thread 별 TLAB bump 할당, [7.1.2](701-object-layout.md#712-메모리-공간)). evacuation 과 세대 수집 (sticky mark-bit) 은 구현의 선택 사항이며, 이를 쓰지 않는 수집기에서 write barrier 의 slow path 는 아무 일도 하지 않습니다. 단, 루트 (이 절의 shadow stack) 와 write barrier ([7.4.2](#742-write-barrier-incremental-update)) 는 이 사양의 계약대로 emit 해야 합니다. 그래야 컴파일러 산출물을 바꾸지 않고 수집기를 교체할 수 있습니다.

### 에러 전파와의 공존

[에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 에러는 throwing 함수의 반환값으로 전달됩니다 ([8.6.5](../008-abi/806-calling-convention.md#865-throwing-함수의-반환-값-기반-에러-전달)). 에러가 전파될 때에도 각 frame 은 일반 반환 경로로 빠져나가며 자신의 shadow stack 항목을 pop 하므로, shadow stack 은 에러 처리를 위한 별도의 보정 없이 항상 실제 호출 스택과 일치합니다.

## 7.4.2. Write Barrier: Incremental-Update

### Write barrier 의 필요성

[GC 알고리듬](703-gc-algorithm.md) 의 [sticky mark-bit](../000-meta/007-glossary.md#sticky-mark-bit) generational 방식에서는 다음과 같은 동작이 일어납니다.

young GC 는 sticky 객체를 건너뜁니다. 그렇다면 sticky (old) 객체가 young 객체를 가리키게 되는 경우에는 어떻게 될까요? Young GC 가 그 young 객체를 mark 하지 못하고 free 처리해 버리는 문제가 발생합니다.

이 문제를 해결하기 위한 장치가 바로 write barrier 입니다. old 가 young 을 가리키게 되는 순간을 GC 에 알리는 역할을 합니다.

### Incremental-Update 방식

```
*p = q;                        // p (old) now points to q (young)
                                // ↓ barrier
if (p.sticky && !q.sticky) {
    // p is sticky but q is young → log
    remembered_set.add(p);
}
```

> p (old) now points to q (young): p (old) 가 q (young) 를 가리키게 됨. p is sticky but q is young: p 가 sticky 인데 q 가 young.

매 포인터 쓰기마다 위와 같은 검사가 인라인으로 들어가게 됩니다.

이 barrier 가 가져오는 효과는 다음과 같습니다. young GC 는 remembered_set 도 root 로 처리하게 되며, remembered_set 에 등록된 객체부터 mark 를 시작하면 그 객체가 가리키는 young 객체까지 함께 mark 됩니다.

### Remembered Set: TLS 버퍼

remembered set 은 thread-local 버퍼에서 시작합니다. 각 thread 는 자기 log buffer 에 add 하는 방식으로 동작하고, 버퍼가 꽉 차면 flush 하여 전역 remembered set 으로 옮기는 식입니다. 이렇게 함으로써 thread 간 동기화 비용을 최소화할 수 있습니다.

flush 의 동시성 안전성은 GC 의 lock-free 구조로 처리됩니다 (정확한 구현은 런타임이 결정합니다).

### sticky 비트 검사의 인라인 비용

위 barrier 는 모든 포인터 쓰기마다 추가 비용이 들어갑니다.

```
*p = q  →  store + branch + (may add to log)
```

이 추가 비용은 sticky 비트 인라인 체크 분기 1 개에 해당하며, 대부분의 경우 `false` 가 됩니다 (한쪽 분기는 거의 일어나지 않습니다).

따라서 대부분의 쓰기는 barrier 를 그대로 통과하게 됩니다 (sticky 가 켜지지 않았거나 q 도 sticky 인 경우입니다). 명령어로 환산하면 5 ~ 6 개 정도의 수준입니다.

### 컴파일러의 emit

배리어는 컴파일러가 LLVM IR 에 직접 emit 하거나, 런타임의 inline 함수로 호출하는 방식으로 구현됩니다. 자세한 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 는 [Extern "C" 심볼](../008-abi/805-extern-symbols.md) 에서 정의하고 있습니다.

```
extern "C" {
    fn rok_gc_wb_slow(obj: Ptr<u8>);   // remembered set 에 add (slow path)
}
```

fast path 는 인라인이고, slow path 만 함수 호출로 처리됩니다.

## 7.4.3. Concurrent Mark 와의 결합

[동시 수집](705-concurrent-collection.md) 의 concurrent mark 가 진행되는 동안에도 mutator (사용자 코드) 는 계속해서 실행됩니다.

mutator 의 포인터 쓰기가 mark 를 망치지 않도록 SATB (Snapshot-At-The-Beginning) barrier 를 추가로 사용합니다.

자세한 SATB 메커니즘은 [동시 수집](705-concurrent-collection.md) 에서 정의하고 있습니다.

## 7.4.4. Safepoint Polling

GC pause 나 동기화가 필요한 시점에 모든 mutator thread 가 안전한 자리에 도달해야 합니다. 이를 위해 컴파일러는 매 함수와 루프 안에 safepoint poll 을 emit 합니다.

```
poll:
    if tls.yield_flag {
        call rok_runtime_yield()       // GC 안전 지점에 도달했음을 알림
    }
```

TLS (thread-local storage) 의 `yield` flag 를 검사하는 방식이며, 이 값은 보통 `false` 입니다. GC 가 `yield` flag 를 set 하면 다음 poll 에서 thread 가 `yield` 되어 GC 와 협력하게 됩니다.

이러한 메커니즘을 통해 모든 thread 가 적절한 시점에 GC 와 동기화됩니다.

자세한 ABI 는 [Extern "C" 심볼](../008-abi/805-extern-symbols.md) 에서 정의하고 있습니다.

## 7.4.5. Shadow Stack 의 walk

GC 가 mark 를 시작할 때의 동작은 다음과 같습니다.

```
for each thread T:
    frame = T.rok_tls_shadow_stack_top
    while frame != null:
        for i in 0..frame.count:
            ref = *frame.slots[i]          // slot 은 word 의 주소
            if ref != null:
                mark(ref)
        frame = frame.prev
```

각 thread 의 shadow stack 을 walk 하여 살아 있는 모든 GC 객체 변수를 mark queue 에 add 합니다.

이 작업이 끝나면 mark 알고리듬 ([GC 알고리듬](703-gc-algorithm.md)) 이 그 객체들에서 출발하여 자식들을 따라가게 됩니다.

## 7.4.6. TLS 의 핵심 슬롯

각 thread 는 다음 TLS 슬롯을 가집니다. 슬롯마다 런타임이 export 하는 `thread_local` global 하나입니다.

| 슬롯                        | 의미                                                                                      |
|-----------------------------|-------------------------------------------------------------------------------------------|
| `rok_tls_yield_flag`        | safepoint 에서 `yield` 해야 하는지                                                        |
| `rok_tls_tlab_cursor`       | [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 의 다음 할당 위치 |
| `rok_tls_tlab_limit`        | TLAB 의 끝                                                                                |
| `rok_tls_shadow_stack_top`  | shadow stack 체인 머리 (가장 안쪽 frame)                                                  |
| `rok_tls_wb_log_cursor`     | write barrier log 의 위치                                                                 |
| `rok_tls_mutator_state`     | 현재 mutator 상태 (running / GC-cooperating)                                              |

자세한 TLS 슬롯 정의는 [TLS 슬롯](../008-abi/803-tls-slots.md) 에서 정의하고 있습니다.
