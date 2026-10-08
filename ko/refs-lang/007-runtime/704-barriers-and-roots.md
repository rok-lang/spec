# 7.4. 배리어와 루트 추적

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 가 정확히 동작하려면 어디부터 mark 를 시작해야 하는지 (루트) 와 프로그램 실행 중에 발생하는 변경 (write barrier) 을 추적해야 합니다. 본 문서는 이 두 메커니즘을 정의합니다.

## 7.4.1. 루트 추적: Shadow Stack via `llvm.gcroot`

### Shadow stack 선택

가장 흔하게 쓰이는 두 가지 루트 추적 방식은 다음과 같습니다.

| 방식                         | 장점                       | 단점                           |
|------------------------------|----------------------------|--------------------------------|
| LLVM statepoint              | 정밀 stackmap              | x86_64 / aarch64 / ppc 만 지원 |
| Shadow Stack (`llvm.gcroot`) | 모든 LLVM 백엔드 자동 지원 | 추가 스택 메모리 사용          |

[플랫폼 타깃](../000-meta/006-platform-targets.md) 의 Tier 1 에는 32-bit ARM (`armv7`), 32-bit x86 (`i686`), WASM 도 포함되어 있습니다. LLVM statepoint 는 이들을 지원하지 않으므로, 모든 Tier 1 에 적용할 수 있는 방식은 [shadow stack](../000-meta/007-glossary.md#shadow-stack) 뿐이라고 할 수 있습니다.

> [!IMPORTANT]
> rok 의 루트 추적은 `llvm.gcroot` intrinsic 을 기반으로 한 shadow stack 입니다. 모든 Tier 1 타깃 (32-bit ARM / x86 / WASM 포함) 에서 동작이 보장됩니다.

### Shadow Stack 의 구조

각 OS thread 는 별도의 shadow stack 을 가집니다.

```
shadow_stack:
  ├── frame N      (current function)
  │   ├── slot 0   ← gcroot variable 1
  │   ├── slot 1   ← gcroot variable 2
  │   └── ...
  ├── frame N-1    (caller)
  │   └── ...
  └── ...
```

> current function: 현재 함수. caller: 호출자. gcroot variable: gcroot 등록된 변수.

함수 진입 시점에 그 함수의 gcroot 변수들을 위한 frame 을 push 하고, 함수가 종료되는 시점에 frame 을 pop 합니다. GC 는 mark 시에 모든 thread 의 shadow stack 을 walk 하여 살아 있는 변수 슬롯들을 root 로 mark 하게 됩니다.

### `llvm.gcroot` 사용

컴파일러는 GC 객체를 가리키는 지역 변수마다 LLVM IR 에 `llvm.gcroot` intrinsic 을 emit 합니다.

```
%root = alloca i8*           ; gcroot 변수
call void @llvm.gcroot(i8** %root, i8* null)
; ... 사용 ...
```

LLVM 백엔드가 이 intrinsic 을 인식하여 shadow stack 에 등록하는 코드를 자동으로 생성해 줍니다. 이 메커니즘은 모든 LLVM 백엔드 (x86, ARM, RISC-V, WASM, ...) 에서 자동으로 작동합니다.

### setjmp / longjmp 와의 공존

[에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 `try` / `catch` / `throw` 는 내부적으로 `setjmp` / `longjmp` (또는 그에 준하는 대체 메커니즘) 으로 구현될 수 있습니다.

shadow stack 은 `longjmp` 시점의 부분 unwind 를 자연스럽게 처리할 수 있습니다. setjmp 가 저장한 깊이까지의 frame 만 valid 한 것으로 간주하면 되기 때문입니다.

이는 LLVM statepoint 가 setjmp/longjmp 와 충돌하는 문제 (statepoint 는 일반 stack frame 만 가정합니다) 를 우회할 수 있는 방법이기도 합니다.

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
    for each frame in T.shadow_stack:
        for each slot in frame:
            if *slot != null:
                mark(*slot)
```

각 thread 의 shadow stack 을 walk 하여 살아 있는 모든 GC 객체 변수를 mark queue 에 add 합니다.

이 작업이 끝나면 mark 알고리듬 ([GC 알고리듬](703-gc-algorithm.md)) 이 그 객체들에서 출발하여 자식들을 따라가게 됩니다.

## 7.4.6. TLS 의 핵심 슬롯

각 thread 는 고정된 TLS 슬롯을 가지고 있습니다.

| 슬롯               | 의미                                                                                      |
|--------------------|-------------------------------------------------------------------------------------------|
| `yield_flag`       | safepoint 에서 `yield` 해야 하는지                                                        |
| `tlab_cursor`      | [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 의 다음 할당 위치 |
| `tlab_limit`       | TLAB 의 끝                                                                                |
| `shadow_stack_top` | shadow stack 의 현재 위치                                                                 |
| `wb_log_cursor`    | write barrier log 의 위치                                                                 |
| `mutator_state`    | 현재 mutator 상태 (running / GC-cooperating)                                              |

자세한 TLS 슬롯 레이아웃은 [TLS 슬롯](../008-abi/803-tls-slots.md) 에서 정의하고 있습니다.
