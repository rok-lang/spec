# 7.4. 배리어와 루트 추적

이 절은 [GC](../000-meta/007-glossary.md#gc-garbage-collector)의 mark 출발점인 루트와, old → young 참조를 기록하는 write barrier를 정의합니다.

> [!NOTE]
> 이 절은 비규범(참고용)입니다. 참조 런타임의 GC를 설명하며, 어느 릴리스에서든 바뀔 수 있습니다. 규범은 [8.1](../008-abi/801-object-header-abi.md)에서 ABI로 고정한 객체 레이아웃뿐입니다.

## 7.4.1. 루트 추적: 컴파일러가 직접 내보내는 shadow stack

### shadow stack 선택

흔히 쓰는 루트 추적 방식은 다음 두 가지입니다.

| 방식                     | 장점                     | 단점                           |
|--------------------------|--------------------------|--------------------------------|
| LLVM statepoint          | 정밀 stackmap            | x86_64 / aarch64 / ppc만 지원  |
| Shadow Stack (직접 내보냄) | 모든 타깃에서 같은 코드  | 추가 스택 메모리와 push/pop    |

[플랫폼 타깃](../000-meta/006-platform-targets.md)의 Tier 1에는 32비트 ARM(`armv7`), 32비트 x86(`i686`), WASM도 들어 있습니다. LLVM statepoint는 이 타깃을 지원하지 않습니다. 그래서 Tier 1 전체에 적용할 수 있는 방식은 [shadow stack](../000-meta/007-glossary.md#shadow-stack)뿐입니다.

LLVM의 `gc "shadow-stack"` 전략(`llvm.gcroot`)은 쓰지 않습니다. 이 전략의 체인 머리 `llvm_gc_root_chain`은 프로세스 전체에 하나뿐인 전역 변수입니다. 그래서 이 전략으로는 스레드마다 shadow stack을 둘 수 없습니다.

> [!IMPORTANT]
> Rok은 shadow stack으로 루트를 추적합니다. 프레임 push / pop 코드는 컴파일러가 직접 내보냅니다. 체인 머리는 스레드마다 하나씩 있는 `rok_tls`의 offset 0 슬롯(`rok_tls_shadow_stack_top`)입니다([8.3.8](../008-abi/803-tls-slots.md#838-컴파일러-출력-형식)). 모든 Tier 1 타깃(32비트 ARM / x86 / WASM 포함)에서 같은 방식으로 동작합니다.

### shadow stack의 구조

OS 스레드마다 자기 shadow stack이 있습니다. shadow stack은 각 함수의 기계 스택에 놓인 프레임을 `prev`로 이은 연결 리스트입니다.

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

`slots`에는 GC 참조 값이 아니라 **GC 참조를 담은 word의 주소**가 들어갑니다. 그 word는 지역 변수의 스택 슬롯이나 spill 슬롯입니다. 배열 원소나 `struct` 필드 안의 word도 같은 방식으로 가리킵니다. 그래서 변수에 대입할 때 프레임을 따로 갱신하지 않습니다.

### 컴파일러 출력 규칙

프레임은 GC 참조를 들고 있으면서 safepoint에 닿을 수 있는 함수에만 필요합니다([8.4.2](../008-abi/804-safepoint.md#842-삽입-위치)). 반환하지 않는 함수(panic)의 호출과, 할당도 block도 하지 않는 런타임 함수([8.5.2](../008-abi/805-extern-symbols.md#safepoint가-아닌-함수))의 호출은 세지 않습니다. 프레임이 필요한 함수에는 컴파일러가 다음 코드를 직접 내보냅니다.

루트는 safepoint를 넘어 살아 있는 지역 변수에만 필요합니다.

1. 진입할 때 GC 참조 word를 0으로 채웁니다.
2. 기계 스택에 `ShadowFrame`을 잡습니다. 함수를 지나는 각 경로의 첫 safepoint 앞에서 `slots`에 그 word의 주소를 기록합니다. 이어서 `prev`에 현재 `rok_tls_shadow_stack_top` 값을 저장하고 머리를 이 프레임으로 바꿉니다. 이 과정이 연결(link)입니다. 연결은 함수 진입 시점까지 앞당겨도 됩니다. 경로가 합쳐지는 곳에서는 모든 경로가 연결했거나 어느 경로도 연결하지 않았어야 합니다. safepoint에 닿지 않는 경로는 연결하지 않아도 됩니다.
3. 프레임을 연결한 경로에서는 모든 `return` 직전에 머리를 `prev`로 되돌립니다. 이 과정이 연결 해제(unlink)입니다. 에러 반환도 일반 `return`이므로 같은 경로를 지납니다. panic은 프로세스를 끝내므로 연결을 해제하지 않아도 됩니다.
4. back-edge poll([8.4.8](../008-abi/804-safepoint.md#848-루프-back-edge-poll))만 넘어 살아 있고 다른 safepoint는 넘지 않는 지역 변수가 있습니다. 이런 변수는 poll의 slow path에서만 등록해도 됩니다. 이때는 두 번째 프레임을 씁니다. slow path는 런타임을 호출하기 직전에 그 프레임의 word를 쓰고 프레임을 연결합니다. 죽은 변수의 word에는 null을 씁니다. 호출 직후에는 연결을 해제합니다. 이런 word는 다른 곳에서 쓰지 않습니다. 그 경로에서 아직 연결하지 않은 규칙 2의 프레임도 함께 연결하고 해제합니다. 이 방식은 다음 규칙을 전제로 합니다. 스레드는 컴파일러가 세는 safepoint(호출, 할당, 진입 poll)나 back-edge poll에서만 수집을 위해 멈춥니다. 따라서 safepoint가 아닌 런타임 함수([8.5.2](../008-abi/805-extern-symbols.md#safepoint가-아닌-함수))는 멈춤을 기다리면 안 됩니다. 이런 함수는 할당도, 블로킹도, 폴링도 하지 않습니다.

liveness로 보아 변수가 더 이상 쓰이지 않으면 그 루트 슬롯이 가리키는 word를 null로 지워도 됩니다. 그 word를 다시 읽지 않는 한 의미는 바뀌지 않습니다. 이렇게 하면 프레임이 죽은 객체를 붙잡아 두지 않습니다.

GC는 mark할 때 모든 스레드의 체인 머리에서 출발해 프레임을 차례로 따라갑니다. 그리고 각 `slots[i]`가 가리키는 word의 값을 루트로 mark합니다.

### 루트가 가리키는 객체는 옮기지 않음

루트(shadow stack, `static`, 등록된 스레드 루트)가 직접 가리키는 객체는 그 수집에서 옮기지 않습니다. 그래서 블로킹 런타임 호출이나 `extern "C"` 함수에 넘긴 GC 포인터는 **호출이 끝날 때까지 인자가 루트에 있기만 하면** 제자리에 고정됩니다. C 코드가 호출이 끝난 뒤에도 그 포인터를 보관하면 `rok_gc_pin`([8.5.1](../008-abi/805-extern-symbols.md#851-gc-영역))으로 고정해야 합니다.

코루틴 프레임은 GC 힙 객체입니다([7.8](708-coroutine.md)). 그래서 그 안의 변수는 shadow stack이 아니라 프레임 객체의 scan table로 추적합니다.

### 수집기와 맺는 계약

루트(이 절의 shadow stack)와 write barrier([7.4.2](#742-write-barrier-incremental-update))는 컴파일러가 이 사양대로 내보냅니다. 수집기는 컴파일러 산출물에 이 둘 말고는 요구하지 않습니다.

### 에러 전파와 공존

[에러 제어 흐름](../003-expressions/314-error-control-flow.md)의 에러는 에러를 던지는 함수의 반환값으로 전달됩니다([8.6.5](../008-abi/806-calling-convention.md#865-던지는-함수의-반환-값-기반-에러-전달)). 에러가 전파될 때도 각 프레임은 일반 반환 경로로 빠져나가며 자기 shadow stack 항목을 꺼냅니다. 그래서 shadow stack은 에러 처리용 보정 없이도 항상 실제 호출 스택과 일치합니다.

## 7.4.2. write barrier: incremental update

### write barrier가 필요한 이유

minor 수집은 old 객체 안으로 들어가지 않습니다([7.3.2](703-gc-algorithm.md#732-sticky-mark-bit-세대-수집)). old 객체가 young 객체를 가리키면 write barrier가 이 사실을 수집기에 알립니다. 알리지 않으면 그 young 객체는 회수됩니다.

### Object-remembering 방식

barrier는 **목적지 객체 `p`의 헤더만** 검사합니다. 살아남은(old) 객체에는 `STICKY`와 함께 `UNLOGGED` 비트([8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags오프셋-4-u32))가 켜져 있습니다. 그 객체에 처음 쓰는 순간 barrier가 객체 전체를 remembered set에 기록합니다.

```
*p = q;
                                // ↓ barrier (store 앞이어도 됨)
if (p.gc_flags & UNLOGGED) {
    rok_gc_wb_slow(p);          // UNLOGGED 를 지우고 p 를 remembered set 에
}
```

- 저장하는 값 `q`의 헤더(`q.sticky` 등)는 **읽지 않습니다**. `q`는 `any` 슬롯의 인라인 정수이거나 `str` 값의 내부 포인터일 수 있습니다. 그래서 이 값을 객체 헤더로 읽으면 안전하지 않습니다.
- barrier는 store 앞에 두어도 됩니다.
- slow path `rok_gc_wb_slow(obj)`는 `UNLOGGED`를 atomic `fetch_and`로 지웁니다. 동시 mark 중이면 그 객체의 현재 참조를 SATB 버퍼에도 기록합니다([7.5.3](705-concurrent-collection.md#753-satb-배리어)). 그래서 여러 스레드가 동시에 써도 객체를 기록하는 스레드는 하나뿐입니다. slow path는 safepoint이면 안 됩니다. 그 안에서 수집이 시작되면 안 된다는 뜻입니다.
- young GC는 remembered set의 객체를 루트로 보고, 그 객체가 가리키는 young 객체까지 mark합니다. 수집이 끝나면 기록된 객체에 다시 `UNLOGGED`를 켭니다.

### Card marking

8 KiB 이상의 old 객체는 객체 전체가 아니라 512바이트 **card** 단위로 기록합니다. 그런 객체에는 `UNLOGGED` 대신 `CARDED` 비트([8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags오프셋-4-u32))를 켭니다.

- 그런 객체에는 card table(512 B마다 1바이트)이 있고, card table은 객체 뒤에 할당됩니다. card table 포인터는 헤더 바로 앞 word에 있습니다.
- 인라인 fast path는 `p` 헤더의 `UNLOGGED | CARDED`를 한 번에 검사합니다(load 하나, 분기 하나).
- 순서: `UNLOGGED`이면 먼저 slow path를 호출하고, 이어서 `CARDED`이면 card를 표시합니다. 동시 mark([7.5](705-concurrent-collection.md)) 중에는 old LOS 객체가 두 비트를 함께 가질 수 있습니다.
- `CARDED`이면 barrier는 호출 없이 인라인 코드로 card를 dirty로 표시합니다. 대상은 쓴 slot이 걸치는 card, 곧 `(slot - p) >> 9`부터 `(slot + len - 1 - p) >> 9`까지입니다. 그래서 barrier는 slot 주소와 저장하는 값의 크기(`len`)를 받습니다.
- `UNLOGGED`이면 위의 `rok_gc_wb_slow(p)`를 호출합니다(변함 없음).
- minor 수집은 `CARDED` 객체의 dirty card만 훑고, 그 card를 깨끗한 상태로 되돌립니다.

barrier가 필요 없는 store는 다음과 같습니다.

| store                          | 이유                                        |
|--------------------------------|---------------------------------------------|
| 방금 할당한 객체의 초기화      | 할당과 store 사이에 GC가 없어 young 그대로  |
| `static`에 쓰기                | static은 root                               |
| raw `Ptr<T>` 메모리            | GC가 추적하지 않음                          |
| atomic (참조가 아닌 값)        | GC 참조가 아님                              |

### remembered set

스레드마다 4 KiB log chunk를 둡니다(TLS 슬롯 `rok_tls_wb_log_cursor` / `rok_tls_wb_log_limit`). 런타임은 가득 찬 chunk를 lock-free 스택에 올립니다. 수집할 때와 스레드가 끝날 때는 모든 chunk를 올립니다. 수집기는 그 스택을 비우면서 기록된 객체를 루트로 씁니다.

### 인라인 비용

fast path는 `p` 헤더 load 하나와 분기 하나(`UNLOGGED | CARDED`)입니다. 대개 young 객체이거나 이미 기록된 객체라서 두 비트가 모두 꺼져 있습니다. 그러면 그대로 통과합니다.

### 컴파일러 출력

fast path는 컴파일러가 인라인 코드로 내보내고, slow path만 런타임 함수를 호출합니다([8.5](../008-abi/805-extern-symbols.md)).

```
extern "C" {
    fn rok_gc_wb_slow(obj: Ptr<u8>);   // UNLOGGED 를 지우고 remembered set 에 add (slow path, safepoint 아님)
}
```

## 7.4.3. safepoint polling

수집은 모든 mutator 스레드가 safepoint에 멈춰 있는 동안 진행합니다. 컴파일러는 8.4.2의 예외를 빼고 함수와 루프마다 safepoint poll을 내보냅니다([8.4](../008-abi/804-safepoint.md)).

```
poll:
    if tls.yield_flag {
        call rok_runtime_yield()       // GC 안전 지점에 도달했음을 알림
    }
```

`rok_tls_yield_flag` 값은 보통 0입니다. 수집기가 이 플래그를 켜면 각 스레드는 다음 poll에서 멈춥니다.

## 7.4.4. shadow stack 순회

GC는 mark를 시작할 때 다음과 같이 동작합니다.

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

GC는 각 스레드의 shadow stack을 따라가며 살아 있는 모든 GC 객체 변수를 mark queue에 넣습니다.

그다음 mark 알고리듬([GC 알고리듬](703-gc-algorithm.md))이 그 객체들에서 출발해 자식을 따라갑니다.

## 7.4.5. TLS의 핵심 슬롯

스레드마다 다음 TLS 슬롯이 있습니다. 슬롯 하나는 런타임이 내보내는 `thread_local` 전역 변수 하나입니다.

| 슬롯                        | 의미                                                                                      |
|-----------------------------|-------------------------------------------------------------------------------------------|
| `rok_tls_yield_flag`        | safepoint에서 `yield`해야 하는지                                                          |
| `rok_tls_tlab_cursor`       | [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer)의 다음 할당 위치  |
| `rok_tls_tlab_limit`        | TLAB의 끝                                                                                 |
| `rok_tls_shadow_stack_top`  | shadow stack 체인 머리 (가장 안쪽 프레임)                                                |
| `rok_tls_wb_log_cursor`     | write barrier log의 위치                                                                  |
| `rok_tls_mutator_state`     | 현재 mutator 상태 (running / GC-cooperating)                                              |

TLS 슬롯의 자세한 정의는 [TLS 슬롯](../008-abi/803-tls-slots.md)에 있습니다.
