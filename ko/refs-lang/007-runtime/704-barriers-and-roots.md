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

### 루트가 가리키는 객체는 옮기지 않음

루트 (shadow stack, `static`, 등록된 스레드 root) 가 직접 가리키는 객체는 그 수집에서 옮기지 않습니다. 그래서 blocking 런타임 호출이나 `extern "C"` 함수에 넘긴 GC 포인터는 **호출이 끝날 때까지 인자가 루트에 있기만 하면** 제자리에 고정됩니다. C 코드가 호출이 끝난 뒤에도 그 포인터를 보관하려면 `rok_gc_pin` ([8.5.1](../008-abi/805-extern-symbols.md#851-gc-영역)) 으로 고정해야 합니다.

코루틴 frame 은 GC heap 객체이므로 ([7.8](708-coroutine.md)) 그 안의 변수는 shadow stack 이 아니라 frame 객체의 scan table 로 추적됩니다.

### 초기 구현

이 장의 GC 설계 (Immix, sticky mark-bit, 동시 수집) 는 목표 설계입니다. 현재 구현의 heap 은 Immix 식 mark-region 이며 (32 KiB block, 128 B line, thread 별 TLAB bump 할당, [7.1.2](701-object-layout.md#712-메모리-공간)), sticky mark-bit 세대 수집 (7.4.2 의 object-remembering barrier 와 큰 객체의 card marking, 스레드별 remembered-set log) 과 full 수집의 opportunistic evacuation ([7.3.1](703-gc-algorithm.md#731-immix-mark-region)) 을 기본으로 켭니다. 아직 없는 것은 동시 mark 입니다 ([7.5](705-concurrent-collection.md) 는 목표 설계). evacuation 은 다음 위에 놓입니다.

- `any` payload 의 box 여부를 tag 의 bit 31 로 판정 ([8.2.7](../008-abi/802-scan-table.md#827-any-슬롯)).
- 컬렉션 버퍼의 원소별 배열 table ([8.2.3](../008-abi/802-scan-table.md#823-직렬화-포맷)). enum / union payload 처럼 참조인지 알 수 없는 word 는 `AMBIGUOUS_WORD` 로, 가리키는 객체를 살리되 옮기지 않습니다.
- 위의 루트 고정 규칙과 `rok_gc_pin`.
- 주소를 수로 쓴 객체 (parking / 동기화 key) 의 `NOMOVE` 비트 ([8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags-offset-4-u32)).

단, 루트 (이 절의 shadow stack) 와 write barrier ([7.4.2](#742-write-barrier-incremental-update)) 는 이 사양의 계약대로 emit 해야 합니다. 그래야 컴파일러 산출물을 바꾸지 않고 수집기를 교체할 수 있습니다.

### 에러 전파와의 공존

[에러 제어 흐름](../003-expressions/314-error-control-flow.md) 의 에러는 throwing 함수의 반환값으로 전달됩니다 ([8.6.5](../008-abi/806-calling-convention.md#865-throwing-함수의-반환-값-기반-에러-전달)). 에러가 전파될 때에도 각 frame 은 일반 반환 경로로 빠져나가며 자신의 shadow stack 항목을 pop 하므로, shadow stack 은 에러 처리를 위한 별도의 보정 없이 항상 실제 호출 스택과 일치합니다.

## 7.4.2. Write Barrier: Incremental-Update

### Write barrier 의 필요성

[GC 알고리듬](703-gc-algorithm.md) 의 [sticky mark-bit](../000-meta/007-glossary.md#sticky-mark-bit) generational 방식에서는 다음과 같은 동작이 일어납니다.

young GC 는 sticky 객체를 건너뜁니다. 그렇다면 sticky (old) 객체가 young 객체를 가리키게 되는 경우에는 어떻게 될까요? Young GC 가 그 young 객체를 mark 하지 못하고 free 처리해 버리는 문제가 발생합니다.

이 문제를 해결하기 위한 장치가 바로 write barrier 입니다. old 가 young 을 가리키게 되는 순간을 GC 에 알리는 역할을 합니다.

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
- slow path `rok_gc_wb_slow(obj)` 는 `UNLOGGED` 를 atomic `fetch_and` 로 지웁니다. 그래서 여러 스레드가 동시에 써도 한 스레드만 그 객체를 기록합니다. slow path 는 safepoint 가 아니어야 합니다 (그 안에서 수집이 시작되면 안 됨).
- young GC 는 remembered set 의 객체를 root 로 보고, 그 객체가 가리키는 young 객체까지 mark 합니다. 수집이 끝나면 기록된 객체에 다시 `UNLOGGED` 를 켭니다.

### Card marking 변형

큰 old 객체에는 객체 전체 대신 512 바이트 **card** 단위로 기록하는 변형을 함께 쓸 수 있습니다. 현재 구현은 8 KiB 이상의 old 객체에 `UNLOGGED` 대신 `CARDED` 비트 ([8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags-offset-4-u32)) 를 켭니다.

- 그런 객체는 card table (512 B 마다 1 바이트) 을 가지며, 객체 뒤에 할당됩니다. 그 card table 의 포인터는 헤더 바로 앞 word 에 있습니다.
- inline fast path 는 `p` 헤더의 `UNLOGGED | CARDED` 를 한 번에 검사합니다 (load 하나, 분기 하나).
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

remembered set 은 thread-local 버퍼로 시작해 가득 차면 전역 set 으로 옮겨도 되고 (TLS 슬롯 `rok_tls_wb_log_*`), 처음부터 전역 set 하나를 두어도 됩니다. 현재 구현은 스레드마다 4 KiB log chunk 를 두고 (`rok_tls_wb_log_cursor` / `rok_tls_wb_log_limit`), 가득 찬 chunk 와, 수집이나 스레드 종료 때의 모든 chunk 를 lock-free stack 에 올립니다. 수집기는 그 stack 을 비우며 기록된 객체를 root 로 씁니다.

### 인라인 비용

fast path 는 `p` 헤더의 load 하나와 분기 하나 (`UNLOGGED | CARDED`) 이며, 대부분 둘 다 꺼져 있어 (young 객체이거나 이미 기록됨) 그대로 통과합니다.

### 컴파일러의 emit

배리어는 컴파일러가 LLVM IR 에 직접 emit 하거나, 런타임의 inline 함수로 호출하는 방식으로 구현됩니다. 자세한 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 는 [Extern "C" 심볼](../008-abi/805-extern-symbols.md) 에서 정의하고 있습니다.

```
extern "C" {
    fn rok_gc_wb_slow(obj: Ptr<u8>);   // UNLOGGED 를 지우고 remembered set 에 add (slow path, safepoint 아님)
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
