# 7.8. 코루틴

[제너레이터](../004-functions/411-generators.md), [비동기](../004-functions/412-async.md), [스트림](../004-functions/413-stream.md) 이라는 세 형태는 모두 동일한 코루틴 인프라 위에서 작동합니다. 본 문서는 그 구현 메커니즘을 정의합니다.

## 7.8.1. LLVM Coroutine Intrinsic

> [!IMPORTANT]
> rok 의 코루틴은 LLVM coroutine intrinsic (`llvm.coro.*`) 을 직접 사용합니다.
> Stackless, heap-allocated frame, arch-independent.

이 방식의 특징을 살펴보면, 먼저 stackless 라는 점에서 별도의 스택 할당이 발생하지 않습니다. 또한 heap-allocated frame 이기 때문에 코루틴의 지역 변수가 하나의 frame 객체 안에 모두 담깁니다. 마지막으로 arch-independent 한 성격을 지니므로 LLVM 이 모든 백엔드에 알아서 lower 해 줍니다.

이러한 선택 덕분에 [플랫폼 타깃](../000-meta/006-platform-targets.md) 에서 정의하는 13 개 Tier 1 플랫폼 모두에서 자동으로 작동합니다. arch 별로 별도의 context-switch 어셈블리를 작성할 필요가 전혀 없습니다.

## 7.8.2. 컴파일 흐름

```
yield/async/async-yield fn  →  state machine
                                ↓
                          rok_coro_frame_alloc(size, type_id)   (extern "C")
                                ↓
                          llvm.coro.* intrinsic
                                ↓
                          기계어 (state machine 으로)
```

컴파일러는 각 코루틴 함수를 상태 머신으로 변환하며, LLVM 은 그 상태 머신을 효율적으로 lower 해 줍니다.

## 7.8.3. 코루틴 프레임

각 코루틴은 자신만의 frame 객체를 가집니다.

```
Frame {
    state         : u32        // 현재 실행 위치 (어느 yield/await 후인지)
    locals        : ...        // 코루틴의 지역 변수들
    waker         : Waker      // 누가 깨울지 (async 의 경우)
    parent        : Frame ptr   // 호출자 frame (필요 시)
}
```

LLVM 이 코루틴 안에 흩어져 있는 `alloca` 들을 한데 모아 frame 레이아웃을 구성합니다.

## 7.8.4. Frame 은 GC 객체

코루틴 frame 은 일반 GC 객체로 다뤄집니다 ([객체 레이아웃과 메모리 공간](701-object-layout.md) 참고).

