# 7.8. 코루틴

[제너레이터](../004-functions/411-generators.md), [비동기](../004-functions/412-async.md), [스트림](../004-functions/413-stream.md) 세 형태 모두 동일한 코루틴 인프라 위에서 작동한다. 본 문서는 그 구현 메커니즘을 정의한다.

## 7.8.1. LLVM Coroutine Intrinsic

> [!IMPORTANT]
> nok 의 코루틴은 LLVM coroutine intrinsic (`llvm.coro.*`) 을 직접 사용한다.
> Stackless, heap-allocated frame, arch-independent.

- stackless: 별도의 스택 할당이 없다.
- heap-allocated frame: 코루틴의 지역 변수가 한 frame 객체에 들어간다.
- arch-independent: LLVM 이 모든 백엔드에 lower 한다.

이 선택은 [플랫폼 타깃](../000-meta/006-platform-targets.md) 의 13 개 Tier 1 platform 에서 모두 자동으로 작동한다. arch 별 context-switch 어셈블리가 전혀 필요하지 않다.

## 7.8.2. 컴파일 흐름

```
yield/async/async-yield fn  →  state machine
                                ↓
                          nok_coro_frame_alloc(size, type_id)   (extern "C")
                                ↓
                          llvm.coro.* intrinsic
                                ↓
                          기계어 (state machine 으로)
```

컴파일러는 각 코루틴 함수를 상태 머신으로 변환한다. LLVM 이 그 머신을 효율적으로 lower 한다.

## 7.8.3. 코루틴 프레임

각 코루틴은 frame 객체를 갖는다.

```
Frame {
    state         : u32        // 현재 실행 위치 (어느 yield/await 후인지)
    locals        : ...        // 코루틴의 지역 변수들
    waker         : Waker      // 누가 깨울지 (async 의 경우)
    parent        : Frame ptr   // 호출자 frame (필요 시)
}
```

LLVM 이 코루틴 안의 `alloca` 들을 모아 frame 레이아웃을 만든다.

## 7.8.4. Frame 은 GC 객체

코루틴 frame 은 일반 GC 객체 ([객체 레이아웃과 메모리 공간](701-object-layout.md)) 다.

