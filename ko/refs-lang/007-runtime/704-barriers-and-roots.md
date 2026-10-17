# 7.4. 배리어와 루트 추적

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 의 mark 출발점 (루트) 과 old → young 참조의 기록 (write barrier) 을 정의합니다.

## 7.4.1. 루트 추적: 컴파일러가 직접 emit 하는 Shadow Stack

### Shadow stack 선택

가장 흔하게 쓰이는 두 가지 루트 추적 방식은 다음과 같습니다.

| 방식                     | 장점                     | 단점                           |
|--------------------------|--------------------------|--------------------------------|
| LLVM statepoint          | 정밀 stackmap            | x86_64 / aarch64 / ppc 만 지원 |
| Shadow Stack (직접 emit) | 모든 타깃에서 같은 코드  | 추가 스택 메모리와 push/pop    |

[플랫폼 타깃](../000-meta/006-platform-targets.md) 의 Tier 1 에는 32-bit ARM (`armv7`), 32-bit x86 (`i686`), WASM 도 포함됩니다. LLVM statepoint 는 이들을 지원하지 않으므로, 모든 Tier 1 에 적용할 수 있는 방식은 [shadow stack](../000-meta/007-glossary.md#shadow-stack) 뿐입니다.

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

```
ShadowFrame {
    prev:  ptr          // 호출자의 frame (없으면 null)
    count: usize        // slots 개수 (pointer 폭)
    slots: [ptr; count] // GC 참조 word 의 주소
}
```

`slots` 에는 GC 참조 값이 아니라 **GC 참조를 담은 word 의 주소** 가 들어갑니다. 그 word 는 지역 변수의 스택 슬롯이나 spill 슬롯이며, 배열 원소나 `struct` 필드 안의 word 도 같은 방식으로 가리킵니다. 따라서 변수에 대입할 때 frame 을 따로 갱신하지 않습니다.

### 컴파일러의 emit 규칙

GC 참조를 들고 있고 safepoint 에 닿을 수 있는 함수만 frame 이 필요합니다 ([8.4.2](../008-abi/804-safepoint.md#842-emit-위치)). 반환하지 않는 함수 (panic) 와 할당도 block 도 하지 않는 런타임 함수 ([8.5.2](../008-abi/805-extern-symbols.md#safepoint-가-아닌-함수)) 의 호출은 세지 않습니다. 그런 함수에 컴파일러는 다음 코드를 직접 emit 합니다.

1. 진입 시 GC 참조 word 들을 0 으로 채웁니다.
2. 기계 스택에 `ShadowFrame` 을 잡고 `slots` 에 그 word 들의 주소를 기록합니다. `prev` 에 현재 `rok_tls_shadow_stack_top` 을 저장한 뒤 머리를 이 frame 으로 바꿉니다 (link).
3. 모든 `return` 직전에 머리를 `prev` 로 되돌립니다 (unlink). 에러 반환도 일반 `return` 이므로 같은 경로를 지납니다. panic 은 프로세스를 끝내므로 unlink 가 필요 없습니다.

변수가 더 이상 쓰이지 않게 되면 (liveness 기준), 그 root 슬롯이 가리키는 word 를 null 로 지워도 됩니다. 그 word 를 다시 읽지 않는 한 의미는 바뀌지 않으며, frame 이 죽은 객체를 붙잡아 두지 않게 됩니다.

GC 는 mark 시에 모든 thread 의 체인 머리에서 출발해 frame 을 walk 하며, 각 `slots[i]` 가 가리키는 word 의 값을 root 로 mark 합니다.

### 루트가 가리키는 객체는 옮기지 않음

루트 (shadow stack, `static`, 등록된 스레드 root) 가 직접 가리키는 객체는 그 수집에서 옮기지 않습니다. 그래서 blocking 런타임 호출이나 `extern "C"` 함수에 넘긴 GC 포인터는 **호출이 끝날 때까지 인자가 루트에 있기만 하면** 제자리에 고정됩니다. C 코드가 호출이 끝난 뒤에도 그 포인터를 보관하려면 `rok_gc_pin` ([8.5.1](../008-abi/805-extern-symbols.md#851-gc-영역)) 으로 고정해야 합니다.

코루틴 frame 은 GC heap 객체이므로 ([7.8](708-coroutine.md)) 그 안의 변수는 shadow stack 이 아니라 frame 객체의 scan table 로 추적됩니다.

### 수집기와의 계약

루트 (이 절의 shadow stack) 와 write barrier ([7.4.2](#742-write-barrier-incremental-update)) 는 이 사양대로 emit 합니다. 수집기는 컴파일러 산출물에 이 둘 외의 것을 요구하지 않습니다.

### 에러 전파와의 공존

[에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 에러는 throwing 함수의 반환값으로 전달됩니다 ([8.6.5](../008-abi/806-calling-convention.md#865-throwing-함수의-반환-값-기반-에러-전달)). 에러가 전파될 때에도 각 frame 은 일반 반환 경로로 빠져나가며 자신의 shadow stack 항목을 pop 하므로, shadow stack 은 에러 처리를 위한 별도의 보정 없이 항상 실제 호출 스택과 일치합니다.

## 7.4.2. Write Barrier: Incremental-Update

### Write barrier 의 필요성

minor 수집은 old 객체 안으로 들어가지 않습니다 ([7.3.2](703-gc-algorithm.md#732-sticky-mark-bit-세대-수집)). old 객체가 young 객체를 가리키게 되면 그 사실을 write barrier 가 수집기에 알립니다. 그렇지 않으면 그 young 객체는 회수됩니다.

### Object-remembering 방식

barrier 는 **목적지 객체 `p` 의 헤더만** 검사합니다. 살아남은 (old) 객체에는 `STICKY` 와 함께 `UNLOGGED` 비트 ([8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags-offset-4-u32)) 가 켜져 있으며, 처음 그 객체에 쓰는 순간 객체 전체를 remembered set 에 기록합니다.

```
*p = q;
                                // ↓ barrier (store 앞이어도 됨)
if (p.gc_flags & UNLOGGED) {
    rok_gc_wb_slow(p);          // UNLOGGED 를 지우고 p 를 remembered set 에
}
```

- 저장하는 값 `q` 의 헤더 (`q.sticky` 등) 는 **읽지 않습니다**. `q` 는 `any` 슬롯의 inline 정수이거나 `str` 의 내부 포인터일 수 있어, 객체 헤더로 읽으면 안전하지 않습니다.
- barrier 는 store 앞에 두어도 됩니다.
- slow path `rok_gc_wb_slow(obj)` 는 `UNLOGGED` 를 atomic `fetch_and` 로 지웁니다. 동시 mark 중이면 그 객체의 현재 참조를 SATB buffer 에도 기록합니다 ([7.5.3](705-concurrent-collection.md#753-satb-barrier)). 그래서 여러 스레드가 동시에 써도 한 스레드만 그 객체를 기록합니다. slow path 는 safepoint 가 아니어야 합니다 (그 안에서 수집이 시작되면 안 됨).
- young GC 는 remembered set 의 객체를 root 로 보고, 그 객체가 가리키는 young 객체까지 mark 합니다. 수집이 끝나면 기록된 객체에 다시 `UNLOGGED` 를 켭니다.

### Card marking

8 KiB 이상의 old 객체는 객체 전체 대신 512 바이트 **card** 단위로 기록합니다. 그런 객체에는 `UNLOGGED` 대신 `CARDED` 비트 ([8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags-offset-4-u32)) 를 켭니다.

- 그런 객체는 card table (512 B 마다 1 바이트) 을 가지며, 객체 뒤에 할당됩니다. 그 card table 의 포인터는 헤더 바로 앞 word 에 있습니다.
- inline fast path 는 `p` 헤더의 `UNLOGGED | CARDED` 를 한 번에 검사합니다 (load 하나, 분기 하나).
- 순서: `UNLOGGED` 이면 먼저 slow path 를 부르고, 이어서 `CARDED` 이면 card 를 표시합니다. 동시 mark ([7.5](705-concurrent-collection.md)) 중에는 old LOS 객체가 두 비트를 함께 가질 수 있습니다.
- `CARDED` 이면 barrier 는 호출 없이 inline 으로, 쓴 slot 이 걸치는 card 들, 곧 `(slot - p) >> 9` 부터 `(slot + len - 1 - p) >> 9` 까지를 dirty 로 표시합니다. 그래서 barrier 는 slot 주소와 저장하는 값의 크기 (`len`) 를 받습니다.
- `UNLOGGED` 이면 위의 `rok_gc_wb_slow(p)` 를 부릅니다 (변함 없음).
- minor 수집은 `CARDED` 객체의 dirty card 만 훑고 그 card 를 깨끗이 합니다.

barrier 가 필요 없는 store 는 다음과 같습니다.

| store                          | 이유                                        |
|--------------------------------|---------------------------------------------|
| 방금 할당한 객체의 초기화      | 할당과 store 사이에 GC 가 없어 young 그대로 |
| `static` 에 쓰기               | static 은 root                              |
| raw `Ptr<T>` 메모리            | GC 가 추적하지 않음                         |
| atomic (참조가 아닌 값)        | GC 참조가 아님                              |

### Remembered Set

스레드마다 4 KiB log chunk 를 둡니다 (TLS 슬롯 `rok_tls_wb_log_cursor` / `rok_tls_wb_log_limit`). 가득 찬 chunk 와, 수집이나 스레드 종료 때의 모든 chunk 는 lock-free stack 에 올립니다. 수집기는 그 stack 을 비우며 기록된 객체를 root 로 씁니다.

### 인라인 비용

fast path 는 `p` 헤더의 load 하나와 분기 하나 (`UNLOGGED | CARDED`) 이며, 대부분 둘 다 꺼져 있어 (young 객체이거나 이미 기록됨) 그대로 통과합니다.

### 컴파일러의 emit

fast path 는 컴파일러가 inline 으로 emit 하고, slow path 만 런타임 함수를 부릅니다 ([8.5](../008-abi/805-extern-symbols.md)).

```
extern "C" {
    fn rok_gc_wb_slow(obj: Ptr<u8>);   // UNLOGGED 를 지우고 remembered set 에 add (slow path, safepoint 아님)
}
```

## 7.4.3. Safepoint Polling

수집은 모든 mutator thread 가 safepoint 에 멈춘 동안 진행합니다. 컴파일러는 함수와 루프마다 safepoint poll 을 emit 합니다 ([8.4](../008-abi/804-safepoint.md)).

```
poll:
    if tls.yield_flag {
        call rok_runtime_yield()       // GC 안전 지점에 도달했음을 알림
    }
```

`rok_tls_yield_flag` 는 보통 0 입니다. 수집기가 켜면 각 thread 는 다음 poll 에서 멈춥니다.

## 7.4.4. Shadow Stack 의 walk

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

## 7.4.5. TLS 의 핵심 슬롯

각 thread 는 다음 TLS 슬롯을 가집니다. 슬롯마다 런타임이 export 하는 `thread_local` global 하나입니다.

| 슬롯                        | 의미                                                                                      |
|-----------------------------|-------------------------------------------------------------------------------------------|
| `rok_tls_yield_flag`        | safepoint 에서 `yield` 해야 하는지                                                        |
| `rok_tls_tlab_cursor`       | [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 의 다음 할당 위치 |
| `rok_tls_tlab_limit`        | TLAB 의 끝                                                                                |
| `rok_tls_shadow_stack_top`  | shadow stack 체인 머리 (가장 안쪽 frame)                                                  |
| `rok_tls_wb_log_cursor`     | write barrier log 의 위치                                                                 |
| `rok_tls_mutator_state`     | 현재 mutator 상태 (running / GC-cooperating)                                              |

자세한 TLS 슬롯 정의는 [TLS 슬롯](../008-abi/803-tls-slots.md) 에서 정의합니다.