구체적인 동작 방식은 다음과 같습니다. llvm.coro.alloc 이 호출되면 `rok_coro_frame_alloc` (`extern` "C") 으로 이어지고, `rok_coro_frame_alloc` 은 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 의 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 을 통해 frame 을 할당합니다. 이때 frame 의 type_id 는 코루틴 함수마다 고유하게 부여되며, frame 의 [scan_table](../000-meta/007-glossary.md#scan_table) 은 자식 GC 포인터들의 offset 목록을 담고 있습니다.

이 통합이 가져오는 효과는 다음과 같습니다.

- frame 이 자동으로 GC root 역할을 수행하며, 누군가 frame 을 가리키는 동안에는 살아 있습니다.
- sticky 비트를 통해 long-lived `async` task 가 자연스럽게 추적됩니다 ([GC 알고리듬](703-gc-algorithm.md) 참고).
- frame 타입이 곧 호출 결과의 익명 타입이며, Future / Iterator / Stream 트레잇을 구현합니다 ([4.1.11](../004-functions/401-function-declaration.md#코루틴-함수의-결과-타입)).

## 7.8.5. Generator 와 Async 의 두 lowering

LLVM coroutine 은 두 가지 lowering 방식을 지원합니다.

### Generator: returned-continuation

```
yield fn count() -> i32 {
    yield 1;
    yield 2;
}
```

lowering 은 다음과 같이 진행됩니다.

```
1. call count() → frame 할당, 즉시 frame ptr 반환.
2. frame.next() 호출.
   - state machine 진행.
   - yield 만나면 — 값 반환 + state 저장 후 return.
3. 다음 frame.next() 호출 — 저장된 state 에서 이어서 진행.
```

호출자가 명시적으로 next() 를 호출하며, 호출과 호출 사이마다 control 이 호출자에게 다시 돌아옵니다.

이 모델은 `for x in iterator { ... }` 처럼 호출자가 주도하는 패턴에 해당합니다.

### Async: switched-resume

```
async fn fetch() -> Data {
    let r = await op1();
    process(r)
}
```

lowering 은 다음과 같이 진행됩니다.

```
1. call fetch() → frame 할당, 즉시 Future ptr 반환.
2. Future.poll(waker) 호출.
   - state machine 진행.
   - await 만나면 — Future 를 등록하고 Pending 을 반환.
3. Future 가 ready 되면 — waker 가 task 큐에 다시 push.
4. 어떤 worker 가 fetch frame 을 다시 poll — 이어서 진행.
```

호출자는 `await` 으로 제어를 양보하며, 다른 곳 (스케줄러 또는 I/O driver) 이 결과를 준비해 두면 다시 깨워 줍니다.

이 모델은 스케줄러나 runtime 이 주도하는 패턴에 해당합니다.

## 7.8.6. Stream: `async` + `yield` 결합

[스트림](../004-functions/413-stream.md) 의 `async yield` 는 앞서 살펴본 두 lowering 을 결합한 것입니다.

`yield` 시점에는 generator 처럼 값을 산출하고, `await` 시점에는 `async` 처럼 일시 중단했다가 재개합니다. 같은 frame 안에 두 메커니즘이 공존하며, coroutine state machine 이 어느 종류의 일시 중단인지를 state 비트로 추적합니다.

## 7.8.7. CoroElide: 인라인 가능한 경우 비용 0

LLVM 은 코루틴 호출을 인라인할 수 있다고 판단하면 frame 할당 자체를 생략합니다 (CoroElide).

다음 조건이 충족되어야 합니다.

- 호출자가 그 코루틴을 `await` 또는 next 로 직접 끝까지 진행시킵니다.
- 코루틴이 escape 하지 않습니다 (즉, frame 이 다른 task 로 넘어가지 않아야 합니다).

이 조건이 만족되면 frame 의 heap 할당은 0 이 되고, 일반 함수 호출과 동일한 비용으로 처리됩니다.

따라서 자주 일어나는 짧은 `async` `fn` 호출은 사실상 비용 0 으로 처리할 수 있습니다.

## 7.8.8. 핵심 ABI 함수

```
extern "C" {
    fn rok_coro_frame_alloc(size: usize, type_id: u32) -> Ptr<u8>;
    fn rok_coro_park(frame: Ptr<u8>, waker: Ptr<u8>);   // waker: 불투명 Waker 포인터
    fn rok_coro_unpark(frame: Ptr<u8>);
}
```

각 함수의 역할은 다음과 같습니다. `rok_coro_frame_alloc` 은 TLAB 을 통해 frame 을 할당하고, `rok_coro_park` 은 코루틴을 일시 중단하면서 waker 를 등록하며, `rok_coro_unpark` 은 코루틴을 task 큐에 다시 push 합니다.

자세한 시그니처는 [extern 심볼](../008-abi/805-extern-symbols.md) 에서 정의합니다.

## 7.8.9. `yield` / `await` 의 효과 정리

| 키워드         | lowering                                                                          |
|----------------|-----------------------------------------------------------------------------------|
| `yield expr;`  | state 저장 + `return` expr (Iterator 의 경우) / `return` Pending (Stream 의 경우) |
| `await expr;`  | Future 등록 + state 저장 + `return` Pending                                       |
| `return expr;` | state machine 종료 + 결과 expr 반환                                               |

각 키워드마다 컴파일러가 llvm.coro intrinsic 을 emit 하는 방식이 결정됩니다.

## 7.8.10. Tail Recursion / 스택 깊이

stackless 코루틴이기 때문에 깊은 재귀에서도 OS stack 을 소모하지 않습니다.

```
async fn deep(n: i32) -> i32 {
    if n == 0 { 0 }
    else { 1 + await deep(n - 1) }
}
```

위 코드는 호출이 일어날 때마다 frame 을 새로 할당할 뿐, 일반 stack 은 전혀 쓰지 않습니다. 그래서 수십만 깊이까지도 가능합니다 (다만 heap 사용량은 그만큼 늘어납니다).

이는 일반 함수가 안고 있는 stack overflow 위험과 뚜렷하게 구분되는 코루틴의 장점입니다.

## 7.8.11. 코루틴 프레임의 수명

코루틴 frame 은 살아 있는 동안 GC 가 보존해 줍니다.

각 상황에서의 참조 관계를 정리하면 다음과 같습니다. task 가 진행 중일 때는 frame 이 task queue 에서 참조되고, task 가 `await` 중일 때는 frame 이 I/O driver 또는 timer 큐에서 참조됩니다. task 결과를 기다리는 쪽이 있다면 그쪽의 frame 이 이 frame 을 참조하게 됩니다. 그리고 어디에서도 참조되지 않는 시점이 오면 GC 가 회수합니다.

이렇게 GC 와 자연스럽게 통합되어 있어, 명시적인 cleanup 코드가 거의 필요 없습니다.