- llvm.coro.alloc 이 호출되면 → `nok_coro_frame_alloc` (`extern` "C") 을 호출한다.
- `nok_coro_frame_alloc` 은 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 의 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 을 통해 frame 을 할당한다.
- frame 의 type_id 는 코루틴 함수마다 고유하다.
- frame 의 [scan_table](../000-meta/007-glossary.md#scan_table) 은 자식 GC 포인터들의 offset 목록이다.

이 통합의 효과:

- frame 이 자동으로 GC root 역할을 한다. 누군가 frame 을 가리키는 동안 살아있다.
- sticky 비트로 long-lived `async` task 가 자연스럽게 추적된다 ([GC 알고리듬](703-gc-algorithm.md)).
- Future / Iterator / Stream 트레잇과 자동으로 연결된다. frame 이 그 트레잇 객체 역할을 한다.

## 7.8.5. Generator 와 Async 의 두 lowering

LLVM coroutine 은 두 가지 lowering 을 지원한다.

### Generator: returned-continuation

```
yield fn count() -> i32 {
    yield 1;
    yield 2;
}
```

lowering:

```
1. call count() → frame 할당, 즉시 frame ptr 반환.
2. frame.next() 호출:
   - state machine 진행.
   - yield 만나면 — 값 반환 + state 저장 후 return.
3. 다음 frame.next() 호출 — 저장된 state 에서 이어서 진행.
```

호출자가 명시적으로 next() 를 호출한다. 호출 사이마다 control 이 호출자에게 돌아온다.

이 모델은 `for x in iterator { ... }` 처럼 호출자가 주도하는 패턴이다.

### Async: switched-resume

```
async fn fetch() -> Data {
    let r = await op1();
    process(r)
}
```

lowering:

```
1. call fetch() → frame 할당, 즉시 Future ptr 반환.
2. Future.poll(waker) 호출:
   - state machine 진행.
   - await 만나면 — Future 를 등록하고 Pending 을 반환.
3. Future 가 ready 되면 — waker 가 task 큐에 다시 push.
4. 어떤 worker 가 fetch frame 을 다시 poll — 이어서 진행.
```

호출자는 `await` 으로 양보한다. 다른 곳 (스케줄러 / I/O driver) 이 결과를 준비하면 깨운다.

이 모델은 스케줄러 / runtime 이 주도하는 패턴이다.

## 7.8.6. Stream: `async` + `yield` 결합

[스트림](../004-functions/413-stream.md) 의 `async yield` 는 두 lowering 을 결합한 것이다.

- `yield` 시: generator 처럼 값을 산출한다.
- `await` 시: `async` 처럼 일시 중단했다가 재개한다.

같은 frame 안에 두 메커니즘이 공존한다. coroutine state machine 이 어느 종류의 일시 중단인지를 state 비트로 추적한다.

## 7.8.7. CoroElide: 인라인 가능한 경우 비용 0

LLVM 은 코루틴 호출을 인라인할 수 있으면 frame 할당을 생략한다 (CoroElide).

조건:

- 호출자가 그 코루틴을 `await` / next 로 직접 끝까지 진행한다.
- 코루틴이 escape 하지 않는다 (frame 이 다른 task 로 넘어가지 않음).

이 경우 frame 의 heap 할당이 0 이며, 일반 함수 호출과 같은 비용이 된다.

자주 일어나는 짧은 `async` `fn` 호출은 거의 비용 0 으로 처리된다.

## 7.8.8. 핵심 ABI 함수

```
extern "C" {
    fn nok_coro_frame_alloc(size: usize, type_id: u32) -> Ptr<u8>;
    fn nok_coro_park(frame: Ptr<u8>, waker: Waker);
    fn nok_coro_unpark(frame: Ptr<u8>);
}
```

- `nok_coro_frame_alloc`: TLAB 을 통해 frame 을 할당한다.
- `nok_coro_park`: 코루틴을 일시 중단하고 waker 를 등록한다.
- `nok_coro_unpark`: 코루틴을 task 큐에 다시 push 한다.

자세한 시그니처는 [extern 심볼](../008-abi/805-extern-symbols.md) 에서 정의한다.

## 7.8.9. `yield` / `await` 의 효과 정리

| 키워드         | lowering                                                                          |
| -------------- | --------------------------------------------------------------------------------- |
| `yield expr;`  | state 저장 + `return` expr (Iterator 의 경우) / `return` Pending (Stream 의 경우) |
| `await expr;`  | Future 등록 + state 저장 + `return` Pending                                       |
| `return expr;` | state machine 종료 + 결과 expr 반환                                               |

각 키워드마다 컴파일러가 llvm.coro intrinsic 을 emit 하는 방식이 결정된다.

## 7.8.10. Tail Recursion / 스택 깊이

stackless 코루틴이라 깊은 재귀에서도 OS stack 을 소모하지 않는다.

```
async fn deep(n: i32) -> i32 {
    if n == 0 { 0 }
    else { 1 + await deep(n - 1) }
}
```

위는 호출마다 frame 을 할당한다. 일반 stack 은 쓰지 않는다. 수십만 깊이까지도 가능하다 (heap 사용량만 늘어난다).

이는 일반 함수의 stack overflow 위험과 구분되는 코루틴의 장점이다.

## 7.8.11. 코루틴 프레임의 수명

코루틴 frame 은 살아있는 동안 GC 가 보존한다.

- task 진행 중: frame 이 task queue 에서 참조된다.
- task 가 `await` 중: frame 이 I/O driver / timer 큐에서 참조된다.
- task 결과를 기다리는 쪽: 그 쪽의 frame 이 이 frame 을 참조한다.
- 어디서도 참조되지 않음: GC 가 회수한다.

이렇게 GC 와 자연스럽게 통합되어 명시적 cleanup 코드가 거의 필요 없다.
